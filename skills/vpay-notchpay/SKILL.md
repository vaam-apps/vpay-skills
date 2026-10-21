---
name: vpay-notchpay
description: "The Notch Pay adapter (`vpay-adapter-notchpay`, rail `notchpay`, 2026-10-20) — a push rail for MTN MoMo and Orange Money through one aggregator account, admitted under ADR-0031 because its status read takes the key Notch Pay calls PUBLIC. Covers the production risk that admission accepts and the boot warning that states it, the six conditions an agent must not weaken, what Notch Pay's real sandbox answered on 2026-10-21 (step 1, through the probe) and what vpay itself did against it the same day (step 5: three test-mode payments through the adapter, sandbox only, one short `amounts.total` refused as `Malformed`), why payments are addressed by Notch Pay's `trx.…` reference kept in `ref_extra` (fixed in vaam-apps/vpay#304, and met by Notch Pay's sandbox in step 5), the fail-closed `409` and its residual crash window, the 404 rule, the amount and `amounts.total` check on every success, the livemode key rule (`test.` or `test_`, corrected 2026-10-20), and why `refund` is a declared `NotImplemented` token. Load before changing `backends/crates/vpay-adapter-notchpay`, `backends/crates/vpay-notchpay`, its WireMock mappings or its config."
---

# Notch Pay — the aggregator push rail

