# Every mounted route, and the middleware around them

Assembled in `vpay_api::router` (`backends/crates/vpay-api/src/lib.rs`).
Counts and contents verified against the code on **2026-09-16**.

Nest order in the source is `/v1/oauth`, `/v1/browser`, `/v1`, `/provider`,
then the two conditional `/dash/v1` mounts. axum's path table is
order-independent for distinct prefixes — `/v1/browser` and `/v1/browser/{*rest}`
match more specifically than `/v1/{*rest}` — so the ordering is a reading
convenience, not a correctness device. What _is_ a correctness device is that
**every nest carries its own `.fallback(crate::not_found)`**: without one, axum
flattens the nest into the outer path table and an unmatched `/v1/oauth/…`
matches the _authenticated_ `/v1` wildcard, answering `401` to a caller whose
whole reason for being there is that it has no token yet.
`the_oauth_nest_answers_its_own_404` fails with `left: 401, right: 404` if the
fallback is deleted.

## `/v1` — merchant API

Source: `V1_ROUTES` in `backends/crates/vpay-api/src/v1/mod.rs`.
**21 paths, 33 methods.**

| Method(s)                | Path                                   | Handler                                             |
| ------------------------ | -------------------------------------- | --------------------------------------------------- |
| POST, GET                | `/v1/payment_intents`                  | `payment_intents::{create, list}`                   |
| GET                      | `/v1/payment_intents/{id}`             | `payment_intents::retrieve`                         |
| POST                     | `/v1/payment_intents/{id}/confirm`     | `payment_intents::confirm`                          |
| POST                     | `/v1/payment_intents/{id}/cancel`      | `payment_intents::cancel`                           |
| GET                      | `/v1/events`                           | `events::list`                                      |
| GET                      | `/v1/events/{id}`                      | `events::retrieve`                                  |
| GET                      | `/v1/account_holders`                  | `account_holders::retrieve`                         |
| POST, GET                | `/v1/checkout/sessions`                | `checkout_sessions::{create, list}`                 |
| GET                      | `/v1/checkout/sessions/{id}`           | `checkout_sessions::retrieve`                       |
| POST                     | `/v1/checkout/sessions/{id}/expire`    | `checkout_sessions::expire`                         |
| GET                      | `/v1/refunds/{id}`                     | `refunds::retrieve`                                 |
| POST, GET                | `/v1/customers`                        | `customers::{create, list}`                         |
| GET, POST, DELETE        | `/v1/customers/{id}`                   | `customers::{retrieve, update, delete}`             |
| POST, GET                | `/v1/invoices`                         | `invoices::{create, list}`                          |
| GET, POST, PATCH, DELETE | `/v1/invoices/{id}`                    | `invoices::{retrieve, update, update, delete}`      |
| POST                     | `/v1/invoices/{id}/finalize`           | `invoices::finalize`                                |
| POST                     | `/v1/invoices/{id}/void`               | `invoices::void`                                    |
| POST                     | `/v1/invoices/{id}/mark_uncollectible` | `invoices::mark_uncollectible`                      |
| POST                     | `/v1/invoices/{id}/pay`                | `invoices::pay`                                     |
| POST                     | `/v1/invoice_items`                    | `invoice_items::create`                             |
| GET, POST, PATCH, DELETE | `/v1/invoice_items/{id}`               | `invoice_items::{retrieve, update, update, delete}` |

Things about this table that are decisions rather than accidents:

- **`POST` is Stripe's update.** Stripe's API has no `PATCH`, so a merchant's
  existing client sends `POST`. `PATCH` is mounted **beside it on the same
  handler function** on the two `{id}` paths that have it — they cannot answer
  differently because they are one function.
- **`/checkout/sessions`, not `/checkout_sessions`.** Stripe's own spelling,
  and the object's `object` field is `checkout.session` for the same reason.
- **`expire` is a `POST`, not a `DELETE`** — expiring is a transition that
  returns the object, exactly as `cancel` is for an intent. Nothing is removed.
  It also **emits no event**: you asked for it. The hourly sweep's expiry does
  emit `checkout.session.expired`.
- **`/account_holders` is plural with no `/{id}`.** A mobile-money number is
  not an id vpay mints and nothing is stored to address later; the collection
  is what is searched, via query parameters.
- **There is no collection `GET` on `/v1/invoice_items`.** An invoice's lines
  are read from the invoice (`invoice.lines`, expanded on every render).
- **`GET /v1/refunds/{id}` with no `POST /v1/refunds`** is deliberate and
  unusual — issue #45 decided a refund needs an authoritative read even though
  nothing can create one.

## `/v1/oauth` — the merchant OP

Built inline in `router`, not from a table. Unauthenticated by necessity: the
credential _is_ the request body (RFC 7523 §2.2), so requiring a bearer token
here would be circular.

