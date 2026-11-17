# `GET /v1/account_holders` — the three-way answer and three provisional controls

_Verified against vpay `b747e5d5` (2026-09-23). § "The three controls" and
every 2026-11-17 note were re-read at `c3358352`, after vpay#404 (issue #56
and #152) merged — on an older `master` the controls do not exist.
Version-sensitive claims carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

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
_every_ lookup vpay makes — under the very alarm the flow doc asks an operator
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
- **the controls govern _lookups_, not a path.** A merchant can ask the same
  question through `POST /v1/refunds` and read the answer off the `400`, so
  since 2026-11-17 the scope, the rate-limit budget and the audit journal all
  sit in `ask_rail` and the refund path, not on this route alone;
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

**2. The name and the number are never stored — but since 2026-11-17 the
question is.** ~~Nothing is persisted. Not the name, not the number, not the
fact that the question was asked.~~ _(True until vpay#404, issue #56.)_
Every lookup, from either caller, now writes two `audit_events` rows (see §
"The three controls" below): `initiated` before the rail is asked and a
terminal outcome after, carrying an HMAC of `tenant:canonical MSISDN` — never
the raw number, never the name. The tension this rule used to name — an audit
trail _is_ a record of who asked about whom — was resolved by keying the
target, on a **provisional** answer to RFC-0002 D3.

**3. Logs carry a masked number and never a name.** One line per lookup, at
`info` (`warn` on a rail failure), carrying the rail's code, `+2376••••200`,
whether a holder was found and which caller asked. **The bullet count is fixed, not one per hidden
digit** — a mask whose length revealed the input's length would be a small
oracle for free. ~~A gap, named: nothing writes `charges.payer_ref_masked` yet;
the confirm path stores `NULL`, and this route's mask is the first producer of
the shape in the workspace.~~ **Pending in vpay#406 (issue #79 item 4,
2026-11-17):** the confirm path writes `charges.payer_ref_masked` as
`+237••••67` — the first three and last two digits, four bullets — from
`vpay_core::privacy::payer_msisdn_mask`, and migration `0062` backfills and
constrains it. **That is a different shape from this log mask on purpose:**
#79 decided the stored column only; the log lines keep `+2376••••NNN` because
the log and telemetry shape is RFC-0002 D3, still open. Do not unify them
without that decision.

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

## The three controls — BUILT on 2026-11-17, every value PROVISIONAL

**Read this box before quoting any number below.** PR #396 (issue #56 / #152,
merged on 2026-11-17 as vpay#404, on the owner's decision to merge it on that
basis) implemented RFC-0002's
_recommendations_ so they could be reviewed against working code. RFC-0002's
decision register still reads **D3, D5, D6, D7, D8 and D9 OPEN**. Every value is
a change point, not a decision; `docs/flows/account-holder-lookup.md` § Status
carries the one table of them. Do not describe any of it as "decided".

| RFC-0002 | What ships                                                                                                                                                                                                                  | Where it changes                                                          |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| D3       | `target_repr` = HMAC-SHA256(`tenant_id:canonical MSISDN`) under `privacy.lookup_audit_salt`; rotating the key breaks correlation with every earlier row                                                                     | `account_holders::audit_target_hash`, `privacy.lookup_audit_salt`         |
| D5       | scope `account_holders:read`, implied by **no** payment scope, enforced from the first request after deploy with **no migration window** — breaking for every credential without it                                         | `vpay_api::v1::SCOPE_ACCOUNT_HOLDERS_READ`, `v1::refunds::resolve_target` |
| D6       | per credential (signing-key thumbprint / token `client_id`), `attempts: 10` per `window_seconds: 60`; **every attempt counts, a malformed number included**; `Retry-After` is the remaining window, rounded up; fail closed | `privacy.lookup_rate_limit`, `ask_rail`                                   |
| D7       | `initiated` + one terminal row under one `operation_id`; outcomes `initiated`, `found`, `not_found`, `invalid`, `unsupported`, `rate_limited`, `rail_error`, frozen by a CHECK                                              | a new migration replacing `audit_events_outcome_is_known`                 |
| D9       | class `operational` in `schemas/privacy-inventory.yaml` — and **no retention handler: the journal is unbounded**                                                                                                            | a handler, RFC-0002 Stage 3 PR 7 — not built                              |

**D8, the operator surface, has no answer at all: it is not built.** The only
reader of `audit_events` is `vpay_db::AuditEvents::list_audit_events_by_tenant`,
which no route, CLI or dashboard screen calls — ad-hoc SQL is the only way an
operator sees the journal. GAP-56 stays open on it. The break-glass mechanism
issue #152 asked for is not built either (`GAP-152-BREAK-GLASS`).

**The scope (D5).** `GET /v1/account_holders` requires `account_holders:read`
(not `payments:read`, which served it until 2026-11-17). A refund that names a
`destination` requires it **in addition to** `payments:write` — checked before
the capability branch, so it applies on `orange_money` too, where no lookup is
made. The merchant MCP `create_refund` tool with a destination requires it as
well: since the PR #396 review an MCP tool call runs as the token's own
`client_id` with exactly the token's scopes held to the registration's YAML
(an earlier revision of that PR ran it as an invented `mcp:<merchant>`
principal with the payments scopes only, so a nominated-payee `create_refund`
was always `403`; on `master` before the PR it needed `payments:write` alone). An
implicit `HEAD` to the route is held to the `GET` policy. A credential without
the scope is `403`, `invalid_request_error` / `forbidden` — `/v1`'s one
`ApiError::Forbidden`, which names no scope. There is no `insufficient_scope`
on `/v1`; that code exists only on `/mcp/v1`, for a token with no MCP scope.
_(The flow page said `insufficient_scope` on the pull-request branch; vpay#404
corrected it.)_ Operators must add it to each
`merchant_clients[].scopes` that needs it **before** upgrading — see the
`vpay-ops` skill's `privacy:` section.

