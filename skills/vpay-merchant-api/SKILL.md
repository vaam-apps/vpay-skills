---
name: vpay-merchant-api
description: The vpay HTTP surface — the /v1 merchant API, its route tables, OAuth2 private_key_jwt auth (there are no API keys), the mandatory Idempotency-Key, the Stripe-shaped error envelope, and the routes that are declared in the wire contract but mounted nowhere. Load this before adding, changing, removing or calling any HTTP route, before touching authentication or scopes, and before writing anything that returns an error to a merchant.
---

# The vpay HTTP surface

> **Verified against vpay `7a79684e` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

Four nests, one router function: `vpay_api::router` in
`backends/crates/vpay-api/src/lib.rs`.

| Nest          | Who calls it        | Authenticated by                        |
| ------------- | ------------------- | --------------------------------------- |
| `/v1`         | a merchant's server | bearer token (`require_merchant_token`) |
| `/v1/oauth`   | a merchant's server | the request body _is_ the credential    |
| `/v1/browser` | a payer's browser   | publishable key + a `client_secret`     |
| `/provider`   | a payment rail      | **nothing at all** — by design          |
| `/dash/v1`    | the staff dashboard | staff session → authorization code      |

`/livez` and `/metrics` are on a **different port** (`vpay_api::observability`)
and deliberately not on this router. `/healthz` is, and is the one response in
the crate that is not an error envelope — it answers bare text.

`/dash/v1` belongs to the `vpay-dashboard` skill; the table here lists it for
completeness.

## Routes are rows in a table, not calls to `.route()`

`V1_ROUTES` (`vpay_api::v1`), `BROWSER_ROUTES` (`vpay_api::browser`),
`DASH_ROUTES` (`vpay_api::dash`) and `STAFF_ROUTES` (`vpay_api::staff`) are
`const` slices, and each router is **folded from its table**. The table is the
router's source, not documentation of it — so **adding a route means adding a
row** with its `path`, its `methods` and its `mount` closure.

`methods` is carried beside the handler because axum 0.8 cannot enumerate a
built `Router`. Two tests walk the tables rather than a hand-kept copy:
`every_registered_v1_path_answers_401_without_a_token`
(`backends/tests/integration/tests/payment_intents.rs`) asserts every
`V1_ROUTES` entry sits behind the token boundary, and
`every_browser_route_is_reachable_without_a_merchant_token`
(`browser_checkout.rs`) asserts the opposite for `BROWSER_ROUTES` and pins its
length. Putting a payer route in `V1_ROUTES` breaks the first; that is the
point.

Full table for all four surfaces, plus the middleware order and why it is
load-bearing: [references/routes.md](references/routes.md).

## Authentication: OAuth2, and there are no API keys

`/v1` takes `Authorization: Bearer <access_token>`. A merchant gets one by
minting an RFC 7523 `client_assertion` signed with their own private key and
POSTing it to `/v1/oauth/token` (`client_credentials` + `private_key_jwt`,
ADR-0010). The issuer is derived in one place, `vpay_api::op::issuer_for`, as
`{deployment.public_base_url}/v1/oauth`.

**Do not reach for an API key.** `backends/migrations/0008_create-merchant-api-keys.sql`
was dropped by `0009_drop-merchant-api-keys.sql`: API keys are a deliberately
deleted design, not a missing feature. Publishable keys (`pk_test_…`) exist,
authorise nothing, and are not secrets — they name a tenant on `/v1/browser`.

`require_merchant_token` is mounted **in front of the whole nest, never
per-handler**. That is what makes it impossible to add a `/v1` handler without
it; the failure mode of a per-handler check is a handler that simply has none.
The cost is that the rule only sees the method, which is why scopes are
per-method.

It puts two things on the request: the validated `ResourceClaims` and a
`MerchantScope`. **`MerchantScope` has a private field and no public
constructor** — a handler cannot invent a tenant to query by. There is no
`merchants` table and so no foreign key that would catch an unscoped query, so
that type is the whole tenancy boundary. Repository methods on this path take a
`merchant_id` and have no unscoped variant.

## Scopes, and why refusals are 403

Two strings, defined once in `vpay_api::v1`: `SCOPE_PAYMENTS_WRITE`
(`payments:write`) and `SCOPE_PAYMENTS_READ` (`payments:read`).
`required_scopes(method)` is **fail-closed** — `GET`/`HEAD` accept either;
everything else, including verbs no route answers, requires write.

| Situation                                       | Answer                                                   |
| ----------------------------------------------- | -------------------------------------------------------- |
| no bearer token                                 | `401` `missing_bearer_token`                             |
| valid token, no scope for this method           | `403` `forbidden`                                        |
| valid token, `client_id` in no registration     | `403` — the token is genuine, the _registration_ is gone |
| valid token, object belongs to another merchant | `404`, byte-identical to a nonexistent id                |
| authenticated, path has no route                | `404` `unknown_route`                                    |

The rule to internalise: a statement about the **credential** is 403 (the
caller can inspect their own token); a statement about an **object** is a
uniform 404, because anything else is an enumeration oracle.

A token request naming no `scope` is granted the client's registered `scopes:`
from YAML (RFC 6749 §3.3's "locally defined default", applied in
`op::token::token_handler`). Both SDKs omit `scope`, so every SDK call takes
that path.

## Errors: derive, never decide

A handler returns `Result<_, ApiError>` and uses `?`. It never picks a status
and never writes a merchant-facing sentence: the status is
`category().http_status()` and the `type` is `category().stripe_type()`
(ADR-0011). The one envelope renderer is `pub(crate)`, so a handler in another
crate cannot build one. `stripe-should-retry` is derived from
`Classify::retry()`, **not** from the status.

