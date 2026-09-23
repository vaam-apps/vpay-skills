---
name: vpay-invoices
description: vpay's Invoice and invoice-line objects — the wire shape (twenty-one keys since vpay step A, nineteen before), the draft/open/paid/void/uncollectible state machine that is enforced by compare-and-swap UPDATE statements and multi-column CHECKs rather than by any can_transition_to method, the two writers of paid (the settlement, and pay with paid_out_of_band=true, which records a merchant's unverifiable statement and posts nothing to the ledger), the per-merchant document number that burns no holes, the eight routes (POST and PATCH on /v1/invoices/{id} are one handler), the invoice.* webhook bodies that carry lines.data EMPTY, and the long list of things deliberately not built — no PDF, no e-mail, no tax, no credit note, no dunning, no subscription. Load this before touching /v1/invoices, /v1/invoice_items, the settlement's invoice flip, manual_payments, or migrations 0036/0042/0049.
---

# Invoices and invoice items

> **Verified against vpay `7997536b` (2026-09-23).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

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

**Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged <pending>): manual
payments** — migration `0049`, `manual_payments` (`mp_…`), two new object
keys, a second writer of `paid` and `invoice.paid`, nothing on the ledger. On an
older `master` none of it exists. On the branch, 2026-09-23: `invoices.rs` 29
cases, 29 passed, 0 ignored; `tests/repositories.rs` twelve on invoices.
[§ Paid out of band](#paid-out-of-band--a-statement-not-a-payment).

**The "What is not built" list is the whole reason the flow page exists** —
read it before assuming anything:
[references/not-built.md](references/not-built.md). Short version: no PDF, no
e-mail, no hosted invoice page, no tax, no discount, no credit note, no
dunning, no subscription, no partial payment, no dashboard screen, and
`amount_refunded` is `0` on every invoice in every deployment.

> **Proposed on 2026-09-23, not built:** RFC-0004 (billing on top of
> invoices: products and prices, subscriptions, pending invoice items,
> ~~manual payments,~~ taxes, coupons, PDFs), RFC-0005 (prepaid customer
> balances) and a usage-metering service brief, merged by vaam-apps/vpay#244
> as `7997536b` (`docs/rfc/0004…`, `0005…`,
> `docs/plans/2026-09-23-metering-service.md`), all **Draft** with open
> questions. Every item in the list above is still true. Do not implement one
> of those features from the RFC as though it were decided, and do not
> describe any of it in the present tense. The list of what the RFCs would
> change is in [references/not-built.md](references/not-built.md).
>
> **Corrected 2026-09-23:** this block listed manual payments as proposed.
> Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged <pending>), RFC-0004 is
> **Draft except § 5's `customer` filters and § 6, manual payments**, which
> are accepted through ADR-0024
> (`docs/adr/0024-customer-filters-and-manual-payments.md`, D1–D19) and built.
> Everything else the RFC names — the invoice preview in § 5,
> `/v1/subscriptions`, products, prices, taxes, coupons, PDFs — is still Draft
> and unbuilt.

> **The reason `amount_refunded` is `0` changed on 2026-09-16, and the new one
> is narrower.** ~~No vpay rail can refund, and `POST /v1/refunds` is
> unrouted.~~ All five `/v1/refunds` routes are mounted now (RFC-0003 § 2),
> `vpay_db::Refunds::create` writes rows, and
> `Settlement::apply_refund_succeeded` — which is the only statement that
> moves `invoices.amount_refunded` — is implemented, tested and **called by
> nothing**. The gap is that **nothing settles a pending refund**: there is no
> refund poll ladder (RFC-0003 open question 8), so no code path moves a
> refund out of `pending` and the statement never runs. Getting this
> distinction right matters, because "the route does not exist" and "the route
> exists and its output never reaches the column" send an agent to completely
> different files.

## The two objects

**Invoice — `in_…`, twenty-one keys since vaam-apps/vpay step A (RFC-0004
§§ 5–6, merged <pending>)**, held by
`the_invoice_object_is_the_documented_twenty_one_keys` in `vpay_api::model`.
~~Nineteen keys, held by `the_invoice_object_is_the_documented_nineteen_keys`~~
— this page said that until 2026-09-23, and it is still true of any `master`
older than that merge. Eighteen until `0042` added `amount_refunded`:

`id`, `object`, `customer`, `currency`, `status`, `number`, `amount_due`,
`amount_paid`, `amount_remaining`, `amount_refunded`, `paid_out_of_band`,
`out_of_band_payment`, `due_date`, `description`, `metadata`,
`payment_intent`, `hosted_invoice_url`, `lines`, `status_transitions`,
`created`, `livemode`.

`paid_out_of_band` is Stripe's bool; `out_of_band_payment` is vpay's own —
`null`, or four keys and no `object`: `{id: "mp_…", method, reference,
received_at}`. They agree by construction (true ⇔ a record). Both SDKs decode
them with a default, so a client of the new shape still reads an older
server.

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
  |                           ^
  |                           +-- the settlement of its own intent, or
  |                           +-- pay with paid_out_of_band=true (step A)
  +-- DELETE /v1/invoices/{id} --> gone, lines and all
