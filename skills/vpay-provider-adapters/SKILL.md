---
name: vpay-provider-adapters
description: The vpay provider port (`vpay-provider`) and how to add a payment rail — the `ProviderAdapter` trait and its seven methods, the `Unsupported` vs `NotImplemented` rule that decides which error a missing operation returns, the rail-agnostic refund destination (`RefundDestination`, `RefundTarget`, `parse_destination`), the capability flags the core branches on instead of provider codes, the shared HTTP/token modules, and the shared conformance suite a new adapter must pass. Load this before writing or changing any `vpay-adapter-*` crate, before touching `backends/crates/vpay-provider`, and before adding a rail.
---

# The provider port, and adding a rail

> **Verified against vpay `0799a8d2` (2026-09-18).** Version-sensitive claims below
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

`refund`, `parse_destination` and `account_holder_name` all default to
`Err(ProviderError::Unsupported)` in the trait.

**The trap:** a rail whose API _does_ exist but which you have not written
must **override the default with its own `NotImplemented` token**. Inheriting
`Unsupported` would hide our gap as a fact about the rail, and
`verify-status` would never see it.

~~`vpay_adapter_mtn_momo::Adapter::refund` is the live example, and
`vpay-adapter-orange-money` does the opposite: it does not override `refund`
at all, because Orange documents no refund API for Web Payment.~~
**Corrected 2026-09-16 — both halves of that moved on 2026-09-15 (RFC-0003
§ 5), and the example is now the other rail:**

| Rail           | `supports_refunds` | `refund` answers                                       |
| -------------- | ------------------ | ------------------------------------------------------ |
| `mtn_momo`     | `true`             | a real `POST /disbursement/v1_0/transfer`              |
| `orange_money` | `true`             | `NotImplemented("orange_money::refund")` — the example |

**Neither shipping rail answers `Unsupported` for `refund` any more**, and a
skill, doc or test that says one does is stale. Orange's flag flipped because
an Orange refund _is_ an outbound transfer: the rail refunds, so `Unsupported`
would be a lie about Orange, and what is missing is vpay's work — this
repository holds no Orange transfer specification. The reason changed, not
just the value.

`cargo xtask verify-status` therefore reports **exactly one** token as of
2026-09-16 — Orange's — down from eight on 2026-09-03 and from **two** partway
through 2026-09-15. `orange_money::refund` is the same string that left the
list on 2026-09-03 meaning something else entirely (then: Orange has no refund
API; now: vpay has not written the transfer), so do not read a token's
reappearance as a regression. The gate also gained a **third direction** on
2026-09-15 — a token whose prefix names a shipping rail must live in **that
rail's crate** — because the two older directions compare token _strings_ and
a copy-paste between adapters was invisible to both. See
[references/adding-a-rail.md](references/adding-a-rail.md) § 8.

`Unsupported` is a 409 for the merchant but `Severity::Error` for us:
reaching that arm means the core skipped the `Capabilities` check it is
supposed to branch on first, and logging it at `Conflict`'s default `Info`
would bury our own bug among merchants' typos.

## The seven methods

**`parse_destination` was added on 2026-09-15 (RFC-0003 open question 4) and
`refund` grew a `destination` parameter the same day.** An adapter written
against the six-method shape does not compile against this port.

