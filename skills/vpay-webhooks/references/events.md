# The event vocabulary and its writers

Verified against the code on **2026-09-16**.

## Where the vocabulary lives

The closed set is a **database CHECK**, `type_is_a_documented_event` on
`events.type`. It has been rewritten four times, each by the migration that
added a type:

| Migration                                         | Added                                                        |
| ------------------------------------------------- | ------------------------------------------------------------ |
| `0018_create-events.sql`                          | the original seven                                           |
| `0028`/`0029_events-checkout-session-expired.sql` | `checkout.session.expired`                                   |
| `0034_create-customers.sql`                       | `customer.deleted`                                           |
| `0039_events-customer-created-updated.sql`        | `customer.created`, `customer.updated`, the four `invoice.*` |

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
| `payment_intent.succeeded`      | `vpay_db::settlement::apply_succeeded`                                                            | 2026-09-03              |
| `payment_intent.payment_failed` | `vpay_db::settlement::apply_failed` **and** `vpay_api::v1::payment_intents::persist_decline`      | 2026-09-03 / 2026-09-10 |
| `payment_intent.canceled`       | `vpay_api::v1::payment_intents::cancel_with_event`                                                | 2026-09-10              |
| `charge.refunded`               | **— nothing**                                                                                     | —                       |
| `charge.refund.updated`         | **— nothing**                                                                                     | —                       |
| `checkout.session.expired`      | `vpay_db::checkout_sessions::expire_due` (the hourly sweep only)                                  | 2026-09-04              |
| `customer.created`              | `vpay_api::v1::customers::create_with_event`                                                      | 2026-09-10              |
| `customer.updated`              | `vpay_api::v1::customers::update_once`, under the row's lock                                      | 2026-09-10              |
| `customer.deleted`              | `vpay_db::customers::erase_idle` (retention sweep) **and** `vpay_api::v1::customers::delete_once` | 2026-09-06 / 2026-09-10 |
| `invoice.created`               | `vpay_api::v1::invoices::write_with_event`                                                        | 2026-09-07              |
| `invoice.finalized`             | `vpay_api::v1::invoices::write_with_event`                                                        | 2026-09-07              |
| `invoice.paid`                  | `vpay_db::settlement::apply_succeeded`                                                            | 2026-09-07              |
| `invoice.voided`                | `vpay_api::v1::invoices::write_with_event`                                                        | 2026-09-07              |

### The four with no writer

`payment_intent.created` and `payment_intent.processing` are **progress**, and
vpay writes events for terminal transitions only. The two refund types have no
writer because **no rail in this repository refunds anything** —
`mtn_momo::refund` is `NotImplemented` (MTN refunds are the Disbursements
product, with its own subscription key and token scope that no deployment has
been issued) and Orange Money's Web Payment product documents no refund API at
all, so its adapter inherits the port's `Unsupported`.

The knock-on: nothing writes a `refunds` row, `POST /v1/refunds` is unrouted,
and `vpay_db::Refunds` is two reads and no write. So what the event tests prove
about the refund types is that the _contract_ holds, not that a refund event
works.

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
