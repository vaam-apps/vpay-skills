# Using the official Stripe SDKs against vpay

_Verified against vpay `93c6dfd0` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Source of record: `docs/flows/stripe-sdk-compat.md`. Evidence:
`sdks/stripe-compat`.

vpay's `/v1` object model, form encoding, error envelope and idempotency
semantics **are** Stripe's. Its **authentication is not**: there is no API key,
and every call carries a short-lived bearer minted from an RFC 7523
`private_key_jwt` client assertion (ADR-0010).

## The one seam

`stripe-node` accepts an arbitrary async `config.authenticator`, invoked once
per request attempt with the whole outbound request. That is the only hook the
handshake needs, and `@vaam-apps/vpay-sdk/stripe`'s `createStripeAuthenticator`
fills it.

```js
const authenticator = createStripeAuthenticator({
  baseUrl: "https://api.vpay.example",
  clientId: "acme-cameroon",
  privateKey: readFileSync("./merchant-key.pem", "utf8"),
  kid: "acme-cameroon-2026-08", // only if more than one JWK is registered
});

const stripe = new Stripe("", {
  authenticator,
  host: "api.vpay.example", port: "443", protocol: "https",
});
```

- `new Stripe("", { authenticator })` is supported — stripe-node refuses only
  when **both** a key and an authenticator are given, or neither.
- `host`/`port`/`protocol` move every request off `api.stripe.com`. `basePath`
  is fixed at `/v1/` and is not configurable, which is moot: the generated
  resources use absolute paths and those are exactly vpay's.
- **The authenticator writes one thing: `headers.Authorization`.** It must not
  rewrite the body — `Content-Length` is computed before it runs.

## Compatible, unchanged

|                                           | Why                                                                                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| The resource paths                        | `/v1/payment_intents`, `/{id}`, `/{id}/confirm`, `/{id}/cancel` are the four vpay serves and the four stripe-node hardcodes                                                    |
| Form encoding, nested and indexed         | stripe-node percent-encodes then decodes brackets back, so a space is `%20` and a literal `+` is `%2B` — what `vpay_api::form` requires. Arrays are **indexed** (`expand[0]=`) |
| `Idempotency-Key`                         | stripe-node generates one for **every** v1 POST unconditionally. vpay _requires_ one — stricter than Stripe, and free for a stripe-node user                                   |
| The list envelope and `autoPagingToArray` | Needs only `data[].id` and `has_more`; `ListObject` supplies both plus `url`                                                                                                   |
| The error envelope                        | `{error: {type, code, message, param?}}`, same closed `type` vocabulary                                                                                                        |
| `webhooks.constructEvent`                 | vpay's signature construction is **byte-identical** to Stripe's                                                                                                                |

## Error mapping

stripe-node picks the error class from the **status code first** and consults
`type` only inside 400/404. vpay derives status, `type` and `code` from one
classification (ADR-0011), so this is two designs meeting rather than anything
written to line up.

| vpay answers            | You catch                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `404 resource_missing`  | `StripeInvalidRequestError`, `err.code === "resource_missing"`                                                           |
| `400 invalid_request`   | `StripeInvalidRequestError`, `err.param` names the field                                                                 |
| `400 idempotency_error` | `StripeIdempotencyError`                                                                                                 |
| `401`                   | `StripeAuthenticationError`                                                                                              |
| `403`                   | `StripePermissionError` — **despite** carrying `type: invalid_request_error`, because stripe-node branches on the status |
| `409`                   | `StripeAPIError` — 409 falls through every branch of `generateV1Error`                                                   |
| `502`                   | `StripeAPIError`, **and stripe-node retries it**                                                                         |
| `429`                   | never emitted — nothing constructs `Category::RateLimited`                                                               |

`err.requestId` comes from a **`request-id`** response header; stripe-node
never reads `x-request-id`. vpay emits **both names with one value**.

### `stripe-should-retry`