```rust
fn code(&self) -> &'static str;                   // == the payment_method_types value
fn capabilities(&self) -> Capabilities;
async fn submit(&self, charge, config)      -> Result<Submitted, ProviderError>;
async fn query_status(&self, charge, config)-> Result<ChargeStatus, ProviderError>;
fn parse_callback(&self, body: &[u8])       -> Result<CallbackRef, ProviderError>;
fn parse_destination(&self, raw: &serde_json::Map<String, Value>)
                                            -> Result<RefundTarget, ProviderError>; // default: Unsupported
async fn refund(&self, charge, amount, destination: Option<&RefundTarget>, config)
                                            -> Result<Refunded, _>;   // default: Unsupported
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
  `status` at all, so there is nowhere to put one. **`parse_destination` is
  synchronous for the same reason**, from 2026-09-15: it reads a merchant's
  parameters, and an adapter that could call a rail while "parsing" them
  would turn one refund request into an unbounded number of outbound calls.
  The two are the only pure functions in the error-surface table, and the
  blank `Config`/`Rejected`/`Transport` cells in their columns are as
  load-bearing as the ticks — there, those variants would not merely be
  unused, they would be untrue.
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
`supports_account_holder_lookup`, and — **new on 2026-09-15** —
`refund_destination: RefundDestination` (`Origin` | `Required`).

`Capabilities::is_coherent()` is `!supports_partial_refunds ||
supports_refunds`. It is **mirrored by a real DB CHECK** —
`partial_refunds_imply_refunds` in
`backends/migrations/0002_create-providers.sql`, proven to fire by
`partial_refunds_without_refunds_is_rejected_by_the_database` in
`backends/tests/integration/tests/postgres_smoke.rs`. Belt and braces, not a
substitute either way. `vpay_api::v1::boot::boot_seeds` also refuses an
incoherent pair at boot (`ConfigError::IncoherentCapabilities`, exit 78).

`supports_account_holder_lookup` and `refund_destination` are **deliberately
not persisted**: unlike the other four they have no column in `providers` and
no field on `vpay_db::ProviderSeed`. Nothing reads a capability out of that
table — `vpay_api` resolves an adapter in-process and asks it — so a column
would be a second copy of an answer the linked code already owns.

**`refund_destination` is deliberately NOT in `is_coherent`**, and the
obvious rule ("a rail that cannot refund must declare `Origin`") was
considered and refused: neither value means anything when refunds are off, so
the rule would be an arbitrary sentinel called coherence, and there is no
`providers` column to mirror it with — a Rust-only "coherence" rule with no
database half would be a different kind of thing wearing that method's name.
`a_refund_destination_is_inert_to_coherence` pins the decision and what would
have to change to reverse it. Read the field **only when `supports_refunds`
is true**: on a rail that cannot refund the value is inert, no code path
reaches it, and it must still be a truthful statement about that rail's refund
product — because it is the value that governs the moment `supports_refunds`
flips, and Orange's flipping on 2026-09-15 is not hypothetical.

## The refund destination — `RefundDestination`, `RefundTarget`, `parse_destination`

New on 2026-09-15 (RFC-0003 § 1 and open question 4). A refund on a
mobile-money rail is an **outbound transfer**, so it needs a payee; a refund
on a card or wallet goes back to the instrument that paid. The port carries
that difference as a capability, never as a rail code.

| `refund_destination` | The core, before it calls `refund`                                                                                           | `destination` argument |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| `Required`           | refuses a request **without** a payee                                                                                        | always `Some`          |
| `Origin`             | refuses a request **carrying** one (accepting and dropping it would tell a merchant their nominated payee had been honoured) | always `None`          |

Both rails vpay carries declare `Required` as of 2026-09-16, so the `Origin`
arm ships with **no live example** — `no_shipping_rail_returns_a_refund_to_the_paying_instrument`
in the conformance suite is what stops that premise changing quietly, and its
assertion message lists the three things owed before it may be relaxed.

**The adapter parses the wire shape, not the core.** The merchant sends
`destination[<rail_code>][…]`; the core strips its own envelope — the outer
key is a `code()`, which is core knowledge by definition — and hands the
**rail-scoped inner map** to `parse_destination`. Symmetric with
`parse_callback`, and that symmetry is the whole point: a bank-account
upstream tomorrow is a new adapter method body and **zero core change**. Both
shipping adapters read the key `msisdn`.

**The adapter owns the key; the port owns the number.**
`RefundTarget::mobile_money` is **fallible and canonicalising** (maintainer's
decision, 2026-09-15 — it was infallible before), the field is private and
that constructor is the only way in, so an invalid destination cannot be
built and no adapter re-spells the rule. It **requires a leading `+`**,
unlike `GET /v1/account_holders`, which still takes the bare national form.
The full rules — the `+` asymmetry, why a refusal may never name the number,
and why `parse_destination`'s `Malformed` must be **translated** by its
caller rather than forwarded — are in
[references/adding-a-rail.md](references/adding-a-rail.md) § "A `Required`
rail".

**If the core's invariant is ever broken — a `Required` rail handed `None` —
answer `ProviderError::Config`.** Settled 2026-09-15 by
`vpay_adapter_mtn_momo::Adapter::refund`, the first adapter here to make a
real transfer call (RFC-0003 open question 6). Not `Rejected`, which blames a
rail nobody asked; not `Malformed`, which is about an answer that does not
exist; not `Unsupported` or `NotImplemented`, which on a rail that refunds
are both lies. `Config` is the closest true sentence the enum offers, and
what matters is its classification — it stops the poll ladder, it pages, and
it never reaches a payer as a decline.

## An `Ok` from `refund` is an acceptance. It is **not** a settlement

Read this before writing anything that turns a `Refunded` into a stored
status. `Refunded` has **no status field** and the trait has **no refund
status read** — there is no `query_refund_status` to pair with `query_status`.
So the strongest thing an adapter can mean by `Ok` is _the rail took the
instruction_, and on the one rail that implements it that is literally so:
MTN's Disbursements `transfer` answers `202 ACCEPTED` with an empty body and
its outcome is read back from
`GET /disbursement/v1_0/transfer/{referenceId}`, **a call vpay does not
make**.

A caller that writes `refunds.status = 'succeeded'` on an `Ok` is telling a
merchant money moved on the strength of a response that did not say so.
`pending` is what an `Ok` supports, and `pending` is what `POST /v1/refunds`
writes. RFC-0003 open question 8 stays open because closing it needs a refund
poll ladder that does not exist. **No rail has ever returned money to
anyone**, and `mtn_momo::refund` is WireMock-proven and rail-unproven: no real
MTN Disbursements credential exists in this project, so a deployment reaching
that call today gets `ProviderError::Config` naming the credential it lacks.

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

`ProviderConfig`, `AccountHolder` and `RefundTarget` all have hand-written
redacting `Debug` impls. So do both adapters' credential and wire types, and
so does `vpay_api::v1::refunds`' `CreateParams` — which holds a payee's number
in a raw `serde_json::Map` that has none of its own. Keep it that way.

## Where to go next

- [references/adding-a-rail.md](references/adding-a-rail.md) — the full
  checklist, including two corrections to `docs/flows/provider-port.md` you
  must not follow literally, and what a `Required` rail owes.
- [references/conformance.md](references/conformance.md) — the shared suite,
  the mapping contract a new rail's stubs must satisfy, and the techniques.
- `vpay-mtn-momo` skill — the push rail in detail.
- `vpay-orange-money` skill — the redirect rail in detail.
- `docs/reference/rails.md`, `docs/flows/provider-port.md`,
  `docs/adr/0002-provider-port.md`, `docs/adr/0011-error-modelling.md`.
