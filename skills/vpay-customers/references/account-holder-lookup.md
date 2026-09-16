# `GET /v1/account_holders` — the three-way answer and three unbuilt controls

_Verified against vpay `93c6dfd0` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

"Whose mobile-money account is this number?" Built for issue #47: an integrator
whose refund flow lets a buyer nominate a _different_ number must match the
nominated account's registered name against the buyer's verified one.

Code: `backends/crates/vpay-api/src/v1/account_holders.rs`. Flow page:
`docs/flows/account-holder-lookup.md`.

**This is the only route on `/v1` that returns information about a person who
is not the caller.** Everything below follows from that.

## Since 2026-09-16 the route is not the only caller — and that moves three rules

`POST /v1/refunds` asks the same question about a nominated payee before it
instructs a transfer (`vpay_api::v1::refunds::verify_registered_holder`),
branching on `Capabilities::supports_account_holder_lookup` and never on a
rail code — so it runs on `mtn_momo` and is skipped on `orange_money`, where
RFC-0003 § 1 accepts the destination unverified.

**It goes through `account_holders::ask_rail`, not through the adapter.** It
reached the adapter directly until review on 2026-09-16, which broke two
stated properties at once: `vpay_account_holder_lookups_total` stopped being
*every* lookup vpay makes — under the very alarm the flow doc asks an operator
to set on a sustained `not_found` rate — and a refused payee left **no trace
at all**, because the refusal happens before any row, any attempt and any rail
instruction. `ask_rail` is now the single place that can break rules 3 and 4
below, which is the point of it. It carries a `caller` **`tracing` field**
(`v1_account_holders` / `v1_refunds_create`) and deliberately **not** a second
metric label: the counter stays at one label, and only a log needs the two
apart.

What follows for the rest of this page:

- **the four rules below now describe every lookup vpay makes**, not this
  route's;
- **the reserved decisions are reserved about _lookups_, not about a path.** A
  merchant holding `payments:write` can ask the same question through
  `POST /v1/refunds` and read the answer off the `400`, so a rate limit
  covering only this route would not be the control it reads as;
- **the refund caller never compares a name.** It branches on `Some`/`None`:
  registered means the transfer may be attempted, `Ok(None)` is a `400` naming
  `destination`, and an `Err` keeps its `502`/`500`. vpay holds no verified
  buyer name to compare against — matching names stays the merchant's job,
  which is what this route is for.

## The three-way answer, and why nothing may collapse it

| The port says      | `/v1` answers                                                                     | What it means                                          |
| ------------------ | --------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `Ok(Some(holder))` | `200`, `name: "…"`, `verified: true`                                              | the rail named a holder                                |
| `Ok(None)`         | `200`, `name: null`, `verified: false`                                            | **the rail answered and has no record of this number** |
| `Err(..)`          | the classified status — `502` unreachable, `500` misconfigured, `400` no such API | **nobody asked, or the rail could not answer**         |

The middle row is a fact about the **number**. The bottom row is a fact about
the **lookup**. Issue #47's caller refuses on both, but only one of them is the
buyer's to fix and only one is worth paging an operator about. **A rail that
could not be reached, reported as `Ok(None)`, tells an integrator that a real
person's real account does not exist.**

So: `Ok(None)` is documented on the port method as "the rail has no record,
never we could not ask"; the MTN adapter maps **exactly one** status (`404`) to
it; and `ask_rail` **matches** on the result rather than `?`-ing it — which is
also what lets the counter see the failure, for both callers.

Since 2026-09-16 the bottom row also decides whether a **refund** is refused.
`verify_registered_holder` turns `Ok(None)` into a `400` naming `destination`
and lets an `Err` keep its own class, so an unreachable rail refuses the
merchant's refund as a server error rather than telling them their payee does
not exist.

### The silent-total-failure mode to watch for on the first real call

Two MTN facts are unverified because **this endpoint** has never been called
against the real sandbox — the 2026-09-15 live run covered the token mint,
`requesttopay` and the status query and nothing else: the case of the
`accountHolderIdType` path segment, and whether MTN answers `404` for an
unknown holder at all (it documents `200`, `401` and `500` only).
**Wrong together they are silent.** A wrong path segment makes MTN's gateway
answer `404` to _every_ lookup, and `404 → Ok(None)` renders every one as
`{"name": null, "verified": false}` with HTTP `200`: a total misconfiguration
looks exactly like "nobody in Cameroon is registered", with nothing in
`vpay_account_holder_lookups_total` but a `not_found` rate of 1.0. The first
sandbox call should check a number known to be registered before it trusts a
`not_found`; an operator should alert on a sustained `not_found` rate near 1.0.
Reversing either decision costs one constant
(`vpay_adapter_mtn_momo::ACCOUNT_HOLDER_ID_TYPE`) or one match arm.

