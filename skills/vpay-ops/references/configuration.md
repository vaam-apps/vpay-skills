# Configuration: the CLI layer, the YAML, and what refuses to boot

_Verified against vpay `87166eaf` (2026-10-09) for step 7 of "The boot sequence", and against `a33aac61` (2026-09-29) for "The ten environment variables"; the Notch Pay sentences under `callbacks.register` were read against `82a68f4e` (2026-10-23) plus the pull request that adds the 2026-10-23 Notch Pay sandbox verification page, restamped with its merge, and its default per rail against `9499f09d` plus the pull request that adds `docs/status/verification/2026-10-23-notchpay-defaults-and-wrong-network.md`, restamped with its merge; those sentences were re-read against `4df5e0d2` (2026-10-24), the merge of vaam-apps/vpay#364, for what Notch Pay's first delivered webhooks changed; the rest of this page was last read at `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

ADR-0003. Flow doc `docs/flows/configuration.md`; crate reference
`docs/reference/vpay-config.md`.

## The CLI / env layer

Both serving modes parse one `clap` CLI (`vpay_config::cli`) where every option
auto-resolves from an environment variable, **with an explicit flag beating its
env var**. The modes share one `CommonArgs` via `#[command(flatten)]`, "so they
cannot drift on a flag's name, env var, or default."

| Flag                                      | Env var                       | Default        |
| ----------------------------------------- | ----------------------------- | -------------- |
| `--bind` _(serve only)_                   | `VPAY_BIND`                   | `0.0.0.0:8080` |
| `--database-url`                          | `DATABASE_URL`                | none           |
| `--profile`                               | `VPAY_PROFILE`                | `sandbox`      |
| `--config`                                | `VPAY_CONFIG`                 | none           |
| `--observability-bind`                    | `VPAY_OBSERVABILITY_BIND`     | `0.0.0.0:9090` |
| `--oauth-signing-key-file` _(serve only)_ | `VPAY_OAUTH_SIGNING_KEY_FILE` | none           |
| `--log-filter`                            | `RUST_LOG`                    | `info`         |
| `--log-format` (`json`\|`text`)           | `VPAY_LOG_FORMAT`             | `json`         |
| `--shutdown-grace-seconds`                | `VPAY_SHUTDOWN_GRACE_SECONDS` | `25`           |
| `--worker-concurrency` _(worker only)_    | `VPAY_WORKER_CONCURRENCY`     | `4`            |

`--version` reports the workspace version. **Run `--help` on the binary** if
this table and the binary ever disagree — the flow doc says so of itself.

**There is no env var for the issuer.** It is `deployment.public_base_url` in
the YAML (`vpay_api::op::issuer_for` → `{public_base_url}/v1/oauth`).
`--public-base-url` was accepted, parsed and read by nothing, and was **removed**
2026-09-03 rather than wired. Passing the flag now fails at parse time;
**setting `VPAY_PUBLIC_BASE_URL` does not fail and does nothing** — clap reads
an env var only for a flag it declares.

### The signing key is a file, deliberately

