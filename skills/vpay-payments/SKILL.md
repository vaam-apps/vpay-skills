---
name: vpay-payments
description: vpay's payment domain model — the PaymentIntent lifecycle (which has no failed status), the charge, refund and invoice state machines, the one-charge-per-intent rule, the refund write path and what still never settles, the double-entry ledger and its one live writer, and money as integer minor units. Load this before touching any status, transition, amount or currency, before writing a state machine, a settlement path or a ledger posting, and before assuming a payment can be retried, refunded or moved to a state you have not checked is reachable.
---

# The payment domain

> **Verified against vpay `0799a8d2` (2026-09-18).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

The types live in `vpay-core` — `state.rs`, `money.rs`, `failure.rs`,
`settlement.rs`, `ids.rs` — and that crate deliberately knows nothing about
HTTP, Postgres or any rail. `docs/flows/payment-lifecycle.md` is the flow page.

## The one thing agents get wrong

> **`IntentStatus` has no `failed` variant, and it is not an oversight.**

```text
requires_payment_method | requires_action | processing | succeeded | canceled
```

A rail refusing a charge returns the intent to **`requires_payment_method`**
with `last_payment_error` set. `docs/flows/payment-lifecycle.md`'s own diagram
labels its "failed" box as an _alias_ for exactly that — it is not a state.

The reason is that a merchant must be able to try another rail. `canceled`
means "cancelled before any rail saw it", not "did not work".

`ChargeState`, which is operator-facing and internal, **does** have a real
terminal `failed`. So does `RefundStatus`. Do not carry the charge's vocabulary
onto the intent, and do not add a status because a screen wants one: the label
is simultaneously a `serde` form, a database CHECK, a `schemas/vpay.cstack`
enum and both SDKs' enum, and nothing ties those four together automatically.

## `next_status` is the whole legal-transition rule

`vpay_core::next_status(from, transition) -> Option<IntentStatus>` is one
`const fn`, matched exhaustively in both dimensions so that adding a status or
a verb **fails to compile** rather than falling silently into "illegal".

```text
requires_payment_method + Confirm(Push)     -> processing
requires_payment_method + Confirm(Redirect) -> requires_action
requires_payment_method + Cancel            -> canceled
everything else                             -> None
```

Three things to take from that shape:

- **`Transition` is only the merchant's three verbs** — `Create`,
  `Confirm(ProviderFlow)`, `Cancel`. `Create` answers `None` for every status
  because it has no _source_ status: a new intent is born at
  `IntentStatus::INITIAL`.
- **Rail-driven edges are not here.** `processing → succeeded` and
  `processing → requires_payment_method + last_payment_error` are moved by the
  reconciler from an authenticated status query, never by a request. If you
  find yourself wanting `next_status` to answer for them, you are about to let
  an HTTP request move money.
- **The flow, not the rail's name, selects the next status**
  (`ProviderFlow::status_after_confirm`). Branching on a provider code here is
  the ADR-0002 defect.

**A legal answer is not permission to write it.** Between `next_status` and the
`UPDATE`, another request may have moved the same row, so every write is a
**compare-and-swap** on the row's current status. A transition that reads, then
decides, then writes without the `WHERE status = '<from>'` is a race, and it is
the shape review will not catch.

## One charge per intent, forever

`CREATE UNIQUE INDEX one_charge_per_intent ON charges (payment_intent_id)`
(migration `0004`). `requires_action` is deliberately not confirmable a second
time, and that index is the backstop for the two pages racing one `confirm`.

**Retry means a new PaymentIntent.** Not a second confirm, not a new charge on
the same intent. `Retry::NewAttempt` in the error taxonomy is exactly this
instruction. It is also why releasing an idempotency key after a `5xx` is safe:
the re-executed confirm meets this index and answers `409`.

## Money

Integer minor units, everywhere, with **no floating point in the workspace** —
`clippy::float_arithmetic` is denied. `docs/flows/money.md` is the flow page.

`Money::new(5_000, Currency::Xaf)` is **5 000 FCFA**, not 50.00: XAF is
zero-decimal, universally. The exponent is a property of the _currency_, not of
a deployment or an environment.

Two currencies exist and the second needs explaining: `Xaf` (exponent 0) and
`Eur` (exponent 2). **EUR is there only because MTN's sandbox rejects XAF.** It
is not a market vpay serves, and an agent should not read its presence as
multi-currency support.

- **No negative money.** `Money::new` refuses a negative amount. A refund is
  its own object, never a negative charge.
