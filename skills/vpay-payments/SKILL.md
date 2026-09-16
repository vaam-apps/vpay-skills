---
name: vpay-payments
description: vpay's payment domain model — the PaymentIntent lifecycle (which has no failed status), the charge, refund and invoice state machines, the one-charge-per-intent rule, and money as integer minor units. Load this before touching any status, transition, amount or currency, before writing a state machine or a settlement path, and before assuming a payment can be retried, refunded or moved to a state you have not checked is reachable.
---

# The payment domain

> **Verified against vpay `93c6dfd0` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

The types live in `vpay-core` — `state.rs`, `money.rs`, `failure.rs`,
`settlement.rs`, `ids.rs` — and that crate deliberately knows nothing about
HTTP, Postgres or any rail. `docs/flows/payment-lifecycle.md` is the flow page.

## The one thing agents get wrong

> **`IntentStatus` has no `failed` variant, and it is not an oversight.**

```
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

```
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

## Refunds: reading one is real, the entire write side is absent

`GET /v1/refunds/{id}` renders a ten-key `RefundObject`, and that read is real
— merchant-scoped by a join onto the owning intent, because `refunds` carries
no `merchant_id`.

**Nothing writes a `refunds` row.** Every part of the write side is missing,
and they are missing independently:

- **no route** — `POST /v1/refunds` is declared in the wire contract and
  mounted nowhere;
- **no repository create** — `vpay_db::Refunds` is two reads
  (`get_for_merchant`, `list_for_intent`) and no create;
- **no adapter that can execute one** — `mtn_momo::refund` is
  `NotImplemented` (MTN refunds are the Disbursements product, and no
  deployment holds that credential) and Orange Money inherits the port's
  `Unsupported`, because its Web Payment product documents no refund API;
- **no event writer** — `charge.refunded` and `charge.refund.updated` are in
  the documented vocabulary and **neither has ever been emitted**;
- **no ledger posting** — see below; nothing posts anything at all.

`vpay_db::Settlement::apply_refund_succeeded` does exist and is tested against
Postgres, but it is called by **nothing outside tests** and could not run
anyway, because nothing creates a `pending` refund for it to settle. Every
refund this deployment can render is one an operator or a test put there.

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

## The ledger posts nothing

`vpay-ledger` is double-entry, credit-normal
(`balance(account) = SUM(credit) − SUM(debit)`), with three accounts —
`MerchantPayable`, `PayerClearing`, `PlatformFeeRevenue` — and a `Direction`
that carries the sign so an `Entry::amount` is always a non-negative `Money`.

> **No code path in any shipping binary writes, updates or reads
> `ledger_transactions` or `ledger_entries`.** The tables exist (migration
> `0005`) and are empty of callers. The crate is linked into `vpay-api` and
> `vpay-worker` only so their error composites can `#[from]` `LedgerError`.

What is real is **invariant 1**: `Transaction::validate()` refuses a
transaction whose debits do not equal its credits, or that has fewer than two
legs. It is deliberately **not** a database constraint and will not become one
— `SUM(debit) = SUM(credit)` is an aggregate over sibling rows and a row-level
`CHECK` cannot see them.

The flow doc's other three invariants are **not started**, it says they are
"asserted nightly" and **nothing schedules any assertion**, and invariant 2
(per-merchant balance) is not even computable: `AccountKind` has no
per-merchant dimension, so nothing says which merchant a `MerchantPayable`
posting belongs to. That is also why `GET /v1/balance` is unmounted.

Postings, the charge relationship, the over-refund guard and what building
persistence would actually involve: [references/ledger.md](references/ledger.md).

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
transaction are real and tested. The ledger posts nothing and nothing writes a
refund. Exactly **one** real rail call has ever been made — a EUR `mtn_momo`
intent settled against MTN's sandbox on 2026-09-15. Orange's redirect rail has
never been called, no real payer has ever been prompted, no money has moved.
