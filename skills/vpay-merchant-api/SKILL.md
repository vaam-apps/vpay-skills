---
name: vpay-merchant-api
description: The vpay HTTP surface — the /v1 merchant API, its route tables, OAuth2 private_key_jwt auth (there are no API keys), the mandatory Idempotency-Key, the Stripe-shaped error envelope, and the routes that are declared in the wire contract but mounted nowhere. Load this before adding, changing, removing or calling any HTTP route, before touching authentication or scopes, and before writing anything that returns an error to a merchant.
---

# The vpay HTTP surface

> **Verified against vpay `a33aac61` (2026-09-29).** Version-sensitive claims below
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
and deliberately not on this router. `/healthz` is, and answers bare text
rather than an error envelope. ~~It is the one response in the crate that is
not an error envelope.~~ _(Corrected 2026-09-23: there are others —
`/v1/oauth/token`'s RFC 6749 body, and axum's `405` and tower-http's `413`;
see [references/errors.md](references/errors.md).)_

`/dash/v1` belongs to the `vpay-dashboard` skill; the table here lists it for
completeness.

**Which nests a process mounts depends on `deployment.surfaces`**, since
2026-09-16 (ADR-0022, vpay#183; the ADR's status is still _Proposed_). This
page did not say so until 2026-09-23. `business` mounts `/v1`, `/v1/browser`,
`/provider` and `POST /v1/oauth/token`. `management` mounts `/dash/v1` and the
staff routes. Either surface mounts `/v1/oauth`'s discovery and JWKS, and
`/healthz` is always mounted. An absent key means both surfaces
(`EnabledSurfaces::ALL`), and an empty list is a boot error. A nest the
process does not mount gets the outer honest `404`, not a refusal. So a
`404 unknown_route` from a pod can mean "wrong tier" as well as "no such
route".

## Routes are rows in a table, not calls to `.route()`

`V1_ROUTES` (`vpay_api::v1`), `BROWSER_ROUTES` (`vpay_api::browser`) and
`DASH_ROUTES` (`vpay_api::dash`) are `const` slices, and each of those routers
is **folded from its table**. The table is the router's source, not
documentation of it — so **adding a route means adding a row** with its
`path`, its `methods` and its `mount` closure.

~~…and `STAFF_ROUTES` (`vpay_api::staff`) are `const` slices, and each router
is folded from its table.~~ **Corrected 2026-09-23:** `STAFF_ROUTES` is a
`(path, methods)` list with **no** mount closure, kept **beside**
`staff::routes()`, which builds its eight `.route(…)` calls inline. It was
the same at `d3a8810b`. Its own test checks only that it has eight entries.
So a staff route added to `routes()` and not to the table mounts without
complaint and escapes the boundary walk that reads the table. Add it in both
places.

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

### What can go wrong signing in, and two limitations nobody has decided

_Added 2026-09-29, from `docs/flows/merchant-auth/verification-and-limits.md`
(the page this skill did not cover before); nothing here was re-run._

- **Failures and where they surface**: a wrong private key, an unregistered
  `kid` or a mistyped `client_id` is `401`/`400` `invalid_client` from the token
  endpoint (the SDK returns an authentication error and does not retry); a
  merchant in `disabled_clients` is refused at the token endpoint or on a `/v1`
  route with `401` (one re-auth attempt, then the error); an assertion `exp`
  more than 300 s out is refused by the OP (the SDKs refuse to be configured
  that way); **clock skew beyond 60 s** is `invalid_client`, and the message
  names the failed check only as far as the OP does; a token endpoint at a
  different path than the SDK default is a `404` `unknown_route` envelope, fixed
  by the SDK's `issuer`/`token_endpoint` setting; and a rolling deploy can answer
  `invalid_client` from one replica and success from another (ADR-0010's
  window), left to the merchant's retry policy.
- **The `jti` replay namespace is global, not per merchant.**
  `oauth_client_assertion_jtis.jti` is the primary key on its own, and
  `authkestra_op`'s `record_jti` is given no `client_id` to scope by. A merchant
  whose library used a counter or a timestamp as `jti` could collide with, and
  deliberately pre-spend, another merchant's values. **Onboarding requirement
  until that changes: `jti` MUST be a UUID v4** (both vpay SDKs do this).
  Scoping the key to `(client_id, jti)` needs a migration and an upstream seam
  or a per-client store; that decision is the maintainer's and is open.
  (Checked at `a33aac61`: the primary key is still `jti` alone.)
- **No rate limit in front of `/v1/oauth/token` or `/v1`.** A known `client_id`
  (they are public) costs one `disabled_clients` `SELECT` per token request
  before any signature check, and ADR-0009 leaves `/token` rate limiting to the
  ingress. Confirm the ingress does it before relying on that; nothing in the
  repository enforces it. (The _dashboard's_ staff sign-in does have a limiter —
  that is `vpay-dashboard`.)
- **Verifying a webhook** (the same page): `Vpay-Signature: t=<unix>,v1=<hex>`,
  several `v1=` allowed during a secret rotation, HMAC-SHA256 over the literal
  bytes `"<t>.<raw body>"`, constant-time compare, a 300 s default tolerance,
  and parse the body only afterwards. The verifier does **not** dedupe by
  `event.id`. `vpay-webhooks` is the owner of the scheme.

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
payment twice. Every `DELETE` needs one too — on `/v1/customers/{id}`,
`/v1/invoices/{id}` and `/v1/invoice_items/{id}` (this named only the first
until 2026-09-23).

Do not hand-roll the claim/finish dance — reuse `PostRequest` from
`vpay_api::v1::payment_intents` (`pub(crate)`), which every `/v1` write already
goes through. Pick its `ResponseSubject` deliberately: a body that can name
the payer must not be stored `Verbatim`, or an erasure leaves a copy behind
for 24 hours (issue #111). [references/idempotency.md](references/idempotency.md).

**On `POST /v1/refunds` the claim is the _only_ thing standing between a
merchant's retry and a second payout.** Unlike a charge, which
`one_charge_per_intent` makes unrepeatable, nothing in the schema forbids two
refunds of one intent — two partial refunds are a legitimate thing a merchant
does. Do not weaken or bypass the claim on that route.

## `customer` on the lists — a filter, never a lookup

Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251; ADR-0024
D1–D4, D12), `GET /v1/payment_intents`, `GET /v1/checkout/sessions` and
`GET /v1/refunds` take `customer=cus_…`. `GET /v1/invoices` always did. On an
older server the parameter is **silently ignored and the whole list comes
back** — no vpay release up to and including `v0.5.0` has it (as of
2026-09-23, and still the latest tag on 2026-09-29: `a33aac61` is `v0.5.0-12`), so a client must not treat an unfiltered page as "this customer's
rows".

- **It is in the same `WHERE` as `merchant_id`, and that is the whole
  security property.** Another merchant's real `cus_…`, an id that never
  existed, and one of yours with no rows are the same `200` empty page. A
  `404` or any different answer for a foreign id would be an existence oracle
  over another tenant's payers. Checked for **shape only**.
- **A malformed value is one `400` naming `customer`**, from one helper,
  `vpay_api::v1::customers::filter_param`, which `GET /v1/invoices` now calls
  too. Do not write a second validator.
- **Cursors page inside the filtered set**, as `GET /v1/invoices?customer=`
  does: a cursor naming a row outside the filter pages from that row's
  position in the merchant's whole list.
- **Each list compares a different column**: intents their own
  `customer_id`; sessions their **own** `customer_id` (D12); refunds, which
  have no customer column, their intent's, inside the join the tenant
  predicate already makes.
- ~~**So one payment can be in one list and not the others.** A session
  created with `customer=X` on an intent with no customer stores `X` on the
  session only: its payment is listed by `GET /v1/checkout/sessions?customer=X`
  and **not** by the intent or refund filters. Whether a session's customer
  should be written onto such an intent is ADR-0024's **open question 3** —
  do not "fix" it by changing checkout-session creation.~~ **Corrected
  2026-09-29 (vpay `a33aac61`):** this was true until
  [ADR-0025](https://github.com/vaam-apps/vpay/blob/a33aac61029c5e2d81e77217cd9ea23d43f946ac/docs/adr/0025-session-customer-onto-intent.md)
  (vaam-apps/vpay#253, merged 2026-09-23), which answered question 3. The
  maintainer chose to write the customer through, so the sentence "do not fix
  it" is now backwards: **the fix is built, and it is not yours to undo.**
  - **Since 2026-09-23 a session created with `customer=X` on a customer-less
    intent writes `X` onto the intent**, in the same transaction as the session
    insert (`vpay_db::checkout_sessions::claim_intent_customer`, a
    compare-and-swap on `customer_id IS NULL`). Its payment is then listed by
    all three filters. Do not remove the write to make a test pass, and do not
    add a backfill (below).
  - **A later session on that intent naming another customer is the `400`
    naming `customer`** — the refusal `prepare_create` always gave a session
    that contradicted its intent. Of two concurrent creates naming two
    customers for one customer-less intent, exactly one wins. The loser gets
    that same `400` (via `DbError::IntentCustomerConflict`, one shared
    `customer_contradiction` function, byte for byte) or, if it read the intent
    after the winner committed, the one-open-session `409` that check has
    always answered first.
  - **A create naming a customer erased in the meantime is a `409`**
    (`DbError::CustomerErased`, the same bytes as the pre-check's — ADR-0027).
    The create takes `FOR SHARE` on the customer **before** it touches the
    intent; the order was the reverse until ADR-0027, and the two orders
    deadlocked with `40P01` against an erasure (a `503`). Keep the customer
    first.
  - **Idempotency, because it differs by path.** The race loser's `400` comes
    from `create`, after the key is claimed, so it is **stored** under the
    key; the pre-check's identical `400` **releases** the key. Neither retry
    can succeed, because an intent's customer is never rewritten once set.
  - **Rows from before 2026-09-23 were not backfilled and are still
    inconsistent.** A historical session may name a customer its intent does
    not (an intent can even carry sessions naming two payers), so its payment
    is listed by `GET /v1/checkout/sessions?customer=X` and **not** by the
    intent or refund filters. A backfill would have to guess between payers;
    ADR-0025 § "No backfill" refused to. A merchant who needs a historical
    payment by customer lists the sessions and follows each `payment_intent`.
    (Erasure does reach those payments since ADR-0027 — see `vpay-customers`.)
- **`GET /v1/customers` stays unfiltered**, deliberately (D4): a filter on a
  payer identifier turns a list into a lookup. `customer` filters _by_ an id
  the merchant already holds.
- **`GET /dash/v1/payment_intents` refuses `customer` with a `400`**; before
  step A it silently ignored it and answered every customer's intents. See
  `vpay-dashboard`.

No route was added: 23 paths, 37 methods, as on 2026-09-16. The out-of-band
`pay` is new parameters on an existing route (`paid_out_of_band=true`,
`out_of_band[…]`) — see the `vpay-invoices` skill.

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

| Declared               | Where                        | Reality (2026-09-23)                                                                                                                                                                                                          |
| ---------------------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /v1/balance`      | resource-contract; both SDKs | **Unmounted.** A ledger read path exists since 2026-09-16 (`vpay_db::Ledger::merchant_payable_balance`) and **nothing routes it**; the nest's fallback answers the honest `404` rather than a body vpay would have to invent. |
| `GET /v1/events?type=` | `docs/api/README.md`         | Route served, **filter silently ignored** — `ListParams` in `vpay_api::v1::events` has no `type` field, so a filtered call gets an unfiltered page rather than a `400`.                                                       |

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
rail accepts stays `pending` indefinitely. Only a _refusal_ moves a refund, to
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

Code wins in all five below (this said "all four" over a list of five until
2026-09-23; three of the five carry a strike-through as of 2026-09-29, because
vpay fixed them). Fix the doc in the same commit if you touch the area.

- ~~`docs/api/README.md` § "Served today" says **"thirty-one methods across
  twenty paths"**~~ and ~~`docs/flows/merchant-auth/resource-contract.md` says
  "**All four** refund routes are served"~~. **Corrected 2026-09-23:** both
  were fixed in vpay on 2026-09-16 (vpay#182, `d3a8810b`). This page went on
  listing them as wrong through its `0799a8d2` stamp. The README now says
  **thirty-seven methods across twenty-three paths**, re-counted from
  `V1_ROUTES`, and that still matches on 2026-09-23. The resource contract
  now says **five methods across three paths**.
- ~~`docs/flows/merchant-auth/resource-contract.md`'s `DELETE /v1/customers/{id}`
  row calls it "the only `DELETE` on this API".~~ **Corrected 2026-09-29:**
  vpay fixed that row on 2026-09-24 (vaam-apps/vpay#255), striking the claim
  and writing "one of three". `V1_ROUTES` has mounted `DELETE` on
  `/v1/invoices/{id}` and `/v1/invoice_items/{id}` since 2026-09-07, and all
  three carry an `Idempotency-Key` on a verb with no body.
- `vpay_api::browser`'s module header opens "the **two** routes a payer's
  browser may call", and `browser_checkout.rs`'s header repeats it. There are
  **five**; the assertion further down that same test file says 5 and is right.
- `docs/flows/merchant-auth/resource-contract.md`'s resource table omits
  `/v1/checkout/sessions`, `/v1/invoices` and `/v1/invoice_items`. All three
  are served. The Checkout Session omission is deliberate: the page says so
  and points at `hosted-checkout.md`. ~~The invoice omission is not mentioned
  at all.~~ **Corrected 2026-09-29:** since vaam-apps/vpay#255 (2026-09-24)
  the page names it — "Nor does it list the invoice routes", fifteen of
  `V1_ROUTES`' 37 method-and-path pairs — and points at `invoices.md` §
  "The surface". The table still omits them, now on purpose.
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