**Since 2026-09-16 that failure mode also refuses money.** The same
`Ok(None)` is what `POST /v1/refunds` turns into a `400 destination`, so a
mis-cased path segment would refuse **every** MTN refund with "the nominated
payee is not a registered account" — for payees that are all perfectly real,
and still with nothing but a `not_found` rate of 1.0 to see it by.

## Privacy rules that apply here and nowhere else

**1. A name, and nothing else.** MTN's `basicuserinfo` body is OIDC-shaped and
carries `given_name`, `family_name`, `birthdate`, `locale`, `gender`, `status`.
vpay reads two. The projection is `vpay_adapter_mtn_momo::wire::BasicUserInfo`,
which has **no field** for the rest, so serde drops them at the first point the
bytes become a Rust value and a field MTN adds tomorrow is dropped by the same
rule with no edit. `vpay_provider::AccountHolder` carries one private `String`
and a hand-written `Debug` that redacts even that, so a `{:?}` of a holder — or
of any `Result`/`Option` containing one — cannot print a name.

**2. Nothing is persisted.** Not the name, not the number, not the fact that
the question was asked. There is **no repository call in the module and no
migration behind it**, and none on the refund path's use of `ask_rail` either:
a refund refused for an unregistered payee is refused before any row is
written, so it too leaves nothing but the log line. Do not add one casually:
this is the same fact rule 4 points at from the other side — a merchant
enumerating the number space leaves no record in vpay.

**3. Logs carry a masked number and never a name.** One line per lookup, at
`info` (`warn` on a rail failure), carrying the rail's code, `+2376••••200`,
whether a holder was found and which caller asked. **The bullet count is fixed, not one per hidden
digit** — a mask whose length revealed the input's length would be a small
oracle for free. A gap, named: nothing writes `charges.payer_ref_masked` yet;
the confirm path stores `NULL`, and this route's mask is the first producer of
the shape in the workspace.

**4. No metric label carries the number, the name, the merchant, or even the
rail.** `vpay_account_holder_lookups_total{outcome}`, `outcome` one of
`found` / `not_found` / `unsupported` / `error`
(`vpay_core::metrics::account_holder_outcome`). A Prometheus label is retained,
queryable and shipped wherever the scrape goes. The four outcomes are not
derivable from the HTTP status — `found` and `not_found` are both `200`, which
is why the series exists at all. `unsupported` covers every refusal decided on
`payment_method_type`; `error` covers a **malformed `msisdn`** as well as rail
trouble, which is a merchant's mistake sitting in a label an operator reads as
an outage (a fifth `invalid_request` value is a reserved decision recorded on
the constant).

## The three controls issue #47 asked for — RESERVED, NOT BUILT

As of 2026-09-16 none of these exists, and none has a chosen default. They are
maintainer decisions; taking one quietly in either direction would be wrong.

**Rate limit — not built.** §3 asks for a per-merchant limit and says why:
"unlimited lookup of arbitrary MSISDNs is a name-harvesting oracle", and the
abuse is _exfiltration_ rather than load. Rate limiting in this deployment is an
ingress concern (`docs/flows/provider-port.md` records the same for the callback
route), and a per-merchant bucket inside one handler would be a control nothing
else on this surface has. The maintainer has to decide the shape (per merchant?
per merchant per MSISDN? a daily ceiling?), the enforcement point (ingress or
`/v1`) and what an exceeding merchant is told. Until then, the honest sentence:
**this route lets any merchant with a valid credential turn a list of phone
numbers into a list of names, at whatever rate they can make HTTP requests, and
vpay keeps no record that it happened.** That is the condition on turning the
route on, not a caveat about it.