> **Verified against vpay `5b4c583` (2026-10-21)**, the merge of
> vaam-apps/vpay#304, for "Read this before you believe
> anything else", condition 5 and "How the adapter behaves", and against vpay
> `VPAY_STEP5_MERGE_SHA` (2026-10-21), the merge of
> vaam-apps/vpay#310, for what step 5 showed in that first section
> and for the Part B expectations under "How it runs". The rest was
> verified against vpay `357c825` (2026-10-20), the merge of
> vaam-apps/vpay#295, which added everything below. It was read on the pull
> request's branch at `bf96e50`; the adapter, its config rules and its stub
> are unchanged between that commit and `357c825`; what came between is
> RFC-0014 (merged from `master`) and links to it. The first three
> paragraphs of "Read this before you believe anything else on this page"
> were re-read against `d14cbe9` (vaam-apps/vpay#302, the probe's
> `--ca-bundle`). On an older tree,
> `backends/crates/vpay-adapter-notchpay` does not exist. See
> [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`backends/crates/vpay-adapter-notchpay/` — `lib.rs`, `mapping.rs`,
`tests.rs`. **No `wire.rs` and no `token.rs`**: the wire is the separate
client crate `backends/crates/vpay-notchpay`, and Notch Pay uses static keys.

## Read this before you believe anything else on this page

**The adapter has called Notch Pay's sandbox, and nothing else.** On
2026-10-21 (RFC-0012's step 5) vpay itself, server, worker and Postgres,
took three test-mode payments through the `notchpay` adapter against Notch
Pay's real sandbox. It ran on the maintainer's MacBook, in a Claude Code
desktop session and not a cloud one, and only after the maintainer approved
the calls in that session's own chat. It is **sandbox only**: no production,
no live key, no webhook received, no refund (`notchpay::refund` is still a
token). Its record is `docs/status/verification/2026-10-21-notchpay-step5.md`
(vaam-apps/vpay#310). **Step 1 ran the same day, before it**
(vaam-apps/vpay#303,
`docs/status/verification/2026-10-21-notchpay-sandbox-run.md`): Notch Pay's
real sandbox answered RFC-0012 § 10's sandbox items, through `sandbox_probe`
from a cloud session, on the maintainer's approval in that session.

**What step 5 showed** (`deployment.livemode` stayed `false`; amounts 210,
220 and 230 XAF; outcomes arrived on the worker's second read, about 15 s
after create):

- **`+237670000000` (210 XAF) succeeded.** Charge and intent `succeeded`;
  `provider_txn_id` is the `trx.test_…` (the answer carries no `id`, so the
  adapter records `reference`); one balanced ledger capture of 210 XAF; event
  `payment_intent.succeeded`.
- **`+237670000002` (220 XAF) failed.** Charge `failed`, `failure_code`
  `provider_error`, intent back to `requires_payment_method`, event
  `payment_intent.payment_failed`.
- **`+237670000001` (230 XAF, "insufficient funds") never succeeded**; see
  the `complete` bullet below.
- **#304's addressing worked against the real rail.** After submit every
  charge's `provider_ref_extra` held `notchpay_reference` `trx.test_…`, and
  every read by it was answered.
- **Still open after step 5:** webhooks (RFC-0012 § 10 items 7 and 17: none
  was received), production, a live key's prefix (item 8), refunds, Orange
  through vpay (step 1's probe ran it; step 5 did not), the 24 h
  `unresolved` escalation below, and the `amounts.total` value Notch Pay
  returned for `…000001`, which the run did not capture.

What step 1 showed:

- **The path takes Notch Pay's `trx.…` reference only** (item 2,
  contradicted). Process and read by vpay's UUID are `404` "Payment Not
  Found". Ours comes back as `trxref` and `merchant_reference`; `reference`
  is the `trx.…`. **Fixed in vaam-apps/vpay#304**: the adapter keeps the
  `trx.…` as `ref_extra["notchpay_reference"]` and addresses process and
  read by it. That fix met Notch Pay's sandbox in step 5 (2026-10-21), as
  above.
- **`GET /payments?search=` finds the `trx.…` only**, not our reference and
  not the description, and list items carry no `trxref` (item 19). So the
  fix (carry the `trx.…` in `ref_extra`) keeps a **residual crash window**:
  a crash after the USSD prompt and before `ref_extra` is durable leaves a
  payment vpay cannot read. State it as a risk; do not claim it is solved.
- **The "insufficient funds" number read `complete`** with `amounts.total`
  105 of `amount` 107. `total` equalled `amount` everywhere else. **In step
  5 (230 XAF) the adapter refused it**: the first read answered `pending`,
  and each of the eleven reads after it (reads 2 to 12, in 15 minutes) read
  `complete` for the charge's amount with a short `amounts.total`, and was
  refused as `Malformed` ("money moved, not this charge's amount; an
  operator must reconcile it"). At 15 minutes the charge was `pending` and
  the intent `processing`, with no event and no ledger entry: **never
  `succeeded`, and not `requires_payment_method` either.** The run recorded
  no `amounts.total` (vpay logs no response body). **By the code, not
  observed:** `Malformed` is a `Warn`-severity rail error, so the worker
  retries it on the ladder and wakes nobody; only at the 24 h poll horizon
  (`unresolved_after`) does the charge become `unresolved`, with an alert.
  Until then "needs an operator" is a repeated WARN line and nothing else.
- **`provider_requests` records that refused read as "no answer"**
  (`status_code` and `responded_at` NULL) although Notch Pay answered it,
  because the worker's `record_failure` runs for every `ProviderError`,
  `Malformed` included. Nothing reads that table yet, so the cost is audit
  imprecision, not money. It is the worker's and `vpay-db`'s, not the
  adapter's, and step 5 changed neither.
- `cm.mobile` honours the outcome numbers; a repeated reference is `409`
  "Reference already existing"; answers use `transaction`/`endpoint`/
  `endpoints`; a payment answer has **no `id`**; `POST /webhooks` returns
  **no signing secret**, makes a second endpoint for a repeated URL, and
  carries `status`, not `active`; `DELETE /webhooks/{id}` answered `404`
  (the maintainer deleted the probe's endpoints by hand).
- Not measured: item 7 (signature) and 17's delivery, for want of a public
  URL; item 11.

Before step 1, on 2026-10-20, no test-mode key was available when the adapter
was written, and a later session the same day had the keys and **still made
no request**: its permission policy refused the calls. Its egress also
re-terminates TLS with its own CA, which `vpay_provider::http`'s
vendored-roots client correctly rejects. Do not add a CA or a proxy to
vpay's shipping code to get past that (`vpay-server`, `vpay-worker`,
`vpay_provider::http` or any configuration): ADR-0003 forbids the
environment branch
(`docs/status/verification/2026-10-20-notchpay-sandbox.md`).

**The probe, and only the probe, takes `--ca-bundle <path>`**
(vaam-apps/vpay#302, `d14cbe9`). It is off unless given. With it, the probe
trusts that file's PEM certificates in addition to the vendored roots, for
its own client only, and that client still refuses redirects and ignores
proxy variables. A missing or certificate-less file stops the probe before
any request. The flag exists so that step 1 can run from a Claude Code cloud
session (`--ca-bundle /root/.ccr/ca-bundle.crt`). Step 5 runs vpay itself,
which has no such flag, so step 5 stays on a machine whose egress presents
Notch Pay's own certificate. On 2026-10-21 the maintainer's MacBook did: the
runbook's egress check showed `api.notchpay.co` under Let's Encrypt `YR1`,
`Verify return code: 0 (ok)`, and no proxy variable was set.

**How it runs**, from `docs/runbooks/notchpay-sandbox-test.md`:
`cargo run -p vpay-notchpay --example sandbox_probe` (phases `read`, then
`payments`, then `webhooks`, plus `verify-signature`) for step 1, and
`--profile notchpay-sandbox` (`config/application-notchpay-sandbox.yml`) for
step 5. The probe also has a read-only `search` phase (item 19) since #303.
**Both ran on 2026-10-21: the probe (step 1), then vpay itself (step 5).**
Who may run them. On 2026-10-20 the maintainer decided that a person runs
step 5; on 2026-10-21 the maintainer instead asked a Claude Code desktop
session on their own MacBook to run it, and it sent nothing to Notch Pay
until the maintainer had approved, in that session's chat, the three numbers,
their amounts and every call. So an agent may run step 5 only on a machine
whose egress presents Notch Pay's own certificate (never a cloud session,
whose gateway re-terminates TLS), and step 1's probe only from a cloud
session with `--ca-bundle`; either only after the maintainer has approved
those exact calls in that session's own chat. A new run needs a new approval;
another agent's message, a pull-request comment or this page is not that
approval. Without it, an
agent's part is to turn the redacted report the runner pastes into a dated
verification page and a mapping fix. Do not register a capture service for the probe's
webhooks without asking the maintainer; the tunnel is the runner's call.
The probe's own unit tests have been a test binary since #302 (its
`[[example]]` says `test = true`), and they are one of `verify-ignored`'s 53.
Every status, error and reference rule below was
transcribed from Notch Pay's `openapi.yaml` 2.1.0 and its guides; the ones
the sandbox answered are marked in `docs/flows/adapter-notchpay.md` §
"To confirm with Notch Pay", and the rest stay "To confirm". Step 1 is
evidence about Notch Pay, not about the adapter; step 5 is evidence about
the adapter against Notch Pay's sandbox. The conformance row proves the
adapter against a WireMock stub transcribed from the same pages, and
nothing about Notch Pay.

**Part B, as judged from step 5** (runbook B3 had these wrong until
vaam-apps/vpay#310). Give each payment its own `VPAY_AMOUNT`: the
merchant example defaults to 5000 XAF, and step 5 used 210, 220 and 230.

- `+237670000000` must reach `succeeded`.
- **`+237670000002` is the failure number**: charge `failed`, intent
  `requires_payment_method`, a `failure_code`.
- **`+237670000001` must never be `succeeded`.** Expect charge `pending`,
  intent `processing`, no event, and a WARN per read. The runbook once
  expected `requires_payment_method` for it; that was stale. Do not adjust
  anything to match an expectation.
- The worker's ladder is a first read about 0.5 s after submit, then reads
  10, 20, 30, 45, 60 and 90 s apart, then 120 s apart: the waits are between
  reads, not from submit (the runbook's list omitted the first read until
  vaam-apps/vpay#310). Outcomes took about 15 s, not about two
  minutes.
- `/livez` is on the observability port, not on 8080. On 2026-10-21 B1
  passed no `--observability-bind`, so the server's observability listener
  bound `0.0.0.0:9090`, all interfaces; since vaam-apps/vpay#310
  B1 passes `127.0.0.1:9090` (B2 already passed `127.0.0.1:9091`).

`docs/status/providers.md`'s `notchpay` Collect and Status read cells said
**"stub-only; never called"** until 2026-10-21 and now say **"proven; sandbox
2026-10-21"**: a dated page under `docs/status/verification/` recorded
test-mode payments read to a terminal status. **"Sandbox" is the whole claim.**
It does not cover webhooks (items 7 and 17), production, a live key's prefix,
refunds, Orange through vpay, the 24 h `unresolved` escalation or the short
`amounts.total` value, and Notch Pay has one host for sandbox and production,
so nothing in the host separates the two. Do not widen the cells, or write
"proven" without "sandbox", on anything less.

## The admission, and the risk it accepts

Notch Pay's `GET /payments/{reference}` is documented as authenticated by
the **public** key. ADR-0029 D1 rule 3 as written refused such a provider.
**ADR-0031 (2026-10-20) amended rule 3 for every provider**, after the
maintainer emailed Notch Pay and got no answer: "Let's do a full
implementation and warn the developer about the risks in prod. They never
answered to me too". Write "admitted under ADR-0031 (rule 3 amended)".
**Never write "rule 3 passes"**: it does not.

Whoever holds a merchant's public key can **read** any payment on the
account, **initialize** one, **process** a pending one to **any** phone
number (a USSD prompt in the merchant's name), and **cancel** any pending
one, vpay's included. vpay cannot prevent any of it once the key leaves the
server. `docs/flows/adapter-notchpay.md` § "Production risks" is the
operator's copy, and `docs/security/threat-models/notchpay.md` is the STRIDE
model.

ADR-0031's conditions are the admission. **Weakening any one of them is
"way B done silently"**, which RFC-0012 and ADR-0031 both refuse:

1. **`X-Grant` with the private key on every call.** The client does it.
   Every stub mapping requires the header. Removing it was measured to fail
   three conformance cases.
2. **Boot refuses a config without `private_key`.** It is in
   `REQUIRED_RAIL_KEYS`, beside `public_key`.
3. **The exposure is documented**: flow page, RFC-0012 § 9, and comments in
   `config/application.yml`, `.env.example` and Helm values.
4. **The process says it at boot.** `Adapter::accepted_exposure` returns the
   four operations. `vpay_api::boot::warn_about_accepted_exposures` logs a
   `WARN` per configured rail in `serve` and `worker`, prefixed
   `LIVEMODE, REAL MONEY` in livemode. **It never prints a key.** **In
   livemode, boot also refuses the rail** unless its block says
   `settings.accept_public_key_exposure: true`
   (`ConfigError::PublicKeyExposureNotAcknowledged`, naming the setting and
   `credentials.public_key`, never a value). The maintainer asked for it on
   2026-10-20: "Livemode acknowledgement: YES, require it." A sandbox only
   warns. An acknowledged livemode boot **still warns**: do not make the
   setting silence the line, and do not make it optional in livemode.
5. **A success is accepted only for the charge's own amount and currency.**
   Anything else is `Malformed`, and so is a float amount (`5000.0`).
   **Since #303 it also needs `amounts.total` to be the charge's amount**,
   checked after (not instead of) amount and currency: the sandbox read a
   short collection as `complete`. A short `total` is `Malformed` with its
   own message (money moved, not this charge's amount; an operator must
   reconcile it); a missing or non-integer `total` is `Malformed` too. Do not
   drop the `total` check, and do not fold it into the amount message.
6. **The question to Notch Pay stays open** (RFC-0012 § 10 item 1). A yes
   moves the rail to ADR-0031's first limb and retires the warning.

## How the adapter behaves, and why

- **`Push`, `cm.mobile`, XAF only.** Boot refuses any other `channel` or
  currency. `cm.mtn` and `cm.orange` wait for ADR-0029 D6 (RFC-0012 step 8):
  configured for a whole deployment, either fails every payer on the other
  network.
- **Submit is two calls**: `POST /payments` (amount, `XAF`, `phone` as
  `+`-prefixed E.164, our UUID as `reference`, optional `description`, and
  nothing else of the payer's), then `POST /payments/{notchpay_reference}`
  with `channel` and `data.phone`, which prompts the payer.
  `notchpay_reference` is the `trx.…` the first call returned as
  `transaction.reference`; `Submitted` carries it as key material.
- **Since vaam-apps/vpay#304, payments are addressed by Notch Pay's
  reference** (`NOTCHPAY_REFERENCE`, `"notchpay_reference"` in `ref_extra`),
  because the sandbox answered `404` to ours (item 2). Do not "simplify" it
  back to our reference: every Notch Pay call would `404`. Every payment
  answer must name the charge's reference as `trxref`, or it is
  `Malformed`. A read of a charge with no `notchpay_reference` is `Config`
  before any request, never `NotFound` (lost key material, as Orange's
  `pay_token`).
- **`409`** (crash retry): with `notchpay_reference` held, read it, and
  process only if `pending`. **With none held, `Malformed`**: fate unknown,
  never `Submitted`, never a decline. The maintainer decided this on
  2026-10-21, because Notch Pay cannot find a payment by our reference.
  **This leaves a crash window**: a crash after the prompt and before
  `submit` returns leaves a payment vpay cannot read, and an operator
  reconciles it in Notch Pay's dashboard. Do not claim it closed.
- **The conformance suite selects key material per rail** since #304
  (`submitted_key_material`, beside `documented_declines`), not by flow: a
  push rail is not necessarily addressed by our reference. The submit and
  duplicate cases assert that the key material round-trips, and
  `notchpay_a_duplicate_it_cannot_resolve_fails_closed` pins the `409` rule.
  The stub matches only `/payments/trx\.test_…`.
- **A `404` is `NotFound` only if the message is Notch Pay's "Payment not
  found".** Any other `404` is `Config`: a wrong base path, or an unknown
  Sync account. Reading those as "no such payment" would resubmit a live
  one.
- **Statuses** (RFC-0012 § 4): `pending`, `processing` and `incomplete` are
  `Pending`. `incomplete` is never a success. `complete`, `refunded` and
  `partialy-refunded` (Notch Pay's spelling) are `Succeeded`, with the
  amount and `amounts.total` check, recording `id` or, when absent (always,
  in the sandbox), `reference`. `failed` and `rejected` are
  `provider_error`. **`canceled` is `payer_declined` since #303**: the
  sandbox read it for the canceled-by-user number, the evidence RFC-0012 § 4
  named; the public key can cancel too, so it is evidence, not proof.
  `expired` and `abandoned` are `payer_timeout`. Anything else is
  `Malformed`. `PRODUCED_FAILURE_CODES` is four codes; nothing maps to
  `insufficient_funds`, because the read names no reason.
- **Callbacks**: the five `payment.*` events only; the charge is `data.trxref`
  if it is one of our UUIDs, else `data.reference` (since #304, after the
  measured payment shape; no webhook has been received to confirm the
  event's), and `ref_extra` is empty. The signature
  (`x-notch-signature`) is **not** checked, because the port passes no
  headers. The URL is registered with the account by hand. RFC-0013 is not
  built.
- **`refund` is `NotImplemented("notchpay::refund")`**, declared in
  `docs/status.md`. `supports_refunds` and `supports_partial_refunds` are
  `true` and `refund_destination` is `Origin`, because Notch Pay documents
  them. It is **the first shipping `Origin` rail**: the core refuses a
  `destination` with a `400` before the token. Do not answer `Unsupported`.
  That would be a claim about Notch Pay rather than vpay's gap.

## Configuration, and the trap in it

The boot rules are in `vpay_config::config::REQUIRED_RAIL_KEYS`, whose rows
gained `setting_values`, `currencies` and `test_marked_credentials` on
2026-10-20. **The livemode rule is on the keys**: in livemode a key
containing `test.` or `test_` refuses to boot, and outside livemode a key
with neither refuses to boot (`TEST_KEY_MARKERS`). **Real Notch Pay sandbox
keys carry `test.` (`pk…test.…`, `sk…test.…`), not the documented
`test_`**: measured on the maintainer's test-mode keys on 2026-10-20. The
rule read `test_` alone until then, which refused every genuine sandbox key
and admitted one in livemode. Do not narrow it back to `test_`.
`a_key_shaped_like_notch_pays_real_sandbox_keys_is_a_test_key` pins it. Notch Pay has one host for sandbox and production,
so the host rule cannot help. A fourth column, `livemode_acknowledgement`,
carries condition 4's `settings.accept_public_key_exposure`: required as
`true` in livemode, optional in a sandbox, and refused in either mode if it is
anything but `true` or `false`. Name fixture keys so a secret scanner cannot
read them as Stripe's (`public-key-test_0001` or `public-key-test.0005`,
never `pk_test_…`).

**`config/application.yml` carries the `notchpay` block commented out.** Boot
checks every _configured_ rail, `enabled` or not, so uncommenting it makes
`NOTCHPAY_PUBLIC_KEY` and `NOTCHPAY_PRIVATE_KEY` mandatory for every
deployment of the image. No compose stack enables the rail. `wiremock-notchpay`
exists with no host port. The proof that a Notch Pay config boots is
`vpay-server`'s
`a_notchpay_rail_is_warned_about_at_boot_and_its_keys_are_never_printed`,
which runs the real binary, and for livemode
`a_livemode_notchpay_rail_boots_only_acknowledged_and_still_warns`.

## Not built, and staying that way until its condition holds

Refunds as a call (RFC-0012 § 10 item 6). Model B / Sync / `X-Sync` (§ 10
item 9; not offered). `cm.mtn` / `cm.orange` (step 8). Webhook signature
verification (needs a port change; RFC-0014, vaam-apps/vpay#296, a draft,
designs it, so do not build it inside a Notch Pay change). Registering the webhook (RFC-0013).
Cards and Notch Pay's hosted Collect page (out of scope). The epic,
vaam-apps/vpay#291, stays open.

## Where to go next

- `docs/flows/adapter-notchpay.md`: the rail page, its Production risks and
  its To-confirm list.
- `docs/adr/0031-a-status-read-authenticated-by-a-server-side-key.md` and
  `docs/rfc/0012-notchpay-adapter.md`.
- `docs/security/threat-models/notchpay.md`.
- `docs/reference/vpay-notchpay.md`: the client crate, which says what Notch
  Pay answered and never what it means.
- `vpay-provider-adapters` skill: the port (`accepted_exposure` is its tenth
  method), the conformance suite, and the provider matrix.