stripe-node consults this header **above** its own status rules, and vpay needs
both directions: `409` gets `false` (stripe-node retries every 409
unconditionally, and a lifecycle refusal is not something waiting fixes);
`IdempotencyKeyInFlight`'s `400` gets `true` (stripe-node retries no 4xx, and a
key still in flight is the one refusal that clears itself).

**A replayed response carries the advisory the original carried.** Migration
`0025`'s `idempotency_keys.response_retry` stores the header's own text, read
off the rendered `HeaderMap` at finish and written back at replay. Re-deriving
it from the stored status at replay time was the fix **deliberately not taken**:
ADR-0011 makes one classification the source of status _and_ retry, and a
second derivation running the other way is exactly the drift it exists to
prevent.

## The divergences an integration actually hits

- **No API keys.** `apiKey`, `stripeAccount` and Connect mean nothing.
  `Stripe-Version`, `Stripe-Account`, `Stripe-Context` and `X-Stripe-Client-*`
  are **accepted and ignored** — a `Stripe-Account` is deliberately _not_ a
  `400`, because a documented "Connect is not a thing here" is a better
  diagnostic than a refusal.
- **No dated API version.** vpay advertises none and echoes none, so
  `obj.lastResponse.apiVersion` is `undefined` and pinning `apiVersion` has no
  effect.
- **`payment_method_types` is required and non-empty**, and each entry must
  name a rail this deployment has enabled. A copied
  `automatic_payment_methods: { enabled: true }` is silently dropped and the
  request is then refused for the missing required field, naming it.
- **`confirm: true` on create is refused**, with `param: "confirm"` and a
  message naming `POST /v1/payment_intents/{id}/confirm`. It used to be dropped
  silently, which left a merchant believing they had charged someone.
  `confirm: false` is accepted, because it asks for what the endpoint does.
- **`payment_method_data.type` is a rail code** — `mtn_momo`, with the
  instrument under a key of the same name
  (`payment_method_data[mtn_momo][msisdn]`). TypeScript users need a cast:
  stripe-node's generated types know Stripe's methods, not vpay's rails.
- **The fields that decide _where_ or _when_ money moves are refused, not
  ignored**, with a `400` naming the field in `error.param`: `capture_method`
  with any value but `automatic`, `application_fee_amount`, `transfer_data`,
  `on_behalf_of`. Both POST bodies carry the same refusal set. Ignoring any of
  them would settle a merchant's money at a time, or to an account, they did
  not ask for and could not see in the response.
- **Everything else Stripe sends that vpay does not implement is accepted and
  ignored** — the default, not the exception. Neither `CreateParams` nor
  `ConfirmParams` has `deny_unknown_fields`, because a `400` per field a Stripe
  SDK adds of its own accord would make vpay unusable from the SDKs this work
  exists to support. `setup_future_usage`, `confirmation_method`,
  `receipt_email`, `statement_descriptor`, `customer`, `expand` and `metadata`
  all leave the payment as requested (`metadata` is stored; the rest dropped).
  The line between the two halves is whether the absence is visible in the
  response.
- **`client_secret` is on `create` and `retrieve` only.** Absent from
  `confirm`, `cancel`, `list` and every webhook body. `amount_received`,
  `capture_method` and `confirmation_method` are genuinely absent although
  stripe-node's types declare them present — a type-level lie with no runtime
  effect, since `StripeResource._makeRequest` casts and never validates.
- **`next_action` is only ever `redirect_to_url`**, and only on a redirect
  rail. A push rail leaves it `null`.
- **Currencies are XAF and EUR**, integer minor units.
- **There is no `failed` status.** A refused charge returns the intent to
  `requires_payment_method` with `last_payment_error` set.