`--oauth-signing-key-file` names the RS256 private key (PKCS#8 PEM) the merchant
OP signs `/v1` access tokens with. It is **serve-only**: the worker issues no
tokens, "so mounting the Secret into it would widen its blast radius for no
capability", and `the_worker_is_not_handed_the_signing_key` pins that.

It is a **file, never an env value**, "because that is how a Kubernetes Secret
reaches a pod". Generate one offline:

```bash
cargo xtask gen-signing-key --out ./secrets   # writes oauth-signing-key.pem
```

The _path_ is deliberately visible in `Debug` output
(`the_signing_key_path_stays_visible_in_debug_output`) — "a path is not a
secret, and 'which file did it try' is the first thing an operator needs" —
while the file's contents never enter the CLI types at all. Nothing prints,
logs or stores the private key.

`--shutdown-grace-seconds` is honoured by `serve` only: it races the in-flight
drain against a clock of that length via `serve_with_bounded_drain` and exits
non-zero if the clock wins. The worker accepts and logs it and does nothing with
it. **Neither mode's handling of the _timeout_ case is covered by a test.**

## The boot sequence, cheapest hard failure first

1. Install SIGINT/SIGTERM handlers and the rustls crypto provider.
2. Load and validate the YAML. Missing or invalid → **78**, before any network
   round trip.
   2b. Link the adapters this binary was built with and **join the YAML's rails
   against them** (`adapters_by_code` + `vpay_api::v1::boot::boot_seeds`). A
   configured rail with no linked adapter is
   `ConfigError::ProviderWithoutAdapter` → **78**, still before any network
   round trip.
3. Load the RS256 signing key and derive the issuer from
   `deployment.public_base_url`, "so the key stamps the same `iss` the OP
   advertises". A missing flag, a missing file, a non-RSA file, or a key under
   2048 bits each → **78**, **before the database connection** — which is why
   all three are covered by subprocess tests that need no Docker. "A server that
   cannot sign can mint no merchant token; it would bind a port, answer
   `/healthz` with a cheerful 200, and refuse every real request."
4. Connect to Postgres and run migrations. Unreachable → **69**.
5. Reconcile `currencies` and `providers` (`vpay_db::ConfigReconcile::reconcile`,
   one transaction, advisory-locked). Fatal.
6. Announce the key as active in `oauth_signing_keys`
   (`ensure_active_signing_key`, advisory-locked). Fatal: "a process whose key
   is not published mints tokens nothing can verify."
7. ~~Sweep expired client-assertion `jti`s and expired `idempotency_keys` once —
   both non-fatal boot-time stopgaps, "because there is no worker job loop to
   schedule either properly". The worker sweeps nothing.~~ **Corrected
   2026-10-09: there is no step 7.** Both boot sweeps were removed in Step 4
   (2026-09-03) and the worker's hourly `sweep_expired` job runs them (the `jti`
   delete keeps a spent row five minutes past `exp`, ADR-0028; see
   `vpay-reconciler`). `vpay-server` sweeps nothing at boot, `vpay-server worker`
   sweeps on its schedule, and a failing sweep is a failed job, not a refusal to
   boot. The numbering below is kept so that a citation of a later step by number
   still resolves. vpay's own `docs/flows/configuration.md` still lists the boot
   sweep as step 7 at `87166eaf`; trust the code.
8. Bind the listener, **then** build the token validator — it needs the port
   actually bound (`--bind 127.0.0.1:0` is a real configuration) and validates
   over loopback against this process's own `/v1/oauth/jwks.json`.
9. Serve.

**Configuration is the authority; the tables are the mirror.** A rail present in
the seed is upserted; a rail **absent** from it is set `enabled = false` rather
than deleted, "because a rail that has ever taken money must stay nameable
forever."

Capabilities come from the **adapter** (`flow`, `supports_refunds`,
`supports_partial_refunds`, `delivers_callbacks`, `requires_ip_allowlist`); the
`enabled` flag comes from the YAML. `providers.display_name` is **derived** from
the code (`mtn_momo` → `Mtn Momo`) rather than configured, because the port has
no `display_name()` — "a placeholder that is honest about being one".

## `ProviderHost`

`backends/crates/vpay-config/src/config.rs`. The resolved per-rail
configuration: `host` (`url` + `label`), `currency`, `settings`, `credentials`,
`enabled`, optional `callback_url`, and — since 2026-10-22 (RFC-0013) —
optional `callbacks: { register: ensure | check | off }`, whose `register`
is itself optional since 2026-10-23, and which since 2026-10-23 (RFC-0014)
also takes `verify: on | off`.

**It hand-writes its own `Debug`, which redacts `credentials` and prints
`settings` in full.** A derived `Debug` would format a rail secret into any log
line that printed a `Config`. The redaction is tested, including that the
`[redacted]` marker itself is not dropped.

