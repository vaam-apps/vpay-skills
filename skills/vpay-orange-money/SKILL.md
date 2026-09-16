---
name: vpay-orange-money
description: "The Orange Money Cameroun adapter (`vpay-adapter-orange-money`) — the Web Payment redirect rail, and the loudest caveat in the repository: it has never been called. Covers the token endpoint living at the host root while payments carry the environment in the path, why `submit` sends the same value as `return_url` and `cancel_url`, the `payment_url` validation, why a missing `pay_token` is `Config` and never `NotFound`, the fail-closed callback nobody verifies, and why `refund` is a declared `NotImplemented` token since 2026-09-15 rather than the `Unsupported` it answered before. Load before changing anything under `backends/crates/vpay-adapter-orange-money` or its WireMock mappings."
---

# Orange Money — the redirect rail

> **Verified against vpay `93c6dfd0` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

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
- **Refunds are a third case and the sharpest one**: the rail has never been
  called *and* the transfer it would be called with has no specification here
  at all. See `refund` below before writing a line of it.

The full "still unverified" list is in
[references/unverified.md](references/unverified.md). Read it before changing
anything in this crate.

## Capabilities

`flow: Redirect`, **`supports_refunds: true` since 2026-09-15** (it was
`false`), `supports_partial_refunds: false`, `delivers_callbacks: true`,
`requires_ip_allowlist: false`, `supports_account_holder_lookup: false`,
**`refund_destination: RefundDestination::Required`**.

`supports_partial_refunds: false` is *decided*, not left over from the flip.
`Capabilities::is_coherent` and migration `0002`'s
`partial_refunds_imply_refunds` both permit `true` now, and permitted is not
decided: `supports_refunds` rests on one known thing — that an Orange refund
is a transfer — and **nothing else about Orange transfers is known here**: no
endpoint, no body, no amount semantics, no minimum, no limit. So neither value
is a true statement about Orange, and the tie-break is merchant-visible
direction: `false → true` is additive, while withdrawing a `true` merchants
had integrated against is a breaking change made on a guess.

**The core reads that flag since 2026-09-16 and did not before**, because
`POST /v1/refunds` was unrouted: `vpay_api::v1::refunds::resolve_amount`
refuses anything but the intent's whole amount on this rail with a `400`
naming `amount`, on the capability and never on a provider code (ADR-0002),
pinned by `a_rail_without_partial_refunds_refuses_a_partial_amount`.

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

## `refund` is a declared `NotImplemented` token — and the MEANING inverted

```rust
Err(ProviderError::NotImplemented("orange_money::refund"))
```

~~`refund` is deliberately not overridden: the port's default is
`Err(ProviderError::Unsupported)`, the permanent answer for a rail with no
refund API. `supports_refunds: false`. There is no `orange_money::*` token in
`docs/status.md` and there must not be one.~~ **Corrected 2026-09-15 (RFC-0003
§ 5).** Every clause of that is now false, and the *reason* inverted rather
than the value alone:

| until 2026-09-15                               | since 2026-09-15                                            |
| ---------------------------------------------- | ----------------------------------------------------------- |
| `supports_refunds: false`                      | **`true`**                                                   |
| the port's default, `Unsupported`              | an override answering `NotImplemented("orange_money::refund")` |
| a fact about **Orange** — "this rail has no refund API" | an admission about **vpay** — work someone owes       |
| no `orange_money::*` token in `docs/status.md` | **the one token `verify-status` counts**                     |

The maintainer decided what an Orange refund *is*: an outbound **transfer**
back to a payee. Orange makes transfers, so the rail can refund and
`Unsupported` — which the core branches on as a permanent capability answer —
would now be a lie about Orange rather than an admission about us. A rail that
will never support an operation must not be described with the same token as
work someone still owes.

`orange_money::refund` is the **single** `NotImplemented` token in the
workspace as of 2026-09-16 — not two, not zero — and it is the same token
meaning a different thing than it did on 2026-09-03, when it left this list
for the opposite reason. `cargo xtask verify-status` fails in both directions,
and since 2026-09-16 in a third: a token whose prefix names a rail this
workspace ships an adapter for must be carried by that rail's crate.

### Why there is no wire call, and what unblocks it

