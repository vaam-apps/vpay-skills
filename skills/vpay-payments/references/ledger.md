# The ledger

Read the status line before anything else.

> **Nothing in vpay writes a ledger posting.** `ledger_transactions` and
> `ledger_entries` exist (migration `0005`) and **no code path in any shipping
> binary inserts, updates or reads a row in either.** As of 2026-09-16 the only
> occurrences of those table names outside the migration are a comment in
> `vpay_db::config_reconcile` and a list of nineteen table names in a
> `vpay-db` routing test.

What _is_ real: `vpay-ledger`'s types and **invariant 1**, the balancing check.
That crate is linked into `vpay-api` and `vpay-worker` for exactly one reason —
so `ApiError` and `JobError` can `#[from]` its `LedgerError` (ADR-0011 makes a
composite wrap every error its callees can produce). `Transaction::validate()`
is called by nothing outside the crate's own tests and doctests.

So: an agent asked to "record the capture in the ledger" is being asked to
build persistence that does not exist, not to call something.
`docs/flows/ledger.md` describes postings in the present tense and is a
**design document** for most of its length; its own Status section says
"Persistence and invariants 2–4 are not started."

## The model

`backends/crates/vpay-ledger/src/lib.rs`, and it is small — four types and one
method.

```rust
enum Direction   { Debit, Credit }
enum AccountKind { MerchantPayable, PayerClearing, PlatformFeeRevenue }
struct Entry       { account: AccountKind, direction: Direction, amount: Money }
struct Transaction { entries: Vec<Entry> }
```

**The convention, and it is the thing to get right:**

> `balance(account) = SUM(credit) − SUM(debit)`

`merchant_payable` is **credit-normal** — a positive balance is money the
merchant received, and a debit _decreases_ it. Get this backwards and every
posting in the tables below inverts.

The chart of accounts is **not merchant-configurable**: it is part of the
settlement model, not of a deployment. Three accounts, and the enum is closed.

- `MerchantPayable` — money owed to the merchant.
- `PayerClearing` — money received from payers, not yet allocated.
- `PlatformFeeRevenue` — vpay's own fee income.

`Entry::amount` is a `vpay_core::Money`, so a posting cannot be negative: the
_direction_ carries the sign, never the amount. That is the double-entry
discipline, and it is why `Money::new` refusing negatives is load-bearing here
rather than merely tidy.

## The postings (designed; nothing writes them)

**Capture of 5 000 XAF, no fee**

| Account            | Direction | Amount |
| ------------------ | --------- | ------ |
| `payer_clearing`   | debit     | 5000   |
| `merchant_payable` | credit    | 5000   |

**Capture of 5 000 XAF with a 100 XAF platform fee**

| Account                | Direction | Amount |
| ---------------------- | --------- | ------ |
| `payer_clearing`       | debit     | 5000   |
| `merchant_payable`     | credit    | 4900   |
| `platform_fee_revenue` | credit    | 100    |

