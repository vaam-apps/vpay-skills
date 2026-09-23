---
name: vpay-customers
description: The vpay Customer object and the account-holder lookup — phone-first identity, the at-least-one-identifier rule, the address object whose GPS half is two integer microdegree columns, the two-shape erasure that must reach every copy of a payer vpay kept (eight tables, not one), the twelve-month retention sweep, the three customer.* events, and GET /v1/account_holders' three-way answer with its three reserved-but-unbuilt privacy controls. Load this before touching /v1/customers, /v1/account_holders, the address, anything that stores or logs a payer identifier, or the sweep_idle_customers job.
---

# Customers, addresses, erasure, and the account-holder lookup

> **Verified against vpay `b747e5d5` (2026-09-23).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

This is the only table in vpay whose entire content is **another person's
personal data**, and `GET /v1/account_holders` is the only route on `/v1` that
returns information about a person who is not the caller. Almost everything
here is a privacy rule with a gate behind it, not a CRUD shape. Flow docs:
`docs/flows/customers.md` plus the five pages in `docs/flows/customers/`, and
`docs/flows/account-holder-lookup.md`.

## State of it, as of 2026-09-23

**REAL.** The five `/v1/customers` routes, the address including both
coordinate columns, both erasure branches, the retention sweep, all three
`customer.*` events, and `GET /v1/account_holders` are built and proven
against a real Postgres and the shipping router. `customers.rs` in
`backends/tests/integration/tests/` is 24 cases.

**NOT real, and do not build on it:** no deployment has ever run the retention
sweep (no vpay has been up for twelve months); the account-holder lookup has
never called MTN's real sandbox, only a WireMock container; the lookup has no
rate limit, no audit log and no dedicated scope (three reserved maintainer
decisions, below); neither SDK has run against a live vpay for this resource
(dated ⛔/⛔ rows in `docs/sdks/parity.md`); and **two issue #113 questions are
open that an agent must not default** — whether the checkout page ever asks a
payer for a location, and whether a merchant reads back a coarser coordinate
(`docs/plans/issue-113-notes/decision.md`).

## The object is nine keys, and a tenth only on an erased customer

`id`, `object`, `name`, `email`, `phone`, `address`, `metadata`, `created`,
`livemode` — plus `deleted: true`, **absent** rather than `false` on a live
one.

The count is held by `the_customer_object_is_the_documented_nine_keys` in
`vpay_api::model`. That test also names the internals that must never reach
the wire: `last_used_at`, `updated_at`, `seq`, `merchant_id`, `anonymized_at`.
Adding any of them puts a payer's data into every `customer.*` webhook body,
signed and stored in `events` for ever. `last_used_at` in particular is the
retention clock and is deliberately not a field.

**At least one of `name`, `email`, `phone` is always present** (`0034`'s
`at_least_one_identifier`); an address does not count. Clearing the last one
must be refused **above the statement**, with a `400` naming all three: the
CHECK raises `23514`, which `vpay_db::classify_write` deliberately leaves as
`DbError::Query` → `Category::Storage`, so the merchant would get a `503`
telling them to wait for a perfectly healthy database.

**Phone is the identity but is not a key.** Stored by default (no flag),
canonicalised to `237600000200` — twelve digits, no `+` — through
`vpay_api::v1::account_holders::canonical_msisdn`, the same function the lookup
uses. It does **not** deduplicate: two creates with one number make two
`cus_…`, because the unique index that would change that is the index that
would make cross-merchant identity possible.

## The address is the formal address AND the GPS point

One nested object, eight components, all nullable, all rendered, or `null`.
Stripe's six strings plus `latitude_microdeg` and `longitude_microdeg`, both
**integers** — a deliberate divergence from Stripe.

**They are integers because no float may exist here.** CrateStack's
`Value::from_plain_json` demotes any non-`i64` JSON number to `f64`, and
ADR-0007 denies float arithmetic workspace-wide. `4.061` is a `400` naming
`address` saying to send `4061000` — refused, never rounded. Resolution is
~0.11 m; the columns are `BIGINT`. Four rules that bite:

| Rule                                                                                   | Where                                                                                  |
| -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Both coordinates or neither — half a point is a `400` naming `address`                 | `validated_address`, backstopped by `address_coordinates_are_both_or_neither` (`0041`) |
| Ranges ±90 000 000 / ±180 000 000, symmetric                                           | `checked_microdeg`; `address_*_microdeg_range`                                         |
| `country` is two letters, **upper-cased on the way in**, shape-checked against no list | `address_country_is_iso_3166_1_alpha_2`                                                |
| An update **replaces the address whole**; it never merges components                   | one flag over all eight columns in `vpay_db::customers::update_in_tx`                  |

