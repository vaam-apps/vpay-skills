# The ledger

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Read the status lines before anything else. **This page said the opposite
until 2026-09-16** and the correction is the whole of what changed.

> ~~Nothing in vpay writes a ledger posting. No code path in any shipping
> binary inserts, updates or reads a row in `ledger_transactions` or
> `ledger_entries`.~~ **Corrected 2026-09-16 (RFC-0003 § 4): the ledger has
> its first writer.** `backends/crates/vpay-db/src/ledger.rs` exists. One
> write, `post_in_tx`, and one read, `Ledger::merchant_payable_balance`.

> **Still true, and it is what an agent must not fabricate:** nothing
> schedules the **nightly assertion** of invariants 2–4.
> `docs/flows/ledger.md` says they are "asserted nightly". They are not
> asserted at all. And **no rail has ever returned money to anyone**, so the
> refund posting below has never run outside a test.

`post_in_tx` is `pub(crate)` and has **no entry on any public trait**, so what
a consumer of `vpay-db` can name is the business operation —
`Settlement::apply_succeeded`, `Settlement::apply_refund_succeeded` — and
never the raw double entry. There is deliberately **no pooled variant**: a
`post` that opened its own transaction would make "the ledger agrees with the
charge" a property of whoever remembered to call it in the right place.

_That rule was briefly broken and is worth knowing about: between the two
halves of RFC-0003 § 4 the function was reachable through a `pub`
`TxRepositories::post_ledger_transaction_in_tx`, which let any caller holding a
`PendingTransaction` post an arbitrary balanced transaction against an
arbitrary `charge_id` under an id of its choosing, with no settlement anywhere
near it — while that trait's own doc claimed the opposite. The method is
gone. **Do not reintroduce a public posting entry point.**_

## The two call sites, and which one actually runs

| Posting     | Written by                           | `Transaction` builder                        | Reached in a shipping binary?                          |
| ----------- | ------------------------------------ | -------------------------------------------- | ------------------------------------------------------ |
| **CAPTURE** | `Settlement::apply_succeeded`        | `Transaction::capture(merchant, gross, fee)` | **yes** — `vpay_worker`'s settle path calls it         |
| **REFUND**  | `Settlement::apply_refund_succeeded` | `Transaction::refund(merchant, amount)`      | **no** — nothing settles a refund, so nothing calls it |

Those two, and nothing else, which is what makes every sentence about "the
caller's transaction" checkable by reading one file. The refund half is real,
tested against Postgres, and **unreachable**: `POST /v1/refunds` creates a
`pending` refund and there is no refund poll ladder to move it (RFC-0003 open
question 8). Do not write a page, a status row or a demo that implies a refund
posting has ever been made.

`Transaction::validate()` — which had existed, been tested, and been called by
nothing that moves money since the crate was written — is called by
`post_in_tx` **before its first statement**, returning `DbError::Ledger` rather
than panicking: an unbalanced transaction reaching it is a vpay bug, and
ADR-0007's answer to a vpay bug is an error the caller must handle. The
caller's transaction is untouched when it fires, so the settlement that raised
it rolls back whole.

## The model

`backends/crates/vpay-ledger/src/lib.rs`, and it is small — four types and one
method.

