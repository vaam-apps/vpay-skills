# The event vocabulary and its writers

_Verified against vpay `93c6dfd0` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Verified against the code on **2026-09-16**.

## Where the vocabulary lives

The closed set is a **database CHECK**, `type_is_a_documented_event` on
`events.type`. It has been rewritten by each migration that
added a type:

| Migration                                         | Added                                  |
| ------------------------------------------------- | -------------------------------------- |
| `0018_create-events.sql`                          | the original seven                     |
| `0028`/`0029_events-checkout-session-expired.sql` | `checkout.session.expired`             |
| `0034_create-customers.sql`                       | `customer.deleted`                     |
| `0036_create-invoices.sql`                        | the four `invoice.*` types             |
| `0039_events-customer-created-updated.sql`        | `customer.created`, `customer.updated` |

`0039` is the current one. There is no Rust enum: `vpay_db::EventRow::r#type`
and `vpay_api::model::EventObject::kind` are both `String`. The SDKs carry a
`KnownEventType` for ergonomics, but the authority is the CHECK.

**Adding a type is therefore a migration plus a writer, in the same commit**
(the lockstep rule, migration `0023`). Adding it to the CHECK alone produces a
type the SDKs will list and nothing will ever send — which is the exact state
`payment_intent.canceled` was in for a week of releases, and the reason the
rule exists.

## Who writes what

| Type                            | Written by                                                                                        | Since                   |
| ------------------------------- | ------------------------------------------------------------------------------------------------- | ----------------------- |
| `payment_intent.created`        | **— nothing**                                                                                     | —                       |
| `payment_intent.processing`     | **— nothing**                                                                                     | —                       |
| `payment_intent.succeeded`      | `Settlement::apply_succeeded`                                                                     | 2026-09-03              |
| `payment_intent.payment_failed` | `vpay_db::settlement::apply_failed` **and** `vpay_api::v1::payment_intents::persist_decline`      | 2026-09-03 / 2026-09-10 |
| `payment_intent.canceled`       | `vpay_api::v1::payment_intents::cancel_with_event`                                                | 2026-09-10              |
| `charge.refunded`               | `vpay_api::v1::refunds::write_pending_refund`                                                     | 2026-09-16              |
| `charge.refund.updated`         | `vpay_api::v1::refunds` — `cancel_once`, `update_once`, `fail_with_event`                         | 2026-09-16              |
| `checkout.session.expired`      | `vpay_db::checkout_sessions::expire_due` (the hourly sweep only)                                  | 2026-09-04              |
| `customer.created`              | `vpay_api::v1::customers::create_with_event`                                                      | 2026-09-10              |
| `customer.updated`              | `vpay_api::v1::customers::update_once`, under the row's lock                                      | 2026-09-10              |
| `customer.deleted`              | `vpay_db::customers::erase_idle` (retention sweep) **and** `vpay_api::v1::customers::delete_once` | 2026-09-06 / 2026-09-10 |
| `invoice.created`               | `vpay_api::v1::invoices::write_with_event`                                                        | 2026-09-07              |
| `invoice.finalized`             | `vpay_api::v1::invoices::write_with_event`                                                        | 2026-09-07              |
| `invoice.paid`                  | `Settlement::apply_succeeded`                                                                     | 2026-09-07              |
| `invoice.voided`                | `vpay_api::v1::invoices::write_with_event`                                                        | 2026-09-07              |

### The two with no writer

`payment_intent.created` and `payment_intent.processing` are **progress**, and
vpay writes events for terminal transitions only.

~~The two refund types have no writer because no rail in this repository
refunds anything — `mtn_momo::refund` is `NotImplemented` and Orange Money's
Web Payment product documents no refund API at all, so its adapter inherits
the port's `Unsupported`. The knock-on: nothing writes a `refunds` row,
`POST /v1/refunds` is unrouted, and `vpay_db::Refunds` is two reads and no
write.~~ **Corrected 2026-09-16 — every clause of that is now false.** All
five `/v1/refunds` routes are mounted, `vpay_db::Refunds::create` writes rows,
neither rail answers `Unsupported` any more, and both refund types have
writers. What the paragraph got right is the only thing that has not changed:
**no rail has ever returned money to anyone.**