The replace-whole rule is the one a merchant gets wrong silently: sending
`address[line1]` and no coordinate **clears the point**.

**Every customer type's `Debug` is hand-written, and re-deriving one is
`E0119` — a `cargo check` failure, not a test to keep green.** That covers
`vpay_db::{CustomerRow, CustomerAddress, NewCustomer, CustomerPatch}`,
`vpay_api::model::{CustomerObject, AddressObject}` and `vpay-api`'s request
types (`CreateParams`, `UpdateParams`, `AddressParam`/`AddressParams`,
`ValidCreate`).

## Erasure: read this before touching any payer field

`DELETE /v1/customers/{id}` always succeeds or answers the uniform `404`;
there is **no `409`** any more. Nothing references the customer → the row is
**hard-deleted** and a later `GET` is a `404`. A payment intent, checkout
session or invoice references it → it is **anonymised in place**: nine text
identifier columns become the literal `[redacted]`, the two coordinate columns
become `NULL`, `anonymized_at` is stamped, `GET` answers `200` with
`deleted: true`. `metadata` is untouched — it is the merchant's data.

**The erasure must cover every copy vpay kept, and "every copy" means eight
tables, not one** — as of 2026-09-23. ~~six tables~~ _(Corrected 2026-09-23:
six was right until 2026-09-19, when vpay#211 added `payment_intents` (its
decline-text pair) and a second `webhook_deliveries` statement
(`response_excerpt`); step A added `manual_payments` on 2026-09-23.)_ Getting this wrong is the most damaging defect available in
this area. Details, the marker CHECK's `IS NOT DISTINCT FROM` subtlety, and
the closed idempotency race: [references/erasure.md](references/erasure.md).

**Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251) there is a
copy a _merchant_ writes: the out-of-band payment reference.**
`out_of_band[reference]` on `POST /v1/invoices/{id}/pay` — a cheque number
beside the drawer's name, a transfer reference carrying the payer's — lands in
`manual_payments.reference`, the `invoice.paid` body in `events.data`, and the
stored `pay` response. It is classified `payment_reference`, `subject: payer`,
`control: redact` — unlike `invoices.description`, which is the merchant's
note about their own bill and stays. The erasure redacts all of them in its own
transaction (`vpay_db::customers::redact_out_of_band_references`), and a
**new** reference on an erased payer's invoice is a `400` naming
`out_of_band[reference]` (the payment without one still records). That
refusal is race-free only because of a **lock order**: the paying transaction
takes `FOR SHARE` on the customer row as its first statement, every erasure
takes `FOR UPDATE` on it first, and nothing takes an invoice lock and then a
customer lock. Add a writer that locks an invoice before its customer and you
have a deadlock the suite's race case
(`an_erasure_racing_an_out_of_band_payment_neither_deadlocks_nor_leaves_the_reference`)
exists to catch.

## The twelve-month retention sweep

`vpay_worker::handlers::sweep_idle_customers` — its own `jobs.kind`
(`0034`), seeded at boot on the singleton dedupe key `sweep:customers`
(`vpay_worker::jobs::SWEEP_CUSTOMERS_DEDUPE_KEY`), hourly, 100 rows
a pass (`CUSTOMER_PAGE`). The horizon is `CUSTOMER_IDLE_AFTER`,
`Duration::days(365)`, and it **must not become configurable**: a retention
period is a promise to a payer, not an operator setting.

"Idle" is `customers.last_used_at`. "Used" means created, updated, or named
by a payment intent, a checkout session **or an invoice** — every one of those
paths calls `vpay_db::Customers::touch_last_used`. **A missing stamp does not
fail loudly: it makes a live customer look idle and erases it twelve months
later with nothing in any log.** If you add a path that names a customer, stamp
the clock. The stamp is monotonic (`WHERE last_used_at < now`) so a process
with a slow clock cannot rewind it.

**Paying an invoice does not stamp it** — hosted `pay` never did, and the
out-of-band `pay` added by vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged
in vaam-apps/vpay#251) does not either; only creating the invoice does, through
`resolve_for_attachment`. `docs/flows/invoices.md` lists it as an existing
gap. Listing by `customer=` stamps nothing either, correctly: a read is not a
use.

The sweep's guard is `anonymized_at IS NULL`. Without it an anonymised
customer stays idle for ever and the job emits an hourly `customer.deleted`
about a payer already erased. Known thin spot (2026-09-12, open): the sweep's
own case has no address fixture, so **no test drives a coordinate through
`erase_idle`**; the property holds only transitively, via the marker CHECK.