```rust
enum Direction   { Debit, Credit }
enum AccountKind {
    MerchantPayable { merchant_id: String },   // the payload is new, 2026-09-15
    PayerClearing,
    PlatformFeeRevenue,
}
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

- `MerchantPayable { merchant_id }` — money owed to **one** merchant.
- `PayerClearing` — money received from payers, not yet allocated. vpay's own,
  **pooled across every tenant**, so it carries no merchant.
- `PlatformFeeRevenue` — vpay's own fee income. Pooled for the same reason.

**The merchant is a variant payload, not a field on `Entry`, and the
difference is the whole point.** A field would admit a `payer_clearing`
posting carrying a merchant (meaningless) and a `merchant_payable` posting
carrying none (the gap that made invariant 2 uncomputable, reintroduced). The
sum type makes exactly one of three variants tenant-scoped, which is what
`ledger_entries_merchant_id_iff_merchant_payable` mirrors in SQL (migration
`0045`). Use `AccountKind::merchant_payable("acme")` rather than the struct
literal; `merchant_id()` answers `Some`/`None`. **`AccountKind` is no longer
`Copy`**, unlike `Direction` — the id is an owned `String`.

`Entry::amount` is a `vpay_core::Money`, so a posting cannot be negative: the
_direction_ carries the sign, never the amount. That is the double-entry
discipline, and it is why `Money::new` refusing negatives is load-bearing here
rather than merely tidy.

## The postings

Both are built by a constructor on `Transaction`, and neither takes free-form
entries — `capture(merchant_id, gross, fee)` and `refund(merchant_id, amount)`.
The capture posting **is written in a shipping binary**; the refund posting has
never run outside a test.

**Capture of 5 000 XAF, no fee**

| Account            | Direction | Amount |
| ------------------ | --------- | ------ |
| `payer_clearing`   | debit     | 5000   |
| `merchant_payable` | credit    | 5000   |

A `fee` of `None` **and** a fee of zero both produce the two-leg form: a
zero-amount leg would post a row that moves no money and would make the number
of entries depend on whether a merchant is on a zero-rate plan. A fee above the
gross is `LedgerError::Money` — vpay never posts a capture that leaves the
merchant owing money.

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

**Two legs, never three**, and `Transaction::refund` has **no `fee`
parameter** for a caller to be tempted by — see "The refund fee reports and
posts nothing" below.

## How a posting relates to a charge and an intent

`ledger_transactions.charge_id TEXT NOT NULL REFERENCES charges (id)`.

So a ledger transaction hangs off a **charge**, not off an intent and not off a
refund. That follows from one-charge-per-intent: the charge is the thing a rail
acted on, and invariant 4 ("every succeeded charge has exactly one capture
transaction") is stated in those terms.

Two consequences of the schema as written, both **resolved on 2026-09-15 in
ways worth knowing about**:

- ~~A refund posting has no column to hang off; if you build it you are also
  adding a `refund_id` column.~~ **Corrected 2026-09-16: no column was added.**
  `post_refund` looks the charge up **inside the transaction**, from the intent
  the refund belongs to, rather than reading `refunds.charge_id` — that column
  is nullable (migration `0017`) and is filled by whoever created the refund,
  and an attribution that depends on a writer having remembered is not an
  attribution. For a `succeeded` intent the lookup always finds a row (one
  charge per intent, forever), so `None` is a broken invariant and becomes
  `DbError::WriteMatchedNoRow`.
- ~~`vpay_core::ids` has no ledger prefix — you would be choosing one.~~
  **Corrected 2026-09-16:** `ids::ledger_transaction_id()` mints `lt_…`, and
  migration `0046` added an `id_length` CHECK (1–64) on **both** tables — the
  two halves of the gap migration `0045`'s header left open. Each entry's id is
  its transaction's id with the leg's index appended, which is injective
  because the index is decimal and decimal contains no `_`
  (`the_entry_id_derivation_is_injective`).

  **A minted id does not make `ledger_transactions_pkey` a guard of invariant 4.** A random id cannot be. What stops a charge growing a second capture
  transaction is `apply_succeeded`'s compare-and-swap on the charge still being
  live, which stops the settlement running twice at all. A second, independent
  guard needs a schema change, and that is a maintainer's decision.

`ledger_entries` carries `currency_code TEXT NOT NULL REFERENCES currencies
(code)` and `CONSTRAINT amount_non_negative CHECK (amount >= 0)`. A transaction
does not carry a currency; each leg does, which is why invariant 1 is stated
"per currency".

## The four invariants, and where each is (or is not) enforced

`docs/flows/ledger.md` names four.

| #   | Invariant                                                                   | Status as of 2026-09-16                                                                               |
| --- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 1   | per transaction: `SUM(debit) = SUM(credit)`, **per currency**               | **Enforced on every write** — `Transaction::validate()`, called by `post_in_tx` before any statement  |
| 2   | per merchant: `balance(merchant_payable) = Σ captures − Σ fees − Σ refunds` | **Computable since 2026-09-15, asserted by nothing** — `Ledger::merchant_payable_balance` is the read |
| 3   | `amount_refunded` equals the sum of succeeded refunds for that intent       | **Asserted by nothing**                                                                               |
| 4   | every succeeded charge has exactly one capture transaction                  | **Asserted by nothing** — `apply_succeeded`'s CAS is what makes it hold, not a check                  |

> The flow doc says they are "asserted nightly". **There is no nightly
> assertion.** Nothing schedules an invariant check on any of 2, 3 or 4; that
> sentence describes the intent. ~~Invariant 1 is the only one that exists.~~
> **Corrected 2026-09-16:** invariant 1 is now _enforced on a live write path_
> rather than only tested, and invariant 2 stopped being uncomputable — but
> **no runner asserts any of 2–4**, and writing one is work, not a doc change.

### Invariant 1 became per-currency on 2026-09-15, and it was wrong before

`validate()` summed minor units across **every leg whatever its currency**
until then, so a posting mixing XAF and EUR could balance in the sum and in no
book, and committed. It now walks `Currency::ALL` and balances each
currency's book separately, reporting `LedgerError::Unbalanced { currency,
debits, credits }` for the first currency that differs.

Neither shipping call site can build a mixed-currency posting, so this is a
guard rather than a fix for a live defect — and the guard is the point:
`ledger_entries.currency_code` is per row, so nothing in the schema stops one.
`a_mixed_currency_posting_is_refused_and_writes_nothing` is the assertion.

Related, and on the same day: `post_refund` **refuses a refund whose
`currency_code` disagrees with its intent's** (`DbError::RefundCurrencyMismatch`,
`Category::Internal`). Both halves of the `Money` come off `refunds`, and by
the time the posting runs the settlement has already added `refund.amount` to
two figures denominated in the _intent's_ currency, neither of which looks at a
currency — **so when the codes differ there is no correct posting to make.**
The refusal aborts the settlement, rolling back the flip to `succeeded`, both
counters and the invoice, and leaving the refund `pending` for a human.

### Invariant 1 is deliberately not a database constraint, and will not become one

`SUM(debit) = SUM(credit)` is an **aggregate over every `ledger_entries` row
sharing a `transaction_id`**. A row-level SQL `CHECK` evaluates one row at a
time and cannot see its siblings, so no schema grammar — raw SQL included, not
just CrateStack's subset — can express it without a trigger.

Migration `0005` says in a comment why it does not fake one: application
enforcement already exists and is tested, and a hand-written deferred
constraint trigger would be new, unexercised logic duplicating that check in
SQL with no test proving it fires. **`post_in_tx` being the only writer is
what makes "application-enforced" true rather than aspirational** — which is
also why a public posting entry point may not come back. If you ever want the
database half too, add the trigger **with a test that proves it rejects an
unbalanced insert.**

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

### Invariant 2 became computable on 2026-09-15 — and is still asserted by nothing

~~`AccountKind` has no per-merchant dimension, so "per merchant:
`balance(merchant_payable) = …`" is not answerable from this model, and
`ledger_entries` has no `merchant_id` column either.~~ **Corrected 2026-09-16.
Both halves landed together**, which is what migration `0005`'s own GAP comment
required — a column added without the Rust field would have been inventing
structure the code did not have:

- `AccountKind::MerchantPayable { merchant_id }` on the Rust side;
- `ledger_entries.merchant_id TEXT` (migration `0045`), with
  `ledger_entries_merchant_id_iff_merchant_payable` making the column present
  **exactly** on `merchant_payable` rows, a length CHECK, and a partial index
  on `(merchant_id, currency_code)`;
- `Ledger::merchant_payable_balance(merchant_id, currency_code)` as the read.

Three things about that read, because each is a trap:

- **It returns `i64`, not `Money`.** A merchant's payable balance is
  legitimately negative while a refund has outrun its capture, and `Money` is
  non-negative by construction.
- **A merchant with no postings answers `0`**, not an absence — the sum uses
  `COALESCE`, so "no rows" and "rows summing to zero" are deliberately the same
  answer.
- **It is not merchant-scoped in the tenancy sense.** `merchant_id` is _what is
  being asked about_, not a permission. A `/v1` handler that ever routes this
  must scope the caller to the merchant it passes, exactly as `vpay_db::refunds`
  does.

**Nothing asserts the invariant itself.** Being able to compute a balance is
not the same as checking it against `Σ captures − Σ fees − Σ refunds`, and no
job, cron or test does.

`GET /v1/balance` is still unmounted. ~~There is no ledger read path.~~ There
is one now; **nothing routes it**, and it answers one currency at a time, which
is a decision a handler would have to make.

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

RFC-0003 § 4 left that decision standing on 2026-09-15, and made it
structural: **`Transaction::refund` has no `fee` parameter at all**, so a
caller cannot pass one by mistake.

**If you ever make the fee post, invariant 2 changes in the same commit.**
`Σ fees` is capture-time `platform_fee_revenue` today; a merchant-borne refund
fee adds a second kind of term and the invariant has to say which. A fee that
debits `merchant_payable` while invariant 2 still reads only capture fees is an
invariant that quietly stops holding — and there is now a real balance read
that would quietly stop agreeing with it.

There is also nothing to report yet: **`refunds.fee` is written by nothing**,
because only the settlement writes it and nothing settles a refund.

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

**No `SELECT ... FOR UPDATE` exists anywhere in vpay for this**, and none is
needed: the row-lock-and-recheck behaviour is a property of the CHECK plus
MVCC, not of anything vpay wrote. ~~No application path can even reach the
constraint outside a test issuing raw SQL.~~ **Corrected 2026-09-16: it is
reachable from a merchant request now** — `POST /v1/refunds` reserves the
amount through `Refunds::create` in the same transaction that writes the row,
so the CHECK is what answers a `409 over_refund`.

The refund sequence, as built: increment `amount_refund_pending` on creation;
on success, in one transaction, decrement pending, increment refunded, add to
`invoices.amount_refunded` and post the REFUND transaction; on failure or
cancel decrement pending only — **nothing was posted, so no reversal entry is
needed**, which is exactly why the reservation column exists rather than
posting optimistically and unwinding.

**Only the first step of that ever runs.** `Settlement::apply_refund_succeeded`
does all of the success step, ledger posting included, and is tested against
Postgres — but it is **reached by nothing a merchant can cause**, because
nothing moves a refund out of `pending` (RFC-0003 open question 8). So
`invoices.amount_refunded` is `0` in every deployment, and always has been.

## Landmine: the ledger enums are still native Postgres enums

Migration `0037` converted vpay's other native enums to `TEXT` + CHECK because
CrateStack's generated row decoders read an enum column with
`try_get::<String>()`, so **a native enum column fails to decode on every read
through that layer**. It deliberately left `account_kind` and `direction`
alone, because no CrateStack query touches the ledger tables.

So if you ever read `ledger_entries` through the CrateStack layer, it will fail
on every row, and the fix is a migration converting both types — not a change
to the query. `post_in_tx` is hand-written sqlx and casts explicitly
(`$3::account_kind`, `$4::direction`), so it is unaffected.

`schemas/vpay.cstack` does declare `model LedgerTransaction` and
`model LedgerEntry`; like `PaymentIntent`, `Charge` and `Refund`, they are
compiled, type-checked **sketches that nothing queries**. `LedgerEntry` gained
`merchant_id String?` on 2026-09-15 to mirror migration `0045`, and its GAP
note was replaced by the record of the gap closing — deliberately, because the
note's own condition was "a mirror of a Rust field, not a paper-over", and that
is the condition the variant payload met. The `iff` CHECK is **multi-column**,
so `cratestack migrate baseline` skips it in both directions; `postgres_smoke.rs`
is what proves it fires.

## If you are asked to work on the ledger

~~This section listed the pieces of persistence that did not exist.~~ **Most of
them landed on 2026-09-15 (RFC-0003 § 4).** What is left, so you can scope
honestly:

- **A runner for invariants 2, 3 and 4.** This is the big one, it is what
  `docs/flows/ledger.md` already claims happens nightly, and nothing schedules
  it. Invariant 2's read exists; 3 and 4 have none.
- **A caller for the refund posting.** It is written and unreachable, and the
  blocker is not the ledger: it is that nothing settles a `pending` refund
  (RFC-0003 open question 8), which needs a refund poll ladder and therefore a
  refund status read on the provider port. That is work, not a doc comment.
- **A route for `merchant_payable_balance`**, if `GET /v1/balance` is the task
  — with the caller scoped to the merchant it asks about, and a decision about
  currency, since the read answers one at a time.
- **A second, independent guard for invariant 4**, if that is wanted. The
  minted `lt_…` id is not one; the guard today is `apply_succeeded`'s
  compare-and-swap, and anything stronger needs a schema change that is a
  maintainer's decision.
- **Invariant 1 in the database**, if you want belt and braces: a deferred
  constraint trigger **with a test that proves it rejects an unbalanced
  insert**.

What you must **not** do: add a `pub` posting method (one existed briefly and
was removed for good reason), post outside the settlement transaction that
makes the posting true, or move an invariant's row to ✅ because the code that
would satisfy it exists. A posting written by nothing is not an invariant held.

Then the ledger row in `docs/status/backend.md` and `docs/flows/ledger.md`'s
Status section, in the same commit.
