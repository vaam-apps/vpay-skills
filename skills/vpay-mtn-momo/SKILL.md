---
name: vpay-mtn-momo
description: The MTN MoMo Cameroon adapter (`vpay-adapter-mtn-momo`) — a push rail, and the only rail vpay has ever called for real. Covers the Collections token mint and the JSON-grant-body fix a real sandbox run found, the pure response-outcome functions (409 means accepted, 404 means NotFound), the wire-type serde and redaction rules, and the Disbursements `transfer` that `refund` became on 2026-09-15 — written, WireMock-proven and never once called against MTN. Load before changing anything under `backends/crates/vpay-adapter-mtn-momo` or its WireMock mappings.
---

# MTN MoMo — the push rail

> **Verified against vpay `0799a8d2` (2026-09-18).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`backends/crates/vpay-adapter-mtn-momo/` — `lib.rs` (transport + trait impl),
`token.rs` (auth), `wire.rs` (MTN's JSON), `mapping.rs` (MTN's vocabulary →
the core taxonomy).

Push: the payer is prompted on their own handset, **we** supply the
reference (`X-Reference-Id`), and status stays queryable by that reference
indefinitely. Both are what the poll ladder is built on.

**Status as of 2026-09-16**: the one rail vpay has ever called for real, and
only on its **charge path**. A EUR PaymentIntent was created, confirmed and
settled against **MTN's sandbox** on 2026-09-15. No production rail, no real
payer, and MTN's sandbox rejects XAF — EUR is load-bearing in
`config/application-live.yml`. That run exercised the token mint,
`requesttopay` and the status query, **and nothing else**: not
`basicuserinfo`, and not the Disbursements `transfer` `refund` became on the
same day. See `docs/status/verification/2026-09-15.md` and
`docs/runbooks/live-sandbox-test.md`.

## Two products, two subscription keys, two token scopes

`token.rs`'s `Product` is the value that makes this a type rather than a
convention. It has been in `docs/flows/adapter-mtn-momo.md` § "Credential
hierarchy" since Step 3 and **nothing acted on it until Disbursements was
built on 2026-09-15**.

| `Product`       | path segment   | `credentials.*` / `settings.*`                                                   | what it serves                                   |
| --------------- | -------------- | -------------------------------------------------------------------------------- | ------------------------------------------------ |
| `Collections`   | `collection`   | `subscription_key`, `api_key`, `api_user`                                        | `requesttopay`, the status read, `basicuserinfo` |
| `Disbursements` | `disbursement` | `disbursement_subscription_key`, `disbursement_api_key`, `disbursement_api_user` | `transfer` — the refund, the only money-out call |

**Both path segments are singular and neither matches the product's English
name**; a plural in either is a 404 from the gateway. `target_environment` is
the one setting the two products share.

**A missing Disbursements key does not fall back to the Collections one**,
deliberately: the failure that convenience creates is silent and lands on the
money-out path — Collections' Basic credentials sent to the Disbursements
token mint produce a 401 that reads as "MTN refused our partner credentials",
pages, and names nothing that is actually wrong.

**Two mechanisms keep a Collections bearer off a `transfer`, and the review of
2026-09-15 corrected which is primary.** The primary one is structural:
`Adapter` holds two named cache fields and `Adapter::slot` matches on the
product. The product discriminator in `Credentials::fingerprint` is **defence
in depth** for the map-or-single-slot design this adapter nearly took.
Removing either one alone leaves the whole crate and all 67 conformance cases
green — one unit test each is the whole of the evidence
(`a_collections_bearer_is_never_served_to_a_disbursement`,
`a_products_bearer_is_stored_where_only_that_product_can_read_it`), and
**neither mechanism is reachable from the conformance suite**, which
configures different keys per product so the copy-pasted-configuration case is
never constructed.

## The token mint — the bug a WireMock suite could not see

`POST {base}/{product}/token/`, HTTP Basic `api_user:api_key`, plus
`Ocp-Apim-Subscription-Key` and `X-Target-Environment`, **plus a JSON body**:

```rust
const CLIENT_CREDENTIALS_JSON: &str = r#"{"grant_type":"client_credentials"}"#;
// .header(CONTENT_TYPE, "application/json").body(CLIENT_CREDENTIALS_JSON)
```