| Method | Path                                         |
| ------ | -------------------------------------------- |
| POST   | `/v1/oauth/token`                            |
| GET    | `/v1/oauth/.well-known/openid-configuration` |
| GET    | `/v1/oauth/jwks.json`                        |

`POST /v1/oauth/token` renders **RFC 6749 §5.2's error body**
(`{"error":…,"error_description":…}`), _not_ the Stripe envelope — every OAuth
client in existence parses that shape. It is the one place in the crate where
an error is not an `ApiError`, other than `/healthz`.

`/v1/oauth` is **not configurable**: a deployment that moved it would silently
break every merchant who took the SDK default. There is **no rate limit** on
`/token`; ADR-0009 leaves it to ingress, and nothing in this repository
verifies that ingress actually limits it.

## `/v1/browser` — the payer's browser

Source: `BROWSER_ROUTES` in `backends/crates/vpay-api/src/browser/mod.rs`.
**Five routes, four of them `GET`.** Pinned at exactly five by
`every_browser_route_is_reachable_without_a_merchant_token`.

| Method | Path                                        | Credential presented                      |
| ------ | ------------------------------------------- | ----------------------------------------- |
| GET    | `/v1/browser/payment_intents/{id}`          | publishable key + intent `client_secret`  |
| POST   | `/v1/browser/payment_intents/{id}/confirm`  | publishable key + intent `client_secret`  |
| GET    | `/v1/browser/checkout/sessions/{id}`        | publishable key + session `client_secret` |
| GET    | `/v1/browser/checkout/sessions/{id}/return` | publishable key + session `return_token`  |
| GET    | `/v1/browser/checkout/origins`              | publishable key alone                     |

The credential ladder is the design (`browser::checkout_sessions`' module
docs). The session secret rides in a **URL fragment**, which never leaves the
browser, so it may expand the intent _with_ its own `client_secret`. The
`return_token` rides in a **query string** — it must, because a fragment does
not survive a rail's redirect — so it gets strictly less: enough to render an
outcome, not enough to confirm. The two are **sibling path patterns read by two
different handlers**, which is what makes "a `return_token` cannot reach the
intent's `client_secret`" a routing fact rather than a review promise.

**Every refusal is the same 404.** `browser::authenticate` has four ways to
fail and answers all of them byte-identically. That is the entire
confidentiality property of an unauthenticated surface: a distinct answer for
"unknown publishable key" enumerates a deployment's merchants, one for "wrong
secret" separates existence from authorisation. `/checkout/origins` answers
`200 {"origins": []}` for an unknown key for the same reason.

**No rate limiting here, deliberately.** What stands between a guesser and an
intent is 160 bits of `client_secret`, the uniform 404, and one-charge-per-intent
— not a counter. A per-process limiter across N replicas is a limit of N times
what it claims.

**This is the only nest with CORS.** `CorsLayer` with `allow_origin(Any)`,
credentials off, `GET`/`POST`/`OPTIONS`, and `allow_headers([CONTENT_TYPE])`
only — notably **not** `Idempotency-Key` or `Authorization`, because allowing
either invites a browser to send one.
`cors_is_mounted_on_the_browser_nest_and_on_no_other` fails if a second nest
gets one. The merchant `/v1` nest gets none: nothing legitimate calls it from a
browser, and a permissive header there would invite a merchant to put a bearer
token in a page.

## `/provider` — rail callbacks

Source: `vpay_api::provider_callback`. One route.

| Method | Path                        |
| ------ | --------------------------- |
| POST   | `/provider/{code}/callback` |

The path is not free: `vpay_config::ProviderHost::effective_callback_url`
derives `{public_base_url}/provider/{code}/callback`, and that is what both
adapters have been sending in `X-Callback-Url` / `notif_url` since Step 3.
`the_mounted_path_is_the_one_the_rails_are_told_to_call` is the join between
the two crates.