## Events

Written in the **same transaction** as the change they describe. All three
have been in `type_is_a_documented_event` since migration `0039`.

| Write                                             | Event              | Writer                                            |
| ------------------------------------------------- | ------------------ | ------------------------------------------------- |
| `POST /v1/customers`                              | `customer.created` | `vpay_api::v1::customers::create_with_event`      |
| `POST /v1/customers/{id}`, when something changes | `customer.updated` | `vpay_api::v1::customers::update_once`            |
| `DELETE /v1/customers/{id}`                       | `customer.deleted` | `delete_once` → `vpay_db::customers::erase_in_tx` |
| the retention sweep                               | `customer.deleted` | `vpay_db::customers::erase_idle`                  |

Nothing is emitted for a bodiless no-op update, a second `DELETE` on an
already-erased customer, a `404`, or `touch_last_used`.

`customer.deleted` carries the **redacted** object — ids, `created`,
`livemode`, the merchant's `metadata`, `deleted: true`, no identifier. That
reversed what the flow doc said until 2026-09-10.

The update takes `SELECT … FOR UPDATE` inside the emitting transaction, and
that is load-bearing, not defensive: `metadata` is merged key-wise in Rust, so
without the lock two concurrent updates lose a key and `customer.updated`
describes a state the database does not hold. Removing `FOR UPDATE` fails two
named tests.

## The routes

| Method   | Path                 | Notes                                                                                 |
| -------- | -------------------- | ------------------------------------------------------------------------------------- |
| `POST`   | `/v1/customers`      | `name`, `email`, `phone`, `address[…]`, `metadata[…]`                                 |
| `GET`    | `/v1/customers/{id}` | answers an erased customer too, with `deleted: true`                                  |
| `POST`   | `/v1/customers/{id}` | the update. **There is no `PATCH`** — unlike `/v1/invoices/{id}`                      |
| `GET`    | `/v1/customers`      | `limit`, `starting_after`, `ending_before`; erased rows **are** listed; **no filter** |
| `DELETE` | `/v1/customers/{id}` | `{"id":…,"object":"customer","deleted":true}`, or a `404`                             |

**Listing a customer's payments is a filter on the _other_ lists, not a
route here.** Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged
in vaam-apps/vpay#251), `customer=cus_…` filters `GET /v1/payment_intents`,
`GET /v1/checkout/sessions` and `GET /v1/refunds` (and `GET /v1/invoices`
always did), inside the same `WHERE` as `merchant_id`, so a foreign or unknown
`cus_…` is an empty page and never a `404`. An erased customer keeps its
`cus_…` and the filter still finds everything that names it. **`GET
/v1/customers` stays unfiltered** (ADR-0024 D4): a filter _by_ an id the
merchant holds is not a search _for_ a payer. Rules and the one consequence
(a session-named customer on a customer-less intent is invisible to the intent
and refund filters, ADR-0024 open question 3): `vpay-merchant-api`.

Update has three states per field: absent leaves it, a value sets it, the
empty string (`name=`) clears it. `metadata` has **two** — merged key-wise, a
key sent empty is removed, and the fifty-key bound is on the **result**, not
the request. Every query is merchant-scoped in SQL; another merchant's `cus_…`
is byte-identically the same `404` as an id that never existed.

**`DELETE` requires an `Idempotency-Key`** — but so do `DELETE
/v1/invoices/{id}` and `DELETE /v1/invoice_items/{id}`; all three go through
`PostRequest::read` → `IdempotencyKey::from_request_parts`, which refuses a
missing header. (The refusal's own sentence says "required on every POST to
/v1", which is stale prose in `vpay_api::idempotency::missing`.) What **is**
unique to this resource is that its three write routes pass
`ResponseSubject::Customer { id }` to `vpay_db::Idempotency::store` where every
other route passes `Verbatim` — see [references/erasure.md](references/erasure.md).
_(Qualified 2026-09-23: since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged
in vaam-apps/vpay#251) a third variant, `ResponseSubject::OutOfBandInvoice { customer_id }`,
covers the out-of-band `pay` and a `POST /v1/invoices/{id}` on an invoice paid
out of band — the same issue-#111 race on a body carrying a reference. The
customer routes are no longer the only non-`Verbatim` callers.)_

## `GET /v1/account_holders`

A name lookup with a **three-way** answer that nothing may collapse, three
privacy rules that apply only here, and three controls issue #47 asked for that
are **reserved maintainer decisions and are not built**:
[references/account-holder-lookup.md](references/account-holder-lookup.md).
