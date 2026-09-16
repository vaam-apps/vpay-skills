# Configuration: the CLI layer, the YAML, and what refuses to boot

_Verified against vpay `7a79684e` (2026-09-16). Version-sensitive claims
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
7. Sweep expired client-assertion `jti`s and expired `idempotency_keys` once —
   both non-fatal boot-time stopgaps, "because there is no worker job loop to
   schedule either properly". The worker sweeps nothing.
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
`enabled`, optional `callback_url`.

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

This is a provider-code match outside an adapter crate — which ADR-0002 forbids
— and it is a deliberate, recorded interim (ADR-0012) carrying a comment saying
so. It moves behind the port the day `ProviderAdapter` grows a
`required_settings()` hook. **It is the only sanctioned one; do not add a
second.**

## The seven environment variables

```bash
grep -o '${[A-Z_]*}' config/application.yml | sort -u
```

As of 2026-09-16: `MERCHANT_WEBHOOK_SECRET`, `MTN_API_KEY`, `MTN_API_USER`,
`MTN_SUBSCRIPTION_KEY`, `ORANGE_CLIENT_ID`, `ORANGE_CLIENT_SECRET`,
`ORANGE_MERCHANT_KEY`. **The list grows as features land** — re-derive it for
the image you are deploying rather than copying this one.

All seven are needed by **both** modes, including `serve`, which never delivers
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
