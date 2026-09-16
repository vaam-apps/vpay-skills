---
name: vpay-payments
description: vpay's payment domain model — the PaymentIntent lifecycle (which has no failed status), the charge, refund and invoice state machines, the one-charge-per-intent rule, and money as integer minor units. Load this before touching any status, transition, amount or currency, before writing a state machine or a settlement path, and before assuming a payment can be retried, refunded or moved to a state you have not checked is reachable.
---

# The payment domain

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
labels its "failed" box as an *alias* for exactly that — it is not a state.

The reason is that a merchant must be able to try another rail. `canceled`
means "cancelled before any rail saw it", not "did not work".

`ChargeState`, which is operator-facing and internal, **does** have a real
terminal `failed`. So does `RefundStatus`. Do not carry the charge's
vocabulary onto the intent, and do not add a status to `IntentStatus` because a
screen wants one — `vpay_api::dash::payment_intents` refuses an unknown status
filter with a `400` naming `status` rather than answering an empty page, and
there are two `status` vocabularies (the REST list and the CrateStack
procedure) held together by
`the_two_intent_status_vocabularies_are_one_vocabulary` in
`backends/crates/vpay-db/src/schema/search_payment_intents.rs`.

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
  because it has no *source* status: a new intent is born at
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

Migration `0036` applies the same device to invoices — one invoice per intent,
forever.

## Money

Integer minor units, everywhere, with **no floating point in the workspace** —
`clippy::float_arithmetic` is denied. `docs/flows/money.md` is the flow page.

`Money::new(5_000, Currency::Xaf)` is **5 000 FCFA**, not 50.00: XAF is
zero-decimal, universally. The exponent is a property of the *currency*, not of
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
- **Case is a real trap.** The wire is lowercase (`xaf`), the database and the
  adapters are uppercase (`XAF`), and `Currency::from_code` accepts **uppercase
  only** because ingress normalises first.
- `to_provider_string()` is the **single conversion point** to a rail's major
  units (`5000 EUR` → `"50.00"`, `5000 XAF` → `"5000"`).
  `to_provider_minor()` is the same conversion for a rail whose API takes a
  JSON number — it returns the minor units unchanged, so a rail expecting major
  units must be sent the *string* form. Sending `to_provider_minor()` to a rail
  that wants major units overcharges by 100× on a two-decimal currency with
  nothing downstream able to detect it.

## Refunds: the object exists, the write does not

`GET /v1/refunds/{id}` renders a ten-key `RefundObject`. **Nothing writes a
`refunds` row** — `POST /v1/refunds` is unrouted, `vpay_db::Refunds` is two
reads and no write, `mtn_momo::refund` is `NotImplemented` and Orange answers
`Unsupported`. Every refund this deployment can render is one an operator or a
test put there.

The `fee` field is the part with an invariant (issue #46):

| `fee`  | Means                                        |
| ------ | -------------------------------------------- |
| `null` | the rail reported no fee. **Not zero.**      |
| `0`    | the rail reported the movement was free      |
| *n*    | the rail charged *n* minor units             |

**An adapter must not invent one**, and `amount` is **never net of the fee** —
`amount` is the payer's money. `a_reported_fee_never_moves_the_payers_amount`
exists because `amount: row.amount - row.fee.unwrap_or(0)` once passed every
other test in the crate.

## Invoices

`draft → open → paid | void | uncollectible`, plus `draft → void` and a
`DELETE` that removes a draft entirely. `paid` is only reachable with
`amount_remaining = 0` (`paid_means_nothing_remaining`, migration `0036`).

**There is deliberately no `can_transition_to` on `InvoiceStatus`.** A method
there would be a second copy of a rule that has to be in the statement to be
enforced at all — every transition in `vpay_db::invoices` is a compare-and-swap
`UPDATE ... WHERE status = '<from>'`, and a Rust guard beside it is the thing a
future writer calls *instead of* taking the lock. `vpay_core::settlement::settle`
is the counter-example that earns its place, because it decides something no
`UPDATE` can express.

Full tables, the charge lifecycle, the failure taxonomy and the crash-safety
ordering rule: [references/state-machines.md](references/state-machines.md).

## Status, as of 2026-09-16

The state functions, `Money`, the failure taxonomy and the settlement
transaction are real and tested. Exactly **one** real rail call has ever been
made — a EUR `mtn_momo` intent settled against MTN's sandbox on 2026-09-15.
Orange's redirect rail has never been called, no real payer has ever been
prompted, and no money has moved.