```

Every right-hand state is terminal. Nothing in vpay moves an invoice out of
`paid`, `void` or `uncollectible`, and no route tries — so an out-of-band
payment recorded in error **cannot be undone**; there is no route for it and
none is planned by ADR-0024.

**`open → paid` has two writers since vaam-apps/vpay step A (RFC-0004 §§ 5–6,
merged <pending>)**: the settlement transaction, and the out-of-band
compare-and-swap. Both match `status = 'open'`; the second also carries
`NO_LIVE_INTENT`, so a settlement and an out-of-band payment cannot both win.
Before that merge the settlement was the only writer, and code or prose that
assumes `paid` implies "a rail collected money" is wrong from then on.

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

Details of the CHECKs (five in `0036`, more since), the numbering, `NO_LIVE_INTENT`, and the
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
| `paid_out_of_band_means_paid` (`0049`, step A)                   | the flag on anything but a `paid` row                                                                                     |
| `paid_names_how` (`0049`, step A)                                | a `paid` row with neither an intent nor the flag — storable before `0049`, when only the settlement wrote `paid`          |
| `paid_out_of_band_is_never_refunded` (`0049`, step A)            | `amount_refunded > 0` on an invoice paid out of band                                                                      |

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

| Route                                  | Methods                          | Notes                                                                                                                                           |
| -------------------------------------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `/v1/invoices`                         | `POST`, `GET`                    | list takes `customer`, `status`, standard cursor                                                                                                |
| `/v1/invoices/{id}`                    | `GET`, `POST`, `PATCH`, `DELETE` | `POST`/`PATCH` are **one handler**; both draft-only, as is `DELETE`                                                                             |
| `/v1/invoices/{id}/finalize`           | `POST`                           |                                                                                                                                                 |
| `/v1/invoices/{id}/void`               | `POST`                           | open only                                                                                                                                       |
| `/v1/invoices/{id}/mark_uncollectible` | `POST`                           |                                                                                                                                                 |
| `/v1/invoices/{id}/pay`                | `POST`                           | `success_url`, `cancel_url` — sent, or from `merchant_clients[].invoices`; **or**, since step A, `paid_out_of_band=true` and no URL — see below |
| `/v1/invoice_items`                    | `POST`                           | **no collection `GET`**                                                                                                                         |
| `/v1/invoice_items/{id}`               | `GET`, `POST`, `PATCH`, `DELETE` | writes are draft-parent-only                                                                                                                    |

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

## Paid out of band — a statement, not a payment

Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged <pending>).
`POST /v1/invoices/{id}/pay` with `paid_out_of_band=true` and optional
`out_of_band[method|reference|received_at]` (`method` defaults to `other`, so
Stripe's bare flag works) moves an `open` invoice to `paid`. What an agent must
not get wrong:

- **Nothing is posted to the ledger, and nothing verifies it.** It is the
  merchant's word. Never add a posting "for completeness", never describe it
  as money vpay saw, and never read `status = 'paid'` as "a rail collected".
- **Refused while a live intent is attached** (`NO_LIVE_INTENT`, the same
  `409` as `void`), and **terminal**: there is no undo.
- **Only `/v1` writes it**; no operator can. `bank_transfer` is a label, not a
  rail — matching transfers is RFC-0007, Draft.
- **`reference` is personal data**, redacted by customer erasure everywhere it
  was copied (`vpay-customers`).

Every refusal, the composite foreign key, the canceled-intent consequence and
the evidence: [references/paid-out-of-band.md](references/paid-out-of-band.md).

## The four events, and the divergence that will bite a merchant

`invoice.created`, `invoice.finalized`, `invoice.paid` (**the settlement
transaction**, not a write afterwards — and, since vaam-apps/vpay step A
(RFC-0004 §§ 5–6, merged <pending>), also the out-of-band `pay`, in its own
transaction) and `invoice.voided`, each written inside the transaction of the
transition it describes; a refused transition writes none. All four entered
`type_is_a_documented_event` in `0036` **with their writer in the same
commit**; the second writer of `invoice.paid` needed no vocabulary change. A
merchant tells the two apart by the body: `paid_out_of_band: true` and a
filled `out_of_band_payment` on the out-of-band writer's.

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

Verified 2026-09-16, both ungated. ~~**`docs/flows/invoices.md` is missing
from the table in `docs/flows/README.md`** — every other flow page is listed,
and nothing checks that table.~~ **Corrected 2026-09-23:** vaam-apps/vpay
step A (RFC-0004 §§ 5–6, merged <pending>) adds the row; on an older `master`
it is still missing, and nothing checks that table either way. And **the
page's own
`[What is not built](#what-is-not-built)` link is a dead anchor**: there is no
such heading, the list is bold prose under `## Status`, and
`cargo xtask verify-links` resolves destination _paths_ only ("anchors and
http(s) URLs are not checked").

One more stale claim to ignore: `vpay_api::v1::invoices::create`'s doc comment
and `docs/flows/invoices.md` § Events both say `customer.created` and
`customer.updated` "have no writer and are deliberately absent from the
database's event vocabulary". **Migration `0039` added both on 2026-09-10 and
both have writers.**
