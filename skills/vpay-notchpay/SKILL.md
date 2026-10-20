---
name: vpay-notchpay
description: "The Notch Pay adapter (`vpay-adapter-notchpay`, rail `notchpay`, 2026-10-20) — a push rail for MTN MoMo and Orange Money through one aggregator account, admitted under ADR-0031 because its status read takes the key Notch Pay calls PUBLIC. Covers the production risk that admission accepts and the boot warning that states it, the six conditions an agent must not weaken, why every mapping is 'To confirm' (nothing has ever called Notch Pay), the read addressed by vpay's own reference with an empty `ref_extra`, the 404 rule, the amount check on every success, the `test_` livemode key rule, and why `refund` is a declared `NotImplemented` token. Load before changing `backends/crates/vpay-adapter-notchpay`, `backends/crates/vpay-notchpay`, its WireMock mappings or its config."
---

# Notch Pay — the aggregator push rail

> **Verified against vpay `b8be107` (2026-10-20)**, `master` before the
> adapter, **and read on `bf96e50`**, the head of vpay's
> `claude/notchpay-adapter` branch, which carries everything below and is
> not on `master` yet. VERSIONING.md says stamp a commit that stays on
> `master`, so the stamp is the base. Restamp when the branch merges. Until
> then, check that `backends/crates/vpay-adapter-notchpay` exists in your
> tree before trusting a line here. See
> [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`backends/crates/vpay-adapter-notchpay/` — `lib.rs`, `mapping.rs`,
`tests.rs`. **No `wire.rs` and no `token.rs`**: the wire is the separate
client crate `backends/crates/vpay-notchpay`, and Notch Pay uses static keys.

## Read this before you believe anything else on this page

**Nothing has ever called Notch Pay.** Not the sandbox, not production. No
test-mode key was available when the adapter was written, so RFC-0012's
delivery step 1 (the sandbox session) and step 5 (the first test-mode call)
**did not happen**. Every status, error and reference rule below is
transcribed from Notch Pay's `openapi.yaml` 2.1.0 and its guides, and every
one is "To confirm" (`docs/flows/adapter-notchpay.md` § "To confirm with
Notch Pay"). The conformance row proves the adapter against a WireMock stub
transcribed from the same pages, and nothing about Notch Pay.

`docs/status/providers.md`'s Collect and Status read cells say
**"stub-only; never called"**. They move to "proven; sandbox" plus a date
**only** with a dated page under `docs/status/verification/` recording a
test-mode call read to a terminal status. Do not move them on anything less.

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
6. **The question to Notch Pay stays open** (RFC-0012 § 10 item 1). A yes
   moves the rail to ADR-0031's first limb and retires the warning.

## How the adapter behaves, and why

- **`Push`, `cm.mobile`, XAF only.** Boot refuses any other `channel` or
  currency. `cm.mtn` and `cm.orange` wait for ADR-0029 D6 (RFC-0012 step 8):
  configured for a whole deployment, either fails every payer on the other
  network.
- **Submit is two calls**: `POST /payments` (amount, `XAF`, `phone` as
  `+`-prefixed E.164, our UUID as `reference`, optional `description`, and
  nothing else of the payer's), then `POST /payments/{reference}` with
  `channel` and `data.phone`, which prompts the payer. A `409` on the first
  is the crash-retry case: read, then process only if `pending`.
- **Every call addresses the payment by vpay's reference, and `ref_extra` is
  always empty.** That is the specification's form. Notch Pay's guide uses
  its own `trx.…` reference instead, and which one the live path wants is
  RFC-0012 § 10 item 2, unanswered. RFC-0012 § 3 had designed "prefer Notch
  Pay's reference when held". The build dropped it, and RFC-0012 records the
  change. Do not "fix" `ref_extra` to carry an id: the conformance suite's
  push-rail duplicate case requires it empty.
- **A `404` is `NotFound` only if the message is Notch Pay's "Payment not
  found".** Any other `404` is `Config`: a wrong base path, or an unknown
  Sync account. Reading those as "no such payment" would resubmit a live
  one.
- **Statuses** (RFC-0012 § 4): `pending`, `processing` and `incomplete` are
  `Pending`. `incomplete` is never a success. `complete`, `refunded` and
  `partialy-refunded` (Notch Pay's spelling) are `Succeeded`, with the
  amount check. `failed`, `canceled` and `rejected` are `provider_error`.
  `canceled` is **not** `payer_declined`, because the public key can cancel
  too. `expired` and `abandoned` are `payer_timeout`. Anything else is
  `Malformed`. `PRODUCED_FAILURE_CODES` is three codes, because the read
  names no reason.
- **Callbacks**: the five `payment.*` events only, `data.reference` must be
  one of our UUIDs, and `ref_extra` is empty. The signature
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
containing `test_` refuses to boot, and outside livemode a key without
`test_` refuses to boot. Notch Pay has one host for sandbox and production,
so the host rule cannot help. A fourth column, `livemode_acknowledgement`,
carries condition 4's `settings.accept_public_key_exposure`: required as
`true` in livemode, optional in a sandbox, and refused in either mode if it is
anything but `true` or `false`. Name fixture keys so a secret scanner cannot
read them as Stripe's (`stub-public-key-test_0001`, never `pk_test_…`).

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
