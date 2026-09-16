# Wire objects

Every wire DTO lives in one module: `vpay_api::model`
(`backends/crates/vpay-api/src/model.rs`). They are rendered, never returned as
errors, and never deserialised from a merchant's request — request shapes are
separate `Deserialize` structs in each `v1::*` handler module.

## The rule that governs every object here

**There is no `#[serde(skip_serializing_if)]` in this module**, deliberately.
Every key is present on every render, so an SDK can model a field as
*required and nullable* and an absent key never means "unknown". `name: null`
on an `account_holder` is a **meaningful answer** — "the rail does not know
this number" — which an omitted key could not express.

Two exceptions, both argued in place:

- `CheckoutSessionForPayer::merchant` — present only on the payer surface;
- `CustomerObject::deleted` — present only on an erased customer.

If you add a field, add it unconditionally. `cargo xtask verify-serde` checks
the *naming* convention (snake_case on the wire, with a table of exemptions in
ADR-0016) but not this rule; the rule is held by the module's own tripwire
tests, which pin each object's exact key set (e.g.
`the_refund_object_is_the_documented_ten_keys`). **Adding a key to an object
breaks its tripwire test on purpose** — update the count and the SDKs together.

## Id prefixes

`vpay_core::ids` owns all of them, with `is_well_formed(prefix, id)` and a
minting function per type.

| Prefix  | Object                                  |
| ------- | --------------------------------------- |
| `pi_`   | PaymentIntent                           |
| `ch_`   | Charge (internal — never on the wire)   |
| `re_`   | Refund                                  |
| `evt_`  | Event                                   |
| `cs_`   | CheckoutSession                         |
| `cus_`  | Customer                                |
| `in_`   | Invoice                                 |
| `ii_`   | InvoiceItem                             |
| `stf_`  | Staff member                            |
| `cred_` | Credential                              |

Also in `ids`: `CLIENT_SECRET_INFIX` (`_secret_`), `client_secret_suffix()`
(160 bits from the OS CSPRNG), `client_secret(id, suffix)`, and
`return_token()`. Cursor paging validates the prefix, so a `pi_…` cursor on
`/v1/events` is a `400` naming the parameter rather than an empty page.

## PaymentIntent — `PaymentIntentObject`

| Field                  | Type                    | Notes                                              |
| ---------------------- | ----------------------- | -------------------------------------------------- |
| `id`                   | string                  | `pi_…`                                             |
| `object`               | `"payment_intent"`      |                                                    |
| `amount`               | integer                 | **minor units** — see `vpay-payments`              |
| `currency`             | string                  | **lowercase on the wire**, uppercase in the database |
| `status`               | enum                    | `IntentStatus` — five values, **no `failed`**      |
| `payment_method_types` | string[]                | rail codes: `mtn_momo`, `orange_money`             |
| `next_action`          | object or null          | redirect rails only                                |
| `last_payment_error`   | object or null          | `{ code, message }`                                |
| `metadata`             | object of string→string |                                                    |
| `description`          | string or null          |                                                    |
| `customer`             | string or null          | `cus_…`; accepted-and-dropped before S4a, stored since |
| `created`              | integer                 | Unix **seconds**                                   |
| `livemode`             | boolean                 |                                                    |

`PaymentIntentWithSecret` is the same object `#[serde(flatten)]`ed plus
`client_secret` — twelve keys plus one. It is returned by `create`, `confirm`
and the browser reads; the plain object is what a list renders.

`NextAction` is `#[serde(tag = "type")]` with exactly one variant today:
`redirect_to_url` → `RedirectToUrl { url, return_url }`. A rail given no
`return_url` renders `return_url: null`, not a missing key.

`LastPaymentErrorObject.code` is a **`String`, not `vpay_core::FailureCode`**,
and that is deliberate on both SDKs too: the column is text, the vocabulary is
owned by the core and may grow, and a value that failed to parse would make a
merchant's `GET` answer `500` instead of showing them why their payment failed.
The vocabulary is enforced where it is *written*, not on the read path.

## Refund — `RefundObject`

Exactly ten keys: `id` (`re_…`), `object`, `amount`, `currency`,
`payment_intent`, `status` (`RefundStatus`), `reason`, `metadata`, `created`,
`fee`.

