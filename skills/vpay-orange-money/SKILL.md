---
name: vpay-orange-money
description: The Orange Money Cameroun adapter (`vpay-adapter-orange-money`) — the Web Payment redirect rail, and the loudest caveat in the repository: it has never been called. Covers the token endpoint living at the host root while payments carry the environment in the path, why `submit` sends the same value as `return_url` and `cancel_url`, the `payment_url` validation, why a missing `pay_token` is `Config` and never `NotFound`, the fail-closed callback nobody verifies, and why `refund` is `Unsupported` rather than unbuilt. Load before changing anything under `backends/crates/vpay-adapter-orange-money` or its WireMock mappings.
---

# Orange Money — the redirect rail

`backends/crates/vpay-adapter-orange-money/` — `lib.rs`, `token.rs`,
`wire.rs`, `mapping.rs`.

## Read this before you believe anything else on this page

**Orange has never been called.** As of 2026-09-16, no request from this
repository has ever reached Orange Money. Every proof is a WireMock container
answering the way `docs/flows/adapter-orange-money.md` says a rail answers —
**and that flow doc is itself reconstructed from Orange Developer's public
overview and community SDKs, not from a vendor specification.** The
error-body shapes in particular are inferred.

`docs/status.md` puts it as "⛔ `orange_money` never called";
`docs/status/verification/2026-09-15.md` repeats it after the MTN sandbox run.

Two consequences:

- **A stub that answers the way we guessed is not evidence.** The MTN
  sandbox run found two real bugs the WireMock suite could not see, because
  the stub accepted the request the adapter actually sent. The same class of
  bug is sitting in this crate unobserved.
- **Do not describe this rail's behaviour in the present tense** in code,
  docs or commit messages. Say what the stub does, and say what is assumed.
  The existing comments do; keep it that way.

The full "still unverified" list is in
[references/unverified.md](references/unverified.md). Read it before changing
anything in this crate.

## Capabilities

`flow: Redirect`, `supports_refunds: false`, `supports_partial_refunds:
false`, `delivers_callbacks: true`, `requires_ip_allowlist: false`,
`supports_account_holder_lookup: false`.

## `token_url` — OAuth at the host root, payments under a path

```
base_url          = https://api.orange.com/orange-money-webpay/dev
token endpoint    = https://api.orange.com/oauth/v2/token
payment endpoints = {base_url}/v1/webpayment, {base_url}/v1/transactionstatus
```

Orange puts the environment in the **path** of the payment API but serves
OAuth from the **host root**, so the two URLs cannot both be derived by
appending. `token_url(base)` does `Url::join("/oauth/v2/token")` — scheme,
host, port and userinfo kept; path, query and fragment replaced.

**Deriving both from one configured value, rather than adding a second YAML
key, is what stops a deployment pointing its payments at one host and its
tokens at another.** A `base` that is not an absolute URL is
`ProviderError::Config` — a mistake in `providers[].host.url`, exit 78.

The grant is `grant_type=client_credentials`, **form-encoded**, written out
as a `&'static str` rather than using reqwest's `form` feature (which would
add `serde_urlencoded` to every binary to encode eleven constant bytes).
**MTN's token endpoint rejects this spelling** — see the `vpay-mtn-momo`
skill. Do not "unify" the two.

`EXPIRY_MARGIN` 60 s. A missing `expires_in` is **not** an error and means
"use it once, do not cache it", not an invented default. The fingerprint
hashes `client_id` **and** `client_secret`, so rotating only the secret
evicts the bearer on the next call.

## `submit` — `POST {base_url}/v1/webpayment`

`WebPaymentRequest`: `merchant_key`, `currency`, `order_id`, `amount`,
`return_url`, `cancel_url`, `notif_url`, `lang`. `order_id` is our own
`reference_id` rendered. `amount` is a JSON **number**
(`Money::to_provider_minor`); MTN's equivalent field is a decimal string.
`currency` is **the amount's own**, not `ProviderConfig::currency` — if a
route ever hands this rail another currency, the rail refusing it is the
correct outcome.

### `return_url` and `cancel_url` get the same value, deliberately

Both carry `ChargeRef::return_url` verbatim. The core fills that field; **the
adapter never invents it**, and a charge with none is `ProviderError::Config`
— a redirect rail must be told where the payer goes next.

Orange's page distinguishes the two ("paid" vs "the payer gave up") and
**vpay cannot yet tell those apart on the way back**: the outcome comes from
the authenticated status query, not from which link the payer clicked, and a
charge the payer abandoned is `Pending` until it expires. Sending two
different URLs would be a claim this system does not check.