**Refund of 2 000 XAF** (the platform fee is _not_ refunded — Stripe's default)

| Account            | Direction | Amount |
| ------------------ | --------- | ------ |
| `merchant_payable` | debit     | 2000   |
| `payer_clearing`   | credit    | 2000   |

## How a posting relates to a charge and an intent

`ledger_transactions.charge_id TEXT NOT NULL REFERENCES charges (id)`.

So a ledger transaction hangs off a **charge**, not off an intent and not off a
refund. That follows from one-charge-per-intent: the charge is the thing a rail
acted on, and invariant 4 ("every succeeded charge has exactly one capture
transaction") is stated in those terms.

Two consequences of the schema as written:

- **A refund posting has no column to hang off.** There is no `refund_id` on
  `ledger_transactions`, so the designed refund posting would have to attach to
  the original charge. Nothing has decided that; if you build the refund
  posting you are also adding a column.
- **`ledger_transactions.id` is supplied by the caller.** There is no default:
  `schemas/vpay.cstack` modelled `Cuid @default(dbgenerated())`, Postgres has
  no native CUID generator, and inventing one to fake a default would be the
  plausible-but-fabricated failure mode this repository is organised against.
  `vpay_core::ids` has no ledger prefix either — you would be choosing one.

`ledger_entries` carries `currency_code TEXT NOT NULL REFERENCES currencies
(code)` and `CONSTRAINT amount_non_negative CHECK (amount >= 0)`. A transaction
does not carry a currency; each leg does, which is why invariant 1 is stated
"per currency".

## The four invariants, and where each is (or is not) enforced

`docs/flows/ledger.md` names four. Only the first exists.

| #   | Invariant                                                                   | Status                                                  |
| --- | --------------------------------------------------------------------------- | ------------------------------------------------------- |
| 1   | per transaction: `SUM(debit) = SUM(credit)`, per currency                   | **Implemented and tested** in `Transaction::validate()` |
| 2   | per merchant: `balance(merchant_payable) = Σ captures − Σ fees − Σ refunds` | **Not started, and not computable** — see the gap below |
| 3   | `amount_refunded` equals the sum of succeeded refunds for that intent       | **Not started**                                         |
| 4   | every succeeded charge has exactly one capture transaction                  | **Not started**                                         |

The flow doc says they are "asserted nightly". **There is no nightly
assertion.** Nothing schedules an invariant check; that sentence describes the
intent.

### Invariant 1 is deliberately not a database constraint, and will not become one

`SUM(debit) = SUM(credit)` is an **aggregate over every `ledger_entries` row
sharing a `transaction_id`**. A row-level SQL `CHECK` evaluates one row at a
time and cannot see its siblings, so no schema grammar — raw SQL included, not
just CrateStack's subset — can express it without a trigger.

Migration `0005` says in a comment why it does not fake one: application
enforcement already exists and is tested, and a hand-written deferred
constraint trigger would be new, unexercised logic duplicating that check in
SQL with no test proving it fires. If persistence lands and this becomes a real
gap, add the trigger **then, with a test that proves it rejects an unbalanced
insert.**

`Transaction::validate()` also refuses fewer than two legs
(`LedgerError::TooFewEntries`) — a single-legged transaction cannot balance and
is not a double entry. Both failures are `Category::Internal`,
`Severity::Page`, `Retry::Never`, with codes `ledger_unbalanced` and
`ledger_degenerate`: no caller builds a ledger transaction from merchant input,
so an unbalanced one is this code's own invariant failing, which is the most
expensive kind of bug this system can have. The merchant is told nothing about
the ledger.

Tests: `a_capture_with_a_fee_balances`, `an_unbalanced_transaction_is_rejected`,
`a_single_legged_transaction_is_rejected`, plus the doctests on `validate`.

### Invariant 2 cannot be computed from the type as it stands

**`AccountKind` has no per-merchant dimension.** Three variants, and nothing
says _which_ merchant a given `MerchantPayable` posting belongs to — so "per
merchant: `balance(merchant_payable) = …`" is not answerable from this model.
`ledger_entries` has no `merchant_id` column either, so the gap is in real SQL
now, not only in the design sketch.

Fixing it needs **a new field on the Rust type and a matching column**. Adding
a `merchant_id` column to the schema alone would be inventing structure the
code does not have, which migration `0005`'s own GAP comment refuses to do.

This is also why `GET /v1/balance` is unmounted: there is no ledger read path,
and the model could not answer a per-merchant balance if there were.

## The refund fee reports and posts nothing, and that is a decision

Issue #46, decided 2026-09-05. `RefundObject::fee` is **reported to the
merchant and posted nowhere**: none of the three postings above gains an entry
and `platform_fee_revenue` is untouched.

The reason is that a posting rule has to answer _who pays_, and that answer is
not vpay's — it is a marketplace judgement about a specific order (the platform
eats it on a platform error, the merchant on theirs), and a rail's response does
not contain it. Writing a rule now would choose one answer for every
deployment. There is also a plainer reason: **no rail reports a refund fee to
this repository today**, so any rule would be written against a number that has
never existed and tested against a fixture.

`fee_borne_by` and `fee_settlement_ref` are deliberately **not** on the object
either: vpay reports what the movement cost; who eats it is the integrator's.

**If you ever make the fee post, invariant 2 changes in the same commit.**
`Σ fees` is capture-time `platform_fee_revenue` today; a merchant-borne refund
fee adds a second kind of term and the invariant has to say which — at which
point it also finally needs the per-merchant dimension `AccountKind` does not
have. A fee that debits `merchant_payable` while invariant 2 still reads only
capture fees is an invariant that quietly stops holding.

## The over-refund guard (not one of the four)

Two layers, and it is worth knowing which does what:

- **`no_over_refund CHECK (amount_refunded + amount_refund_pending <= amount)`**
  on `payment_intents` (migration `0003`). This guarantees, unconditionally and
  including under concurrency, that **no committed row can be over-refunded**:
  two `UPDATE`s racing to increment `amount_refund_pending` serialise on the row
  lock Postgres's MVCC already takes, and the second re-evaluates the CHECK
  against the first's committed value. Proven to fire by
  `over_refund_is_rejected_by_the_database` in
  `backends/tests/integration/tests/postgres_smoke.rs`.
- **`Money::checked_sub`** in Rust, which rejects an arithmetic result that
  would go negative (`refunding_more_than_captured_is_rejected`). Narrower and
  non-concurrent: it stops one refund larger than what remains, and says nothing
  about two racing each other.

**No `SELECT ... FOR UPDATE` exists anywhere in vpay for this.** Nothing
persists a refund, so no application path can even reach the constraint outside
a test issuing raw SQL. The row-lock-and-recheck behaviour is a property of the
CHECK plus MVCC, not of anything vpay wrote.

The designed refund sequence, for when it is built: increment
`amount_refund_pending` on creation; on success, in one transaction, decrement
pending, increment refunded and write the ledger transaction; on failure
decrement pending only — **nothing was posted, so no reversal entry is needed**,
which is exactly why the reservation column exists rather than posting
optimistically and unwinding.

`vpay_db::Settlement::apply_refund_succeeded` implements the _database_ half of
the success step (refund `pending → succeeded`, plus `invoices.amount_refunded`)
and is tested against Postgres — but it is **called by nothing outside tests**,
writes **no ledger row**, and could not run in production anyway because nothing
creates a `pending` refund to settle.

## Landmine: the ledger enums are still native Postgres enums

Migration `0037` converted vpay's other native enums to `TEXT` + CHECK because
CrateStack's generated row decoders read an enum column with
`try_get::<String>()`, so **a native enum column fails to decode on every read
through that layer**. It deliberately left `account_kind` and `direction`
alone, because no CrateStack query touches the ledger tables.

So if you ever read `ledger_entries` through the CrateStack layer, it will fail
on every row, and the fix is a migration converting both types — not a change
to the query. Reading them through hand-written sqlx is unaffected.

`schemas/vpay.cstack` does declare `model LedgerTransaction` and
`model LedgerEntry`; like `PaymentIntent`, `Charge` and `Refund`, they are
compiled, type-checked **sketches that nothing queries**.

## If you are asked to build ledger persistence

The pieces that do not exist yet, so you can scope honestly: an id scheme and
prefix; a repository trait and impl in `vpay-db` (nothing may live outside that
crate — `cargo xtask verify-repositories`); a call site inside the settlement
transaction, because a posting written outside the transaction that made it true
is worse than none; the per-merchant dimension on `AccountKind` and its column;
a decision about what a refund posting hangs off; invariants 2–4 and whatever
actually runs them; and, if you want invariant 1 in the database too, a deferred
constraint trigger **with a test that proves it rejects an unbalanced insert**.

Then `docs/status/backend.md`'s "Ledger balancing invariant" row (🟡 today) and
`docs/flows/ledger.md`'s Status section, in the same commit.
