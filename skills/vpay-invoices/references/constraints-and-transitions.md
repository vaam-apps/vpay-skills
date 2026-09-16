# Transitions, numbering, paying, and the settlement flip

_Verified against vpay `7a79684e` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Code: `backends/crates/vpay-db/src/invoices.rs` (the statements),
`backends/crates/vpay-api/src/v1/invoices.rs` (the handlers),
`backends/crates/vpay-db/src/settlement.rs` (`flip_invoice`,
`apply_refund_succeeded`), `backends/crates/vpay-core/src/state.rs`
(`InvoiceStatus`). Schema: `0036_create-invoices.sql`,
`0042_invoices-amount-refunded.sql`.

## Three enforcers, and none of them is a validation function

1. **`invoices_status_enum_check`** closes the vocabulary at five labels. It is
   `TEXT` + CHECK, never a native Postgres enum, and it is the **first enum
   CHECK in this repository created under the name CrateStack generates** — so
   the declared constraint and the live one are one object to the diff engine
   rather than a drop-and-add pair. (`0032` had to rename `providers.flow`'s
   after the fact; do not repeat that.)
2. **Every transition is a compare-and-swap.** `UPDATE invoices SET … WHERE id
= $1 AND merchant_id = $2 AND status = '<from>'`. "Matched no row" _is_ the
   refusal. The API only ever reads the invoice **after** a refusal, to decide
   whether the answer is a `404` or a `409`.
3. **Five multi-column CHECKs** make the combinations a broken transition would
   produce unstorable. They are the guard that survives a future writer who
   forgets rule 2.

All of rule 3's constraints — and `0042`'s two — are **invisible to
`cratestack migrate baseline` in both directions** (the introspector filters
`array_length(c.conkey, 1) = 1`). The drift report cannot notice one going
missing. `the_invoice_invariants_are_enforced_by_the_database_itself` in
`backends/tests/integration/tests/invoices.rs` writes the row each one exists
to refuse, straight past the API and the repository, and `postgres_smoke.rs`
carries the inventory of every multi-column CHECK in the schema.

## What each transition's statement actually names

| Route                   | Statement            | `WHERE status =`                      | Extra                                                                                 |
| ----------------------- | -------------------- | ------------------------------------- | ------------------------------------------------------------------------------------- |
| `finalize`              | `finalize_in_tx`     | `'draft'`                             | takes the number under the sequence row's lock, freezes lines, sums `amount_due` once |
| `void`                  | `void_in_tx`         | `'open'` **only**                     | `AND NO_LIVE_INTENT`                                                                  |
| `mark_uncollectible`    | `mark_uncollectible` | `'open'`                              | `AND NO_LIVE_INTENT`                                                                  |
| `pay` → `attach_intent` | `attach_intent`      | `'open'`                              | `AND NO_LIVE_INTENT`                                                                  |
| `DELETE`                | `delete_draft`       | `'draft'`                             | removes the invoice and its lines                                                     |
| settlement              | `flip_invoice`       | `'open'`, keyed on **its own intent** | emits `invoice.paid` in the same transaction                                          |

**A draft cannot be voided** — see the SKILL.md warning; two doc comments in
the repository claim otherwise and are stale.

**An invoice with no lines is refused at `finalize`** with a `400`. A
deliberate divergence from Stripe, which finalizes a zero-amount invoice and
marks it paid: vpay has no "paid without a payment" transition, and a
zero-amount `open` invoice would be a document nobody can pay because `pay`
would have to mint an intent for zero.

**`finalize` refuses above `2^53 - 1` minor units**, naming `invoice`. `pay`
mints its intent through `PaymentIntents::insert` rather than
`POST /v1/payment_intents`' own `parse_amount`, so the bound that stops a JSON
number silently rounding did not apply. Measured: ninety-one lines at both
`invoice_items` ceilings finalized `200` at 9 100 000 000 000 000. It is
enforced at `finalize` rather than at `pay` (which would leave an `open`
invoice nobody could pay) or at the line write (already committed by then).

## `NO_LIVE_INTENT` — one payment at a time

```
(invoices.payment_intent_id IS NULL OR <the intent is canceled>)
```

While an intent is attached and **not `canceled`**, `pay`, `void` and
`mark_uncollectible` all answer `409` naming the intent. The way back is
`POST /v1/payment_intents/{id}/cancel`.

The condition is `canceled` and deliberately **not** "not `processing`": a
rail-declined intent lands back on `requires_payment_method`, which is also
where a fresh unconfirmed intent sits, so "not processing" would let a merchant
mint a second intent one millisecond after the first. It is also consistent
with the standing rule that a retry is a _new_ PaymentIntent (`AGENTS.md`).

