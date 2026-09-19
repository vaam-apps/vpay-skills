---
name: vpay-ops
description: Configuration, deployment and observability for vpay — the YAML layering and its boot-time refusals, the exit-code contract, ProviderHost and credential redaction, the single vpay-server binary and its three modes, the three published images, the Helm chart and what it deliberately does not render, and the metrics that exist but have never been scraped. Load this before changing config, the Dockerfiles, the chart, or anything that claims vpay runs somewhere.
---

# vpay ops

> **Verified against vpay `0799a8d2` (2026-09-18).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

## Read this before you write a sentence about running vpay

**No pod has ever run this.** Not a real cluster, not kind, not minikube.
`deploy/helm/vpay/README.md` says so in its second paragraph and again at
length in its Status section.

**No Prometheus has ever scraped a vpay process.** Every metric named below is
emitted on a real seam and **no series has ever been watched over time**. The
chart's `ServiceMonitor` and `PrometheusRule` are both `false` by default, and
no alert rule in this repository has ever been evaluated against real data.

**No `kubectl` or `helm` command in any runbook has been run against a
cluster** (`docs/runbooks/README.md`). `docs/runbooks/migrations.md` is the one
page whose central SQL _is_ executed — by the test suite, against a real
Postgres — and even its `kubectl` commands have been run nowhere.

**No backup of any vpay database has ever been taken**, no restore has ever
been performed, and no restore drill has ever run. ADR-0013 is `Proposed` and
every number in it is proposed, not measured.

What _has_ run: compose stacks, CI runners, and — once, on 2026-09-15 — a
single MTN **sandbox** push (`docs/runbooks/live-sandbox-test.md`). Describing
anything else in the present tense is the most damaging thing you can write
here.

## The configuration model

ADR-0003: administration is YAML in git, not an admin UI. "Configuration gets
code review, diffs and rollback. Changes take a deploy rather than a click,
which is the point."

Five steps, in order (`docs/flows/configuration.md`):

1. Load `config/application.yml`, overlay `config/application-{profile}.yml`
   from the same directory.
2. Resolve `${VAR}` against the process environment. **An unresolved
   placeholder is fatal, never an empty string** — "an empty subscription key
   otherwise fails much later and much more confusingly."
3. Validate.
4. Reconcile `currencies` and `providers` into the database in **one
   transaction**, taking `pg_advisory_xact_lock lock_keys::CONFIG_RECONCILE` so
   N replicas booting at once cannot interleave. (The _config hash_ half of
   this step is **not implemented** — nothing records or compares one.)
5. Only then bind the port.

**A validation failure exits non-zero without serving traffic.** "A payment
gateway that boots half-configured is worse than one that does not boot."

### A profile selects a config FILE, never a code path

ADR-0003, and it is the rule most likely to be violated by accident:

- A sandbox **environment** — yes. Two deployments, each with its own config
  file and its own database. Same binary, **same image digest**.
- A sandbox **mode** — no. No `if (sandbox)`, no `NODE_ENV` check, no
  profile-selected bean.

Because Spring Boot is the idiom being borrowed, the trap it makes easy is
named outright: **`@Profile("!prod")`, `@ConditionalOnProperty` on business
logic and profile-specific bean overrides are all `if (sandbox)` wearing a
dependency-injection costume.** Profiles may select _values_; never _beans that
behave differently_.

Depth: [references/configuration.md](references/configuration.md) — the env-var
table, the boot order, `ProviderHost`, and every rule that refuses to boot.

## The exit-code contract

| Exit                      | Means                                                 |
| ------------------------- | ----------------------------------------------------- |
| **78** (`EX_CONFIG`)      | fix your configuration or your deploy. Not transient. |
| **69** (`EX_UNAVAILABLE`) | wait for Postgres (or a rail).                        |
| **64** (`EX_USAGE`)       | a caller-shaped problem.                              |
| **77** (`EX_NOPERM`)      | authentication / forbidden.                           |

