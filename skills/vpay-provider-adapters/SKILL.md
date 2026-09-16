---
name: vpay-provider-adapters
description: The vpay provider port (`vpay-provider`) and how to add a payment rail — the `ProviderAdapter` trait and its six methods, the `Unsupported` vs `NotImplemented` rule that decides which error a missing operation returns, the capability flags the core branches on instead of provider codes, the shared HTTP/token modules, and the shared conformance suite a new adapter must pass. Load this before writing or changing any `vpay-adapter-*` crate, before touching `backends/crates/vpay-provider`, and before adding a rail.
---

# The provider port, and adding a rail

> **Verified against vpay `f063ee96` (2026-09-15).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

One trait, `vpay_provider::ProviderAdapter`, in
`backends/crates/vpay-provider/src/lib.rs`. The core owns the payment
lifecycle, ledger, reconciliation and failure taxonomy; an adapter owns
exactly one rail's wire protocol (ADR-0002).

**`if provider == "mtn_momo"` outside a `vpay-adapter-*` crate is a defect,
and nothing greps for it.** The structural guard is that the port is only
ever held as `Box<dyn ProviderAdapter>` — which is also why the trait is
`#[async_trait]` rather than using native `async fn`, since that is not
dyn-safe. The core branches on capability _values_, never on a code.

## The headline: `Unsupported` vs `NotImplemented`

Two ways for an operation to be absent, and they are not interchangeable.
Getting this wrong is the single most likely mistake in an adapter.

|                     | `ProviderError::Unsupported`                              | `ProviderError::NotImplemented("crate::fn")`                              |
| ------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------- |
| Means               | **The rail has no such API, permanently**                 | **Unbuilt work on our side**                                              |
| Whose fact          | the rail's                                                | ours                                                                      |
| Capability flag     | `false`                                                   | stays **`true`** — the rail can do it                                     |
| How to produce it   | do **not** override the trait method; inherit the default | **override** the method and return your own token                         |
| Category / severity | `Conflict` (409), severity overridden to `Error`          | `NotImplemented`                                                          |
| Seen by             | nothing — it is a settled answer                          | `cargo xtask verify-status`, which fails if it is not in `docs/status.md` |

`refund` and `account_holder_name` both default to
`Err(ProviderError::Unsupported)` in the trait.

**The trap:** a rail whose API _does_ exist but which you have not written
must **override the default with its own `NotImplemented` token**. Inheriting
`Unsupported` would hide our gap as a fact about the rail, and
`verify-status` would never see it. `vpay_adapter_mtn_momo::Adapter::refund`
is the live example — it returns
`ProviderError::NotImplemented("mtn_momo::refund")` while
`supports_refunds` stays `true`, because MTN _does_ refund (Disbursements
product) and we have not built it. `vpay-adapter-orange-money` does the
opposite: it does not override `refund` at all, with a comment where the impl
would be, because Orange documents no refund API for Web Payment.

`Unsupported` is a 409 for the merchant but `Severity::Error` for us:
reaching that arm means the core skipped the `Capabilities` check it is
supposed to branch on first, and logging it at `Conflict`'s default `Info`
would bury our own bug among merchants' typos.

## The six methods

```rust
fn code(&self) -> &'static str;                   // == the payment_method_types value
fn capabilities(&self) -> Capabilities;
async fn submit(&self, charge, config)      -> Result<Submitted, ProviderError>;
async fn query_status(&self, charge, config)-> Result<ChargeStatus, ProviderError>;
fn parse_callback(&self, body: &[u8])       -> Result<CallbackRef, ProviderError>;
async fn refund(&self, charge, amount, config)     -> Result<Refunded, _>;   // default: Unsupported
async fn account_holder_name(&self, msisdn, config)-> Result<Option<AccountHolder>, _>; // default: Unsupported
```

- **The error-surface table in the trait's own doc comment is the contract.**
  It is a markdown grid, one row per `ProviderError` variant and one column
  per method. An adapter may raise fewer variants than a row allows; it may
  **not** raise a variant the table does not give it. There is deliberately
  no `ProviderError::retryable()` — retry policy is `Classify::retry` and a
  second oracle beside it is what ADR-0011 exists to prevent.
- **`parse_callback` is synchronous so that it cannot make a network call.**
  A callback is a hint; an adapter that could fetch something while
  "parsing" one could smuggle a status out of an unauthenticated request. It
  returns identifiers only — returning a status is a design error, and both
  shipping adapters' callback wire types have **no field** for the rail's
  `status` at all, so there is nowhere to put one.
