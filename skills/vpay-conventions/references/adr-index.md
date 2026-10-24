# The ADR index, and what is still undecided

_Verified against vpay `87166eaf` (2026-10-09) for the ADR-0026 and ADR-0028
rows, the notes on them, "Not accepted" and the numbering section; the rest of
this page was last read at `a33aac61` (2026-09-29). **The RFC list below was
extended on 2026-10-20 for RFC-0012 and RFC-0013, read on a vpay branch that
merged to `master` the same day as `b8be107` (vaam-apps/vpay#285), where both
rows still hold, and for RFC-0014, merged the same day as `0977616`
(vaam-apps/vpay#296); no stamp moved for either.** The RFC-0012 row's
sentence on what the adapter has called was re-read on 2026-10-21 against vpay
`e28292d`, the merge of vaam-apps/vpay#310 (RFC-0012's
step 5); the rest of that row is unchanged. The RFC-0014 row's clause on
what Notch Pay's check has been verified against was re-read on 2026-10-24
against vpay `4df5e0d2`, the merge of vaam-apps/vpay#364. Version-sensitive
claims carry the date they became true — see
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`docs/adr/`. An ADR is **immutable once accepted**: to change a decision you
write a new ADR that supersedes it. You never edit one.

The one documented exception is ADR-0016's serde exemption _table_, which the
ADR itself calls "the one part of this document a routine change touches —
adding or removing a row is a change to an accepted decision's data, not to
the decision."

## The index — Accepted unless the row says otherwise

| ADR                                                         | Decision, in one line                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0001 record-architecture-decisions                          | Numbered, immutable ADRs in `docs/adr/`; supersede, never edit.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| 0002 provider-port                                          | One `ProviderAdapter` trait; **providers are rows in a table, never enum variants**; the core branches on capability _values_, never a provider code. Adding a rail is an INSERT plus an adapter crate, never a schema migration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 0003 yaml-configuration                                     | All administration is YAML in git with `${ENV}` placeholders, validated at boot and reconciled in one transaction; validation failure exits non-zero without serving traffic; the dashboard cannot change any of it.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| 0004 musl-mimalloc                                          | Static musl binaries into `FROM scratch` with mimalloc. **Superseded in part by 0014** — only the architecture named in its Decision. _Addendum 2026-09-20:_ its "two binaries, two images" framing is obsolete since issue #77 (2026-09-07) — one binary, `vpay-server`; musl, mimalloc and `FROM scratch` stand.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| 0005 rustls-only                                            | rustls everywhere; `openssl`, `openssl-sys`, `native-tls` banned in `deny.toml`; TLS verification never disabled, including against stub hosts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| 0006 no-mocks-in-main-processes                             | No test double reachable from the shipping binary; a stub rail is a WireMock host in configuration. Gate: `verify-no-mocks`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| 0007 lint-policy                                            | A panic in a payment path is a defect. Deny `unwrap`/`expect`/`panic`/`todo`/`unimplemented`/float arithmetic; forbid `unsafe`; tests exempt via `clippy.toml`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| 0008 dashboard-scope                                        | The dashboard **observes and acts on records**; it never administers configuration and never holds a merchant secret key. Every write produces an `audit_log` row. _Addendum 2026-09-20:_ **none of those writes is built** — `/dash/v1` mounts `GET` and nothing else. The Decision stands; it is not implemented.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| 0009 dashboard-oidc-provider                                | vpay runs Authkestra's `authkestra-op` **in-process as its own OP**; authorization-code + PKCE. _Its "Scope boundary" paragraph is superseded by 0010; its audience half by 0017._                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| 0010 merchant-auth-private-key-jwt                          | `/v1` authenticates merchants with OAuth2 `client_credentials` + `private_key_jwt`, not API keys — because `SqlxOpStore::find_client` hardcodes `token_endpoint_auth_method: None` and `jwks: None`, so a DB-backed registry cannot serve `private_key_jwt` at the pinned version.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| 0011 error-modelling                                        | Errors typed at the leaves, composed per layer, classified once, `anyhow` only at the edge. See [errors.md](errors.md).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| 0012 rail-configuration-requirements-in-config              | **Interim.** `vpay_config::config::REQUIRED_RAIL_KEYS` is the _one_ sanctioned place outside an adapter crate where a provider code is matched on; it moves behind the port the day the port grows a `required_settings()` hook.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| 0014 builder-host-musl-triple                               | `backends/Dockerfile` passes `--target` set to the **builder's own host triple** read from `rustc -vV`, never a hardcoded `x86_64-unknown-linux-musl` — hardcoding it on an arm64 host "fails outright". Supersedes 0004's architecture only.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| 0015 sdk-parity                                             | `sdks/rust` and `sdks/nodejs` are held to parity per capability, machine-checked by `verify-sdk-parity` against `docs/sdks/parity.md`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| 0016 engineering-standards                                  | Six engineering standards, three machine-checked. See the table in `SKILL.md`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| 0017 staff-authentication                                   | How a staff member signs in to `/dash/v1`: argon2id with a deployment pepper, RFC 6238 TOTP with a sealed secret, a strictly-increasing replay guard, mandatory enrolment, a one-time password. **Supersedes the audience half of 0009** — the literal `vpay:dash/v1` is retired.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 0018 cross-tenant-admin-reads                               | A cross-tenant **read-only** admin role for `/dash/v1` (`is_admin` on `staff_members`). `/dash/v1` still answers `403` to every non-`GET`. Extends 0017.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| 0019 credential-model                                       | Credentials become their own object, generic over kind and subject. Amends 0017's decision 1 about _where the material lives_, not about how a staff member proves identity. Does not touch 0018.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 0020 privacy-controls-and-evidence                          | Six GDPR workstreams share **one** personal-data inventory and one evidence architecture (actor / tenant / target / outcome / time / correlation). Was a second `0018` until 2026-09-16 — see below.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| 0021 flutter-checkout-plugin                                | A Flutter checkout plugin as a payer surface that calls the hosted page and reads the outcome by **polling, never a URL**. ~~a native Activity / UIViewController calling the hosted page~~ _Corrected 2026-09-23:_ D5 was revised in place on 2026-09-16 (vpay#185) — the in-app `WebView` is gone on every platform, and the page opens in the **payer's own browser** (Custom Tab, `SFSafariViewController`, the macOS default browser).                                                                                                                                                                                                                                                                                                                                                                                                                |
| 0022 surface-isolation-and-independent-scaling              | **Proposed, not Accepted.** One image, two tiers: `deployment.surfaces` selects which surfaces a process mounts (absent = all, so existing deployments upgrade unchanged). Autoscaled `-server`; `-management` and `-dashboard` fixed. `/v1/oauth/token` is business-only. The binding constraint is `MAX_CONNECTIONS` (10/process), not CPU — see the `connection-budget` guard. Three decisions are left to the maintainer.                                                                                                                                                                                                                                                                                                                                                                                                                              |
| 0023 tauri-checkout-plugin                                  | **Proposed, not Accepted** (2026-09-22; every decision T1–T7 taken by the implementing agent). A Tauri v2 checkout plugin for Android, iOS and web that inherits 0021's D1–D9 unchanged: the state machine is guest-JS (T1); the crate is **its own Cargo workspace**, outside `just ci` (T2, T7); iOS cannot report `stopUrlReached`, so every iOS checkout ends `dismissed` (T6). Four questions are left to the maintainer.                                                                                                                                                                                                                                                                                                                                                                                                                             |
| 0024 customer-filters-and-manual-payments                   | Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251); **Accepted, D1–D19**, 2026-09-23. `customer=` filters the intent, session and refund lists in the same `WHERE` as `merchant_id` (no oracle; `GET /v1/customers` stays unfiltered); `pay` with `paid_out_of_band=true` records a merchant's unverifiable statement, posts nothing to the ledger. ~~**Open question 3** (write a session's customer onto a customer-less intent?) is undecided.~~ _(Answered 2026-09-23 by ADR-0025, below; ADR-0024 is not edited and still reads "Still open".)_ Not on `master` before that merge. ~~`0023` (the Tauri plugin) is not in this table — not re-verified here~~ _(0023 added above, 2026-09-23.)_                                                                                                                               |
| 0025 session-customer-onto-intent                           | **Accepted** (2026-09-23, vpay#253). A session naming a `customer` on a customer-less intent writes it onto the intent, in the insert's own transaction. Answers ADR-0024's open question 3. No backfill. See the notes below the table.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| 0026 the-database-clock-schedules-jobs                      | ~~**Accepted in part** (2026-09-23; vpay#256, merged 2026-09-24). The rule is the maintainer's; D1–D8 are the implementing agent's, marked as needing the maintainer's confirmation.~~ **Corrected 2026-10-09: Accepted (2026-10-08).** The rule is the maintainer's decision of 2026-09-23; the maintainer confirmed D1–D8 on 2026-10-08 (vaam-apps/vpay#256 merged 2026-09-24; the confirmation reached `master` with vaam-apps/vpay#271). Scheduling methods take a `Duration`; facts keep the application's clock. D8's `jti` item is concluded by ADR-0028. See the notes below the table.                                                                                                                                                                                                                                                            |
| 0027 erasure-reaches-through-checkout-sessions              | **Accepted** (2026-09-23; vpay#257, merged 2026-09-24). Erasure also reaches payments through `checkout_sessions.customer_id`, via `vpay_db::customers::PAYERS_INTENTS`. See the notes below the table.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| 0028 a-spent-jti-outlives-the-validators-leeway             | **Accepted** (2026-10-08; vaam-apps/vpay#271, merged 2026-10-09). The housekeeping sweep keeps a spent client-assertion `jti` for `vpay_worker::CLIENT_ASSERTION_JTI_RETENTION` (5 minutes) past its `exp`, computed by Postgres, because the validator accepts an assertion for 61 s after `exp`. Concludes ADR-0026 D8's `jti` item. See the notes below the table.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| 0029 card-and-bank-adapters-and-payment-method-routing      | **Accepted in part** (2026-10-17, vaam-apps/vpay#275). D1–D11 are RFC-0004 § 7 and RFC-0008 as merged: one adapter per provider under four hard rules, the capabilities the port will grow (`method_types`, `amount_limits`, `supports_off_session` + `charge_stored`, `ProviderFlow::Instructions`), stored methods bound to their minting rail and decoupled from subscriptions, the cheapest capable rail first from configured fees, every decision recorded per charge, no cascading. **D12–D17 answer the RFCs' open questions and need the maintainer's confirmation**; D12 (`card` plus a `card.brand` attribute, not `visa_card`/`mtn_card`) refuses a spelling the maintainer proposed. **2026-10-23: D6, D7 and D10 are built**, D12/D13 confirmed for wallets, D8/D9 reopened by RFC-0008's revision. See the notes below the table.           |
| 0031 a-status-read-authenticated-by-a-server-side-key       | **Accepted** (2026-10-20, vaam-apps/vpay#295, on `master` since `357c825`). Amends ADR-0029 D1's **third rule only**, for every provider. A status read authenticated by a key the provider calls public is admissible on six conditions: every secret sent on every call, boot refusing a config without it, the exposure documented and logged at boot (`ProviderAdapter::accepted_exposure`), the amount checked on every success, and the written question kept open. Notch Pay is "admitted under ADR-0031", never "rule 3 passes". Building such an adapter without the conditions is "way B done silently", refused.                                                                                                                                                                                                                                |
| 0032 a-rail-issued-reference-is-committed-before-the-prompt | **Accepted** 2026-10-23: asked whether to mark it Accepted, the maintainer answered "Yes". Merged as vaam-apps/vpay#345 (`3633ca0c`), **Proposed** until then. The maintainer had asked how to close Notch Pay's crash window and then said to build the answer; D1–D6 are the implementing agent's, accepted as written. Notch Pay's side was measured through `sandbox_probe` (vaam-apps/vpay#350); vpay itself has not run the split against Notch Pay. Extends ADR-0002: an optional `ProviderAdapter::prepare` and `Capabilities::submit_steps` (`One` \| `PrepareThenSubmit`). On a two-step rail the core commits what `prepare` returned on the charge before it writes the `submit` row, so the prompt is never sent for a payment vpay cannot name; no `submit` row is the proof nobody was prompted (`PriorSubmits`). Built on `notchpay` only. |

### Notes on 0025–0028 (0025–0027 added 2026-09-29 against vpay `a33aac61`; the 0026 bullet corrected and 0028 added 2026-10-09 against `87166eaf`)

- **0025** — the write is a compare-and-swap on `customer_id IS NULL`
  (`claim_intent_customer`, in the session insert's own transaction), so a second
  session naming another customer is the existing `400` naming `customer`, and of
  two concurrent creates exactly one wins. It answers ADR-0024's open question 3
  **without editing ADR-0024**, which still reads "Still open" because ADRs are
  immutable. **No backfill**: rows from before 2026-09-23 stay inconsistent across
  the three `customer=` filters, because a backfill would have to guess between
  sessions that named different payers. It left open whether erasure should go
  through `checkout_sessions.customer_id`; ADR-0027 answers that.
- **0026** — the rule ("the database's clock must be the one clock for anything
  the database compares") is the maintainer's decision. **D1–D8** are how the
  implementing agent applied it, and the ADR said the boundaries — which instants
  are scheduling and which are facts (D6), and what was left out (D8) — "need the
  maintainer's confirmation in review". ~~The status line on vpay `master` at
  `a33aac61` still reads "Accepted in part"; a vpay PR confirming it was in flight
  when this page was written, so do not describe D1–D8 as maintainer-decided until
  `master` says so.~~ **Corrected 2026-10-09:** the status line now reads
  `Accepted (2026-10-08)`. The maintainer confirmed D1–D8 on 2026-10-08, after
  vaam-apps/vpay#256 merged, and the text of D1–D8 is unchanged. The edit reached
  `master` with vaam-apps/vpay#271 and touched the Status block only; the ADR is
  otherwise as it was, so "immutable" survives. Two findings from the review that
  preceded the confirmation are recorded in ADR-0028 rather than in ADR-0026: D8's
  `jti` item is concluded there, and **D6's list of comparisons omits the checkout
  confirm gate**. Applied: `enqueue_in_tx`, `record_attempt` and
  `live_charges_stale_since` take a `Duration` and SQL computes `now() ± $d`;
  `oldest_runnable_run_at` became `oldest_runnable_age`. Facts
  (`checkout_sessions.created_at`/`expires_at`, `customers.last_used_at`, the `t=`
  in a webhook signature) keep the application's clock. Two same-class columns were
  named and left out (D8): `oauth_signing_keys.expires_at` and
  `oauth_client_assertion_jtis.expires_at`. _(Since 2026-10-09 the second is
  concluded by ADR-0028 and takes a `Duration` too. `oauth_signing_keys.expires_at`
  is untouched and still open.)_
- **0027** — the guard is the intent's own customer being `NULL` or the erased one;
  an old intent whose sessions named two payers is redacted by either payer's
  erasure; a session create takes `FOR SHARE` on its customer **before** the
  intent (the old order deadlocked with erasure, `40P01`) and refuses a just-erased
  customer with `409`; erasure writes nothing onto the intent.
- **0028** — the defect: `authkestra-op` 0.7.1's `verify_client_assertion` leaves
  `jsonwebtoken`'s default **60 s leeway** (vpay cannot set it; the dependency is
  pinned `=0.7.1`), and the comparison is on whole seconds, so an assertion is
  accepted for up to **61 s after its `exp`**. The hourly `sweep_expired` used to
  delete `oauth_client_assertion_jtis` rows at `expires_at < now()`, with
  `expires_at` the client's raw `exp`. A sweep landing inside those 61 s deleted the
  `jti` of an assertion the API still accepted, and a captured, **already-spent**
  assertion could be replayed once for a second merchant access token. The window
  is `max(0, 61 s + db_clock_offset − api_clock_offset)` after `exp`; about 1.7 % per
  assertion at zero skew; nothing was logged. The fix:
  `ClientAssertions::delete_expired_client_assertion_jtis` takes
  `retain_after_exp: Duration` and the statement is
  `expires_at < now() - ($1::BIGINT * INTERVAL '1 microsecond')`, on the database's
  clock (ADR-0026 D1's shape). The worker passes
  `vpay_worker::CLIENT_ASSERTION_JTI_RETENTION`, 5 minutes: 60 s leeway, 1 s
  truncation and about 239 s of skew budget. `expires_at` stays the client's raw
  `exp` (a fact, D6) and `record_jti` and the migrations are unchanged. Two tests in
  `backends/tests/integration/tests/merchant_token_flow.rs` pin it from both sides,
  and **an assertion older than the horizon is refused by the validator, so a
  leeway bump past 5 minutes fails CI** and the constant must grow first. **Not
  covered:** a database clock more than about four minutes ahead of an API
  replica's, which nothing checks. Migration `0011`'s header comment still says no
  cleanup job exists; it is checksummed and stays stale. See `vpay-merchant-api` and
  `vpay-reconciler`.

### Note on 0029 (added 2026-10-17, against the vaam-apps/vpay#275 branch)

- **0029** — accepts a decision, not a capability. On the day it was written
  the port had none of the identifiers it names, there was no `pm_…` object
  and no `routing_decisions` table, and `payment_method_types` held rail
  codes — so an adapter written against it is written against a design, as
  RFC-0010 (Stripe) and RFC-0011 (PayPal) are. Each capability "lands with
  its first consumer, never ahead of one" (D2); do not add one to the port
  without the adapter that declares it and the conformance case that proves
  it. **D12–D17 are proposed**: read the Status line on `master` before
  describing `card`/`collapse`/`routing.requires` as the maintainer's
  decision. _(2026-10-23: D12's two wallet spellings and D13's `collapse` are
  confirmed, D8 and D9 reopened; ADR-0029's Status says so with the quotes.
  Later that day RFC-0008's R8 was answered, which settles D9 (score, then
  rail code), D8 in substance (the fee is a score component, not the sort
  key) and RFC-0008's clarification of D11 (no measured quantity moves
  traffic: health by R8.3, the completion rate by "Go with (a)"; D11's own
  "one charge per intent" is unchanged).
  `card`'s spelling and D14–D17 are still proposed. The table's "Nothing it
  decides is built" stopped being true that day: D6, D7 and D10 are.)_
  _(2026-10-24: **D10 amended by the maintainer**, "Both" and then "Go with
  A": the decision's outcome, the charge's type and rail, is on the `/v1`
  intent as `payment_method_details`; the `routing_decisions` record is still
  on no `/v1` object. ADR-0029's Status and D10 carry it.)_ RFC-0004 §§ 1–4 and 8–13, RFC-0006 and RFC-0007 stay Draft.
  _(Added 2026-10-20.)_ D1's third rule says _secret_-authenticated rather than
  _authenticated_ because of Notch Pay, whose status read takes a public key;
  RFC-0012 was gated by it, waiting on Notch Pay's written answer (the
  maintainer's decision of 2026-10-20). **Superseded the same day by ADR-0031**
  (vaam-apps/vpay#295, merged as `357c825`), which amends that rule for
  every provider after Notch Pay did not answer.

## Not accepted — read the status line before you build on it

**ADR-0022 and ADR-0023 are `Status: Proposed`** (as of 2026-09-23, and still
on 2026-09-29, vpay `a33aac61`), and their rows above say so. Both were written
and taken by implementing agents; neither has been put to the maintainer as a
whole. Build on what their code does, not on the ADR having been accepted.

~~**ADR-0026 is `Status: Accepted in part`** (as of vpay `a33aac61`, 2026-09-29).
Its rule is the maintainer's; its eight decisions D1–D8 are the implementing
agent's and say they "need the maintainer's confirmation in review". The row
above says so. A vpay PR that confirms them was in flight when this page was
written; until `master` says "Accepted", the unconfirmed half is D6–D8, the line
between scheduling instants and facts. Do not move a fact (D6) onto the
database's clock, or a scheduling instant off it, on the strength of this
page.~~ **Corrected 2026-10-09: ADR-0026 is `Status: Accepted (2026-10-08)`** and no
longer belongs in this section; the maintainer confirmed D1–D8 (vaam-apps/vpay#271
carried the status edit). The instruction survives as a rule rather than a caveat:
do not move a fact (D6) onto the database's clock, or a scheduling instant off it,
without reading D6 and ADR-0028 § "Related findings" first.

**ADR-0013 database-backups-and-retention is `Status: Proposed`.** Its own
first bullet:

> Nothing here is implemented. **No backup of any vpay database has ever been
> taken**, no restore has ever been performed, and no restore drill has ever
> run. This ADR records obligations the schema already creates and proposes a
> policy to meet them; **every number in it is proposed, not measured.**

The recovery objectives, the 30-day PITR window and the 90-day full retention
are all proposed values nobody has measured against anything. The ADR exists
because "the schema already commits vpay to holding things whose loss cannot be
recovered from anywhere else" — money, the only evidence an operator would have
about money, and replay protection, "a security control whose loss is not
visible in any dashboard".

`docs/runbooks/restore-from-backup.md` had every SQL statement in it executed
against a scratch Postgres — but nothing about backups, PITR or the restore
itself has been exercised.

## RFCs — proposals, not decisions

As of 2026-10-20 there are **fourteen** (this list named only 0001 and 0002
until 2026-09-23; 0003 arrived 2026-09-16 with vpay#178, 0004–0008 on 2026-09-23
with vpay#244, 0009 on 2026-10-16, 0010 and 0011 on 2026-10-17, and 0012,
0013 and 0014 on 2026-10-20 — it said "thirteen" for part of 2026-10-20,
"eleven" from 2026-10-17 until 2026-10-20, and
"eight" before that):

- `docs/rfc/0001-settlement-and-payouts.md` — **Draft**.
- `docs/rfc/0002-gdpr-policy-and-operator-decisions.md` — **Under review**.
- `docs/rfc/0003-refunds-destinations-and-the-first-ledger-postings.md` —
  **Draft** by its own status line. It landed in vpay#178 alongside the refund
  path it proposes; read `docs/status/` for what that built, not the RFC.
- `docs/rfc/0004-billing-on-top-of-invoices.md` — **Draft, except § 5's
  `customer` filters and § 6 (manual payments)**, accepted 2026-09-23 as
  ADR-0024, **and § 7 (one adapter per provider)**, accepted 2026-10-17 as
  ADR-0029 and **not built**. Everything else in it — subscriptions, the
  invoice preview — is unbuilt; `/v1/subscriptions` does not exist.
- `docs/rfc/0005-prepaid-customer-balances.md`,
  `0006-card-processing-without-a-psp.md`,
  `0007-bank-transfer-reconciliation.md` — **Draft**. None is built.
- `docs/rfc/0008-payment-method-routing.md` — **Accepted** 2026-10-17 as
  ADR-0029, less its § 7 (health); **revised 2026-10-23** ("a score of each
  payment solution for each checkout", the maintainer) with § 4's sort
  superseded, and **slice 1 built** the same day: method types beside rail
  codes, the scored router with formula `provisional-1`, and
  `routing_decisions`. **Its R8 questions were answered on 2026-10-23**
  ("go with your defaults"): the estimated fee is the next score component;
  the completion rate is **reported** per rail and type for the operator to
  set `routing.weights` by hand, never scored ("Go with (a)", § R8.1; neither
  is built); the operator alone sets weights until RFC-0009; health reports
  and never moves traffic, and no measured quantity does; the confirm follows
  the rail the page showed while it is eligible (**not built**: the confirm
  re-ranks today); no exploration; the tie-break to the lower rail code is
  kept. _(This bullet
  listed 0008 among the Drafts until 2026-10-17, and said "not built …
  no router exists" until 2026-10-23.)_
- `docs/rfc/0009-provider-accounts-per-merchant.md` — **Draft**, proposed
  2026-10-16. It starts from a measured gap, not a feature request: provider
  credentials are configured once per deployment, so pass-through holds only on
  a deployment that serves one merchant. See `vpay-provider-adapters`.
- `docs/rfc/0010-stripe-adapter.md` — **Draft**, proposed 2026-10-17. The
  design of `vpay-adapter-stripe` (rail code `stripe`, hosted Checkout as a
  `Redirect` rail) and of the two core changes any `Origin`-refunding rail
  needs. No crate, mapping or stub exists. See `vpay-provider-adapters`.
- `docs/rfc/0011-paypal-adapter.md` — **Draft**, proposed 2026-10-17. The
  design of `vpay-adapter-paypal` (rail code `paypal`, Orders v2 as a
  `Redirect` rail whose `query_status` **captures** an approved order,
  because PayPal's wallet is not auto-captured on approval). No crate,
  mapping or stub exists. See `vpay-provider-adapters`.
- `docs/rfc/0012-notchpay-adapter.md` — **Draft**, proposed 2026-10-20. **Since
  `357c825` (2026-10-20, vaam-apps/vpay#295) it is admitted under ADR-0031
  and built**: the adapter exists,
  has called only Notch Pay's sandbox (RFC-0012's step 5, 2026-10-21), and the
  `vpay-notchpay` skill is its briefing.
  What follows is the RFC as it stood on `master`, **not buildable as
  written**: Notch Pay (a Cameroon mobile-money aggregator)
  fails ADR-0029 D1's third hard rule on its own documentation: its payment
  status read accepts the key Notch Pay calls public, and that rule says
  _secret_-authenticated, worded for exactly this provider. The maintainer
  decided on 2026-10-20 to ask Notch Pay in writing (the RFC's way A) and to
  leave the rule as written; the adapter is not written until Notch Pay
  answers, and no answer is in the repository as of 2026-10-20. The design is
  `vpay-adapter-notchpay`, rail code `notchpay`, a `Push` rail. **What exists
  is only the client crate**, `backends/crates/vpay-notchpay` (2026-10-20, 34
  unit tests), which no adapter uses and no binary links. See
  `vpay-provider-adapters`.
- `docs/rfc/0013-vpay-registers-its-own-callbacks.md` — **Draft**, proposed
  2026-10-20; **steps 2–4 built 2026-10-22 for Notch Pay** (its § "As built";
  `ensure_callbacks`, `vpay-server callbacks ensure`). _(The rest of this
  entry, from "nothing in it is built" to "neither of which exists", is as
  written on 2026-10-20; both names exist now.)_ ~~**nothing in it is
  built**.~~ On a rail that registers its callback
  URL with the provider account (Stripe, PayPal, Notch Pay) vpay would register
  its own endpoint, in a worker job and never on the boot path; MTN and Orange
  take the URL per request and would be asked nothing. It proposes
  `Capabilities::callback_url_mode` and `ProviderAdapter::ensure_callback_endpoint`,
  **neither of which exists**. Its § 11 lists findings about vpay's own
  documentation (for one, the claim that MTN retries a callback, which MTN's
  documentation contradicts); ~~vpay has not yet corrected them~~
  **updated 2026-10-21:** vaam-apps/vpay#319 corrects items 1–3 and leaves 4
  and 5, and **2026-10-22:** vaam-apps/vpay#326 corrects 4 and 5, which
  completes step 0 — see `vpay-troubleshooting`'s known-wrong-docs page. See `vpay-provider-adapters`.
- `docs/rfc/0014-verifying-rail-callbacks.md` — **Draft; the port and
  Notch Pay's check built on 2026-10-23** (against `master` `82a68f4e` plus
  the RFC-0014 branch), stub-proven, and since 2026-10-24 checked offline
  against four real Notch Pay webhooks, all `Verified`; none has reached a
  running vpay. _(This said "stub-proven; never verified against a real
  Notch Pay delivery" until 2026-10-24.)_
  `ProviderAdapter::verify_callback` and `Capabilities::callback_check`, run
  after the charge is found; the verdict gates only the pull-forward, so
  callbacks stay hints. The secret is a credential under port-fixed names,
  not § 5's list; six boot refusals, all `ConfigError` exit 78
  (`IncoherentCallbackCheck`, `CallbackSigningSecretMissing`,
  `CallbackSigningSettingUnread`, `CallbackVerifyOffWithSecret`, and from the
  same day's security review `CallbackSigningPreviousWithoutCurrent` and
  `CallbackSigningSecretWhitespace`); a declared check that is not performed
  fails closed at the route; a match under the previous secret is
  `verified_previous`. Notch Pay
  was the first consumer, not Orange: Orange's `BodyToken` (open question 2)
  is not built, nor are Stripe's and PayPal's. The RFC's § "Built" records
  every choice it left open. It **lifts RFC-0012 § 2's "hash key is evidence
  only, never vpay configuration" refusal**, by the maintainer's decision of
  2026-10-23, for this purpose only. See `vpay-provider-adapters`.

## Maintainer decisions recorded in place, not as an ADR status

These are live open questions sitting inside Accepted documents. An agent that
"tidies" one has reversed a decision nobody took.

| Where                                                                 | What is open                                                                                                                                                                                                                                                                                                                                                               |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `docs/flows/errors.md`, the policy-table note                         | Splitting `Category::Idempotency` so an in-flight key can answer `409` as Stripe does. "An ADR-level change and deliberately left as a maintainer decision."                                                                                                                                                                                                               |
| ADR-0017                                                              | Four in-place reservations, including whether the credential lifetime default should move and one the ADR explicitly declines to take ("that is a maintainer decision and this ADR does not take it").                                                                                                                                                                     |
| `deploy/helm/vpay/README.md` + `docs/runbooks/provider-error-rate.md` | Whether `VpayProviderErrorRateHigh` should exclude declines. It counts every non-successful port call, which on mobile money includes `charge_declined` — a large, normal share of traffic. To be decided **with measured traffic**, together with the threshold.                                                                                                          |
| `docs/flows/hosted-checkout/not-built-and-not-proven.md`              | Whether `checkout_not_configured` should answer `503` rather than `500`. A truthful `503` needs a new `Category` or `Configuration` moving — an ADR-0011-level change, **left to the maintainer**. _(Corrected 2026-09-23: this row cited the overview page, and read as though the maintainer had decided it.)_                                                           |
| ~~ADR-0024, open question 3~~                                         | ~~Whether a checkout session's `customer` should be written onto a customer-less intent. Undecided as of 2026-09-23.~~ **Answered the same day by ADR-0025** (yes — see its row above); ADR-0024 itself still reads "Still open" because ADRs are immutable. **Struck 2026-09-29: this is no longer an open question, and an agent must not treat the write as optional.** |
| `docs/flows/deployment.md`                                            | Deleting or archiving the retired `ghcr.io/vaam-apps/vpay-worker` GHCR package — "written here as a task with an owner rather than as a fact". Nobody holds the `delete:packages` scope.                                                                                                                                                                                   |

## The duplicate ADR number — resolved 2026-09-16

~~**Two files are both numbered `0018`**:
`docs/adr/0018-cross-tenant-admin-reads.md` and
`docs/adr/0018-privacy-controls-and-evidence.md`. They are unrelated
decisions.~~

**Resolved by [#172](https://github.com/vaam-apps/vpay/pull/172), 2026-09-16.**
The second file is now `docs/adr/0020-privacy-controls-and-evidence.md`, and
`docs/adr/` runs ~~`0001`–`0022`~~ ~~`0001`–`0024`~~ ~~`0001`–`0027`~~ `0001`–`0028`
(as of 2026-10-09; 0023 on 2026-09-22, 0024 and 0025 on 2026-09-23, 0026 and 0027
on 2026-09-24, 0028 on 2026-10-09 — merge dates) with **no gap and no duplicate**. `0018` means
_cross-tenant admin reads_ and nothing else.

The section is kept because the **method** below is what a reader needs, and
because a six-week-old checkout still has the duplicate — on such a tree,
`0018` is ambiguous and this paragraph is how you tell which document someone
meant.

ADR-0021's own header records how it was resolved, and the method is worth
copying: the author ran `git ls-tree origin/master docs/adr/` (20 entries for
19 numbers), checked every **open PR's diff** with `gh pr diff --name-only`,
found that PR #172 already renumbers the second `0018` to
`0020-privacy-controls-and-evidence.md`, and took `0021` — "the next number
free of both the tree and every open PR's diff". The header is titled
**"Number checked at branch time, not assumed."**

So: **before you write a new ADR, check the tree _and_ every open PR.**
~~As of 2026-09-16 `0020` is reserved on a branch and not on `master`.~~

**ADR-0022 is the worked example, and the method was vindicated within hours.**
Its author found `0020` still an unused gap in the tree and took `0022` anyway
— monotonic above `0021` — precisely because the plan page
`docs/plans/2026-09-13-flutter-plugin.md:419` calls a future decision
"ADR-0020 shaped" (line 419 when ADR-0022 cited it; the sentence is at line
578 as of vpay `b747e5d5`), and reusing the number would have made that sentence point
at the wrong document.

**Then [#172](https://github.com/vaam-apps/vpay/pull/172) landed** (after
ADR-0022 merged as `9653ee94`, the same day) and claimed `0020` for
`0020-privacy-controls-and-evidence.md` — the document that had been waiting
for it — which also retired the duplicate `0018`. Had ADR-0022 filled the gap,
#172 would have had to renumber twice.

**Current state of `docs/adr/` as of 2026-10-09: `0001`–`0028`, no gap, no
duplicate** (it was ~~`0001`–`0027`~~ on 2026-09-29, ~~`0001`–`0024`~~ on 2026-09-23
and `0001`–`0022` on 2026-09-16). ~~`0028` is the next number free **on `master`**,
and an open vpay PR is expected to take it (the same reservation pattern ADR-0025
to ADR-0027 each recorded)~~ — it was taken by vaam-apps/vpay#271 on 2026-10-09.
`0029` is the next number free **on `master`** at `87166eaf`; open PRs were not
checked here, so check them, as below, before you take it. ADR-0023's own header
records the same check — `ls docs/adr` and every open PR's files — so the
method is now the habit, not the exception. The lesson stands and is now demonstrated rather than asserted:
**a gap in the numbering is a reservation, not an opening.** Check the tree
_and_ every open PR before you take a number.