**The rate limit (D6).** Counted before the rail call in `rate_limit_windows`
under scope `lookup:merchant_key`, one fixed-window budget per credential shared
by the route, nominated-payee refunds and MCP `create_refund` — so a client's
`/v1` token and its MCP calls spend one budget. Over budget: `429`,
`rate_limit_error` / `rate_limit`, `Retry-After` = seconds left in the window.
A database failure while counting is a `500`. Since PR #396 each row stores its
own `window_expires_at` (migration `0061`; the journal is `0060`), so a 60 s lookup sweep no
longer deletes staff sign-in counters still inside their 300 s window. **Still
not a substitute for ingress limiting**: it bounds one key, not a merchant with
many keys or many merchants. Neither merchant SDK surfaces `Retry-After`
(two dated `⛔` rows in `docs/sdks/parity.md`).

**The audit journal (D3/D7/D9).** `audit_events`, append-only **in the
database**: triggers refuse `UPDATE`, `DELETE` and `TRUNCATE` for every role
(`restrict_violation`) — a guard against mistakes, not tamper-proofing, since
the owner can disable a trigger; the generated `@vpay/dash-client` has no
create/update/delete on `auditEvents` (`@@internal`). **Which writes fail
closed is one rule:** a row that precedes or accompanies a disclosure fails
closed (`initiated` before the rail is asked; the terminal row before the name
is released — if it cannot be written the answer is withheld with a `500`); a
row that records a refusal (`rate_limited`, `invalid`) is best-effort, logged at
`error`, keyed over the canonical number. A retention handler, when D9 is
decided, will need a deliberate exception to the trigger.

**Proven** by `credential_without_account_holders_read_scope_is_refused_403`,
`a_nominated_payee_refund_without_account_holders_read_is_refused_403`,
`a_head_to_account_holders_is_held_to_the_get_scope_policy`,
`rate_limit_blocks_and_retry_after_is_provided_and_second_credential_unaffected`,
`a_nominated_payee_refund_spends_the_lookup_rate_limit_budget`,
`audit_table_holds_hmac_target_representation_and_no_raw_pii` (integration),
`the_audit_journal_refuses_update_delete_and_truncate` (`vpay-db`), and the
two MCP seam tests. **The MCP nominated-payee path is proven at its seams only**
— no integration test drives `create_refund` with a destination end to end.

### What this section said until 2026-11-17

The three controls were "RESERVED, NOT BUILT", with no chosen default. The
reasoning below is kept because it is still why the controls are scoped to
lookups and why every value is provisional.

~~**Rate limit — not built.**~~ §3 asks for a per-merchant limit and says why:
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
route on, not a caveat about it. _(No longer true per credential since
2026-11-17; still true for a merchant holding many keys, absent ingress
limiting.)_

**Widened 2026-09-16, and this is the part to carry into any design.** It is no
longer only this route. `POST /v1/refunds` asks the same question under
`payments:write`, and a `400` naming `destination` versus a `200` is a
registration oracle with no name attached but the same enumeration shape — and
it leaves no row either, since the refusal precedes every write. **A limit
scoped to `GET /v1/account_holders` would therefore be a control that looks
like one.** Scope the decision to lookups, at `ask_rail` or at ingress across
both paths.

~~**Audit log — not built, and it contradicts rule 2.**~~ A per-merchant,
per-MSISDN trail _is_ a stored record of who asked about whom, which is
precisely the record rule 2 declines to keep. Which wins is a policy choice.
`MerchantScope` is already bound on the handler, so the key such a log would
need is in hand the day it is decided. Same widening as the rate limit: the
refund path makes lookups too, so a trail that covered only this route would
under-report its own subject.

~~**A dedicated scope (`identity:read`) — not built.** The route is served under
`payments:read` like every other `GET`.~~ (It was built as
`account_holders:read`, not `identity:read`, and it did refuse every existing
credential on the day it landed — the warning below is exactly what happened.) Adding one is a three-place change that
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
this surface.** Elsewhere it is read off the row; there is no row for the
answer (rule 2 — the `audit_events` rows of 2026-11-17 record the question, not
an object), so the only available value would be the deployment's
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
`Unsupported`, because RFC-0003 § 5 decided an Orange refund _is_ a transfer
back. Two flags on one rail, two different shapes — see the `vpay-orange-money`
skill, and do not carry either correction across to the other.

The flag is deliberately **not persisted**: no column in `providers`, no field
on `vpay_db::ProviderSeed`. Nothing reads a capability out of that table.

`/v1` refuses an unknown rail, a _disabled_ rail and an incapable rail with a
**byte-identical `400`** — telling them apart would let a merchant enumerate
which rails a deployment has configured but switched off.

## ~~The tenant is bound and unused, on purpose~~ — used since 2026-11-17

~~The handler's signature is `_scope: MerchantScope`. Nothing is scoped by it,
because nothing is stored or read.~~ It was bound so that the auth middleware
must have resolved a tenant (a failure there is a `500`, fail-closed) and so
the value an audit log would key on was in hand the day that decision was
taken. Since vpay#404 it is used: the tenant keys the `audit_events`
row and the HMAC input, and the handler also takes a `CallerCredential` (the
signer's thumbprint or the token's `client_id`) for the scope check and the
rate-limit budget. Deleting either parameter now breaks the controls, not just
the fail-closed property.