Boot is ordered **cheapest hard failure first**, so the stage tells you the
cause: YAML → adapter join → signing key → Postgres → reconcile → announce key
→ sweep → bind → validator → serve. A missing `--config`,
`--oauth-signing-key-file` **or** `--database-url` is `78` in both `serve` and
`worker`. That last one used to exit `1`; issue #87 made it typed on
2026-09-10, specifically so "`78` means the operator forgot something, `69`
means wait for Postgres" is a rule an operator can hold.

## One binary, three modes

Since 2026-09-07 (issue #77) there is exactly one shipping binary,
`vpay-server`. It was two (`vpay-server`, `vpay-worker-bin`).

| Mode              | What it does                                                                                                                                                                                                                          | Signing key       |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| _(no subcommand)_ | serves the API on `--bind`                                                                                                                                                                                                            | **required**      |
| `worker`          | runs `vpay_worker::run_loop` — claims `poll_charge`, polls the rail on a ladder, commits charge + intent + one event in a single transaction, reaps stranded leases at boot and on a timer, prints one `job loop gauge` line a minute | refused, two ways |
| `staff add`       | creates a dashboard account                                                                                                                                                                                                           | not used          |

**Both serving modes call a payment rail**: the server when a merchant confirms
an intent, the worker on the poll ladder. Whether that rail is MTN, Orange or a
WireMock stub **is a line in `config/application.yml`** — never a code branch.

`cargo run -p vpay-server -- --help` and `… -- worker --help` are more
trustworthy than any table if the two disagree.

## Three images

`ghcr.io/vaam-apps/vpay-{server,dashboard,checkout}`.

| Image            | Base             | Contents                                                                                |
| ---------------- | ---------------- | --------------------------------------------------------------------------------------- |
| `vpay-server`    | `scratch`        | one static musl binary + `config/` baked at `/config`. **Runs both backend workloads.** |
| `vpay-dashboard` | `node:22-alpine` | the Next standalone server                                                              |
| `vpay-checkout`  | `node:22-alpine` | vpay's own hosted/embedded payment page                                                 |

**`ghcr.io/vaam-apps/vpay-worker` is retired and has not been deleted.** The
package is frozen at the last `:edge` and `sha-<40 hex>` `release.yml` pushed
to it (2026-09-04) and nothing publishes to it any more. Deleting it needs a
`delete:packages` scope this repository has never held, so it is recorded as a
task with an owner rather than as a fact. **The failure mode that leaves: a
package nothing publishes to still answers a `docker pull`, so a deployment
that has not been upgraded keeps silently running 2026-09-04's worker binary
against a newer server image and a newer schema.**

### What `FROM scratch` costs you

No shell, no package manager, no writable path (ADR-0004). Three consequences
that show up everywhere:

- **No `HEALTHCHECK` and no `kubectl exec` debugging** — everything is observed
  from outside the container.
- **Configuration is baked in**, so a config change is a rebuild or an overlay
  mount, never an edit in place.
- **`USER 65532:65532` is a raw UID**, because `scratch` has no `/etc/passwd`.

Depth: [references/deployment.md](references/deployment.md) — the chart, the
Secrets, the overlay `subPath` rule, the routing defects, and — added
2026-09-19, never run — how `release.yml` publishes the chart itself to GHCR.

## Observability, in one paragraph

Two listeners. `/livez` and `/metrics` on `--observability-bind` (default
`0.0.0.0:9090`, on **both** modes); `/healthz` stays on `--bind` and stays the
readiness probe. Thirteen metric names, all emitted, **none ever scraped**.
Traces are deliberately absent. Depth:
[references/observability.md](references/observability.md).

## The MTN currency note — do not misread it

`config/application.yml` puts `mtn_momo` on **`currency: EUR`**, not XAF, and
`application-sandbox.yml` inherits it. The reason is in the file: **MTN's real
sandbox rejects XAF and accepts EUR only.**

The demo overlay (`.e2e/application-demo.yml`, written by `just gen-demo-keys`)
puts _both_ rails on XAF, because `/v1` refuses a confirm whose intent currency
is not the rail's settlement currency and one currency for both rails is what
makes the demo shop's MTN button payable.

> **Do not read that as "MTN accepts XAF".** It does not.

Configuration either way — never a code branch. Amounts against a EUR profile
are notional; no FX happens and none is implied.