**Nothing authenticates this route.** Neither MTN nor Orange signs a callback
or sends a shared secret. **There is no signature verification to find, and
adding a naive one would be inventing a scheme the rails do not implement.**
The handler is written around that: it never writes charge or intent state.
The only thing it can do is bring an already-queued `poll_charge` job forward,
and not even that if the job is already due within `PULL_FORWARD_FLOOR` (10 s,
the poll ladder's fastest rung). `CallbackRef::ref_extra` — Orange's
`notif_token` / `pay_token` — is **discarded**, because writing rail key
material from an unauthenticated request is the one thing this route must not
do.

| Case                                       | Answer                                              |
| ------------------------------------------ | --------------------------------------------------- |
| `code` names no adapter this process links | `404`, byte-identical to the router's fallback      |
| body is not a notification this rail sends | `400` (plus a `warn` carrying the adapter's reason) |
| reference names no charge here             | `202`                                               |
| reference names a charge                   | `202`                                               |

The last two are identical on purpose: a non-2xx makes the rail retry forever,
and a distinct answer would be an oracle for "does this charge exist".

**There is no rate limit**, and the module says so plainly: a caller who knows
a live `provider_reference_id` can hold one charge at roughly one rail request
per worker claim. Bounded only by the 16 KiB body limit and the dedupe key.

## `/dash/v1` — the dashboard (owned by `vpay-dashboard`)

Two `GET` reads from `DASH_ROUTES`, eight unauthenticated staff routes merged
in from `STAFF_ROUTES`, and the CrateStack procedure transport `nest_service`d
at the same prefix. The whole nest is **conditional** on the deployment having
both a `dashboard_validator` and a `dashboard` binding; a deployment with no
`dashboard_client` mounts nothing and every `/dash/v1/…` path falls through to
the outer honest 404.

| Method | Path                                      | Notes                                           |
| ------ | ----------------------------------------- | ----------------------------------------------- |
| GET    | `/dash/v1/payment_intents`                | `DASH_ROUTES`, behind `require_dashboard_token` |
| GET    | `/dash/v1/payment_intents/{id}`           | same                                            |
| POST   | `/dash/v1/staff/login`                    | `STAFF_ROUTES`, outside the token layer         |
| POST   | `/dash/v1/staff/totp`                     | same                                            |
| POST   | `/dash/v1/staff/password`                 | same                                            |
| GET    | `/dash/v1/staff/session`                  | same                                            |
| GET    | `/dash/v1/staff/session/stage`            | same                                            |
| POST   | `/dash/v1/staff/logout`                   | same                                            |
| GET    | `/dash/v1/oauth/authorize`                | same                                            |
| POST   | `/dash/v1/oauth/token`                    | same                                            |
| POST   | `/dash/v1/$procs/searchPaymentIntents`    | CrateStack transport                            |
| POST   | `/dash/v1/$procs/searchRefunds`           | CrateStack transport                            |
| POST   | `/dash/v1/$procs/searchWebhookDeliveries` | CrateStack transport                            |
| POST   | `/dash/v1/$procs/searchCustomers`         | CrateStack transport                            |
| POST   | `/dash/v1/$procs/searchCheckoutSessions`  | CrateStack transport                            |

`/dash/v1` is **read-only structurally, not by promise**:
`require_dashboard_token` refuses any method that is not `GET`/`HEAD` with a
`403` _before_ the router matches, so a write mounted here would be a refused
request rather than an unlogged one (ADR-0008 wants an `audit_log` row per
dashboard write and none exists). The `$procs` transport is POST-only and
therefore uses a **separate** middleware,
`require_dashboard_procedure_token`.

> **Docs↔code disagreement (code wins).** `backends/crates/vpay-db/src/schema.rs`'s
> test is named `no_generated_model_route_is_mounted_only_the_one_procedure_is`
> and asserts "the one procedure this transport mounts"; `vpay_api::router`'s
> comment and `schema/search_payment_intents.rs`'s module docs say the same.
> That was true for Lane C. `procedure_router` mounts **every** procedure in
> the `ProcedureRegistry`, and `schemas/vpay.cstack` now declares **five**.
> Only `searchPaymentIntents` is exercised over HTTP by any test.

## Middleware, and why the order is load-bearing

Outer stack, applied to the whole router in `router`:

1. `discard_unusable_request_id` — vet the caller's own `x-request-id`
2. `SetRequestIdLayer::x_request_id(MakeRequestUuid)` — mint one if none survived
3. `mirror_request_id_header` — copy it onto the response under Stripe's
   `request-id` spelling
4. `TraceLayer` with `make_request_span` — open the span carrying `request_id`
5. `PropagateRequestIdLayer::x_request_id`

That order is what makes `Category::Internal`'s "Contact support with the
request id" a promise a merchant can act on: the id exists before the span is
built and reaches the caller afterwards. Nothing in `vpay_api::error` generates
an id of its own — a second id would appear in the log and in no header, which
is worse than none.

Per-nest layers, **inside** each nest:

- `track_http_metrics` on _every_ nest individually, and on the outer router
  **before** the `.nest()` calls. `Router::layer` wraps only the routes that
  exist when it is called, so the outer copy covers `/healthz` and the outer
  404 only. Moving that line below the nests would double every `/v1` count and
  label half of them `unmatched`. A request's route _pattern_ only exists once
  the nest has matched, which is why the layer cannot live outside.
- `RequestBodyLimitLayer` **outside** the auth layer on `/v1`, `/dash/v1` and
  the `$procs` transport, so an anonymous caller cannot make the process buffer
  a body before the 401.
- `CorsLayer` on `/v1/browser` and nowhere else.
- The `$procs` transport carries its **own** `.fallback(not_found)` added after
  its token layer, and it is not optional: `Router::nest_service` registers a
  _catch-all_ in the outer path table, which beats the fallback router where
  `Router::nest` put `dash::routes`' fallback. Without it, `GET
/dash/v1/not_a_route` with a valid token answers `403` instead of `404`.