Deleting `NO_LIVE_INTENT` from `attach_intent` was measured (2026-09-07
review): every wire case stayed green while **two concurrent `pay` requests
minted two intents and two hosted URLs for one bill**. The delivered suite paid
twice _in sequence_, so the handler's own read answered and the statement was
never under test. `two_concurrent_pays_attach_exactly_one_intent` puts both
requests in flight over the socket and is what fails under that mutation now.

## Numbering: `{prefix}-{000001}`, and no holes

The prefix is eight upper-case characters of **Crockford's alphabet**, minted
once per merchant by their first finalize and stored in
`invoice_number_sequences` (`prefix_shape` pins the regex). Crockford because
this is the one identifier in the system a human types back — off a paper
receipt, into a bank transfer reference, over the phone — and `i`, `l`, `o`,
`u` are absent, so there is no `1`/`l` and no `0`/`O` to get wrong.

**Random rather than derived from `merchant_id`**, which would put the
deployment's internal tenant identifier on every invoice. Two merchants sharing
a prefix is cosmetic: `invoices_merchant_number_key` is scoped to one merchant.

**`invoice_number_sequences` is an ordinary table row and NOT a Postgres
`SEQUENCE`.** `nextval` is non-transactional by design, so a finalize that
failed after taking a number would leave a hole. Stripe's numbering has holes;
that is not defensible here, because a Cameroonian merchant's invoice numbers
are read by a tax authority that treats a missing number as a destroyed
document. Two concurrent finalizes serialise on the `INSERT … ON CONFLICT
(merchant_id) DO UPDATE`'s lock and, under `READ COMMITTED`, the second
re-reads the committed value. `two_concurrent_finalizes_take_consecutive_numbers`
and `a_refused_finalize_does_not_burn_a_number` are the two halves; deleting
the `ON CONFLICT` clause fails the first.

## Paying, and where the URLs come from

`POST /v1/invoices/{id}/pay` mints an ordinary `pi_…` for `amount_remaining`,
creates an ordinary **hosted checkout session** for it, attaches the intent
with a compare-and-swap, and answers the invoice with a `hosted_invoice_url`.
**No new rail code and no invoice page** — `hosted_invoice_url` is a checkout
session URL, not a document.

Stripe's `pay` charges a stored payment method; vpay has none on this market,
so it hands back a page, and `0028`'s `urls_match_ui_mode` requires both URLs
on a hosted session. They may be configured per merchant (issue #91 D2,
2026-09-10) under `merchant_clients[].invoices.{success_url,cancel_url}` —
`vpay_config::oauth::InvoiceDefaults`. Resolution order is **request, then
configuration, then refuse**:

| Sent    | Configured | Result                                              |
| ------- | ---------- | --------------------------------------------------- |
| both    | either     | the request's, always                               |
| one     | the other  | the request's for one, the configured for the other |
| neither | both       | the configured pair                                 |
| neither | neither    | one `400` naming **both** parameters                |

The request winning is what makes the key safe to add to a running deployment:
every request that worked before behaves identically after. vpay never invents
one — guessing a "thank you" page sends a paying customer to a `404`.
Configured values are validated **at boot** (bounded, `http(s)`, a host, no
embedded credentials, `https` under `deployment.livemode`) **and again at
request time by the same function** a passed URL goes through, so one rule
decides where a payer may be sent.

## The settlement flip

The settlement transaction — the one that takes the charge terminal — marks the
invoice `paid` and emits `invoice.paid` **inside itself**. Not a second write
afterwards: that would leave a window in which the intent is `succeeded` and
the invoice still says money is owed, and a crash in that window would make it
permanent. **An invoice has no poller, no sweep and no job that would ever
notice** — a checkout session at least has a page that polls.

`apply_succeeded_pays_the_invoice_the_intent_was_for` and
`an_aborted_settlement_leaves_the_invoice_open_and_emits_nothing` are the two
halves. Keying the flip on anything but its own intent left all seven delivered
invoice cases green while a settlement paid an invoice it was never bound to;
`a_settlement_pays_only_the_invoice_its_own_intent_is_bound_to` is what fails
now.

`vpay_worker::handlers::invoice_snapshot` projects the paid row for the event
body before the statement runs. If the projection is wrong the settlement
re-evaluates the compare-and-swap inside its transaction and writes **no event
at all**, so it can never tell a merchant about a payment that did not happen —
only fail to tell them about one that did.

**Refunds are written in the refund's own settlement transaction**
(`apply_refund_succeeded`), as `amount_refunded = amount_refunded + $n` — an
expression over the row's own column, not a total read first — so two
concurrent refunds add up and `refunded_at_most_paid` refuses the over-refund
rather than clamping it. `invoice.paid` is **not** re-emitted: the invoice did
not transition. None of this path is reachable today (see
[not-built.md](not-built.md)).
