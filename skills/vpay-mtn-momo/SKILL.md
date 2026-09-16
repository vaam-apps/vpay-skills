---
name: vpay-mtn-momo
description: The MTN MoMo Cameroon adapter (`vpay-adapter-mtn-momo`) — a push rail, and the only rail vpay has ever called for real. Covers the Collections token mint and the JSON-grant-body fix a real sandbox run found, the pure response-outcome functions (409 means accepted, 404 means NotFound), the wire-type serde and redaction rules, and why `refund` is the workspace's only remaining NotImplemented token. Load before changing anything under `backends/crates/vpay-adapter-mtn-momo` or its WireMock mappings.
---

# MTN MoMo — the push rail

> **Verified against vpay `93c6dfd0` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`backends/crates/vpay-adapter-mtn-momo/` — `lib.rs` (transport + trait impl),
`token.rs` (auth), `wire.rs` (MTN's JSON), `mapping.rs` (MTN's vocabulary →
the core taxonomy).

Push: the payer is prompted on their own handset, **we** supply the
reference (`X-Reference-Id`), and status stays queryable by that reference
indefinitely. Both are what the poll ladder is built on.

**Status as of 2026-09-15**: the one rail vpay has ever called for real. A
EUR PaymentIntent was created, confirmed and settled against **MTN's
sandbox**. No production rail, no real payer, and MTN's sandbox rejects XAF —
EUR is load-bearing in `config/application-live.yml`. See
`docs/status/verification/2026-09-15.md` and
`docs/runbooks/live-sandbox-test.md`.

## The token mint — the bug a WireMock suite could not see

`POST {base}/collection/token/`, HTTP Basic `api_user:api_key`, plus
`Ocp-Apim-Subscription-Key` and `X-Target-Environment`, **plus a JSON body**:

```rust
const CLIENT_CREDENTIALS_JSON: &str = r#"{"grant_type":"client_credentials"}"#;
// .header(CONTENT_TYPE, "application/json").body(CLIENT_CREDENTIALS_JSON)
```

