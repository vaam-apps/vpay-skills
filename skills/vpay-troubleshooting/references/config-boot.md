# Boot failures, exit codes, and configuration refusals

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

For the configuration model itself see **vpay-ops**. This page is the lookup:
the process will not start, or a request is refused, and you need the cause.

## Read the exit code first

| Exit | `sysexits.h`     | Means                                                               |
| ---- | ---------------- | ------------------------------------------------------------------- |
| `78` | `EX_CONFIG`      | **fix your configuration / your deploy.** Not transient.            |
| `69` | `EX_UNAVAILABLE` | **wait for Postgres.** A rail or storage dependency is unreachable. |
| `64` | `EX_USAGE`       | a caller-shaped problem (`InvalidRequest`, `Idempotency`).          |
| `77` | `EX_NOPERM`      | `Authentication` / `Forbidden`.                                     |
| `1`  | —                | anything the chain gave nothing classifiable for.                   |

That split is a contract an operator and a supervisor can hold: "`78` means the
operator forgot something, `69` means wait for Postgres." Preserve it. A
2026-09-10 fix (issue #87) existed only because a missing `--database-url` used
to exit `1` — `main` raised a bare `anyhow` error there and the exit-code
classifier had nothing to read. It now raises
`StartupError::MissingDatabaseUrl` **at both call sites**, and there are two
subprocess tests, one per mode, each of which fails if _its_ site reverts.

## `78` — which of the six?

The boot order is "cheapest hard failure first", so the stage tells you the
cause. In order (`docs/flows/configuration.md`):

1. Signal handlers and the rustls crypto provider.
2. **Load and validate the YAML.** Missing or invalid `--config` / `VPAY_CONFIG`
   → `78`, before any network round trip.
   2b. **Join the YAML's rails against the linked adapters.** A configured rail
   with no linked adapter is `ConfigError::ProviderWithoutAdapter` → `78`, still
   before any network round trip.
3. **Load the RS256 signing key** and derive the issuer. A missing flag, a
   missing file, a file that is not an RSA private key, or a key under 2048 bits
   each → `78`, **before the database connection** — which is why all of those
   are covered by subprocess tests that need no Docker. "A server that cannot
   sign can mint no merchant token; it would bind a port, answer `/healthz` with
   a cheerful 200, and refuse every real request."
4. **Connect to Postgres and run migrations.** Unreachable → `69`. A modified
   applied migration → `78` (`DbError::Migrate` is `Configuration`, not
   `Storage`: "a broken migration is a deploy problem, not a transient one") —
   see [migrations-database.md](migrations-database.md).
5. Reconcile `currencies` and `providers` from configuration, one transaction,
   advisory-locked. Fatal on failure.
6. Announce the key as active in `oauth_signing_keys`. Fatal: "a process whose
   key is not published mints tokens nothing can verify."
7. Sweep expired client-assertion `jti`s and `idempotency_keys` once —
   non-fatal, boot-time stopgaps.
8. Bind the listener, **then** build the token validator (it needs the port
   actually bound, and validates over loopback against this process's own
   `/v1/oauth/jwks.json`).
9. Serve.

## `78` from an unresolved `${VAR}`

**An unresolved placeholder is fatal, never an empty string** — "an empty
subscription key otherwise fails much later and much more confusingly."

Seven variables are referenced by `config/application.yml` and must be supplied
by **every** deployment, **in every mode**:

```text
MTN_SUBSCRIPTION_KEY  MTN_API_KEY  MTN_API_USER
ORANGE_MERCHANT_KEY   ORANGE_CLIENT_ID  ORANGE_CLIENT_SECRET
MERCHANT_WEBHOOK_SECRET
```

Including `serve`, **which never delivers a webhook** but loads and validates
the same document — which is why `backends/apps/vpay-server/tests/cli.rs` had to
start setting `MERCHANT_WEBHOOK_SECRET`.

The list **grows as features land.** To regenerate it for the image you are
deploying:

```bash
grep -o '${[A-Z_]*}' config/application.yml | sort -u
```

`.env.example` lists them. `compose.e2e.yml` sets all seven on both services,
and `compose.demo.yml` layers on top and inherits them.

## `78` from a missing required rail key

`vpay_config::config::REQUIRED_RAIL_KEYS` refuses to boot a rail missing a key
its adapter cannot work without. **A present-but-empty value counts as
missing.**

- MTN: `settings.target_environment`, `settings.api_user`,
  `credentials.subscription_key`, `credentials.api_key`.
- Orange: `credentials.merchant_key`, `credentials.client_id`,
  `credentials.client_secret`.

`target_environment` is required because MTN answers a wrong one with a **500
whose body says `NOT_ALLOWED_TARGET_ENVIRONMENT`** — "a failure that reads as
'the rail is broken' rather than 'our YAML is'."

This table is a provider-code match outside an adapter crate, which ADR-0002
forbids; it is a deliberate, recorded interim (ADR-0012) and it carries a
comment saying so.

## A flag that is accepted and does nothing, and one that is refused

`--public-base-url` was removed 2026-09-03. The two halves behave differently:

- **`--public-base-url` now fails at parse time**, loudly, on an unknown
  argument. A chart or compose file that still sets the flag breaks on upgrade.
- **`VPAY_PUBLIC_BASE_URL` does not fail.** clap reads an environment variable
  only for a flag it declares, so a stale variable in a Secret or a compose file
  is **silently ignored: no error, no effect, and nothing tells anyone.**

The issuer comes from **`deployment.public_base_url` in the YAML**
(`vpay_api::op::issuer_for` → `{public_base_url}/v1/oauth`). YAML is the only
spelling that ever worked.

Same shape: `VPAY_OAUTH_SIGNING_KEY_FILE` in the environment is **ignored rather
than refused** on the worker, deliberately — "a shared env block must not
`CrashLoopBackOff` a worker". Both _flag_ spellings are refused (clap does not
declare it on the `worker` subcommand; `vpay_config::cli`'s `SERVE_ONLY_FLAGS`
catches `vpay-server --oauth-signing-key-file … worker`, which clap's derive
cannot express and which "parsed and was read by nothing for the length of one
review pass").

`--shutdown-grace-seconds` is a partial exception of the same kind: `serve`
actually bounds its drain with it; `worker` accepts and logs it and does nothing
with it.

## Checkout and origin refusals

| Symptom                                                              | Cause                                                                                                                                                                |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /v1/checkout/sessions` answers `checkout_not_configured`       | The deployment has no `checkout.public_base_url`. It answers **500**, not 503 — recorded as a maintainer's decision, not an oversight.                               |
| `create` is a `400` naming `payment_intent`                          | The intent must be `requires_payment_method`, have no charge, and have no other open session. **One open session per intent is a database index**, not just a check. |
| The payer's page says "invalid link"                                 | The `url` lost its `#fragment` — something copied it through a redirect, a logger, or a link with a `?query` and no fragment.                                        |
| The embedded iframe is an empty box                                  | The page never painted, so no `vpay:resize` arrived. **Browser console, not server log.**                                                                            |
| The framed page says "This page will not load here"                  | The framing origin is not in that merchant's `checkout_origins`.                                                                                                     |
| `session.url` resolves to nothing                                    | `checkout.public_base_url` and the page's actual origin disagree. **Nothing logs a port; compare the two by hand.**                                                  |
| The order never turns `paid` although vpay says `succeeded`          | Your webhook endpoint. Check the signing secret on both sides, and that you are verifying the **raw** bytes.                                                         |
| Every token request is `invalid_client` and everything else is right | Your assertion's `aud`. It must be **vpay's own token endpoint**, not the URL you POST to, if your server reaches vpay by an internal name.                          |

### `checkout_origins` must be spelled canonically

A `checkout_origins` entry becomes `Content-Security-Policy: frame-ancestors`.
`https://Shop.example` and `https://shop.example:443` pass every other rule and
are then **dropped silently** by the page's own filter, leaving the merchant
unable to embed with nothing to read. So `ConfigError::NonCanonicalCheckoutOrigin`
refuses at boot rather than normalising — lower-cased host, IDNA to ASCII,
default port elided — "so the file and the running policy stay the same
document, and the message names what to write instead."

An empty `checkout_origins` list is the default and means **no site may embed
that merchant's page** — fail-closed, "the only shape that is safe to default
to."