- **`submit` must report a duplicate as `Submitted`, never as an error.**
  That is what makes same-reference retry safe after a crash.
- **`query_status` must remain callable indefinitely**, long after any payer
  prompt expired. A rail with no record is `ChargeStatus::NotFound`, which is
  **not** an error and **not** a decline.
- Only `ProviderError::Rejected` says the money did not move. `Transport` and
  `Malformed` leave the charge's fate **unknown**, which is the distinction
  the whole port is arranged around.

## Capabilities

`flow` (`Push` | `Redirect`), `supports_refunds`,
`supports_partial_refunds`, `delivers_callbacks`, `requires_ip_allowlist`,
`supports_account_holder_lookup`.

`Capabilities::is_coherent()` is `!supports_partial_refunds ||
supports_refunds`. It is **mirrored by a real DB CHECK** —
`partial_refunds_imply_refunds` in
`backends/migrations/0002_create-providers.sql`, proven to fire by
`partial_refunds_without_refunds_is_rejected_by_the_database` in
`backends/tests/integration/tests/postgres_smoke.rs`. Belt and braces, not a
substitute either way. `vpay_api::v1::boot::boot_seeds` also refuses an
incoherent pair at boot (`ConfigError::IncoherentCapabilities`, exit 78).

`supports_account_holder_lookup` is **deliberately not persisted**: unlike
the other four it has no column in `providers` and no field on
`vpay_db::ProviderSeed`. Nothing reads a capability out of that table —
`vpay_api` resolves an adapter in-process and asks it — so a column would be
a second copy of an answer the linked code already owns.

## What the port owns, and an adapter must not re-implement

- **`vpay_provider::http`** — `client()` / `client_with_timeouts()`. Vendored
  Mozilla roots via `webpki_roots`; **`reqwest::Client::new()` _panics_ in
  the `FROM scratch` runtime image** (ADR-0004), so this is the only correct
  client. It also _removes_ two reqwest defaults: redirects are **not
  followed** (a 3xx arrives intact and must be refused as `Malformed`, never
  taken — taking it would replay your credentials at whatever `Location`
  named) and proxy environment variables are ignored. `read_rail_body`
  bounds every response at `MAX_RAIL_BODY_BYTES` (256 KiB);
  `Response::text()`/`bytes()` read to end of stream and let the peer decide
  how much memory a worker allocates. `path_segment()` percent-encodes
  anything interpolated into a URL path.
- **`vpay_provider::token`** — `fingerprint(&[&str])` (length-prefixed
  SHA-256; hash the **secret** halves too, so rotating only a secret evicts
  the cache on the next call), `usable_until()`, `CachedToken` with a
  redacting `Debug`. The refresh margin is **per-rail** and passed in.
- **`vpay_provider::measured::Measured`** — the counter and histogram. It is
  applied **once**, in `vpay_api::v1::boot::adapters_by_code`, the single
  funnel every rail call passes through. Do not instrument inside an adapter.
  One increment per **port call**, not per HTTP request; `parse_callback` is
  not counted.

Adapters take the process's one `reqwest::Client` by clone. Neither shipping
adapter derives `Default` — `Default` would have to invent a client.

Every deployment gets `DEFAULT_CONNECT_TIMEOUT` (5 s) and
`DEFAULT_REQUEST_TIMEOUT` (20 s), carried on `ProviderConfig` rather than on
the client because one client is shared by every rail. **Apply
`config.request_timeout` per request.** See the timeout lesson in
[references/conformance.md](references/conformance.md) — Orange did not, and
the suite said it was fine.

`ProviderConfig` and `AccountHolder` both have hand-written redacting `Debug`
impls. So do both adapters' credential and wire types. Keep it that way.

## Where to go next

- [references/adding-a-rail.md](references/adding-a-rail.md) — the full
  checklist, including two corrections to `docs/flows/provider-port.md` you
  must not follow literally.
- [references/conformance.md](references/conformance.md) — the shared suite,
  the mapping contract a new rail's stubs must satisfy, and the techniques.
- `vpay-mtn-momo` skill — the push rail in detail.
- `vpay-orange-money` skill — the redirect rail in detail.
- `docs/reference/rails.md`, `docs/flows/provider-port.md`,
  `docs/adr/0002-provider-port.md`, `docs/adr/0011-error-modelling.md`.