This is a fix, not an oversight. Until 2026-09-04 both were read from
_deployment_ settings falling back to the notification endpoint — one answer
per deployment to a per-charge question, so a merchant's `return_url` was
stored on `charges`, echoed back on `next_action`, and **never sent to the
rail that would act on it**. The two settings keys are gone; nothing shipped
set them.

`lang` is **the only defaulted field in the body** (`DEFAULT_LANG = "fr"`).
Defaulted rather than required because it selects wording on Orange's page
and nothing else; refusing a payment over a missing display language would be
the worse failure.

### The `payment_url` is validated before it can reach a browser

`mapping::checked_redirect_url` refuses anything that is not `http://` or
`https://` and anything over 2048 characters (the `charges.redirect_url`
column limit). **A refusal never quotes the URL.**

Every field of the 2xx body is `Option` even though the flow doc shows all
three present, so a missing `pay_token` produces a precise `Malformed`
**naming the field** rather than a serde error naming a line and column of a
body we must never log.

## A missing `pay_token` is `Config`, never `NotFound`

`query_status` POSTs `{order_id, amount, pay_token}` to
`/v1/transactionstatus`. All three are required by the rail: **`order_id`
alone does not authorise the read**, which is why `pay_token` must be
committed before the payer could act (`docs/flows/crash-safety.md`).

```
no pay_token in ref_extra  ->  ProviderError::Config   (NOT ChargeStatus::NotFound)
```

The difference decides whether a payer's money is looked for. `NotFound` is
the rail saying nothing has happened yet; a charge whose `pay_token` we lost
is the opposite case — the rail may well have settled it — and it is a case
for a human, not for the poll ladder.

The rest of the ladder, the status table and the callback are in
[references/the-three-calls.md](references/the-three-calls.md).

## `parse_callback` fails closed — and nothing checks `notif_token`

Requires `order_id` (parsing as a UUID we could have generated) **and** a
non-blank `notif_token`; either missing is `Malformed`. `CallbackBody` has
**no `status` field** — a callback is a hint, and the cheapest way to
guarantee an adapter cannot leak a status out of an unauthenticated request
is to give it nowhere to put one.

**The `notif_token` is never compared to anything.** The adapter holds no
state, so it cannot — and since Step 8 (2026-09-04) **the callback route does
not either**: `vpay_api::provider_callback` **discards the returned
`ref_extra`** rather than merging unverified rail material onto the charge.
The comparison is unbuilt and `ref_extra` repair from a callback is
unavailable. The adapter's fail-closed check is therefore **load-bearing in
production, not only in tests**: it is the only thing between an
unauthenticated POST and a queued poll.

## `refund` is not overridden at all

There is a comment where the impl would be:

```rust
// `refund` is deliberately not overridden: the port's default is
// `Err(ProviderError::Unsupported)`, which is the permanent answer for a
// rail with no refund API. See the module doc.
```

Orange documents no refund API for Web Payment. `Unsupported` is a
**permanent capability answer** the core branches on via
`supports_refunds: false` — **not** `NotImplemented`, because there is
nothing to build and nothing anyone owes. There is no `orange_money::*` token
in `docs/status.md`, and there must not be one.

`supports_account_holder_lookup: false` is the same shape: "Orange's Web
Payment product documents no account-holder lookup", not "we have not written
one". Orange has a KYC/customer product elsewhere; its route is unconfirmed
from this repository and belongs on the flow doc's "to confirm" list, not in
a `true` nobody can honour.

Both are asserted behaviourally by the conformance suite
(`a_rail_without_the_refund_capability_answers_unsupported` and the five
account-holder cases), so "Orange has no such API" is checked, not skipped.

## The hosted page and the payer window are the STUB's

`wiremock/orange/mappings/stub-hosted-page.json` serves
`/stub-hosted-page/{pay_token}` with a Pay link and a Cancel link so a
browser can finish the redirect leg, and since Step 9 a browser does. **The
real rail stores `return_url` and `cancel_url` against the `pay_token` at
submit and renders them from its own state; the stub does not.** The
four-rung `PENDING` chain the stub serves, and the four conformance cases
over it, are facts about the stub and not measurements of Orange — see
[references/unverified.md](references/unverified.md).

## See also

- `vpay-provider-adapters` skill — the port, the `Unsupported` vs
  `NotImplemented` rule, the conformance suite.
- `docs/flows/adapter-orange-money.md`, `docs/reference/rails.md`.