**No Orange transfer API is documented anywhere in this repository — not even
reconstructed — and none was invented.** The three calls the adapter does make
came from Orange Developer's public overview plus community SDKs that agree
with each other; for transfers no such source exists here, so an endpoint path
and a request body would be *invented*, in the money path, on a rail nobody
has ever called. That is the failure mode `CLAUDE.md` names first.

**What unblocks the token is item 5 of
`docs/flows/adapter-orange-money.md`'s "To confirm with Orange Cameroun"
list** — rewritten 2026-09-15 from "Refund/disbursement availability" to "the
transfer product: which one, on what endpoint, with what request body, under
which credential, and with what amount rules". RFC-0003 § 5 settled the
*availability*; what is open is the **specification**, and it is now the one
item on that list that blocks a shipping token. It is not answerable from
anything in this repository, which is the point of leaving it there rather
than guessing. More reading does not close it.

`destination` arrives `Some` here (the rail declares `Required`) and is
**ignored rather than read**: there is no call to put it in, and reading it
would be the first half of a pretence.

### `parse_destination` IS built, on a rail whose `refund` is a token

`destination[orange_money][msisdn]`, parsed by this adapter — `POST /v1/refunds`
calls it since 2026-09-16 and then answers the merchant the token. The two
answer different questions and only one needs Orange's transfer spec: *who* a
refund is addressed to is vpay's own merchant-facing parameter (RFC-0003 § 1)
and is fully known; *how* an Orange transfer body would carry that payee is
not known here at all.

`DESTINATION_MSISDN_KEY` is the only place in the crate that spells the key,
and it is **not** a field of Orange's API — there is no documented Orange
transfer body here to have taken a name from. A missing key, a non-string
value and a blank string are `Malformed` (blank is absent: a blank value is a
lost one, not a choice). The *number* is validated by
`RefundTarget::mobile_money`, not here — the adapter owns the key, the port
owns the number — so a bare `600000200` is refused here while
`GET /v1/account_holders` accepts it, and **no message ever contains the
value**.

`Required` rests on the same sentence `supports_refunds` is true by: there is
no "back the way it came" on a redirect rail where `payer_ref` is `None` and
vpay never learns who paid, so a refund here is addressed or it is nowhere.
It was declared `Required` *while* `supports_refunds` was still `false`, on
purpose, so the flip would be one line about refunds and not also a fresh
guess about destinations.

### `supports_account_holder_lookup: false` is the shape `refund` USED to have

This one is still a permanent capability answer: "Orange's Web Payment product
documents no account-holder lookup", not "we have not written one". The
adapter overrides nothing and inherits `Unsupported`. Orange has a
KYC/customer product elsewhere; its route is unconfirmed from this repository
(item 8 of the same "to confirm" list) and belongs there, not in a `true`
nobody can honour. **Do not carry the refund correction across to this flag.**
One consequence for the money path: `vpay_api::v1::refunds` skips
`verify_registered_holder` entirely on this rail, so **an Orange refund
destination is accepted unverified** (RFC-0003 § 1).

### What the conformance suite actually asserts now

`a_rail_without_the_refund_capability_answers_unsupported` still exists, but
**its `Unsupported` arm no longer runs on any rail** — no rail declares the
capability off. It is kept so a rail added or flipped tomorrow is checked
rather than silently skipped (and because live pages cite it by name), and the
property it exercised moved to
`a_rail_with_no_refund_api_takes_the_default_and_answers_unsupported` in
`vpay-provider`, on a stub that overrides nothing. **The name now describes
the dead arm.**

Live on `orange_money` is the *token* arm of `claims_refunds`: an unbuilt
refund on a rail advertising refunds must answer
`NotImplemented("<its own code>::refund")` — never `Unsupported`, never `Ok`,
and never a copy-pasted token naming another rail, since `verify-status`
compares strings and cannot tell which adapter answered. The five
account-holder cases still assert the `Unsupported` inheritance for the
lookup.

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
  `NotImplemented` rule (which this rail is now the workspace's only live
  example of), the conformance suite.
- `vpay-mtn-momo` skill — the other rail's refund went the **opposite** way on
  the same day: a real Disbursements `transfer`, WireMock-proven and never
  called against MTN.
- `docs/flows/adapter-orange-money.md`, `docs/reference/rails.md`,
  `docs/rfc/0003-refunds-destinations-and-the-first-ledger-postings.md`.
