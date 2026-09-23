# Errors: typed at the leaves, composed per layer, classified once

_Verified against vpay `b747e5d5` (2026-09-23). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Decision record: `docs/adr/0011-error-modelling.md`. Readable version:
`docs/flows/errors.md`. Gate: `cargo xtask verify-errors`.

## The invariant

> Every error is classified exactly once, by the crate that raises it, and
> every boundary — HTTP envelope, worker retry, process exit code, log line —
> is derived from that classification, never re-decided.

Two errors of the same kind therefore always get the same status, the same
retry policy and the same severity, whichever handler or job they surface
through.

## Read this before you touch a composite

**Delegation is wholesale, not `category()`-only.** This is the single
mistake the shape is designed to prevent, and the reason is concrete
(`backends/crates/vpay-worker/src/error.rs`, module header):

> Forwarding `category()` alone would silently drop a leaf's deliberate
> override — `ProviderError::Rejected` overrides `retry()` to
> `Retry::NewAttempt` while its category (`Category::Conflict`) defaults to
> `Retry::Never`, so a category-only delegation would **dead-letter a declined
> charge** instead of letting the intent's own state machine decide.

So a composite delegates in **every** `Classify` method — `category`, `code`,
`retry`, `severity`, `public_message` — not just the first one.

## The three tiers

```text
 rail / Postgres / YAML / caller input
          │
 TIER 1 — leaf errors, one thiserror enum per crate concern
   MoneyError · LedgerError · ConfigError · DbError · ProviderError
   RailFailure · AuthRejection · UnknownCurrency
   each: impl vpay_core::error::Classify { fn category(&self) -> Category }
          │ #[from] / #[source]
 TIER 2 — composite errors, one per layer
   vpay_api::ApiError (HTTP)   → Classify delegates
   vpay_worker::JobError (jobs) → Classify delegates
          │ consumed, never re-classified
 TIER 3 — boundaries
   IntoResponse → the Stripe envelope (status, type, code, message)
   JobError::decision → RetryAfter{alert} / Terminal / DeadLetter
   main(): ExitCode from the anyhow chain (.context at each step)
   tracing level from Severity
```

`anyhow` lives only in the bottom box, and only in `backends/apps/*`. A
library crate that lists `anyhow` under `[dependencies]` fails the gate.

## `Category` is the whole policy table

`vpay_core::error::Category`. Everything else is derived from it unless a leaf
overrides one column **with a comment saying why**.

| Category         | Whose problem    | HTTP | Stripe `type`           | default `code`           | Retry         | Severity | Exit |
| ---------------- | ---------------- | ---- | ----------------------- | ------------------------ | ------------- | -------- | ---- |
| `InvalidRequest` | caller           | 400  | `invalid_request_error` | `invalid_request`        | never         | info     | 64   |
| `Authentication` | caller           | 401  | `authentication_error`  | `invalid_token`          | never         | info     | 77   |
| `Forbidden`      | caller           | 403  | `invalid_request_error` | `forbidden`              | never         | info     | 77   |
| `NotFound`       | caller           | 404  | `invalid_request_error` | `resource_missing`       | never         | info     | 1    |
| `Conflict`       | caller (state)   | 409  | `invalid_request_error` | `invalid_state`          | never         | info     | 1    |
| `Idempotency`    | caller           | 400  | `idempotency_error`     | `idempotency_key_in_use` | never         | info     | 64   |
| `RateLimited`    | caller (pace)    | 429  | `rate_limit_error`      | `rate_limit`             | after backoff | warn     | 1    |
| `Rail`           | the rail         | 502  | `api_error`             | `provider_unavailable`   | after backoff | warn     | 69   |
| `Storage`        | us (Postgres)    | 503  | `api_error`             | `service_unavailable`    | after backoff | error    | 69   |
| `Configuration`  | operator         | 500  | `api_error`             | `misconfigured`          | never         | error    | 78   |
| `NotImplemented` | us (honest stub) | 501  | `api_error`             | `not_implemented`        | never         | error    | 1    |
| `Internal`       | us (a bug)       | 500  | `api_error`             | `internal_error`         | never         | **page** | 1    |

**Do not edit that table in `docs/flows/errors.md`.** It is transcribed
literally into a `vpay-core` test, so the document and the code fail together.

`Retry` has a third value, `NewAttempt`, for operations that must not be
repeated as-is but may be started over (a failed charge: retry means a new
`PaymentIntent`). No category defaults to it; a leaf sets it explicitly.

Exit codes follow `sysexits.h` where one fits: `EX_CONFIG` 78, `EX_UNAVAILABLE`
69, `EX_USAGE` 64, `EX_NOPERM` 77.