**The body is load-bearing and so is its content type.** Measured against the
real sandbox on 2026-09-15 (commit `f063ee96`, PR #177), on **Collections**;
the Disbursements mint is assumed to behave the same way and has never been
called:

| what was sent                                               | what MTN's gateway answered                                                   |
| ----------------------------------------------------------- | ----------------------------------------------------------------------------- |
| no body at all                                              | **HTTP 411 Length Required**                                                  |
| `grant_type=client_credentials` form-encoded                | **HTTP 200, an HTML "Request Rejected" page** — not JSON, not an error status |
| `{"grant_type":"client_credentials"}` as `application/json` | a token                                                                       |

The form-encoded spelling is the one **`vpay-adapter-orange-money` uses**.
The two rails must NOT be assumed interchangeable. (An HTTP/2-only theory was
tested and disproven; the ALPN change made under it was reverted.)

~~The stub pins this: `wiremock/mtn/mappings/token.json` carries
`"bodyPatterns": [{ "contains": "client_credentials" }]`, so a bodyless or
form-encoded mint cannot match the stub.~~ **Corrected 2026-09-16: that
matcher did not hold it.** `grant_type=client_credentials` — the exact
form-encoded spelling PR #177 fixed — _also_ contains the substring, so the
regression would have gone green. Since 2026-09-15 the mapping is

```json
"bodyPatterns": [{ "equalToJson": { "grant_type": "client_credentials" } }],
"headers": { "Content-Type": { "contains": "application/json" } }
```

on **both** minting mappings, Collections' and Disbursements' — `equalToJson`
refuses a form body because a form body is not JSON, and the header matcher
refuses it a second time. (The third mapping, the one that answers 401 to the
`bad-key` subscription key, matches on that header alone.) Proven by
mutation: swap the adapter's `.body(CLIENT_CREDENTIALS_JSON)` for `.form(..)`
and every wire case fails.

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

## `refund` — written since 2026-09-15, and **never once called against MTN**

~~`refund` is `Err(ProviderError::NotImplemented("mtn_momo::refund"))`, the
workspace's only remaining token.~~ **Corrected 2026-09-15 (RFC-0003 § 5):**
it is a real `POST {base}/disbursement/v1_0/transfer`. The token is retired
and `verify-status` no longer counts it. Grep any page that still says
otherwise — the adapter's own module header was the most-read of the stale
copies and was fixed on review.

**Both of the next two paragraphs belong on any page that mentions this.
Either one alone is wrong.**

**The call is real.** A Disbursements subscription key, a separately scoped
bearer from `POST /disbursement/token/`, `X-Reference-Id` and the body's
`externalId` both set to `charge.reference_id`, and
`wire::Transfer { amount, currency, externalId, payee: { partyIdType:
"MSISDN", partyId } }` — `RequestToPay` with **`payee` in place of `payer`**,
a separate struct on purpose: one struct with a runtime `#[serde(rename)]`
would put the two products' bodies one boolean apart, and a `payer` on a
transfer is money leaving to the wrong party. `amount` is a decimal string on
both products. **No `X-Callback-Url`** — `parse_callback` reads a _charge_
reference, so a Disbursements notification would find no charge and be
`Malformed` on every delivery; the header goes in when a refund poll job
exists to pull forward.

**The rail has never answered it. Nothing in this repository has ever called
MTN's Disbursements product — not in production, not against the sandbox, not
once — and no REAL Disbursements credential exists in the project.** Every
deployment leaves `config/application.yml`'s three keys empty except the
e2e/demo compose stack, whose values are stubs aimed at a `wiremock/wiremock`
container (since 2026-09-16, so the SDKs' live refund suites reach the `202`).
So every assertion about this method — seven conformance cases and thirteen
unit tests, per `docs/status.md` read 2026-09-16 — is against a stub written
from MTN's own documentation, and **a body, header set or status table
faithful to that documentation but not to the rail would pass all of it**.
Writing "MTN refunds work" is this repository's cardinal sin;
`docs/status.md` § "`mtn_momo::refund` is written, WireMock-proven and
rail-unproven" is the long form.

**One of those seven is weaker than its name.**
`a_rail_without_the_refund_capability_answers_unsupported` describes an arm
that **no longer runs on any rail** — since 2026-09-15 no rail declares
`supports_refunds: false`. On MTN what is left is the weak half: whatever a
refunding rail answers, it is not "this rail cannot refund". The name is kept
on purpose, because live pages and a dated verification record cite it and
renaming would orphan a record that must not be rewritten. Do not cite it as
evidence that a refund works; the case that carries that weight is
`a_refund_on_a_rail_that_refunds_reaches_the_rail_and_is_accepted`, and it
reaches a WireMock container, not MTN. The `vpay-provider-adapters` skill's
conformance page has the full accounting of which case proves what.

Those three keys are also **three new `${VAR}` names** every environment
loading `config/application.yml` must define — the list went seven to ten on
2026-09-15. Empty is fine; **absent is an unresolved placeholder and exit 78
on both binaries**, before any key check runs.

### `refund_outcome(status, body)` — the money-out table

Pure, like `submit_outcome`. Put new behaviour here, not in the request
builder.

| status        | outcome                                                                                     |
| ------------- | ------------------------------------------------------------------------------------------- |
| `202`         | `Ok(Refunded { ref_extra: empty, fee: None })` — **accepted, not settled**                  |
| **`409`**     | **`Ok`**, same value — the rail already has this reference                                  |
| `400`         | `Rejected` with `mapping::failure_code(code)` — Collections' vocabulary                     |
| `401` / `403` | `Rejected{ProviderAccountBlocked}` — the **Disbursements** credentials                      |
| `500`         | `mapping::internal_error(..)` — the same `CONFIGURATION_CODES` table                        |
| other `5xx`   | `Transport`                                                                                 |
| `404`         | `Config` — likeliest symptom of a base URL with no Disbursements product                    |
| `3xx`         | `Malformed` — never followed; a hop would replay the key, bearer **and the payee's number** |

Three things about that table are worth not re-deriving.

**A 202 is `Ok` and means the rail took the instruction, not that the payee
has the money.** `transfer` is asynchronous exactly as `requesttopay` is, its
outcome is read from `GET /disbursement/v1_0/transfer/{referenceId}`, and this
adapter does not make that call because `ProviderAdapter` has no
`query_refund_status` and `Refunded` has no status field. A write path that
marked a refund `succeeded` on an `Ok` from here would assert something no MTN
response has said.

**409 → `Ok` is safe only under the reference invariant below.** It says "this
transfer was already instructed", which is what a crash retry needs to hear.

**The failure vocabulary is Collections' `ErrorReason`, reused as a stated
assumption.** MTN publishes no schema document for the Disbursement API at all
(re-checked 2026-09-11), so there is nothing to compare a Disbursements table
against.

### The reference the transfer carries — an invariant the core owes

`X-Reference-Id` and `externalId` are both `charge.reference_id`, and **that
is only correct if the core hands this method the refund's own
`provider_reference_id`** (RFC-0003 § 3 step 2; `refunds.provider_reference_id`,
migration `0017`). The port cannot express the difference — `refund` takes a
`ChargeRef`, which carries one reference — so one of the two has to be the
refund's, and it is this one. Hand it the _charge's_ reference instead and the
failure is silent: a second partial refund reuses a reference MTN has seen,
gets `409 RESOURCE_ALREADY_EXIST`, and is reported **accepted** — a refund the
merchant is told happened and for which no money moved.
`the_transfer_is_addressed_by_the_reference_the_core_supplied` pins what the
method does.

### The destination — `Required`, and a `None` is `Config`

`capabilities().refund_destination` is `RefundDestination::Required` (and
`supports_partial_refunds` is `true` here, unlike Orange). MTN has no "send it
back the way it came": the collection and the disbursement are different
products with different subscription keys.

`parse_destination` reads `destination[mtn_momo][msisdn]` and owns **only that
key** — `DESTINATION_MSISDN_KEY`, the one place in the workspace that spells
it. It is vpay's merchant-facing parameter (RFC-0003 § 1), not a field of
MTN's API, which is why it need not match `payee.partyId`. A missing key, a
non-string value (a JSON number is **refused, not coerced** — a leading `+` or
`0` does not survive one) and a blank string are each `Malformed`. The _number_
is validated by `RefundTarget::mobile_money`, not here: the adapter owns the
key, the port owns the number, and an adapter cannot construct an invalid
`RefundTarget` at all. So a bare `600000200` is refused here while
`GET /v1/account_holders` accepts it. **No message ever contains the value.**

A `None` destination is `ProviderError::Config` — RFC-0003 open question 6,
left to "the first adapter to make a real transfer call" and decided here. Not
`Rejected` (blames a rail nobody asked), not `Malformed` (there is no answer),
and not `Unsupported`/`NotImplemented`, which are both lies now.

### `fee: None`, and nothing settles a pending refund

MTN's documented transfer response has no fee field and its `202` carries an
empty body, so `Refunded::fee` is `None` — "the rail did not report a fee",
where `Some(zero)` would say "the rail said it was free". Collapsing them is
the exact defect issue #46 was filed about, and whether Disbursements reports
a fee at all is unverified, because the product has never been called.

It could not be written down anyway: there is **no refund poll ladder**
(RFC-0003 open question 8), so a transfer this rail **accepts** leaves the
refund `pending` indefinitely — `invoices.amount_refunded` never moves and
`refunds.fee` is written by nothing. Only a refusal moves a refund, and it
moves it to `failed`; `succeeded` is reached by
`Settlement::apply_refund_succeeded` alone and nothing a merchant can do
reaches it. **Do not write a page that narrates a refund reaching a payee.**

## See also

- [references/failure-mapping.md](references/failure-mapping.md) — the three
  vocabularies row by row, and what each is evidenced by.
- [references/account-holder-lookup.md](references/account-holder-lookup.md)
  — real code, never called against the real rail, two assumptions that fail
  quietly together, and the **second caller it gained on 2026-09-16**: the
  refund path refuses an unregistered payee through the same port method.
- `vpay-provider-adapters` skill — the port and the conformance suite.
- `docs/flows/adapter-mtn-momo.md`, `docs/reference/rails.md`,
  `docs/flows/failures.md`.