### The two refund types, in detail — emitted since 2026-09-16

They were in the original seven of `0018` and nothing wrote them for the whole
of this repository's history until RFC-0003 § 2 mounted the routes. Four things
a merchant handler has to know:

1. **`charge.refunded` is written for a refund that is only `pending`**, in
   the transaction that creates the row and reserves the amount against the
   intent (`write_pending_refund`). The transition it reports is "this charge
   now has a refund against it", which has happened. Its body carries
   `status: "pending"`.
2. **There is no event that says the money arrived, and there will not be one
   today.** Nothing settles a pending refund — there is no refund poll ladder
   (RFC-0003 open question 8) — so no code path in this repository moves a
   refund out of `pending`. A handler that blocks on
   `charge.refund.updated` with `status: "succeeded"` blocks for ever.
   `vpay_db::Settlement::apply_refund_succeeded` is the method that would
   write that transition, it is fully implemented and tested, and it is called
   by no shipping binary.
3. **`charge.refund.updated` says a refund that already existed changed** —
   Stripe's own split. Three writers, all in `vpay_api::v1::refunds`:
   - `cancel_once` — the compare-and-swap, the released reservation and the
     event, in one transaction;
   - `update_once` — the metadata merge under the row's lock. **A request
     that carries no `metadata` at all writes nothing and emits nothing**; it
     reads the object back unchanged, because an event about a change that did
     not happen is a webhook a merchant has to work out how to ignore;
   - `fail_with_event` — reached **inside the create request**, when the rail
     answers `Rejected`, `NotImplemented`, `Unsupported` or `Config`. The
     refund goes `pending` -> `failed`, the reservation goes back, the event
     is written, and the merchant gets an error response rather than a `201`.
     So on `orange_money`, whose `refund` is a declared `NotImplemented`
     token, **a create emits `charge.refunded` and then
     `charge.refund.updated` for the same refund inside one HTTP request**,
     and the merchant sees an error. A handler must tolerate that ordering
     arriving out of order, because delivery is unordered.

   The one `ProviderError` split that does **not** emit: `Transport` and
   `Malformed` mean vpay does not know what the rail did with the
   instruction, so nothing is released, no event is written, the refund stays
   `pending` with its reservation held, and the merchant is answered `201`
   with that pending refund. Reconcile it against the rail by its
   `provider_reference_id` — **nothing in this repository will move it.**
4. **`apply_refund_succeeded` deliberately emits neither**, and that is not an
   oversight to fix by adding an `INSERT` there. Emitting one needs the wire
   object `vpay-api` shapes, which is the caller's to supply; a caller that
   settles a refund through that method owes the merchant a
   `charge.refund.updated`, and the method's signature is what would have to
   carry it, as `erase_customer_in_tx`'s does.

**Both bodies are the ten-key `RefundObject`** — `id`, `object`, `amount`,
`currency`, `payment_intent`, `status`, `reason`, `metadata`, `created`, `fee`
— rendered by the same type `GET /v1/refunds/{id}` returns.
`the_refund_object_is_the_documented_ten_keys` is the tripwire, and it is
there because an eleventh key reaches a signed body stored in `events` for
ever. **The payee is not among the ten**, and cannot be: the `destination` a
merchant sends is persisted in no column at all (retention is RFC-0003 open
question 3, undecided). vpay cannot tell an operator which payee a refund went
to; the rail's records can, by `provider_reference_id`.

### Two Stripe invoice types deliberately absent from the list

`invoice.marked_uncollectible` and `invoice.payment_failed` are **not** in the
CHECK, because nothing writes them. `POST /v1/invoices/{id}/mark_uncollectible`
is a single statement with no transaction to put an event in, and a failed
intent leaves the invoice `open` with the merchant already receiving
`payment_intent.payment_failed`. A merchant learns about a write-off from
`GET /v1/invoices?status=uncollectible`.

If you add either, you are adding a transaction as well as a type.

## Why `payment_intent.payment_failed` has two writers