**There is deliberately no `ProviderError::retryable()`.** Retry policy is
`Classify::retry` and a second oracle beside it is exactly what ADR-0011
exists to prevent. The worker reads `Classify` exclusively.

## The canonical example

`backends/crates/vpay-worker/src/error.rs`:

```rust
#[derive(Debug, thiserror::Error)]
pub enum JobError {
    #[error(transparent)] Db(#[from] DbError),
    #[error(transparent)] Provider(#[from] ProviderError),
    #[error(transparent)] Money(#[from] MoneyError),
    #[error(transparent)] Ledger(#[from] LedgerError),
    Poisoned { /* … */ },
    Exhausted { job_id: Uuid, attempts: u32 },
}

impl Classify for JobError {
    fn category(&self) -> Category {
        match self {
            Self::Db(e)       => e.category(),   // named, never `_ =>`
            Self::Provider(e) => e.category(),
            Self::Money(e)    => e.category(),
            Self::Ledger(e)   => e.category(),
            // An inconsistent job row is an invariant this code was supposed
            // to guarantee when it wrote the row. `Internal` is the only
            // category that pages.
            Self::Poisoned { .. } => Category::Internal,
            // `Rail`, not `Internal` or `Conflict`: nothing is broken on our
            // side and no caller did anything wrong — the rail never answered.
            Self::Exhausted { .. } => Category::Rail,
        }
    }
    fn code(&self)     -> &'static str { /* same shape */ }
    fn retry(&self)    -> Retry        { /* same shape */ }
    fn severity(&self) -> Severity     { /* same shape */ }
}
```

Every `#[from]` variant is named in **every** method that matches on `self`.
Every non-delegating arm carries a comment explaining the override. That is
the whole pattern.

## What `verify-errors` actually refuses

1. A `pub` type in `backends/crates` that derives `thiserror::Error` **or** is
   named `*Error` / `*Rejection`, with no `impl Classify` in the same crate.
   The scan skips `#[cfg(test)]` blocks and `tests/` directories, and the impl
   must itself be outside test code. `backends/apps` and the SDKs are outside
   the scan by design.
2. A library crate listing `anyhow` under `[dependencies]`.
3. **A `#[from]` variant a composite answers for with a wildcard instead of
   naming.** For every `#[from]` variant, each `Classify` method that
   _discriminates_ on `self` must name `Self::<Variant>` explicitly.

**Five spellings count as discriminating** — `match self`, `match *self`,
`match &self`, `if let Self::`, `matches!(self`. All five are checked because
searching only for `match self` made the rule **opt-out**: an `if let` ladder's
trailing `else` answers for an unnamed leaf exactly as a `_ =>` arm does. The
hole was proven shut live, by deleting `ApiError`'s `Self::Db(e) => e.code()`
arm and watching the gate refuse, and is held by
`a_from_variant_swallowed_by_an_if_let_ladder_is_reported`, which fails if
anyone narrows the list back.

## Attach a cause; never `format!` it in

Before Step 7, `ProviderError::Transport` and `Malformed` were
`Transport(String)`, so every adapter flattened `reqwest`'s error with
`format!`. `reqwest`'s own `Display` for a timeout is _"error sending request
for url (…)"_ — **the word _timeout_ is one link further down the chain** — so
MTN's log line named the URL and not the fault. Orange had noticed and
hand-walked `Error::source()` into a `String`, which is the same information
rebuilt by hand in one of the two adapters.

Both are now struct variants carrying a `context: String` and an optional
`#[source] RailFailure`:

```rust
ProviderError::transport_from("mtn_momo: the request to the rail failed", error)
```