**Widened 2026-09-16, and this is the part to carry into any design.** It is no
longer only this route. `POST /v1/refunds` asks the same question under
`payments:write`, and a `400` naming `destination` versus a `200` is a
registration oracle with no name attached but the same enumeration shape — and
it leaves no row either, since the refusal precedes every write. **A limit
scoped to `GET /v1/account_holders` would therefore be a control that looks
like one.** Scope the decision to lookups, at `ask_rail` or at ingress across
both paths.

**Audit log — not built, and it contradicts rule 2.** A per-merchant,
per-MSISDN trail _is_ a stored record of who asked about whom, which is
precisely the record rule 2 declines to keep. Which wins is a policy choice.
`MerchantScope` is already bound on the handler, so the key such a log would
need is in hand the day it is decided. Same widening as the rate limit: the
refund path makes lookups too, so a trail that covered only this route would
under-report its own subject.

**A dedicated scope (`identity:read`) — not built.** The route is served under
`payments:read` like every other `GET`. Adding one is a three-place change that
fails **silently** when the places disagree (see `SCOPE_PAYMENTS_WRITE`'s own
doc comment: the string an operator writes in a registration, the string the OP
mints, the string the middleware checks), and it would refuse every existing
merchant credential on the day it landed.

## The wire, and two more shapes that are decisions

```http
GET /v1/account_holders?msisdn=237600000200&payment_method_type=mtn_momo
```

```json
{ "object": "account_holder", "payment_method_type": "mtn_momo",
  "name": "David Mbarga", "verified": true }
```

Four keys, **always all four**. `name` is present-and-null when the rail has no
record, never omitted, because both SDKs model it as a required nullable field
and a dropped key is a decode failure in a merchant's client. `verified` is
`true` exactly when `name` is present — redundant on purpose, because it is
what an SDK branches on — and it is **not** a claim that anything was
cryptographically verified.

**There is no `livemode`, and that is a departure from every other object on
this surface.** Elsewhere it is read off the row; there is no row here and never
will be (rule 2), so the only available value would be the deployment's
configuration read at render time — the same field name carrying a weaker
guarantee. Recorded as a decision, not settled: if the maintainer wants it, it
is one line in `vpay_api::model::AccountHolderObject` plus a row in each SDK.

`msisdn` takes the three spellings `+2376XXXXXXXX`, `2376XXXXXXXX`, national
`6XXXXXXXX`; anything else is a `400` naming `msisdn`. `canonical_msisdn` is
**shared with `/v1/customers`** — the same function canonicalises a customer's
stored `phone` — and is deliberately **not** shared with
`frontends/apps/checkout/src/lib/msisdn.ts`: the browser's copy is a form
affordance, this one is a trust boundary.

## Capability, not provider code

`Capabilities::supports_account_holder_lookup` is what both callers branch on
(ADR-0002) — there is no `if code == "mtn_momo"` arm anywhere. `mtn_momo`
declares `true`; `orange_money` declares `false` and inherits
`ProviderError::Unsupported`, **not** a `NotImplemented` token, because
**nothing about Orange's _lookup_** is unbuilt work someone owes: the flag is a
permanent answer, and Orange's equivalent route is merely unconfirmed from this
repository. A rail that _does_ expose a lookup and has not written one must
declare `true` and override the method with its own token, so `verify-status`
sees the gap.

**That sentence used to read "nothing about Orange is unbuilt work someone
owes", and has been too broad since 2026-09-15.** Orange's `refund` is exactly
such work: `supports_refunds` flipped `false → true` and the adapter now
answers a declared `NotImplemented("orange_money::refund")` rather than
`Unsupported`, because RFC-0003 § 5 decided an Orange refund *is* a transfer
back. Two flags on one rail, two different shapes — see the `vpay-orange-money`
skill, and do not carry either correction across to the other.

The flag is deliberately **not persisted**: no column in `providers`, no field
on `vpay_db::ProviderSeed`. Nothing reads a capability out of that table.

`/v1` refuses an unknown rail, a _disabled_ rail and an incapable rail with a
**byte-identical `400`** — telling them apart would let a merchant enumerate
which rails a deployment has configured but switched off.

## The tenant is bound and unused, on purpose

The handler's signature is `_scope: MerchantScope`. Nothing is scoped by it,
because nothing is stored or read. It is bound so that the auth middleware must
have resolved a tenant (a failure there is a `500`, fail-closed) and so the
value an audit log would key on is in hand the day that decision is taken.
Deleting the parameter would silently remove both properties.
