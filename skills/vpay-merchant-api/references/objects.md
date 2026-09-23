# Wire objects

_Verified against vpay `b747e5d5` (2026-09-23). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Every wire DTO lives in one module: `vpay_api::model`
(`backends/crates/vpay-api/src/model.rs`). They are rendered, never returned as
errors, and never deserialised from a merchant's request — request shapes are
separate `Deserialize` structs in each `v1::*` handler module.

## The rule that governs every object here

**`#[serde(skip_serializing_if)]` is the exception in this module, never the
default**, deliberately. Every other key is present on every render, so an SDK can model a field as
_required and nullable_ and an absent key never means "unknown". `name: null`
on an `account_holder` is a **meaningful answer** — "the rail does not know
this number" — which an omitted key could not express.

The module header says there is none "anywhere below". There are **five**
attributes on **four** types in `vpay-api/src/model.rs`, all on the payer
surface or an erased customer, each argued in place:

- `CheckoutSessionForPayer::merchant` — present only on the payer surface;
- `RailSpec::display_name`, and both of `RailDisplayName`'s `en`/`fr` — the
  payer surface's per-rail descriptor, since 2026-09-16 (vpay#186); omitted
  when the deployment configured no name, "matching `merchant`'s convention";
- `CustomerObject::deleted` — present only on an erased customer.

_(This said "two exceptions" until 2026-09-23, when the restamp first wrote
"four sites". That counted types, not attributes; `RailDisplayName` carries
two. The `RailSpec` and `RailDisplayName` attributes postdate the `d3a8810b`
stamp.)_

The staff surface (`vpay-api/src/staff/mod.rs`) has four more:
`otpauth_uri`, `enrolment`, `access_token` and `access_token_expires_at`,
each on a sign-in or enrolment response rather than a `/v1` object.

If you add a field, add it unconditionally. `cargo xtask verify-serde` checks
the _naming_ convention (snake_case on the wire, with a table of exemptions in
ADR-0016) but not this rule; the rule is held by the module's own tripwire
tests, which pin each object's exact key set (e.g.
`the_refund_object_is_the_documented_ten_keys`). **Adding a key to an object
breaks its tripwire test on purpose** — update the count and the SDKs together.

## Id prefixes

`vpay_core::ids` owns all of them, with `is_well_formed(prefix, id)` and a
minting function per type.

| Prefix  | Object                                                                                                                                                                       |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pi_`   | PaymentIntent                                                                                                                                                                |
| `ch_`   | Charge (internal — never on the wire)                                                                                                                                        |
| `re_`   | Refund                                                                                                                                                                       |
| `evt_`  | Event                                                                                                                                                                        |
| `cs_`   | CheckoutSession                                                                                                                                                              |
| `cus_`  | Customer                                                                                                                                                                     |
| `in_`   | Invoice                                                                                                                                                                      |
| `ii_`   | InvoiceItem                                                                                                                                                                  |
| `stf_`  | Staff member                                                                                                                                                                 |
| `cred_` | Credential                                                                                                                                                                   |
| `lt_`   | Ledger transaction (internal — never on the wire; added 2026-09-15)                                                                                                          |
| `mp_`   | Manual (out-of-band) payment — on the wire only as `invoice.out_of_band_payment.id`; since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251, 2026-09-23) |

Also in `ids`: `CLIENT_SECRET_INFIX` (`_secret_`), `client_secret_suffix()`
(160 bits from the OS CSPRNG), `client_secret(id, suffix)`, and
`return_token()`. Cursor paging validates the prefix, so a `pi_…` cursor on
`/v1/events` is a `400` naming the parameter rather than an empty page.

## PaymentIntent — `PaymentIntentObject`

| Field                  | Type                    | Notes                                                  |
| ---------------------- | ----------------------- | ------------------------------------------------------ |
| `id`                   | string                  | `pi_…`                                                 |
| `object`               | `"payment_intent"`      |                                                        |
| `amount`               | integer                 | **minor units** — see `vpay-payments`                  |
| `currency`             | string                  | **lowercase on the wire**, uppercase in the database   |
| `status`               | enum                    | `IntentStatus` — five values, **no `failed`**          |
| `payment_method_types` | string[]                | rail codes: `mtn_momo`, `orange_money`                 |
| `next_action`          | object or null          | redirect rails only                                    |
| `last_payment_error`   | object or null          | `{ code, message }`                                    |
| `metadata`             | object of string→string |                                                        |
| `description`          | string or null          |                                                        |
| `customer`             | string or null          | `cus_…`; accepted-and-dropped before S4a, stored since |
| `created`              | integer                 | Unix **seconds**                                       |
| `livemode`             | boolean                 |                                                        |

`PaymentIntentWithSecret` is the same object `#[serde(flatten)]`ed plus
`client_secret` — **thirteen** keys plus one, as the table above lists and
`the_browser_wrapper_is_the_thirteen_keys_plus_the_client_secret` pins.
_(This said "twelve keys plus one" until 2026-09-23. `customer` became the
thirteenth on 2026-09-06 (S4a). `model.rs`'s own doc comments on
`PaymentIntentWithSecret` and `ExpandableIntent` still say "twelve".)_ It is returned by `create`, `confirm`
and the browser reads; the plain object is what a list renders.