`Display` renders `context` alone, so the body cap and the rail's name stay in
the one-line message; the chain is rendered once, at the boundary that logs it,
through `vpay_core::error::source_chain` (`ApiError::log`'s `source_chain`
field, and `vpay_worker`'s `jobs.last_error`). What an operator now sees for a
timeout is `sending the request: error sending request for url (…): operation
timed out`.

`a_transport_failures_source_chain_reaches_the_reqwest_error` in
`vpay-adapter-mtn-momo` **fails if anyone goes back to `format!`.**

Four constructors, and choosing between them is the whole decision:

|           | the rail answered (badly), no library error to attach | there is a library error |
| --------- | ----------------------------------------------------- | ------------------------ |
| transport | `transport`                                           | `transport_from`         |
| malformed | `malformed`                                           | `malformed_from`         |

All four carry a doctest asserting `Display` renders `context` alone while the
cause stays reachable through `Error::source()` — the property that would
silently die the day someone reverts.

One deliberate exception: a `serde_json` parse failure is **not** attached as a
source. Its own text is the whole diagnostic and belongs in `context`, where a
one-line log shows it.

## Three port rows worth knowing without reading the trait doc

Which variant each `ProviderAdapter` operation may raise is a table in the
port's own rustdoc, and `#![warn(clippy::missing_errors_doc)]` on
`vpay-provider` makes a method that loses its `# Errors` section fail
`cargo clippy -- -D warnings`.

- `parse_callback` raises `Malformed` and nothing else. It touches no network,
  holds no credential and reads no configuration.
- `query_status` raises `Rejected` **only** when the rail refuses _our_ partner
  credentials. A declined charge is `Ok(ChargeStatus::Failed)` and a rail with
  no record is `Ok(ChargeStatus::NotFound)` — neither is an error.
- `submit` never raises `Unsupported`: a rail that cannot take a payment is not
  a rail. ~~`Unsupported` belongs to `refund` alone, where it is a permanent
  capability answer rather than unbuilt work.~~ **Corrected 2026-09-16:** the
  port's table gives `Unsupported` to **three** operations — `refund`,
  `parse_destination` and `account_holder_name` — and **no shipping rail
  answers it for `refund` any more.** Orange flipped `supports_refunds` to
  `true` on 2026-09-15 (RFC-0003 § 5: an Orange refund is an outbound transfer,
  so the refusal stopped being a fact about the rail) and now answers a declared
  `NotImplemented("orange_money::refund")`; MTN's Disbursements call is written.
  The distinction the old sentence was teaching is still the right one —
  `Unsupported` is a permanent capability answer, `NotImplemented` is work vpay
  owes — it just no longer has a live example on `refund`. The remaining live
  `Unsupported` is on `account_holder_name`, which Orange inherits.

`ProviderError::Rejected` is the seam between system errors and business
outcomes. A rail declining a charge is not a system failure.

## Boundaries

**HTTP.** Handlers return `Result<_, ApiError>` and use `?`. They never
construct an envelope, choose a status, or format a message for a merchant.
The two envelope renderers are `pub(crate)` to `vpay-api`, so a handler
_cannot_ build one by hand — one renderer is structural here, not a
convention. The full `Display` + source chain goes to the log; only
`public_message()` goes to the merchant.

**Worker.** Jobs return `Result<_, JobError>`; the loop calls `decision()` and
logs at `severity()`. The loop does not inspect variants.

**Binaries.** `main` returns `ExitCode` and wraps an
`async fn run() -> anyhow::Result<()>` in which every fallible startup step
gets `.context("what we were doing")`. On `Err`, `main` prints the full chain
to **stderr** with `eprintln!("vpay-server: {error:#}")` — `tracing` may not
be initialised yet when configuration fails — then finds the first
classifiable leaf and exits with `category().exit_code()`, falling back to
`Internal`/`1`.

~~…finds the first classifiable leaf with `find_in_chain::<ConfigError>`
first, then `DbError`…~~ **Corrected 2026-09-23:** `exit_code_for` in
`backends/apps/vpay-server/src/main.rs` tries **four** leaves, in this order:
`StartupError` (a flag or knob the binary was not given, defined in the binary
itself), `ConfigError`, `SigningKeyError`, then `DbError` — last, because a
config naming a dead database is still a config problem. It has had that shape
since Step 1 (2026-09-02); the two-type version was copied from
`docs/flows/errors.md` § Boundaries, which still says it as of vpay `b747e5d5`.
Trust the function, and add a new startup leaf **there**: `find_in_chain` is
typed, so anything it does not name falls through to exit `1`.

## How to add an error

1. Add the variant to the crate's existing enum, or a new `thiserror` enum if
   it is a new concern. Keep `#[source]` on wrapped errors.
2. Classify it: extend the crate's `impl Classify`. If the category's defaults
   are wrong for it, override `code`/`retry`/`severity`/`public_message`
   **with a comment saying why**.
3. If a composite must carry it, add a `#[from]` variant there and delegate in
   **every** method that discriminates on `self`.
4. `just verify`.
5. Test the classification **if it overrides a default** — the override is the
   decision worth pinning.

Two properties to preserve when you add a variant that carries a payload: a
foreign object and a missing object must stay byte-identical
(`a_foreign_object_and_a_missing_object_are_byte_identical`), and an
idempotency key is never echoed past an 8-character hint, in the log only
(`an_idempotency_key_is_never_echoed_past_its_hint`).

## One decision left open

`Category::Idempotency` carries both `idempotency_key_in_use` (a key replayed
with a different body) and `idempotency_key_in_flight` (a key whose first
request has not finished). Both are `400`/`idempotency_error` because the
status comes from the category — **Stripe answers `409` for the second**, which
this policy table cannot express without splitting the category. That is an
ADR-level change and is deliberately left as a maintainer decision;
`ApiError::IdempotencyKeyInFlight`'s doc comment records it. Do not "fix" it in
passing.
