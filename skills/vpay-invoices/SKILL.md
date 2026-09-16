---
name: vpay-invoices
description: vpay's Invoice and invoice-line objects — the nineteen-key wire shape, the draft/open/paid/void/uncollectible state machine that is enforced by compare-and-swap UPDATE statements and five multi-column CHECKs rather than by any can_transition_to method, the per-merchant document number that burns no holes, the eight routes (POST and PATCH on /v1/invoices/{id} are one handler), the invoice.* webhook bodies that carry lines.data EMPTY, and the long list of things deliberately not built — no PDF, no e-mail, no tax, no credit note, no dunning, no subscription. Load this before touching /v1/invoices, /v1/invoice_items, the settlement's invoice flip, or migrations 0036/0042.
---

# Invoices and invoice items

> **Verified against vpay `f063ee96` (2026-09-15).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See VERSIONING.md.

A merchant's bill to one customer, S4b of the data-layer work, built on the
Customer (see the `vpay-customers` skill). Flow page: `docs/flows/invoices.md`.

## State of it, as of 2026-09-16

**REAL.** All eight routes, both objects, the four transitions, per-merchant
numbering, the four `invoice.*` events, payment through the existing hosted
checkout, and the refund arithmetic at the database — proven against a real
Postgres and the shipping router (`backends/tests/integration/tests/invoices.rs`
is 16 cases, `vpay-db`'s `tests/repositories.rs` adds 10, `postgres_smoke.rs`
pins the multi-column CHECK inventory). Both merchant SDKs ship 13 methods each
with a live suite against a real `vpay-server`.

**The "What is not built" list is the whole reason the flow page exists** —
read it before assuming anything:
[references/not-built.md](references/not-built.md). Short version: no PDF, no
e-mail, no hosted invoice page, no tax, no discount, no credit note, no
dunning, no subscription, no partial payment, no dashboard screen, and
`amount_refunded` is `0` in every deployment because **no vpay rail can
refund**.

## The two objects

**Invoice — `in_…`, nineteen keys**, held by
`the_invoice_object_is_the_documented_nineteen_keys` in `vpay_api::model`
(eighteen until `0042` added `amount_refunded`):

`id`, `object`, `customer`, `currency`, `status`, `number`, `amount_due`,
`amount_paid`, `amount_remaining`, `amount_refunded`, `due_date`,
`description`, `metadata`, `payment_intent`, `hosted_invoice_url`, `lines`,
`status_transitions`, `created`, `livemode`.

The same test names the internals that must never reach the wire — `seq`,
`merchant_id`, `updated_at`, `currency_code` — because every key here is signed
into an `invoice.*` body and stored in `events` for ever.

- **`customer` is required**, unlike a payment intent's: an invoice is a bill
  _to somebody_, and `pay` would have no payer to bind an intent to.
- **`currency` is required on create.** Both SDKs had it optional, documented
  as "the server applies a default"; there is no such default, and no stub
  answering `201` could have caught it. The live suite did, first run
  (2026-09-08).
- **`due_date` is advisory. Nothing in vpay reads it**; there is no timer.
- `number` is `null` while a draft. `status_transitions` is `finalized_at`,
  `paid_at`, `voided_at`, `marked_uncollectible_at`.

**Line — `ii_…`, rendered as `"object": "line_item"`, eight keys:** `id`,
`object`, `description`, `quantity`, `unit_amount`, `amount`, `currency`,
`livemode`. Nothing else: no price, no product, no proration, no period. (There
is **no** key-count tripwire on this one, unlike the invoice.)

**The route says `invoice_items` and the object says `line_item`, deliberately.**
Stripe has two objects where vpay has one — an `invoiceitem` there is a pending
charge not yet attached to a document, and vpay has no pending-charge inbox
because that is a subscription feature — so the route keeps Stripe's spelling
and the object keeps Stripe's `lines` spelling.

**`amount` is never a parameter**: it is `quantity * unit_amount`, computed by
the statement and checked by `amount_is_the_product`. Nor is `currency`,
`merchant` or `livemode` — all three are copied off the parent in the same
statement.

## The state machine is the `WHERE` clause

```text
draft --finalize--> open --> paid | void | uncollectible
  |
  +-- DELETE /v1/invoices/{id} --> gone, lines and all
```

Every right-hand state is terminal. Nothing in vpay moves an invoice out of
`paid`, `void` or `uncollectible`, and no route tries.

**There is deliberately no `can_transition_to` method** — not on
`vpay_core::InvoiceStatus`, not anywhere in the codebase. Every transition is a
compare-and-swap `UPDATE invoices SET … WHERE id = $1 AND merchant_id = $2 AND
status = '<from>'`, and "matched no row" **is** the refusal. A Rust guard
beside the write is the thing a future writer calls _instead of_ taking the
lock. If you add a transition, add a statement — not a predicate.

> **A draft cannot be voided.** `void_in_tx`'s `WHERE` names `'open'` alone; a
> draft is _deleted_. This is forced by `number_is_assigned_at_finalize` — a
> voided draft would be a non-draft row with no number, which the database
> refuses — and the statement never attempts it rather than discovering a
> `23514`. **Two doc comments in the repository say otherwise and are wrong**:
> `vpay_core::InvoiceStatus`'s ASCII diagram draws a `draft ──void──> void`
> edge, and `void_in_tx`'s own first line says "Voids a `draft` or `open`
> invoice" before contradicting itself three paragraphs later. The statement is
> the truth. (Found 2026-09-16; neither is gated.)

