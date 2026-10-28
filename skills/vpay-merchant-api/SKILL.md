---
name: vpay-merchant-api
description: The vpay HTTP surface — the /v1 merchant API and its route tables, the two ways it authenticates (a COSE_Sign1 message under a registered Ed25519 key, and, where the deployment's auth.methods permits machine_tokens, a bearer token; never an API key), the merchant MCP endpoint at /mcp/v1, the mandatory Idempotency-Key, the Stripe-shaped error envelope, and the routes declared in the wire contract but mounted nowhere. Load this before adding, changing, removing or calling any HTTP route or MCP tool, before touching authentication or scopes, and before writing anything that returns an error to a merchant.
---

# The vpay HTTP surface

> **Verified against vpay `f885170c` (2026-10-27); the authentication, OP, scopes
> and MCP-auth sections re-read at `3bcf0763` (2026-10-28, after vpay#384 and
> #386).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

One router function, `vpay_api::router` in
`backends/crates/vpay-api/src/lib.rs`, mounts these nests:

| Nest          | Who calls it                 | Authenticated by                                                                                                                                                                                                   |
| ------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `/v1`         | a merchant's server          | a **COSE_Sign1 message** under the merchant's registered Ed25519 key (`EnvelopeLayer`, since 2026-10-26), **or** a bearer token (`aud = vpay:v1`) where `auth.methods` permits `machine_tokens` (since 2026-10-27) |
| `/v1/oauth`   | a merchant's token client    | the request body _is_ the credential; mounted only where `machine_tokens` is permitted (`404` otherwise, since 2026-10-27); mints `/v1` and `/mcp/v1` tokens                                                       |
| `/mcp/v1`     | a merchant's **agent** (MCP) | a bearer token from `/v1/oauth` whose audience is the MCP resource (since 2026-10-27, opt-in surface `mcp`, which needs `machine_tokens`)                                                                          |
| `/v1/browser` | a payer's browser            | publishable key + a `client_secret`                                                                                                                                                                                |
| `/provider`   | a payment rail               | **nothing at all** — by design                                                                                                                                                                                     |
| `/dash/v1`    | the staff console            | a COSE_Sign1 message under a paired console key (since 2026-10-27, vpay#383)                                                                                                                                       |
| `/svc/v1`     | a service (vaam)             | a COSE_Sign1 message under a `service_clients` key (since 2026-10-27, vpay#383)                                                                                                                                    |

_(Until 2026-10-27 this table had five rows and said `/v1` was authenticated by
a bearer token (`require_merchant_token`) and `/dash/v1` by a staff session.
vaam-apps/vpay#380 replaced the first and #383 the second; neither mechanism
exists any more. `/mcp/v1` and `/svc/v1` are new.)_

`/livez` and `/metrics` are on a **different port** (`vpay_api::observability`)
and deliberately not on this router. `/healthz` is, and answers bare text
rather than an error envelope. ~~It is the one response in the crate that is
not an error envelope.~~ _(Corrected 2026-09-23: there are others —
`/v1/oauth/token`'s RFC 6749 body, and axum's `405` and tower-http's `413`;
see [references/errors.md](references/errors.md).)_

`/dash/v1` and `/svc/v1` belong to the `vpay-dashboard` skill; the table here
lists them for completeness.

**Which nests a process mounts depends on `deployment.surfaces`**, since
2026-09-16 (ADR-0022, vpay#183; the ADR's status is still _Proposed_). This
page did not say so until 2026-09-23. `business` mounts `/v1`, `/v1/browser`,
`/provider` and `POST /v1/oauth/token`. `management` mounts `/dash/v1` and the
staff routes. Either surface mounts `/v1/oauth`'s discovery and JWKS, and
`/healthz` is always mounted. An absent key means both surfaces
(`EnabledSurfaces::ALL`), and an empty list is a boot error. **A third value,
`mcp`, exists since 2026-10-27 (vaam-apps/vpay#377, ADR-0033) and is opt-in
only:** it is not in `EnabledSurfaces::ALL`, so an absent key never mounts
`/mcp/v1`, and nothing in the Helm chart or compose sets it (ADR-0033 step 6 is
not built). A pod serving it needs `surfaces: [business, mcp]` or `[mcp]`. A nest the
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

## Authentication: a signed message, a bearer token where the deployment permits one, never an API key

**Since 2026-10-26 (vaam-apps/vpay#380, VP-3/VP-5/VP-7, ADR-0035), every `/v1`
request is a COSE_Sign1 message** signed by an Ed25519 key registered in
`merchant_clients[].cose_keys`, and every answer is a COSE_Sign1 message signed
by one of vpay's two response keys (`VPAY_ENVELOPE_SIGNING_KEY_FILE`,
`..._NEXT_FILE`), which every client pins. It is cratestack's `EnvelopeLayer`,
mounted on the `/v1` router before it is nested, in mode `Required`. The
signature binds the audience (`deployment.public_base_url`), method, route
template, path parameters, canonical query, the operation's contract digest
(`vpay-wire`'s table of 37), the payload type and the `Idempotency-Key`. A GET
has a body: the message. Mechanism, key rotation and the replay cache
(`envelope_nonces`, migration `0055`): vpay `docs/flows/signed-transport.md`.

- ~~**There is no bearer token on `/v1` at all.** `Authorization` is not read;
  a token of any audience is the same coarse `401` as no signature.~~
  **Corrected 2026-10-27:** that holds only where `auth.methods` leaves out
  `machine_tokens` (vaam's `[keypair]`), and even there `Authorization` is now
  read: on a signed request it is the `400` in the `auth.methods` section below. Without a token, `curl`
  cannot make a request — `sdks/nodejs/scripts/vpay-call.mjs` makes one signed
  call from a shell.
- **For a signed request, every refusal before the handler is one coarse,
  unsigned `401`, identical bytes for every cause** (unknown key, disabled client, bad signature, wrong
  audience, a clock more than 300 s off, a replay). It is no longer the Stripe
  envelope: it is `{"code": "UNAUTHORIZED", …}`. Errors **after** the layer
  (validation, `403`, `404`, `409`) are still the Stripe-shaped envelope,
  sealed. Diagnose a `401` from the table in `signed-transport.md` § "What a
  refusal looks like", because the response will not tell you.
- The SDKs map an unsigned answer to `Error::Unsigned` /
  `VpayUnsignedResponseError` and one that fails the pinned keys to
  `Error::Unverified` / `VpayUnverifiedResponseError` (see `vpay-sdks`).

_(Until 2026-10-27 this section said `/v1` takes `Authorization: Bearer
<access_token>` minted at `/v1/oauth/token` by `client_credentials` +
`private_key_jwt` (ADR-0010). That stopped being true on 2026-10-26. The OP
still exists — see "The OP" below. It became true again on 2026-10-27, as a
deployment option, by ADR-0037.)_

**Do not reach for an API key.** `backends/migrations/0008_create-merchant-api-keys.sql`
was dropped by `0009_drop-merchant-api-keys.sql`: API keys are a deliberately
deleted design, not a missing feature. Publishable keys (`pk_test_…`) exist,
authorise nothing, and are not secrets — they name a tenant on `/v1/browser`.

The layer is mounted **in front of the whole nest, never per-handler**, and
`vpay_api::envelope::scope` (the step after it) turns the verified signer into
the merchant, its tenant and its YAML scopes. That is what makes it impossible
to add a `/v1` handler without authentication; the failure mode of a
per-handler check is a handler that simply has none. The scope step only sees
the method, which is why scopes are per-method. _(`require_merchant_token`,
which this paragraph named until 2026-10-27, was deleted by vpay#380.)_

It puts a `MerchantScope` on the request (`MerchantScope::for_signed`).
**`MerchantScope` has a private field and no public constructor** — a handler
cannot invent a tenant to query by. Its `pub(crate)` constructors are named for
their callers (`for_signed`, `for_payer`, `for_dashboard`, and `for_mcp` for an
MCP tool call). There is no
`merchants` table and so no foreign key that would catch an unscoped query, so
that type is the whole tenancy boundary. Repository methods on this path take a
`merchant_id` and have no unscoped variant.

### `auth.methods` decides whether a token is a second way in (since 2026-10-27)

_vaam-apps/vpay#384 and #386, ADR-0037, which **supersedes ADR-0035 in part**.
Source: vpay `docs/reference/vpay-config.md` § "`auth:`",
`docs/flows/signed-transport.md` § "Both ways, when the deployment permits it",
and the doc comment of `vpay_api::envelope::scope`._

`auth.methods` in the YAML lists `keypair`, `machine_tokens`, `hmac_webhooks`
and `human_login`. **Absent means all four**, so a file written before the key
existed permits a bearer token beside a signature. vaam sets `[keypair]` plus
`hmac_webhooks` (until sealed webhooks, VP-8), so on vaam the rest of this page's
"signed only" statements hold unchanged.

| `keypair` | `machine_tokens`  | `/v1` envelope | `/v1/oauth/token` | An unsigned `/v1` request                         |
| --------- | ----------------- | -------------- | ----------------- | ------------------------------------------------- |
| yes       | no                | `Required`     | `404`             | the layer's coarse `401`                          |
| yes       | yes (the default) | `Optional`     | mounted           | validated as a bearer token                       |
| no        | yes               | `Off`          | mounted           | validated as a bearer token; a COSE body is `415` |

- **One caller, one credential.** A signed request that also carries
  `Authorization` is a sealed `400` on **every** deployment, `[keypair]`
  included. Before ADR-0037 the header was ignored on a signed request.
- **A signature that fails to verify is the `401`, never downgraded** to the
  token path.
- **A token caller** is validated for audience `vpay:v1` and this deployment's
  issuer, its `client_id` resolves to a tenant through the YAML, and it gets
  plain, unsealed JSON with no contract lock (no `426`). Its refusals are the
  Stripe-shaped ones from before VP-3: `401` `missing_bearer_token`, `401`
  `invalid_token` (also for a client in `disabled_clients`, even with a token
  minted before the switch), `503` when the JWKS cannot be read, and `403`
  `forbidden` for a token naming no registered client or lacking the scope.
- **Idempotency does not see the credential.** A retry that switches between a
  signature and a token deduplicates on `(merchant_id, Idempotency-Key)`.
- **The SDKs still only seal.** Their token modes are VPA-6, not built.
- `GET /.well-known/vpay-auth` answers `{"methods": [...]}` unauthenticated:
  what the deployment permits and serves (`human_login` is permitted by the
  default and served by nothing in this release; VPA-3 restores staff sign-in).
- Every boot refusal the key adds (`NoWayIn`, `McpNeedsMachineTokens`,
  `MerchantNoCredential`, `AuthMethodDisabled`, `NearMissKey`,
  `MisplacedAuthKey` and the rest) is in `vpay-ops`'s configuration reference.

### Merchant Ed25519 keys: what `cose_keys` is (since 2026-10-26)

_vaam-apps/vpay#379 (VP-2), shipped in v0.15.0. Source: vpay
`docs/status/backend.md`, "Merchant Ed25519 keys are registered…", and
`docs/flows/merchant-auth.md` § Status._

- **`merchant_clients[].cose_keys`** takes a merchant's Ed25519 **public**
  keys: unpadded base64url, 43 characters, with an optional `label` of 1–64
  characters. Boot refuses an entry that does not decode to a valid Ed25519
  point (`ConfigError::MalformedCoseKey`), and a thumbprint registered twice,
  across merchants or within one (`ConfigError::DuplicateCoseKey`). **An empty
  list is legal**, and `config/application.yml` leaves it empty.
- **`vpay_api::merchant_keys`** builds a `MerchantKeyRegistry` and a
  `MerchantKeyResolver`, cratestack-cose's `CoseVerifierResolver` for Ed25519.
  It maps a key to its merchant, tenant and YAML scopes. A `disabled_clients`
  row resolves to nothing, and a failed kill-switch lookup is a `500`, never a
  key and never a miss.
- **`vpay-wire`** (`backends/crates/vpay-wire`) holds `V1_OPERATIONS` and
  `V1_CONTRACTS`, one digest per `/v1` `(method, path)`. The digest is the
  operation's identity, not its body's shape: adding a field to a form body
  does not move it. `vpay_api::v1::contracts` tests the table against
  `V1_ROUTES` in both directions.

~~**None of it is on a request path.** `/v1` still takes the bearer token
above, and `jwks` is still required. … **Do not tell a merchant to sign
requests with these keys, and do not describe the signed transport as built.**
That is VP-3 onward, and none of it is in v0.15.0. ADR-0010 is not
superseded.~~ **Corrected 2026-10-27:** that was true of v0.15.0 (VP-2,
vaam-apps/vpay#379) and stopped being true one PR later. Since vpay#380
(2026-10-26) the resolver above is what the `EnvelopeLayer` verifies every
`/v1` request against, the server signing keys and `envelope_nonces` exist,
and both SDKs sign. ~~**An empty `cose_keys` now means the merchant cannot call
`/v1` at all.** `jwks` is still required in a registration and still read, but
only by the OP (below), until VP-4.~~ **Corrected 2026-10-27 (ADR-0037):** an
empty `cose_keys` means the merchant cannot **sign**. It can still call `/v1`
with a token if it has a `jwks` and the deployment permits `machine_tokens`.
Every registration needs one of the two (`MerchantNoCredential`). `jwks` is no
longer required on its own: with `grant_types` and `allowed_audiences` it is the
OAuth half, required together only for a merchant that has a `jwks` while
`machine_tokens` is permitted, and refused outright when it is not.

### The OP: `/v1` tokens where permitted, the only source of `/mcp/v1` tokens, and two limitations nobody has decided

**The OP is `/v1/oauth/token`, `client_credentials` + `private_key_jwt`
(ADR-0010).** Since ADR-0037 (2026-10-27) it is mounted on the business surface
**iff `auth.methods` permits `machine_tokens`** (otherwise an honest `404`), and
`/v1` reads the `vpay:v1` tokens it mints again. It is the only way to get a
token for the MCP endpoint, which is why `mcp` refuses to boot without
`machine_tokens` (`McpNeedsMachineTokens`). _(From 2026-10-26 until vpay#384 on
2026-10-27 this section said it applied to `/mcp/v1` tokens only and that
nothing on `/v1` read what it mints. That was true for that one day.)_
**VP-4 plans to retire it, and nothing replaces it for `/mcp/v1` yet** —
ADR-0033's "Left to the maintainer" item 4. Do not delete the OP, or its
`jwks` and `oauth_client_assertion_jtis`, without that decision.

_Added 2026-09-29, from `docs/flows/merchant-auth/verification-and-limits.md`;
nothing here was re-run. "`/v1` route" below was history from 2026-10-26 and is
true again for a token caller since 2026-10-27._

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
- **A spent `jti` is kept five minutes past its `exp`, because the validator
  accepts the assertion for 61 s after it** (ADR-0028, vaam-apps/vpay#271,
  2026-10-09). Single use is only as good as the row: `authkestra-op` (`=0.13.1` since 2026-10-26, unchanged here from 0.7.1) leaves
  `jsonwebtoken`'s 60 s leeway at its default, on a whole-second `now`, so an
  assertion verifies until `exp + 61 s`. The worker's hourly sweep used to delete
  rows at `expires_at < now()`, and a sweep landing in those 61 s let a captured,
  already-spent assertion be replayed **once** for a second access token (about
  1.7 % per assertion at zero clock skew, silent). Now the sweep deletes
  `expires_at < now() - retain_after_exp`, computed by Postgres, and the worker
  passes `vpay_worker::CLIENT_ASSERTION_JTI_RETENTION` (5 minutes: 60 s + 1 s +
  about 239 s of skew budget). `expires_at` still stores the client's raw `exp`.
  Two things an agent will trip on: **a test refuses an assertion older than the
  horizon** (`an_assertion_older_than_the_sweep_horizon_is_refused_by_the_validator`),
  so bumping `authkestra-op` or `jsonwebtoken` past a wider leeway fails CI until the
  constant grows; and **what is not covered** is a database clock more than about
  four minutes ahead of an API replica's, which nothing checks. Nothing a merchant
  sends or receives changed. Mechanism and the sweep itself: `vpay-reconciler`.
  _(Until 2026-10-09 the doc comment on vpay's `delete_expired_client_assertion_jtis`
  said an assertion past its `exp` "is refused by `verify_client_assertion` before
  any store is consulted" and that the worker's job loop "does not exist yet". Both
  were wrong, the second since 2026-09-03.)_
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
everything else, including verbs no route answers, requires write. Since
2026-10-26 a signer's scopes are its registered `scopes:` from YAML, read by
`vpay_api::envelope::scope`. Since 2026-10-27 a token caller's are the
intersection of its `scope` claim and the same registration's **current** YAML
`scopes` (`ResourceConfig::token_holds`), so a scope removed from the YAML
stops working for tokens minted before the restart.

| Situation (since 2026-10-26)                             | Answer                                                         |
| -------------------------------------------------------- | -------------------------------------------------------------- |
| unsigned, unknown key, disabled client, replay, …        | `401`, coarse and **unsigned**, identical bytes for every case |
| verified signer, no scope for this method                | `403` `forbidden`, sealed                                      |
| verified signer, client disabled between the two lookups | `403` `forbidden`, sealed                                      |
| verified signer, object belongs to another merchant      | `404`, byte-identical to a nonexistent id                      |
| a path or method that is no operation                    | `404` `unknown_route`, the honest one, not a `401`             |
| a signed request that also carries `Authorization`       | `400`, sealed (since 2026-10-27, every deployment)             |

A **token caller** (since 2026-10-27, only where `machine_tokens` is permitted)
gets the plain, unsealed answers in the `auth.methods` section above:
`401` `missing_bearer_token` or `invalid_token`, `503` on an unreadable JWKS,
`403` `forbidden` for no registered client or no scope.

_(Until the morning of 2026-10-27 the first row said "no bearer token → `401`
`missing_bearer_token`", and the second and third said "valid token". Those
were removed when #380 made `/v1` signed-only, then came back for token callers
with vpay#384 the same day.)_

The rule to internalise: a statement about the **credential** is 403 (the
caller can see its own registration); a statement about an **object** is a
uniform 404, because anything else is an enumeration oracle.

A token request naming no `scope` is granted the client's registered
`scopes:` (RFC 6749 §3.3's "locally defined default", applied in
`op::token::token_handler`), for `/v1` and `/mcp/v1` tokens alike. _(This said
"For `/mcp/v1` only" while that was the OP's one customer.)_

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

## The merchant MCP endpoint, `/mcp/v1` (since 2026-10-27)

_vaam-apps/vpay#377, ADR-0033 step 3 (ADR-0033 is still **Proposed**). Source:
vpay `docs/flows/mcp.md`, `docs/api/mcp-parity.md`,
`docs/reference/vpay-api/mcp.md`, `docs/security/threat-models/mcp.md`._

`POST /mcp/v1` speaks MCP revision `2026-07-28` over Streamable HTTP to a
merchant's **agent**. It is mounted only when `deployment.surfaces` names
`mcp` (above), so a default deployment does not have it.

- **Nine tools, as of 2026-10-27:**
  - `create_payment_intent`, `retrieve_payment_intent`,
    `confirm_payment_intent` and `cancel_payment_intent`;
  - `create_refund`, `retrieve_refund` and `cancel_refund`;
  - `search_payment_intents` and `search_refunds`, the dashboard's existing
    search procedures. These page by `offset`/`limit` and return
    database-shaped summary rows, not the `/v1` list object.

  The other **28** `/v1` routes are dated gaps in `docs/api/mcp-parity.md`:
  customers, checkout sessions, invoices, invoice items, events, account
  holders and the refund metadata update. **Do not tell an agent a tool exists
  for them.**

- **A tool runs the `/v1` operation, never a copy of it.** The tools are
  `@mcp(tool)` procedures in `schemas/vpay.cstack`. Each write's body calls the
  `MerchantOperations` port (`vpay-db`). `vpay-api` implements that port
  (`mcp::MerchantOps`) by running the route's own `*_with` function, which
  vaam-apps/vpay#375 split out of each handler. It passes a
  `PostRequest::for_tool` whose parameters are what the form body would have
  decoded to, so validation, idempotency and the rail call are `/v1`'s. **A
  new rule belongs in the `*_with` function, where both surfaces get it**, not
  in the procedure body.
- **`idempotency_key` is a required argument on every write.** It is claimed
  in the same per-merchant `idempotency_keys` store, under
  `("MCP", "tool/<name>", hash of the arguments)`. A replay returns the first
  answer; the same key with different arguments is `idempotency_key_in_use`.
- **Shapes differ from `/v1` in named places, because MCP schemas take no free
  JSON:**
  - `metadata` is `[{key, value}]`;
  - confirm's payer fields are `payer_fields: [{name, value}]`;
  - outputs rename `next_action.type` to `kind` and
    `payment_method_details.type` to `method_type`;
  - outputs carry no `object` and no `client_secret`.

  A refusal is an `isError` result whose message starts with `/v1`'s own
  `error.code` and `(param)`.

- **Authentication is a bearer token only**: a signed request is not an MCP
  credential, and the `mcp` surface refuses to boot unless `auth.methods`
  permits `machine_tokens` (`McpNeedsMachineTokens`, since 2026-10-27). A
  client in `disabled_clients` is refused on its next call, even with a token
  minted before the switch (ADR-0037).
  - A merchant opts in by adding `{public_base_url}/mcp/v1` to its
    registration's `allowed_audiences`.
  - Its agent gets a token from `/v1/oauth/token` (`client_credentials` +
    `private_key_jwt`) with `audience=` or RFC 8707 `resource=` set to that
    URL. The OP honours `resource` since vpay#377; it silently dropped it
    before.
  - `mcp::MerchantMcpAuth` validates the audience, maps the client to its
    merchant (`MerchantScope::for_mcp`), and lets a write through only for
    `payments:write`.
  - A token for any other audience is `401` with an RFC 9728 challenge naming
    `/.well-known/oauth-protected-resource/mcp/v1`.
  - A foreign `Origin` is refused.
- **CrateStack's generated `procedure_router` mounts every procedure**, so the
  merchant writes would also be routable on `/dash/v1/$procs`. Three things
  stop that, and **removing any of them is a defect**:
  - `only_dashboard_procedures` in `vpay-api/src/lib.rs` allowlists the five
    searches;
  - each write's `@allow` needs `audience == "merchant"`;
  - the dashboard's registry has no port.

  The generated `@vpay/dash-client` still lists the seven procedures.

- **A gate keeps it honest:** `cargo xtask verify-mcp-parity` (the
  seventeenth `just verify` gate, see `vpay-tooling`) fails on a `/v1` route
  with no row in `docs/api/mcp-parity.md`, on an undated gap, and on a declared
  tool no row names. **A `/v1` route you add needs a parity row in the same
  change.**
- **Not built, as of 2026-10-27:**
  - **No MCP resources.** ADR-0033 D5's `jsonb` measurement was never made.
  - **No staff endpoint.** It waits on an upstream CrateStack per-audience
    feature. Its auth design is also open: ADR-0033 assumed the staff OAuth
    that vpay#383 deleted.
  - **No Helm or compose workload.**
  - **No real MCP client or agent host has called it.** The suite speaks
    JSON-RPC over HTTP.
  - **No MCP case drives a confirm to `succeeded`.**
  - **No per-merchant amount ceiling.** That is the maintainer's call.
  - **Nothing issues its tokens once VP-4 retires the OP.** That is an open
    decision.

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
`docs/sdks/parity.md`, and a row naming no capability. Since 2026-10-27 it is
also not done until `docs/api/mcp-parity.md` has a row for it
(`cargo xtask verify-mcp-parity`), and since 2026-10-26 until `vpay-wire`'s
operation table has it, because a route with no operation cannot be signed. Then
`docs/status/backend.md`, then the flow page.

Wire objects, fields, id prefixes and validation bounds:
[references/objects.md](references/objects.md).
