# The error model

_Verified against vpay `f063ee96` (2026-09-15). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

ADR-0011. Two tiers, and one rule that holds them together: **a boundary
derives its answer from `Classify`; it never decides one.**

- Tier 1 — `vpay_core::error`: the `Classify` trait, the `Category` enum, and
  the policy table that maps a category to a status, a Stripe `type`, a default
  code, a retry policy and a log severity.
- Tier 2 — `vpay_api::ApiError`: the HTTP composite, and the single place an
  envelope is rendered. `vpay_worker::JobError` is its sibling for the job
  loop, same shape.

A handler returns `Result<_, ApiError>` and uses `?`. It never picks a status
and never formats a merchant-facing sentence.

## The envelope

```json
{ "error": { "type": "...", "code": "...", "message": "...", "param": "..." } }
```

Built by `error_envelope_with_param` in `vpay_api` — `pub(crate)`, called from
`ApiError`'s `IntoResponse` and nowhere else in production. That visibility is
what makes "two handlers cannot answer the same `DbError` differently"
impossible rather than merely discouraged: a handler in another crate cannot
reach the function at all.

`param` is **omitted, never `null`**, when there is none. Stripe omits it, and
an SDK testing `"param" in error` would be misled by a null. It is rendered
only for `ApiError::InvalidParam`.

`param` is a **field name from vpay's own vocabulary, never a value the caller
sent**. `ApiError::invalid_param` takes an `impl Into<String>`, so nothing in
the type system stops a future handler passing a header or a request body
through it; the 64-character plausible-field-name bound is what makes that
mistake harmless — anything else renders as a fallback, so the envelope cannot
become a reflection channel.

### Two response headers you get for free

- **`stripe-should-retry: true|false`** — derived from `Classify::retry()`,
  asked of the same classification the status came from. It is **not** derived
  from the status, so a `409` refusal and a `409` that heals answer
  differently. stripe-node reads this header _above_ its own status-code rules.
- **`request-id`** — Stripe's spelling, mirrored from `x-request-id` by
  middleware. `vpay_api::error` deliberately has no `request_id` field: a
  second id generated here would appear in the log and in no response header,
  which is worse than none. `Category::Internal`'s message promises the
  merchant a request id, and that header is what the promise points at.

## The `Category` policy table

`vpay_core::error::Category` — twelve variants, and the table is exhaustive by
construction (no `_` arms, so adding a category fails to compile everywhere it
matters).

| Category         | Status  | `stripe_type`           | `default_code`           | `default_retry` | `default_severity` |
| ---------------- | ------- | ----------------------- | ------------------------ | --------------- | ------------------ |
| `InvalidRequest` | **400** | `invalid_request_error` | `invalid_request`        | `Never`         | `Info`             |
| `Authentication` | **401** | `authentication_error`  | `invalid_token`          | `Never`         | `Info`             |
| `Forbidden`      | **403** | `invalid_request_error` | `forbidden`              | `Never`         | `Info`             |
| `NotFound`       | **404** | `invalid_request_error` | `resource_missing`       | `Never`         | `Info`             |
| `Conflict`       | **409** | `invalid_request_error` | `invalid_state`          | `Never`         | `Info`             |
| `Idempotency`    | **400** | `idempotency_error`     | `idempotency_key_in_use` | `Never`         | `Info`             |
| `RateLimited`    | **429** | `rate_limit_error`      | `rate_limit`             | `AfterBackoff`  | `Warn`             |
| `Rail`           | **502** | `api_error`             | `provider_unavailable`   | `AfterBackoff`  | `Warn`             |
| `Storage`        | **503** | `api_error`             | `service_unavailable`    | `AfterBackoff`  | `Error`            |
| `Configuration`  | **500** | `api_error`             | `misconfigured`          | `Never`         | `Error`            |
| `NotImplemented` | **501** | `api_error`             | `not_implemented`        | `Never`         | `Error`            |
| `Internal`       | **500** | `api_error`             | `internal_error`         | `Never`         | **`Page`**         |

Stripe's `type` vocabulary is closed (five strings), so several categories share
one and are told apart by `code` and the status.

Only `Rail`, `Storage` and `RateLimited` are retryable as-is. `Retry` has three
values, and the third matters: `NewAttempt` means _do not repeat this operation
— start over with a new one_ (a new PaymentIntent, per
`docs/flows/payment-lifecycle.md`). `stripe-should-retry` renders `false` for
both `Never` and `NewAttempt`, because neither means "send the same request
again".

`Severity::Page` is separate from `Error` because a `tracing` level cannot tell
them apart; that is why `vpay_alert_events_total` exists as its own counter
rather than a level filter. Metric labels are the `Debug` spelling, so an
alert's label and the JSON log line that produced it are joinable by eye.

### Two category boundaries that get confused

- **A rail _rejecting_ a charge is not `Category::Rail`.** That is a business
  outcome — a `vpay_core::FailureCode` on the charge. `Category::Rail` is "the
  rail could not be reached or answered incoherently".