**This is why Orange's `merchant_key` stays under `credentials`.** It is not a
bearer secret — it travels in the request body — but it is per-merchant
material, and "moving it [to `settings`] would only make it log-visible"
(Step 3's decision 4). The rule to take from it: **`settings` is what you are
willing to see in a log; `credentials` is everything else.**

**`enabled` absent means enabled** (`a_provider_with_no_enabled_line_is_enabled`).
`enabled: false` keeps the rail's host and credentials loaded — so charges
already on it stay observable — while `reconcile` writes `enabled = false` into
`providers`, and a disabled rail cannot be named on a new intent or a confirm.

**`effective_callback_url`** derives `{public_base_url}/provider/{code}/callback`
unless `providers[].callback_url` overrides it. The **effective** value — not
just the override — is put through `validate_host`, so a livemode deployment
cannot hand a live rail a plaintext or stub callback host.

**`callbacks.register`** says what the worker's `ensure_callbacks` job and
`vpay-server callbacks ensure` may do about the rail's callback endpoint at
the provider: `ensure` lists and creates vpay's when it is missing, `check`
lists and never creates, `off` never asks. **Unset — absent block, or `{}` —
it is the rail's own default, which its adapter declares**
(`ProviderAdapter::default_callback_registration`): `ensure`, except
**`notchpay`, which is `off` since 2026-10-23** (the maintainer's "Default to
`off` for Notch Pay"); the worker at start, and `vpay-server callbacks
ensure`, log one `info` line for a rail an unset `register` leaves `off`. _(Until then an absent block meant `ensure` on every
rail.)_ It applies
only to an **enabled** rail whose adapter declares
`CallbackUrlMode::Registered` — `notchpay` today. On a `PerRequest` rail
(`mtn_momo`, `orange_money`) the block is refused at boot,
`ConfigError::CallbacksOnPerRequestRail` → **78**, whatever it says. A
misspelt key inside it is refused (`deny_unknown_fields`), never silently
ignored. `config/application-notchpay-sandbox.yml` sets `check`, so a
worker started against the real sandbox never creates an endpoint by
itself. **On Notch Pay, `ensure` may buy nothing (2026-10-23):** the
sandbox delivered nothing in 21 minutes to an endpoint created through the
same `POST /webhooks` call, with a public URL, for a reason still unknown;
`GET`, `PUT` and `DELETE` by id answered `404`, so it can be removed only in
Notch Pay's dashboard. That is why the default became `off`. An operator who
wants Notch Pay's callbacks registers vpay's URL in the dashboard and leaves
`register` unset; that route delivers (four signed webhooks to a capture
endpoint registered there on 2026-10-24, none yet to a running vpay), and
the endpoint's Hash Key is the signing secret below. _(This said the sandbox
"stored `"url": null`" for the API-created endpoint, and called the
dashboard route "unmeasured too", until 2026-10-24: the listing shows `null`
for a dashboard endpoint that delivers too.)_ Set `check`
instead to have `VpayCallbackEndpointUnconfirmed` watch it, and then the
endpoint must carry vpay's description (`docs/runbooks/callback-endpoints.md`).
Charges settle by polling either way.

**Callback signing secrets** (2026-10-23, RFC-0014). A rail whose adapter
declares `CallbackCheck::Signature` (`notchpay` only) checks every callback
against `credentials.callback_signing_secret`, plus
`credentials.callback_signing_secret_previous` while a rotation is in flight.
Two names the **port** fixes, so boot needs no rail's vocabulary. A rail
credential like any other: a `${VAR}` in livemode (Notch Pay's is
`${NOTCHPAY_HASH_KEY}`, the endpoint's Hash Key from its dashboard), redacted
by `Debug`, never printed. **`callbacks.verify` is an acknowledgement, not a
switch**: a configured secret is checked whatever it says. Boot step 4
(`boot_seeds`) refuses, each **78** and naming the setting, never a value:

| condition                                                                  | error                                   |
| -------------------------------------------------------------------------- | --------------------------------------- |
| an adapter's `callback_check` without `delivers_callbacks`                 | `IncoherentCallbackCheck`               |
| livemode, a `Signature` rail, no secret, and no `callbacks.verify: off`    | `CallbackSigningSecretMissing`          |
| a signing secret (either name) or `verify: off` on a rail that checks none | `CallbackSigningSettingUnread`          |
| `verify: off` beside a configured secret                                   | `CallbackVerifyOffWithSecret`           |
| `callback_signing_secret_previous` without a non-blank current secret      | `CallbackSigningPreviousWithoutCurrent` |
| a non-blank secret (either name) with surrounding whitespace               | `CallbackSigningSecretWhitespace`       |

Outside livemode, no secret boots and `vpay-server serve` logs a `WARN`; with
`verify: off` it warns at every boot in either mode. A blank value is no
secret. The last two rows are the 2026-10-23 security review's, in both
modes: a value created from a file that ends in a newline is refused rather
than keying the HMAC on it (write the Secret with `--from-literal` or
`printf %s`), and a previous secret alone is a rotation whose new key is
missing. **Rotating:** put the new value in `callback_signing_secret` and the
old one in `_previous`, roll, update the provider's dashboard; `serve` warns
at every boot while `_previous` is set. Remove it once
`vpay_provider_callbacks_total{verdict="verified_previous"}` stops
increasing: until then whoever holds the old secret can still pull this
rail's polls forward. A stale secret shows as
`vpay_provider_callbacks_total{verdict="invalid"}` with no `verified`, and
costs callback latency, never money: the poll ladder still settles every
charge.

**`to_provider_config(&Deployment)`** is "the only place a
`vpay_provider::ProviderConfig` is built from configuration, so the server and
the worker cannot disagree about a rail's callback URL or currency." The two
timeouts are constants, not YAML knobs: "no deployment has asked for a different
budget and **a knob nobody sets is a knob nobody has tested**."

## Rules that refuse to boot

| Rule                                                                                                   | Why                                                                                           |
| ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Every merchant's rail host appears in that rail's allowlist                                            | the host allowlist, checked before the FK                                                     |
| Every referenced provider exists and is enabled                                                        | a typo fails at boot, not at first payment                                                    |
| Every merchant registration carries a unique `merchant_id`                                             | the `/v1` tenancy boundary has no foreign key behind it                                       |
| Currency exponent matches the canonical table                                                          | **a 100× amount bug is otherwise silent**                                                     |
| `livemode` ⇒ every host is `https://`                                                                  |                                                                                               |
| **`livemode` ⇒ no host labelled `wiremock`/`stub`/`mock`/`localhost`**                                 | see below                                                                                     |
| `livemode` ⇒ secrets come from `${}`, not literals                                                     | stops a real key reaching git                                                                 |
| `checkout.public_base_url` is a well-formed origin, `https://` under livemode                          | every payer link vpay mints is built on it                                                    |
| every `checkout_origins` entry is an `https://` origin, no path, no duplicate, **spelled canonically** | it becomes `frame-ancestors`; a non-canonical spelling is dropped **silently** by the browser |
| `checkout_origins` without a `checkout.public_base_url`                                                | there is no page for those origins to frame                                                   |
| `merchant_clients[].display_name` non-blank, ≤ 80 chars                                                | it is painted into a heading on a phone-sized page                                            |

### The livemode stub-host rule is the load-bearing one

The flow doc calls it "**the most valuable rule here**", and the reason is the
architecture:

> It is what makes "the code cannot tell a stub from a real rail" safe to live
> with.

ADR-0006 requires that a stub rail be a WireMock **host in configuration**,
reached over HTTP exactly as a real rail is — so by construction no code path
distinguishes them. The only thing standing between a livemode deployment and a
stub is this boot rule. Do not weaken it, and do not add a code-side check
instead: the code-side check is the thing ADR-0006 exists to prevent.

### `REQUIRED_RAIL_KEYS`

Refuses to boot a rail missing a key its adapter cannot work without. **A
present-but-empty value counts as missing.** MTN:
`settings.target_environment`, `settings.api_user`,
`credentials.subscription_key`, `credentials.api_key`. Orange:
`credentials.merchant_key`, `credentials.client_id`,
`credentials.client_secret`.

**Notch Pay, since 2026-10-20** (vaam-apps/vpay#295, on
`master` since `357c825`): `settings.channel`, `credentials.public_key`,
`credentials.private_key`. The row also refuses a `channel` other than
`cm.mobile`, a currency other than `XAF`, and a key whose test-key marker
(`test.` or `test_`; real sandbox keys carry `test.`) disagrees with
`deployment.livemode` _(corrected 2026-10-20: this said "`test_` marker"
alone, which the maintainer's real test keys do not carry)_ (`RailKeyLivemodeMismatch`, which names
the key and never prints it). **In livemode it also refuses the rail unless
`settings.accept_public_key_exposure` is `true`**
(`PublicKeyExposureNotAcknowledged`, ADR-0031 condition 4); a sandbox may
omit it. Those are four new **columns** of the same table, not a second
table. Every boot of a deployment configuring it also logs a `WARN` about the
public key's exposure (ADR-0031), acknowledged or not. The block ships
**commented out** in `config/application.yml`, because uncommenting it makes
`NOTCHPAY_PUBLIC_KEY` and `NOTCHPAY_PRIVATE_KEY` mandatory for every
deployment, and, since 2026-10-23, `NOTCHPAY_HASH_KEY` too, which the block
writes as `credentials.callback_signing_secret` (RFC-0014; the Helm values
comment says "three keys"). **`cm.mobile` is now only the channel of a charge confirmed by the
rail code `notchpay`** (2026-10-23): a charge routed from `mtn_wallet` or
`orange_wallet` is sent `cm.mtn` or `cm.orange` per charge, so the setting
stays `cm.mobile`.

This is a provider-code match outside an adapter crate — which ADR-0002 forbids
— and it is a deliberate, recorded interim (ADR-0012) carrying a comment saying
so. It moves behind the port the day `ProviderAdapter` grows a
`required_settings()` hook. **It is the only sanctioned one; do not add a
second.**

## `routing:` — static weights for the router (since 2026-10-23)

Optional. With no block, every rail weighs `100` for each payment-method type
it serves, and ties go to the lower rail code — so out of the box
`mtn_wallet` routes to `mtn_momo` and `orange_wallet` to `notchpay` when both
rails are configured. That is a tie-break, not a choice; set weights if it is
not the one you want:

```yaml
routing:
  weights:
    - rail: notchpay
      method_type: mtn_wallet # optional; omitted = every type the rail serves
      weight: 300 # 0 ..= 1000; highest total wins
```

The most specific entry wins (rail and type, then rail, then `100`). `0`
ranks a rail last, it does not disable it; `enabled: false` does. **The
weight is the only score component today, and the formula
(`provisional-1`) is provisional**. The maintainer chose on 2026-10-23
(RFC-0008 § R8.1): the estimated fee as the next score component; the
completion rate reported per rail and type, for you to read and set this
block by hand, never scored ("Go with (a)"). Neither is built. Health is not a component (R8.3: it reports only), and the
block stays the operator's until RFC-0009 (R8.2). Refusals, all exit `78`: `UnknownRoutingRail`,
`UnknownMethodType` (reserved `card`/`bank_transfer` say so),
`RoutingWeightTooLarge`, `DuplicateRoutingWeight` (all in `validate_all`),
and at boot step 4 `RoutingWeightOnAnUnservedType` (e.g. `orange_money` with
`mtn_wallet`) and `RailServesNoMethodType` (a linking mistake, not YAML).
Every routed charge's decision, with each rail's score breakdown or exclusion
reason, is in `routing_decisions` (`SELECT * FROM routing_decisions WHERE
charge_id = …`); it is on no `/v1` object.

## The ten environment variables

~~The seven environment variables~~ — **corrected 2026-09-29: ten, as of vpay
`a33aac61`.** vpay's `docs/flows/configuration.md` made the same correction on
2026-09-20 ("Seven … Corrected 2026-09-20: 10") and has gated it since by
`cargo xtask verify-doc-counts` (marker `count:env-vars config/application.yml`).
The three that were missing here are the Disbursements credentials that arrived
with the refund path on 2026-09-16.

```bash
grep -o '${[A-Z_]*}' config/application.yml | sort -u
```

As of 2026-09-29 that prints eleven lines; one is the literal `${VAR}` that the
file's own `#` comments use as an example (comments never reach the resolver),
so the real set is ten: `MERCHANT_WEBHOOK_SECRET`, `MTN_API_KEY`,
`MTN_API_USER`, `MTN_DISBURSEMENT_API_KEY`, `MTN_DISBURSEMENT_API_USER`,
`MTN_DISBURSEMENT_SUBSCRIPTION_KEY`, `MTN_SUBSCRIPTION_KEY`, `ORANGE_CLIENT_ID`,
`ORANGE_CLIENT_SECRET`, `ORANGE_MERCHANT_KEY`. **The list grows as features land**
— re-derive it for the image you are deploying rather than copying this one.

All ten are needed by **both** modes, including `serve`, which never delivers
a webhook but loads and validates the same document. They are set on both
services in `compose.e2e.yml` (which `compose.demo.yml` layers on) and listed in
`.env.example`.

## The checkout page reads its own two files, and they are not these

`frontends/apps/checkout` reads a `branding.yaml` and a `config.yaml` (examples
in `config/checkout/`), by TypeScript in a different process. Nothing in
`vpay-config` reads them. As of 2026-09-16 **no container has been started with
the Helm chart supplying them, because the chart cannot yet.**

## Config changes and in-flight payments

There is no hot reload (ADR-0003), by design. A config change is a deploy. See
`docs/flows/configuration.md` § "Config changes and in-flight payments" before
changing a rail's host or currency on a deployment with live charges.