- **`405` and `413` carry neither the envelope nor the header.** `405` is
  axum's own with an empty body; `413` is tower-http's, `text/plain`. Neither
  passes through `ApiError::into_response`. The consequence is worse than a
  missing header and is **measured, not inferred**
  (`sdks/stripe-compat/src/errors.compat.test.ts`): stripe-node meets a
  non-JSON body by discarding everything it knows and throwing
  `StripeAPIError: Invalid JSON received from the Stripe API` with `statusCode`
  and `headers` both `undefined`. **A merchant cannot tell a 405 from a 413
  from a proxy's HTML 502** — the only thing that survives is `err.requestId`.
  A `method_not_allowed` renderer and an envelope for the body limit would fix
  both; neither exists.
- **`search`, `POST /v1/refunds` and `/v1/balance`** are not routed and answer
  the honest `404 unknown_route`. `GET /v1/refunds/{id}` **is** routed, but
  nothing drives it through stripe-node's `refunds` resource — so
  `stripe.refunds.retrieve()` working is **untested rather than known**, and
  the same is true of `stripe.events.list()`.

## Webhooks

`stripe.webhooks.constructEvent(rawBody, header, secret)` takes the header
**value**, not the request, and verifies
`t=<unix>,v1=<lowercase hex HMAC-SHA256 of "<t>.<raw body>">` with a 300 s
tolerance — byte-identical to vpay's `Vpay-Signature`. vpay's deliverer sends
`Vpay-Signature` **and** `Stripe-Signature` carrying the same string byte for
byte, so `req.headers["stripe-signature"]` works unedited. `Vpay-Signature`
stays the authoritative name in vpay's own documentation.

`sdks/stripe-compat/src/webhooks.compat.test.ts` is an **observation**, not an
argument from the scheme being identical: it makes a payment through the
official package, waits for the worker to settle it, pulls the delivery out of
the WireMock receiver's own request journal (`GET /__admin/requests` — what a
receiver _got_), and passes the recorded bytes and header straight to
`constructEvent`. **The bytes are never re-serialised**: the signature covers a
body, and parse-and-reprint is the commonest way a merchant breaks their own
verification. Both refusals are asserted too.

## There is no Rust twin

`async-stripe` builds its client around a headers/secret pair and has no
per-request async hook equivalent to stripe-node's `RequestAuthenticator`.
Reaching the same result means wrapping its transport in custom middleware,
which was scoped as a follow-up. Rust merchants use `sdks/rust`. Three ⛔ rows
in the gap ledger cover it.

## Status, and what must not be read as proven

**Built and proven against a real stack, 2026-09-03**: the real
`stripe@22.6.1` driven through `createStripeAuthenticator` against a real
`vpay-server` + Postgres + WireMock rails + worker + WireMock receiver, out of
process over TCP. **25 cases, 0 skipped**, via `just demo_port=18080
stripe-compat`; CI runs it in the `e2e (compose)` job. The suite **cannot
skip** — `src/preflight.ts` fails the run when no stack answers.

Not proven:

- **The `stripe-should-retry: true` direction is not observed.** Provoking
  `IdempotencyKeyInFlight` needs two concurrent requests where one holds the
  key long enough to collide, and the only slow operation is a confirm whose
  rail-side delay is keyed by a server-minted reference. A deterministic stage
  would need a test double, which ADR-0006 forbids in a shipping process. The
  _derivation_ is unit-tested in `vpay-api`; its effect on stripe-node is not.
- **The `502` re-POST is reasoning, not a measurement.** A `502` stripe-node
  retries under the same `Idempotency-Key` meets vpay's "one charge per intent,
  forever" rule and comes back a `409`. That is the correct thing to want and
  it is not what happens — stated because it is the predictable consequence of
  the header, not because anything observed it.
- **The rail is a WireMock host.** MTN has never been called through this
  suite. No money has moved. The `succeeded` it polls to is a stub mapping
  driven through the real worker, the real settlement transaction and the real
  `/v1` renderer — but the approval itself is fiction.
- **The receiver is a WireMock host too.** No merchant endpoint has ever been
  POSTed to. Deliveries do go through `vpay_worker::ssrf` since Step 8; the
  compose stack's private receiver is permitted by the sandbox profile's
  `webhooks.allow_private_targets`, not by the guard being absent.