- **`Conflict` is "the object's state forbids it"**, not "you sent something
  wrong". Cancelling an intent that is already `processing` is `Conflict`;
  cancelling one that does not exist is `NotFound`.

## `ApiError`, and the rule that keeps it honest

`vpay_api::ApiError` has seven `#[from]` leaves — `Db`, `Provider`, `Money`,
`Currency`, `Ledger`, `Config`, `Auth` — and its own variants: `UnknownRoute`,
`InvalidParam`, `IdempotencyKeyReused`, `IdempotencyKeyInFlight`, `NotFound`,
`Conflict`, `Forbidden`, `StaffSignInRefused`, `StaffSignInRateLimited`,
`StaffAuth`, `CheckoutNotConfigured`, `CheckoutSessionNotOpen`, `Internal`.

> **A composite never re-classifies.** Every `Classify` method — `category`,
> `code`, `retry`, `severity`, `public_message` — delegates **wholesale** for a
> wrapped variant, not just `category()`.

Forwarding the category alone silently discards a leaf's deliberate override.
`ProviderError::Rejected` overrides `code()` to `charge_declined`, `retry()` to
`Retry::NewAttempt` and `severity()` to whatever the `FailureCode` deserves (a
blocked _partner_ account pages), while its category `Conflict` defaults to
`invalid_state` / `Never` / `Info`. A category-only delegation would answer a
declined charge with the wrong code and log a blocked partner account as one
more merchant typo — and `vpay_worker::JobError` would answer the identical
error differently, which is exactly the drift ADR-0011 exists to stop.

## The four deliberate code overrides

Everything else uses its category's default. These four exist because an SDK
has to branch on the difference:

| Variant                  | Code                                                     | Why the default is not enough                                                                                        |
| ------------------------ | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `UnknownRoute`           | `unknown_route`                                          | "you called an endpoint vpay does not implement" ≠ "that payment intent does not exist"                              |
| `IdempotencyKeyInFlight` | `idempotency_key_in_flight`                              | "your own earlier request is still running" (wait) ≠ `idempotency_key_in_use`, "you changed the body" (fix your bug) |
| `CheckoutNotConfigured`  | `checkout_not_configured`                                | "this deployment has no checkout page" (a permanent capability answer) ≠ `misconfigured`, an outage                  |
| `CheckoutSessionNotOpen` | `checkout_session_expired` / `checkout_session_complete` | "your payer abandoned this checkout" ≠ "this intent is already processing"                                           |

The last is two codes from one variant, chosen by a two-variant `ClosedSession`
enum so the match stays total. It is a `code` and **not** a `param`, because
`param` must name a field of the request and the request that trips this
carries no reference to a session at all.

One retry override: `IdempotencyKeyInFlight` is `AfterBackoff`, because the
first request finishing is what clears it. Its sibling `IdempotencyKeyReused`
stays `Never` — a key reused with a different body never becomes valid.

## What crosses the wire, and what does not

`public_message()` is the **only** thing a caller sees. The full `Display`
**and** the `source` chain go to the log, because a leaf's `Display` names
hosts, tables and library error text on purpose (ADR-0011: `Display` is for
operators). The two are pinned apart by a test that puts a recognisable string
inside a `sqlx::Error` and asserts it appears in the log line and not in the
body.

When you add an error, the questions are: what category is it, does a leaf
override earn its keep, and does `public_message` name anything internal.

Related habits enforced by the same gate:

- an `Idempotency-Key` is echoed back at most **eight characters** — enough to
  tell two of a merchant's own keys apart in a support ticket, not enough to
  reconstruct one;
- a rejected currency code is truncated on the way out, though `Display` keeps
  all of it for the operator.

## The two `/v1` answers that are not `ApiError`

Neither goes through this module, and both are documented as such in
`docs/status/backend.md`:

- **`405 Method Not Allowed`** — axum's route table answers it when a path
  matches and the verb does not. There is no envelope.
- **`413 Payload Too Large`** — tower-http's `RequestBodyLimitLayer` answers it
  before any extractor runs.

Two more responses in the process are deliberately not envelopes:

- **`/healthz`** answers `503` with the bare text `database unreachable`. It is
  an infrastructure probe read by an orchestrator, not by an SDK, and it is the
  one route whose failure must not depend on this module working.
- **`POST /v1/oauth/token`** renders RFC 6749 §5.2's
  `{"error":…,"error_description":…}`, because every OAuth client parses that.

## The gate

`cargo xtask verify-errors` (part of `just verify`) refuses an unclassified
error type — any public type whose name ends in `Error` owes an
`impl Classify` — and refuses `anyhow` in a library crate. As of the
2026-09-11 run it reported 19 error types and 16 `#[from]` variants.

A corollary worth knowing: a **wire DTO** must not be named `…Error`, or the
gate will demand a `Classify` impl for something that is rendered rather than
returned. That is why `vpay_api::model::LastPaymentErrorObject` carries the
`…Object` suffix.