Mapping, the four deliberate code overrides, and the two `/v1` answers that are
not `ApiError`: [references/errors.md](references/errors.md).

## Idempotency is mandatory — a divergence from Stripe

Every `POST` under `/v1` **requires** an `Idempotency-Key`; a missing one is a
`400` naming `idempotency_key`, not a generated fallback. A key the server
invented would differ on every attempt, so a timed-out create would take the
payment twice. `DELETE /v1/customers/{id}` needs one too.

Do not hand-roll the claim/finish dance — reuse `PostRequest` from
`vpay_api::v1::payment_intents` (`pub(crate)`), which every `/v1` write already
goes through. [references/idempotency.md](references/idempotency.md).

**On `POST /v1/refunds` the claim is the _only_ thing standing between a
merchant's retry and a second payout.** Unlike a charge, which
`one_charge_per_intent` makes unrepeatable, nothing in the schema forbids two
refunds of one intent — two partial refunds are a legitimate thing a merchant
does. Do not weaken or bypass the claim on that route.

## Body limits

64 KiB on `/v1` and `/dash/v1` (`V1_BODY_LIMIT_BYTES`), 16 KiB on `/provider`
(`CALLBACK_BODY_LIMIT_BYTES`). Both are mounted **outside** the auth check, so
an anonymous caller cannot make the process buffer before the 401, and both are
`RequestBodyLimitLayer` rather than axum's `DefaultBodyLimit` so the bound
applies whether or not an extractor reads the body.

## Declared in the contract, mounted nowhere

There is **no OpenAPI or Swagger file in this repository.** The wire contract
is `docs/flows/merchant-auth/resource-contract.md`, `docs/api/README.md` and
the two SDKs.

| Declared               | Where                | Reality (2026-09-16)                                                                                                                                                                                       |
| ---------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `GET /v1/balance`      | resource-contract; both SDKs | **Unmounted.** A ledger read path exists since 2026-09-16 (`vpay_db::Ledger::merchant_payable_balance`) and **nothing routes it**; the nest's fallback answers the honest `404` rather than a body vpay would have to invent. |
| `GET /v1/events?type=` | `docs/api/README.md` | Route served, **filter silently ignored** — `ListParams` in `vpay_api::v1::events` has no `type` field, so a filtered call gets an unfiltered page rather than a `400`.                                    |

~~`POST /v1/refunds` is unmounted, and mounting it would mount a route that
can only ever answer `501`.~~ **Corrected 2026-09-16 (RFC-0003 § 2): the
refund resource is mounted for five methods across three paths** —
`POST`/`GET /v1/refunds`, `GET`/`POST /v1/refunds/{id}`,
`POST /v1/refunds/{id}/cancel`, pinned by
`the_refund_resource_is_mounted_for_exactly_five_methods`. The create writes a
real row, reserves the amount and emits `charge.refunded`; it does **not**
answer `501`, and it does **not** mean money came back — see
[references/objects.md](references/objects.md) § Refund and the `vpay-payments`
skill before writing anything that implies it did.

**The operational consequence, because it will cost you a day otherwise:**
there is **no refund poll ladder** (RFC-0003 open question 8), so a refund the
rail accepts stays `pending` indefinitely. Only a *refusal* moves a refund, to
`failed`; `succeeded` is written by `Settlement::apply_refund_succeeded` alone
and **nothing a merchant can cause reaches it.** An integration that creates a refund and
waits for `succeeded` waits forever. `charge.refund.updated` does **not**
rescue it: that event is emitted on a failure, on a metadata `update` and on a
`cancel` — three places, all in `vpay_api::v1::refunds` — and **never on a
settlement**, because nothing settles. And no rail in this repository has
ever returned money to anyone: on Orange the create fails outright
(`NotImplemented("orange_money::refund")`), and on MTN it reaches the
Disbursements `transfer`, which has never been called outside WireMock.

Both SDKs can call `GET /v1/balance`, and it gets the honest envelope.

## Where this repository's docs are wrong

Code wins in all four. Fix the doc in the same commit if you touch the area.

- `docs/api/README.md` § "Served today" says **"thirty-one methods across
  twenty paths"**, last re-counted on 2026-09-07. `V1_ROUTES` has **37 methods
  across 23 paths** as of 2026-09-16 — it is missing
  `/v1/invoices/{id}/mark_uncollectible` and the four refund methods RFC-0003
  § 2 added, and its own increments do not add up either. (This page said
  "33 across 21" until 2026-09-16; the refund routes are the difference.)
- `docs/flows/merchant-auth/resource-contract.md` says "**All four** refund
  routes are served since 2026-09-16". That counts the four RFC-0003 § 2
  _added_ — create, update, list, cancel — beside the read that shipped on
  2026-09-05, and reads as a total. The total is **five methods across three
  paths**; `the_refund_resource_is_mounted_for_exactly_five_methods` is the
  number to trust.
- `vpay_api::browser`'s module header opens "the **two** routes a payer's
  browser may call", and `browser_checkout.rs`'s header repeats it. There are
  **five**; the assertion further down that same test file says 5 and is right.
- `docs/flows/merchant-auth/resource-contract.md`'s resource table omits
  `/v1/checkout/sessions`, `/v1/invoices` and `/v1/invoice_items` entirely.
  All three are served.
- The same table calls `Idempotency-Key` "caller-supplied, else a UUIDv4
  generated per call". The **server requires it**; the UUID is the SDKs'
  client-side behaviour, not a server fallback.

## Before you finish

A `/v1` route change is not done until the SDKs agree: `cargo xtask
verify-sdk-parity` refuses an SDK capability with no row in
`docs/sdks/parity.md`, and a row naming no capability. Then
`docs/status/backend.md`, then the flow page.

Wire objects, fields, id prefixes and validation bounds:
[references/objects.md](references/objects.md).