`fee` is `Option<i64>` and the null-versus-zero distinction is the whole reason
it exists (issue #46). See `vpay-payments` for the invariant it protects.

**Nothing writes a `refunds` row.** Every refund this deployment can render is
one an operator or a test put there, and `fee` is `null` on all of them.

## CheckoutSession — `CheckoutSessionObject`

`id` (`cs_…`), `object` (`"checkout.session"`), `livemode`, `payment_intent`,
`ui_mode`, `status`, `payment_status`, `success_url`, `cancel_url`,
`return_url`, `url`, `customer`, `expires_at`, `created`.

`ui_mode`, `status` and `payment_status` are **plain `String`s on this type**,
not Rust enums. The vocabulary is enforced by database CHECKs where the values
are *written* (`ui_mode_is_known`, `status_is_known`, `payment_status_is_known`,
migration `0028`):

- `status`: `open | complete | expired`
- `payment_status`: `unpaid | paid | failed`

`payment_intent` is an `ExpandableIntent` — `#[serde(untagged)]`, so it
serialises as **a string or a whole object with no wrapper**. `/v1` renders the
id (a merchant already holds the intent they created, and expanding in a list
would repeat every amount per row); `/v1/browser` renders the expanded intent,
because the payer page holds only a session id and needs amount, currency,
status, `payment_method_types`, `next_action` and `last_payment_error` before
it can paint anything.

`CheckoutSessionObject::expired_snapshot(row)` builds the `data.object` of a
`checkout.session.expired` event: it renders `status: "expired"` and `url: None`
while leaving the row itself saying `open`, and leaves `payment_status`
untouched — an expired session that was already `paid` keeps saying so.

## Customer — `CustomerObject`

`id` (`cus_…`), `object`, `name`, `email`, `phone`, `address`, `metadata`,
`created`, `livemode`, and `deleted: true` only when erased.

At least one of `name`/`email`/`phone` is required and **`phone` alone is
enough** (a 2026-09-05 maintainer decision — this market's payers often have no
email). A patch that would clear the last identifier is a `400` naming `name`.
A field sent **empty** is *cleared*, which is a different request from omitting
it; `address[…]` **replaces** the address whole rather than merging.

`AddressObject`: `city`, `state`, `postal_code`, `country`,
`latitude_microdeg`, `longitude_microdeg` — coordinates are integer
microdegrees, consistent with the no-float rule.

## Invoice — `InvoiceObject`

`id` (`in_…`), `object`, `customer`, `currency`, `status`
(`vpay_core::InvoiceStatus`), `number`, `amount_due`, `amount_paid`,
`amount_remaining`, `amount_refunded`, `due_date`, `description`, `metadata`,
`payment_intent`, `hosted_invoice_url`, `lines`, `status_transitions`,
`created`, `livemode`.

`lines` is a `ListObject<InvoiceLineObject>` with `has_more` always `false` and
a `url` of `/v1/invoice_items` — a route that exists — rather than Stripe's
`/v1/invoices/{id}/lines`, which vpay does not serve.

`InvoiceStatusTransitions`: `finalized_at`, `paid_at`, `voided_at`,
`marked_uncollectible_at`, all Unix seconds or null.

`hosted_invoice_url` and `payment_intent` are `null` until
`POST /v1/invoices/{id}/pay`.

**The four `invoice.*` webhook bodies carry `lines.data` EMPTY** while this
object always carries them — a real, deliberate difference between what a
webhook says and what the API says about the same object. See `vpay-webhooks`.

## Event — `EventObject`

`id` (`evt_…`), `object`, `type` (the Rust field is `kind`, serde-renamed),
`created`, `livemode`, `data` (`EventDataObject { object: Value }`).

**One renderer, and that is the point.** `GET /v1/events` and the webhook
deliverer serialise the *same* `EventObject`. Two renderers would let the
documented fallback answer a different question from the one the webhook asked,
and the difference would surface as a merchant's signature check failing
against a body they re-fetched.

## AccountHolder — `AccountHolderObject`

Four keys, always four: `object`, `payment_method_type`, `name` (nullable and
meaningfully so), `verified`. **There is no row behind this object** — it is
rendered from a rail's live answer and never stored.

## Lists and deletions

`ListObject<T>`: `object` (`"list"`), `data`, `has_more`, `url`. Cursor-paged
with `limit` / `starting_after` / `ending_before`; `DEFAULT_LIMIT` is 10 and
`MAX_LIMIT` is 100 (`vpay_api::v1::paging`).

The `/dash/v1/$procs/search*` procedures are **offset-paged instead**, and
their own module docs say what that costs: `OFFSET` is recounted per call, so a
row inserted behind a caller's back shifts the window. Anything that must not
double-count — a reconciliation, an export, a sum — reads the cursor list, not
the procedure.

`DeletedObject<T>` / `DeletedTrue`: `{ id, object, deleted: true }`.

## Validation bounds

Enforced in `vpay_api::v1::payment_intents` and shared by the other handlers:

| Bound                           | Value                       |
| ------------------------------- | --------------------------- |
| `MAX_AMOUNT`                    | `(1 << 53) - 1` (safe in JS) |
| metadata key                    | 40 chars                    |
| metadata value                  | 500 chars                   |
| `description`                   | 1000 chars                  |
| `return_url`                    | 2048 chars                  |
| `last_payment_error.message`    | 512 chars                   |
| raw rail failure text stored    | 2000 chars                  |
| form nesting depth              | 8 (`vpay_api::form`)        |
| request body (`/v1`)            | 64 KiB                      |

Request encoding is `application/x-www-form-urlencoded`, Stripe-shaped:
`metadata[order_id]=1234`, `payment_method_data[mtn_momo][msisdn]=…`, and
**both** the indexed `payment_method_types[0]=…` form the SDKs send and the
unindexed `payment_method_types[]=…` form the curl examples use.