A rail can refuse a charge in two places: at the submit, before anything polled
it (`persist_decline` — the synchronous `409 charge_declined` a merchant gets),
and at a later status query (`apply_failed` — the worker's poll ladder). To a
merchant those are one thing: the payment did not go through, the intent is
back at `requires_payment_method`, `last_payment_error` says why. A second type
would be a type a Stripe-shaped handler has no branch for.

A merchant cannot receive both for one intent: one charge per intent forever,
and the submit path runs only when the rail refused before anything polled it.

Until 2026-09-10 only the poll path emitted, and the submit path was the one
terminal outcome no signed event reported (issue #57). The visible cost was in
`examples/shop`: MTN's documented test number `237600000400` is refused at
submit, so the demo shop's order stayed `unpaid` for ever.

## Same transaction, always

Every writer above puts its `events` row in the **same transaction** as the
transition it reports. There is no other shape in this repository.

Consequences worth knowing before you add one:

- `checkout.session.expired` is written by the sweep's flip, and **not** by
  `POST /v1/checkout/sessions/{id}/expire` — "you asked for it". A session the
  settlement finished already produced a `payment_intent.*` event for the same
  thing.
- `invoice.paid` is **not re-emitted** if a settlement lands on an invoice that
  is no longer `open` — that is a `WARN`, not a second event.
- `cancel` writes no event when its compare-and-swap refuses (a status that
  forbids it, or a charge the rail may still be acting on).
- The pooled, non-transactional `vpay_db::PaymentIntents::cancel` was
  **deleted** rather than left beside the transactional one, so "cancel without
  an event" is no longer expressible. Do not reintroduce a pooled writer beside
  a transactional one.

## The bodies

`data.object` is rendered by the same `vpay_api::model` types the API returns —
one renderer, on purpose. A merchant who missed a webhook is told to re-read
the event at `GET /v1/events/{id}`, and two renderers would let that answer a
different question from the one the webhook asked. The difference would surface
as a signature check failing against a body the merchant re-fetched.

Three bodies differ from what the API would return for the same object, each
for a stated reason:

**`invoice.*` carries `lines.data` EMPTY.** The `/v1` object always carries the
lines. The event's `data` is rendered inside the transaction that wrote the
row, and reading the lines there would put a second query on a connection
already holding the invoice number sequence's row lock. A merchant who needs
the lines reads `GET /v1/invoices/{id}`.

**`checkout.session.expired` carries a snapshot, not the row.**
`CheckoutSessionObject::expired_snapshot` renders `status: "expired"` and
`url: null` while the row it was built from still says `open` — the event
describes the transition, and the sweep's own commit is what makes the row
agree. `url` is `null` because a hosted session's `url` carries its
`client_secret` in the fragment, and a webhook body is stored, delivered
at-least-once and replayed. **`url: null` therefore does not mean the session
was embedded — read `ui_mode`.** `payment_status` is deliberately untouched: an
expired session that was already `paid` keeps saying so.

**`customer.deleted` carries every identifier already `[redacted]`.** vpay
stores event bodies in `events` for ever. Between a merchant's convenience in
identifying the payer and the payer's erasure being real, the erasure wins
(reversed 2026-09-10, issue #68 — older prose saying the body carries `name`,
`email` and `phone` is stale). This is also the one type a merchant cannot
substitute polling for: every other event describes a row that is still there
afterwards, and this one describes an erasure that makes
`GET /v1/customers/{id}` byte-identical to one for an id that never existed.

## The read fallback

`GET /v1/events` and `GET /v1/events/{id}` are merchant-scoped, newest first,
cursor-paged with `limit` / `starting_after` / `ending_before`. A foreign
merchant's id is the same 404 a nonexistent one gets, byte for byte.

**`?type=` is documented in `docs/api/README.md` and deliberately not
implemented.** A filter interacts with the cursor — `has_more` and the `seq`
window both have to be computed over the _filtered_ set or paging silently
skips rows — and half of that is worse than none. Unknown query parameters are
ignored everywhere on this surface, so `?type=…` returns an **unfiltered page**
rather than a `400`. Do not assume a filtered read.

There is no `POST /v1/events` and there will not be one: an event is a record
of something vpay did, and a merchant who could create one could forge their
own history. Retrying a _delivery_ is an operator action against
`webhook_deliveries` (`docs/runbooks/webhook-delivery-failures.md`).