- `Money` is a **boundary type, not a wire type and not a storage type.** The
  wire carries `amount: i64` plus a lowercase `currency` string; the database
  carries `BIGINT` columns plus an uppercase `currency_code TEXT`. `Money` is
  constructed at the provider port, in the confirm path and in `vpay-ledger`.
  **Case is a real trap**: `Currency::from_code` accepts **uppercase only**,
  because ingress normalises first.
- `to_provider_string()` is the **single conversion point** to a rail's major
  units (`5000 EUR` → `"50.00"`, `5000 XAF` → `"5000"`).
  `to_provider_minor()` returns the minor units unchanged, for a rail whose API
  takes a JSON number. Sending `to_provider_minor()` to a rail that wants major
  units overcharges by 100× on a two-decimal currency, with nothing downstream
  able to detect it.

## Refunds: a merchant can create one, and no rail has ever returned money

~~Nothing writes a `refunds` row. `POST /v1/refunds` is mounted nowhere,
`vpay_db::Refunds` is two reads and no create, and neither event type has ever
been emitted.~~ **Corrected 2026-09-16 (RFC-0003, issues #45 and #46): every
clause of that is now false.** What landed:

- **Five routes across three paths** — `POST`/`GET /v1/refunds`,
  `GET`/`POST /v1/refunds/{id}`, `POST /v1/refunds/{id}/cancel`. The create is
  no longer a `404`.
- **`vpay_db::Refunds::create` writes the row** and reserves the amount
  against the intent **in the same transaction**. The `no_over_refund` CHECK
  is what refuses an over-refund, and it is reachable from a merchant request
  for the first time.
- **Both event types are emitted** — `charge.refunded` on create,
  `charge.refund.updated` on update, cancel and failure. Both had been
  documented-but-never-written since migration `0018`.
- **Both rails declare `supports_refunds: true` and neither answers
  `Unsupported`**: `mtn_momo::refund` is a written Disbursements `transfer`
  (2026-09-15) and `orange_money::refund` is a declared `NotImplemented`
  token, because an Orange refund is an outbound transfer this repository has
  no specification for. The reason changed, not just the value.

**What is still false, and blurring it is this repository's cardinal sin:**

> **No rail has ever returned money to anyone.** MTN's Disbursements product
> has never been called from this repository — not in production, not against
> the sandbox, not once — and **no real Disbursements credential exists in the
> project**. `mtn_momo::refund` is WireMock-proven and rail-unproven, so a
> deployment reaching it today gets `ProviderError::Config`.

> **Nothing settles a `pending` refund.** There is no refund poll ladder: the
> port has no refund status read and `Refunded` has no status field (RFC-0003
> open question 8). Every refund the routes create stays `pending` **forever**,
> so `invoices.amount_refunded` never moves, `refunds.fee` is written by
> nothing, and `Settlement::apply_refund_succeeded` — which does exist and is
> tested against Postgres — is still reached by **nothing a merchant can
> cause**. An `Ok` from the rail means the rail _accepted the instruction_ and
> is written as `pending`, deliberately.

Two consequences worth carrying:

- **A refund whose transfer was already instructed cannot be cancelled.** The
  cancel statement carries `NOT EXISTS (… provider_requests … 'refund')`,
  closing a double-payout hole measured on 2026-09-16: a 5 000 charge, one
  full refund MTN accepted, a `200 canceled`, and a second full refund — two
  transfers on the rail's own journal. Since nothing settles refunds, that is
  **every** refund with an attempt row; cancel's remaining subject is a create
  that died before recording its attempt. A merchant reconciles a stuck
  `pending` against the rail by `provider_reference_id` instead.
- **The destination is persisted nowhere.** There is no `destination` column
  on `refunds` and this change did not add one — retention is RFC-0003 open
  question 3 and is **undecided**, so storing it would be answering a question
  reserved for the maintainer. vpay therefore cannot tell an operator which
  payee a refund went to; the rail's records can, by `provider_reference_id`.
  The one exception, recorded rather than hidden: `refunds.failure_raw` stores
  the rail's own refusal text, which a rail is free to echo a payee into.
  Nothing renders that column.

`GET /v1/refunds/{id}` renders a ten-key `RefundObject` — the same renderer all
five routes and both event types use — merchant-scoped by a join onto the
owning intent, because `refunds` carries no `merchant_id`.

The `fee` field is the part with an invariant (issue #46):

| `fee`  | Means                                   |
| ------ | --------------------------------------- |
| `null` | the rail reported no fee. **Not zero.** |
| `0`    | the rail reported the movement was free |
| _n_    | the rail charged _n_ minor units        |

**An adapter must not invent one**, and `amount` is **never net of the fee** —
`amount` is the payer's money. `a_reported_fee_never_moves_the_payers_amount`
exists because `amount: row.amount - row.fee.unwrap_or(0)` once passed every
other test in the crate.

## The ledger has one live writer, and still asserts none of its invariants

`vpay-ledger` is double-entry, credit-normal
(`balance(account) = SUM(credit) − SUM(debit)`), with three accounts —
`MerchantPayable { merchant_id }`, `PayerClearing`, `PlatformFeeRevenue` — and
a `Direction` that carries the sign so an `Entry::amount` is always a
non-negative `Money`.

~~No code path in any shipping binary writes, updates or reads
`ledger_transactions` or `ledger_entries`.~~ **Corrected 2026-09-16 (RFC-0003
§ 4): the ledger got its first writer.** `vpay_db::ledger::post_in_tx` is it,
and it is `pub(crate)` with no entry on any public trait — so a consumer names
the business operation, never the raw double entry. Two call sites, both in
`vpay_db::settlement`:

| Posting     | Written by                           | Reached in a shipping binary?                          |
| ----------- | ------------------------------------ | ------------------------------------------------------ |
| **CAPTURE** | `Settlement::apply_succeeded`        | **yes** — `vpay_worker`'s settle path                  |
| **REFUND**  | `Settlement::apply_refund_succeeded` | **no** — nothing settles a refund, so nothing calls it |

Two more things moved the same day: `AccountKind::MerchantPayable` gained
`merchant_id` as a **variant payload** (mirrored by
`ledger_entries_merchant_id_iff_merchant_payable`, migration `0045`), so
invariant 2 is computable and `Ledger::merchant_payable_balance` exists; and
**`Transaction::validate()` now balances per currency** — it summed minor
units across every leg until 2026-09-15, and a mixed-currency posting
committed.

> **Still absent: any assertion of invariants 2–4.** `docs/flows/ledger.md`
> says they are "asserted nightly". **Nothing schedules any assertion**, and
> that sentence describes the intent. Invariant 1 is application-enforced by
> `validate()`, which `post_in_tx` calls before its first statement; it is
> deliberately **not** a database constraint and will not become one —
> `SUM(debit) = SUM(credit)` is an aggregate over sibling rows and a row-level
> `CHECK` cannot see them.

`GET /v1/balance` is still unmounted. ~~There is no ledger read path.~~ There
is one now — `Ledger::merchant_payable_balance`, since 2026-09-16 — and
**nothing routes it**; a handler that ever does must scope the caller to the
merchant it passes, because that `merchant_id` is what is being asked about
and not a permission.

Postings, the charge relationship, the over-refund guard and what an invariant
runner would still involve: [references/ledger.md](references/ledger.md).

## Invoices

`draft → open → paid | void | uncollectible`. **There is no `draft → void`** —
`void_in_tx`'s `WHERE` names `status = 'open'` alone, and a draft is deleted
rather than voided. Two doc comments in vpay say otherwise and are stale; see
`vpay-invoices`. Plus a
`DELETE` that removes a draft entirely. `paid` is only reachable with
`amount_remaining = 0` (`paid_means_nothing_remaining`, migration `0036`).

**There is deliberately no `can_transition_to` on `InvoiceStatus`.** A method
there would be a second copy of a rule that has to be in the statement to be
enforced at all — every transition in `vpay_db::invoices` is a compare-and-swap
`UPDATE ... WHERE status = '<from>'`, and a Rust guard beside it is the thing a
future writer calls _instead of_ taking the lock. `vpay_core::settlement::settle`
is the counter-example that earns its place: it decides something no `UPDATE`
can express. (`vpay-invoices` owns the resource; this is the state rule.)

Full tables, the charge lifecycle, the crash-safety ordering rule and the
failure taxonomy: [references/state-machines.md](references/state-machines.md).

## Status, as of 2026-09-16

The state functions, `Money`, the failure taxonomy and the settlement
transaction are real and tested. A merchant can create a refund and the ledger
records a capture — ~~the ledger posts nothing and nothing writes a refund~~,
corrected 2026-09-16.

What has **not** changed is the line that matters: exactly **one** real rail
call has ever been made — a EUR `mtn_momo` intent settled against MTN's
sandbox on 2026-09-15. MTN's Disbursements product has never been called at
all, Orange's redirect rail has never been called, no real payer has ever been
prompted, and **no money has moved in either direction**. Mounting a route is
not a rail call; `docs/status.md`'s load-bearing banner is unchanged, and a
skill that narrates this work as narrowing it is the failure this repository is
organised against.