`NextAction` is `#[serde(tag = "type")]` with exactly one variant today:
`redirect_to_url` → `RedirectToUrl { url, return_url }`. A rail given no
`return_url` renders `return_url: null`, not a missing key.

`LastPaymentErrorObject.code` is a **`String`, not `vpay_core::FailureCode`**,
and that is deliberate on both SDKs too: the column is text, the vocabulary is
owned by the core and may grow, and a value that failed to parse would make a
merchant's `GET` answer `500` instead of showing them why their payment failed.
The vocabulary is enforced where it is _written_, not on the read path.

## Refund — `RefundObject`

Exactly ten keys: `id` (`re_…`), `object`, `amount`, `currency`,
`payment_intent`, `status` (`RefundStatus`), `reason`, `metadata`, `created`,
`fee`.

**One renderer for all five refund routes and both event types.** Two would
let the documented webhook fallback answer a different question from the one
the webhook asked, which is the same rule `vpay_api::v1::events` follows for
its own object.

`fee` is `Option<i64>` and the null-versus-zero distinction is the whole reason
it exists (issue #46). See `vpay-payments` for the invariant it protects. It is
`null` on every row in every deployment, and will stay so: only the settlement
writes it, and nothing settles a `pending` refund.

~~Nothing writes a `refunds` row.~~ **Corrected 2026-09-16:
`POST /v1/refunds` does (RFC-0003 § 2).** What the create accepts, which is
_not_ the object's key set:

| Param            | Notes                                                                                                                                                               |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `payment_intent` | required; must be this merchant's `succeeded` intent, else a `404`                                                                                                  |
| `amount`         | **a string on the wire** — the body is form-encoded, and typing it here would hand "not a number" to serde, which answers `param: "body"`. Defaults to what remains |
| `reason`         | optional                                                                                                                                                            |
| `destination`    | a nested map — see below                                                                                                                                            |
| `metadata`       | the usual bounds                                                                                                                                                    |

**`destination` is the eleventh key that is deliberately not on the object.**
The wire shape is `destination[<rail_code>][msisdn]` — nested under the rail's
own code, for `payment_method_data`'s reason: a typed struct here would have to
name `mtn_momo` as a field, and `if provider == "mtn_momo"` outside an adapter
crate is what ADR-0002 forbids. It is **required** on a rail declaring
`RefundDestination::Required` (both rails, today) and **refused** on one that
returns money to the instrument that paid. The **adapter** parses the inner
map; the core's whole business with it is presence.

**It is persisted nowhere**: there is no `destination` column on `refunds`,
because its retention is RFC-0003 open question 3 and is **undecided** — the
customer object settled on twelve months and a refund destination has no
policy. It is in no response, in no event body, and logged only **masked**
(`+2376••••200`). The cost, stated rather than hidden: vpay cannot tell an
operator which payee a refund went to. One qualification, recorded on review
2026-09-16: `refunds.failure_raw` stores the rail's own refusal text and a rail
may echo the payee into it. Nothing renders that column — `RefundObject` is ten
keys and none is a failure field — so it reaches no response and no webhook.

`POST /v1/refunds/{id}` updates **`metadata` and nothing else**: `amount`,
`reason`, `payment_intent` and `destination` each earn a `400` naming the
parameter, refused _before_ the claim so the key is left unspent and the
corrected retry is a fresh request rather than a replay of the refusal.

## CheckoutSession — `CheckoutSessionObject`

`id` (`cs_…`), `object` (`"checkout.session"`), `livemode`, `payment_intent`,
`ui_mode`, `status`, `payment_status`, `success_url`, `cancel_url`,
`return_url`, `url`, `customer`, `expires_at`, `created`.

`ui_mode`, `status` and `payment_status` are **plain `String`s on this type**,
not Rust enums. The vocabulary is enforced by database CHECKs where the values
are _written_ (`ui_mode_is_known`, `status_is_known`, `payment_status_is_known`,
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
A field sent **empty** is _cleared_, which is a different request from omitting
it; `address[…]` **replaces** the address whole rather than merging.

`AddressObject`: `city`, `state`, `postal_code`, `country`,
`latitude_microdeg`, `longitude_microdeg` — coordinates are integer
microdegrees, consistent with the no-float rule.

## Invoice — `InvoiceObject`

`id` (`in_…`), `object`, `customer`, `currency`, `status`
(`vpay_core::InvoiceStatus`), `number`, `amount_due`, `amount_paid`,
`amount_remaining`, `amount_refunded`, `paid_out_of_band`,
`out_of_band_payment`, `due_date`, `description`, `metadata`,
`payment_intent`, `hosted_invoice_url`, `lines`, `status_transitions`,
`created`, `livemode` — **twenty-one keys** since vaam-apps/vpay step A
(RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251), pinned by
`the_invoice_object_is_the_documented_twenty_one_keys`. This list had nineteen
until 2026-09-23 (no `paid_out_of_band`, no `out_of_band_payment`), which is
still the shape of any older `master`.

`out_of_band_payment` is `null`, or `{id, method, reference, received_at}` —
four keys and **no `object`**. `method` is `cash | cheque | bank_transfer |
other`; `reference` is `[redacted]` once the invoice's customer is erased.
**Nothing in vpay verified it**; it is the merchant's statement, echoed.

`lines` is a `ListObject<InvoiceLineObject>` with `has_more` always `false` and
a `url` of `/v1/invoice_items` — a route that exists — rather than Stripe's
`/v1/invoices/{id}/lines`, which vpay does not serve.

`InvoiceStatusTransitions`: `finalized_at`, `paid_at`, `voided_at`,
`marked_uncollectible_at`, all Unix seconds or null.

`hosted_invoice_url` and `payment_intent` are `null` until
`POST /v1/invoices/{id}/pay` mints an intent. An out-of-band `pay`
(2026-09-23) sets neither. An invoice paid out of band therefore carries
`null`, or the canceled earlier attempt's intent and URL, which stay attached
(ADR-0024 D15). Neither is evidence of a rail payment. _(Added 2026-09-23.)_

**The four `invoice.*` webhook bodies carry `lines.data` EMPTY** while this
object always carries them — a real, deliberate difference between what a
webhook says and what the API says about the same object. See `vpay-webhooks`.

## Event — `EventObject`

`id` (`evt_…`), `object`, `type` (the Rust field is `kind`, serde-renamed),
`created`, `livemode`, `data` (`EventDataObject { object: Value }`).

**One renderer, and that is the point.** `GET /v1/events` and the webhook
deliverer serialise the _same_ `EventObject`. Two renderers would let the
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

| Bound                        | Value                                                     |
| ---------------------------- | --------------------------------------------------------- |
| `MAX_AMOUNT`                 | `(1 << 53) - 1` (safe in JS)                              |
| metadata key                 | 40 chars                                                  |
| metadata value               | 500 chars                                                 |
| `description`                | 1000 chars                                                |
| `out_of_band[reference]`     | 1–500 chars (step A, D13)                                 |
| `out_of_band[received_at]`   | ≤ now + 30 s, ≥ `finalized_at`'s second (step A, D11/D14) |
| `return_url`                 | 2048 chars                                                |
| `last_payment_error.message` | 512 chars                                                 |
| raw rail failure text stored | 2000 chars                                                |
| form nesting depth           | 8 (`vpay_api::form`)                                      |
| request body (`/v1`)         | 64 KiB                                                    |

Request encoding is `application/x-www-form-urlencoded`, Stripe-shaped:
`metadata[order_id]=1234`, `payment_method_data[mtn_momo][msisdn]=…`, and
**both** the indexed `payment_method_types[0]=…` form the SDKs send and the
unindexed `payment_method_types[]=…` form the curl examples use.