**The body is load-bearing and so is its content type.** Measured against the
real sandbox on 2026-09-15 (commit `f063ee96`, PR #177):

| what was sent                                               | what MTN's gateway answered                                                   |
| ----------------------------------------------------------- | ----------------------------------------------------------------------------- |
| no body at all                                              | **HTTP 411 Length Required**                                                  |
| `grant_type=client_credentials` form-encoded                | **HTTP 200, an HTML "Request Rejected" page** — not JSON, not an error status |
| `{"grant_type":"client_credentials"}` as `application/json` | a token                                                                       |

The form-encoded spelling is the one **`vpay-adapter-orange-money` uses**.
The two rails must NOT be assumed interchangeable. (An HTTP/2-only theory was
tested and disproven; the ALPN change made under it was reverted.)

The stub now pins this: `wiremock/mtn/mappings/token.json` carries
`"bodyPatterns": [{ "contains": "client_credentials" }]`, so a bodyless or
form-encoded mint cannot match the stub and cannot go green again.

Token facts worth not re-deriving: `REFRESH_MARGIN` 60 s;
`ASSUMED_LIFETIME` 300 s when MTN omits `expires_in`; `expires_in` parses as
a number **or** a quoted string; `minted_at` is read **before the send**, so
the round trip is not credited to the token's life; the fingerprint hashes
`[subscription_key, api_key, api_user]`, with `api_key` in **because** it is
the password — leaving it out meant a key-only rotation kept serving the
bearer minted from the old one. `send_authorized` re-mints **once** on a 401,
then treats a second 401 as `Rejected{ProviderAccountBlocked}`: the
credentials are wrong, not stale, and that pages.

## Read the response with a pure function

`submit_outcome`, `status_outcome` and `account_holder_outcome` are free
functions of `(StatusCode, &str)`. That is the part that has to be right: it
is proven row by row without a network, and the conformance suite then proves
the same rows arrive over a real socket. **Put new behaviour in these
functions, not in the request builder.**

### `submit` — `POST /collection/v1_0/requesttopay`

| status        | outcome                                                            |
| ------------- | ------------------------------------------------------------------ |
| `202`         | `Ok(Submitted)`                                                    |
| **`409`**     | **`Ok(Submitted)`** — the rail already has this reference          |
| `400`         | `Rejected` with `mapping::failure_code(code)`                      |
| `401` / `403` | `Rejected{ProviderAccountBlocked}` — ours, and it pages            |
| `500`         | `mapping::internal_error(..)` — **never a blind retry**, see below |
| other `5xx`   | `Transport`                                                        |
| `404`         | `Config("check base_url")` — the endpoint, not the charge          |
| `3xx`         | `Malformed` — redirects are never followed                         |

**409 → `Ok` is the line that makes same-reference retry safe.**
`X-Reference-Id` is a UUID written to the database _before_ the network call
(`docs/flows/crash-safety.md`), so a worker that crashed mid-submit re-sends
the identical request, MTN answers 409, and the adapter reports the charge as
submitted. Make 409 an error and you have turned crash recovery into a
duplicate charge or a stuck one.

Headers on every submit: bearer, subscription key, target environment,
`X-Reference-Id`, `X-Callback-Url`, and `.timeout(config.request_timeout)`.
The callback header is a **contract the stub matches on** — delete it and the
mapping stops matching, WireMock answers 404, and the adapter raises
`Config("check base_url")`.

### `query_status` — `GET /collection/v1_0/requesttopay/{reference_id}`

`200` → `PENDING` | `SUCCESSFUL` (+ `financialTransactionId`) | `FAILED` (+
mapped reason). An undocumented status string is `Malformed`, never guessed.

**`404` → `ChargeStatus::NotFound`, never `Failed`.** The whole recovery
story rests on that line: `NotFound` tells the reconciler nothing has
happened yet, and a push rail can say it about a charge it is about to
accept. `Failed` would close a charge that may still be alive.
`vpay_worker::recovery_step` needs _three_ consecutive `NotFound`s over ≥60 s
before concluding the rail never received the submit.

### `parse_callback`

**This request is not authenticated in any way.** MTN signs nothing and sends
no shared secret; the only thing between the body and the open internet is
that the callback host is one MTN was told about. That is why the port
returns identifiers and not a status — the most an attacker gains is causing
us to ask MTN, over an authenticated channel, about a charge of ours.
`wire::CallbackBody` has **no `status` field**; do not add one. It prefers
`referenceId` (the rail's copy), falls back to `externalId` (ours, set to the
same UUID by `RequestToPay`), and is `Malformed` if neither parses — failing
closed.

## The 500s that are our own misconfiguration

`mapping::CONFIGURATION_CODES` — `INVALID_CURRENCY`,
`NOT_ALLOWED_TARGET_ENVIRONMENT`, `INVALID_CALLBACK_URL_HOST` — arrive with
**HTTP 500** and are permanent until a human edits configuration. That is the
reason a 500 is never blindly retried here: `internal_error()` maps these to
`ProviderError::Config`, which stops the poll ladder, and everything else on
a 500 to `Transport`. A retry loop against them is an outage that looks like
a flake. All three vocabularies, row by row and with what each is evidenced
by, are in [references/failure-mapping.md](references/failure-mapping.md).

## `wire.rs` — three rules

**Never `#[serde(rename_all = "snake_case")]` in that file.** That attribute
is the workspace convention for types modelling _vpay's own_ wire, so a field
added as `payTo` fails review. These model **MTN's** wire, which is camelCase,
and the per-field `#[serde(rename = "…")]` attributes are what make each one
exact. A blanket `rename_all` would be a no-op masked by those renames on the
fields that have one, and a **silent wire break** on the fields that do not.
Same for `token.rs`'s `TokenResponse`: a rail whose casing coincides with ours
today is not a promise about tomorrow.

**Liberal in, exact out.** A rail that starts quoting a number must not turn
a settled payment into a parse error the poll ladder retries forever (hence
the untagged `Scalar` and `Reason` enums); a request body that drifts from
the documented shape is a charge the rail refuses (hence the exact-equality
assertion in `the_request_body_has_the_documented_shape`). `amount` is a
decimal **string** here; Orange takes the same amount as a JSON number.

**`BasicUserInfo` redacts each name half separately.** It keeps two of MTN's
six fields, so `birthdate`, `locale`, `gender` and `status` have no home and
serde drops them where bytes first become a Rust value. Its `Debug` is
hand-written: it was derived until 2026-09-06, and a mutation showed the cost
— one `tracing::debug!(?parsed)` put a third party's name in an operator's
log and the conformance privacy case did **not** fail, because it grepped for
the joined `"Amina Nkeng"` and the derived `Debug` printed the two halves,
never adjacent. The field _names_ are still printed, or the "four dropped
fields have no home" test would pass vacuously.

## `refund` — the workspace's only remaining `NotImplemented` token

```rust
Err(ProviderError::NotImplemented("mtn_momo::refund"))
```

MTN refunds are the **Disbursements** product: a different subscription key,
a separately-scoped token, and a `transfer` call this adapter does not make.
No deployment holds those credentials.

**`supports_refunds` stays `true` on purpose.** The _rail_ refunds; it is we
who have not built it. `Unsupported` would be a lie about MTN, and
`verify-status` would stop seeing the gap. Declared in `docs/status.md`;
`cargo xtask verify-status` fails in both directions. `POST /v1/refunds` is
**not routed**, so no caller can provoke it today.

`vpay_provider::Refunded::fee` exists and **no adapter populates it**. `None`
means "the rail did not report a fee"; `Some(zero)` means "the movement was
free". Collapsing them is the exact defect issue #46 was filed about.

## See also

- [references/failure-mapping.md](references/failure-mapping.md) — the three
  vocabularies row by row, and what each is evidenced by.
- [references/account-holder-lookup.md](references/account-holder-lookup.md)
  — real code, never called against the real rail, and two assumptions that
  fail quietly together.
- `vpay-provider-adapters` skill — the port and the conformance suite.
- `docs/flows/adapter-mtn-momo.md`, `docs/reference/rails.md`,
  `docs/flows/failures.md`.