A voided invoice **keeps its number** — for the same constraint, and because a
number that vanished is a hole an accountant reads as a destroyed document.

Details of the five CHECKs, the numbering, `NO_LIVE_INTENT`, and the
settlement: [references/constraints-and-transitions.md](references/constraints-and-transitions.md).

## `paid_means_nothing_remaining` and the amounts

All amounts are integer minor units (`docs/flows/money.md`).

| Constraint (`0036`, `0042`)                                      | What it refuses                                                                                                           |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `paid_means_nothing_remaining`                                   | `status <> 'paid' OR amount_remaining = 0` — **this is where "no partial payments" stops being a sentence in a document** |
| `amounts_add_up`                                                 | `amount_paid + amount_remaining = amount_due`                                                                             |
| `number_is_assigned_at_finalize`                                 | `(status = 'draft') = (number IS NULL)`, both halves                                                                      |
| `only_a_live_invoice_has_an_intent`                              | `status <> 'draft' OR payment_intent_id IS NULL`                                                                          |
| `amount_is_the_product` (on `invoice_items`)                     | `amount = quantity * unit_amount`                                                                                         |
| `refunded_at_most_paid`, `amount_refunded_non_negative` (`0042`) | over-refund, and a rebate                                                                                                 |

`amount_refunded` is **gross and sits beside the arithmetic, not inside it**:
it is not subtracted from `amount_paid` and takes no part in `amounts_add_up`.
A refund leaves the invoice `paid`. The alternative makes a fully refunded
invoice read `paid` with the whole bill _remaining_, and `amount_remaining` is
the number `pay` mints an intent for.

**Every one of these is multi-column and therefore invisible to `cratestack
migrate baseline` in both directions.** The drift report cannot be the guard.
`the_invoice_invariants_are_enforced_by_the_database_itself` writes the row
each one exists to refuse, straight past the API and the repository.

## The eight routes

| Route                                  | Methods                          | Notes                                                                     |
| -------------------------------------- | -------------------------------- | ------------------------------------------------------------------------- |
| `/v1/invoices`                         | `POST`, `GET`                    | list takes `customer`, `status`, standard cursor                          |
| `/v1/invoices/{id}`                    | `GET`, `POST`, `PATCH`, `DELETE` | `POST`/`PATCH` are **one handler**; both draft-only, as is `DELETE`       |
| `/v1/invoices/{id}/finalize`           | `POST`                           |                                                                           |
| `/v1/invoices/{id}/void`               | `POST`                           | open only                                                                 |
| `/v1/invoices/{id}/mark_uncollectible` | `POST`                           |                                                                           |
| `/v1/invoices/{id}/pay`                | `POST`                           | `success_url`, `cancel_url` — sent, or from `merchant_clients[].invoices` |
| `/v1/invoice_items`                    | `POST`                           | **no collection `GET`**                                                   |
| `/v1/invoice_items/{id}`               | `GET`, `POST`, `PATCH`, `DELETE` | writes are draft-parent-only                                              |

`POST` and `PATCH` are `invoices::update` mounted twice, so the two cannot
answer differently. Stripe has no `PATCH` and the real `stripe` package sends
`POST`, so `POST` is the one that has to work; `PATCH` is there because a
partial update is what the verb means. (`/v1/customers/{id}` mounts no `PATCH`
at all — the two resources are deliberately inconsistent.) The three
transitions are separate `POST`s and not a `PATCH` with a `status`, so "which
transitions exist" is not something a merchant discovers by trying.

Another merchant's `in_…` is byte-identically the same `404` as an id that
never existed **on every route including the transitions**, which is what stops
a `409` naming a status being an existence oracle. All four writes take an
`Idempotency-Key`, `DELETE` included.

## The four events, and the divergence that will bite a merchant

`invoice.created`, `invoice.finalized`, `invoice.paid` (**the settlement
transaction**, not a write afterwards) and `invoice.voided`, each written
inside the transaction of the transition it describes; a refused transition
writes none. All four entered `type_is_a_documented_event` in `0036` **with
their writer in the same commit**.

> **`invoice.*` webhook bodies carry `lines.data` EMPTY**, while
> `GET /v1/invoices/{id}` carries them. Deliberate: the body is rendered inside
> the transaction that wrote the row, and reading the lines there would put a
> second query on a connection holding the sequence row's lock. Three sites
> render `InvoiceObject::render(&row, &[], None)` — `vpay_api::v1::invoices`'
> transition writer, its create, and
> `vpay_worker::handlers::invoice_snapshot`.

`invoice.marked_uncollectible` and `invoice.payment_failed` are Stripe types
vpay **does not emit** — neither is in the vocabulary, because nothing writes
them (`0023`'s lockstep rule). A failed intent leaves the invoice `open` and
emits nothing on it; the merchant learns from `payment_intent.payment_failed`.

## Two documentation defects in the flow page itself

Verified 2026-09-16, both ungated. **`docs/flows/invoices.md` is missing from
the table in `docs/flows/README.md`** — every other flow page is listed, and
nothing checks that table. And **the page's own
`[What is not built](#what-is-not-built)` link is a dead anchor**: there is no
such heading, the list is bold prose under `## Status`, and
`cargo xtask verify-links` resolves destination _paths_ only ("anchors and
http(s) URLs are not checked").

One more stale claim to ignore: `vpay_api::v1::invoices::create`'s doc comment
and `docs/flows/invoices.md` § Events both say `customer.created` and
`customer.updated` "have no writer and are deliberately absent from the
database's event vocabulary". **Migration `0039` added both on 2026-09-10 and
both have writers.**
