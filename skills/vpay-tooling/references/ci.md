# CI, and the flakiness that was engineered out

_Verified against vpay `f063ee96` (2026-09-15). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Three workflows in `.github/workflows/`: `ci.yml`, `docs.yml`, `release.yml`.
Re-read 2026-09-16.

The shape to internalise: **CI calls `just` recipes rather than copies of their
commands**, so the gate and the local check cannot drift. Where a job does
inline a command (`cargo fmt`, `cargo clippy`, `cargo nextest run` in the
`rust` job) the recipe is a one-liner and the duplication is visible.

## `ci.yml`

Triggers: `push` to **`master`** and every `pull_request`. It said `main` for a
while, which meant the workflow ran only on pull requests and **nothing ever
verified what actually landed**.

`concurrency: ci-<PR number or ref>`, `cancel-in-progress: true` — a newer push
cancels the older run, because nothing in this workflow writes anything
external and a cancelled run leaves nothing half-done.

`permissions: contents: read, pull-requests: read` at workflow level, and no
job overrides it. `pull-requests: read` is there for `dorny/paths-filter`.

`CYPRESS_INSTALL_BINARY` is deliberately **not** set at workflow level. It used
to be `0` there, which also applied to the `e2e` job — whose entire point is to
run Cypress — and that single line failed two runs with "Command 'cypress' not
found". The jobs that do not need the binary opt out individually.

Third-party actions are pinned to full commit SHAs with the tag in a trailing
comment; first-party `actions/*` use tags.

### The jobs

| Job                              | Gated by                 | Runs                                                                                                                                                                                                                         |
| -------------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `changes`                        | —                        | `dorny/paths-filter` with `fetch-depth: 0`. Outputs `rust`, `deny`, `web`, `e2e`, `deploy`                                                                                                                                   |
| `verify` — _self-checks_         | **nothing. Always runs** | the twelve gates in `just verify`'s exact order, then `cargo xtask verify-docs`                                                                                                                                              |
| `rust`                           | `changes.rust`           | `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo nextest run --workspace`, `just test-doc`, `just verify-ignored`                                                               |
| `deny` — _supply chain_          | `changes.deny`           | `EmbarkStudios/cargo-deny-action@v2`, `command: check`                                                                                                                                                                       |
| `web`                            | `changes.web`            | `just audit-web`, `just fmt-check-web`, `just lint-web`, `pnpm -r test`, `just build-storybook`, `just test-storybook`                                                                                                       |
| `e2e` — _e2e (compose)_          | `changes.e2e`            | Cypress install + verify, `just build-checkout-browser`, `just gen-demo-keys`, `docker compose up -d --build`, three readiness polls, `just demo-staff`, `just e2e-specs`, then the stripe-compat, Rust and Node live suites |
| `deploy` — _deploy (helm chart)_ | `changes.deploy`         | `just helm-check`                                                                                                                                                                                                            |

**The `verify` job has no `needs` and no `if`.** Every other job can be skipped
by the path filter; the self-checks cannot. If you are wondering why a
docs-only PR still ran something, that is it — and `just fmt-check-web` runs
prettier over the whole repository, so the `web` filter deliberately includes
`**/*.md` too.

### Where the pins come from

Five jobs resolve the compiler from the toolchain file rather than a floating
`@stable`:

```
sed -n 's/^channel = "\(.*\)"/\1/p' rust-toolchain.toml
```

That `sed` is **anchored**. Reformatting the `channel` line in
`rust-toolchain.toml` breaks CI in five places at once.

The `verify` and `rust` jobs resolve the CrateStack CLI version the same way —
`just --evaluate cratestack_version` — and **error out rather than falling back
to the latest release** if it comes back empty. There is one copy of that
number (`justfile`'s `cratestack_version`), and CI follows it.

The `deploy` job installs `kubeconform` by curl with an explicit
`sha256sum -c` against a literal digest, and pins `helm` to `v3.16.1`.

### Environment worth knowing

The `rust` job sets `VPAY_REQUIRE_NODE: 1`. A webhook signature-parity test
shells out to `node` against the built Node SDK; without the flag it degrades
into a silent skip, which is how a suite goes green while proving nothing. With
`VPAY_REQUIRE_NODE=1` a missing or unusable `node` **fails** that test rather
than skipping it. It is never `Ok(skip)` — the flag only changes the message.

The `rust` job therefore also installs Node and builds
`@vaam-apps/vpay-sdk` before running the Rust suite.

## The readiness-poll lore

`vpay-server`'s runtime image is `FROM scratch` (ADR-0004): no shell, no curl,
no wget, no package manager, and the only executable in the image is
`/vpay-server`. A shell-form `CMD-SHELL` healthcheck cannot run there, and
adding a second static binary just to answer one would reintroduce exactly the
attack surface that ADR removed on purpose.

Consequences, in order:

1. `compose.e2e.yml` gives `vpay-server` **no `healthcheck:`**.
2. `docker compose up --wait` therefore reports it merely **started**, not
   ready. `dashboard`'s `depends_on` is the best ordering available, not a
   guarantee.
3. **Readiness is observed from outside.** CI polls
   `http://localhost:8080/healthz` for a 200, up to 90 × 1 s; then the
   dashboard's `/` (followed to `/login`) up to 60 s; then `vpay-checkout`
   `:3080/healthz` and `vpay-shop` `:3001/healthz`, 90 s each.
4. Every poll failure emits `::error::`, then `docker compose ps` and
   `docker compose logs --no-color <service>` for the service that never
   answered.
5. `if: always()` writes the whole stack's logs to `compose.log` and uploads it
   as the `compose-logs` artifact, then `down -v`.

`just test-e2e` and `just demo-up` do the same thing locally with a 120 s
deadline and `logs --tail 80`, and add a hint the CI job does not: **exit 78 in
that log means a config or CLI prerequisite is missing**, not a crash.

The honest fix — a `--healthcheck` self-check mode on the `vpay-server` binary,
reusing the one executable already in the image to call its own `/healthz` over
loopback — is named in `compose.e2e.yml` and **has not been built** as of
2026-09-16.

## The `audit-web` retry loop, and why it exists

`just audit-web` runs `pnpm audit` twice: `--prod` first (the production
dependency graph only — what a merchant actually receives, and what
`frontends/Dockerfile` ships), then the whole workspace including dev
dependencies. Two runs and not one, because a failure in each is different
news, and every advisory this repository has had on the JS side has been in dev
tooling.

Each run is wrapped in a bounded retry:

- 4 attempts, 90 s apart, with `npm_config_fetch_timeout` raised to 180 000 ms
  from pnpm's 60 s default.
- It retries **only** when the output carries the registry's own failure
  signatures: `ERR_SOCKET_TIMEOUT`, `ERR_PNPM_AUDIT_BAD_RESPONSE`,
  `ECONNRESET`, `EAI_AGAIN`, `FetchError`.
- An **advisory** fails immediately, with pnpm's report already printed. Exit 1
  = advisory. Exit 2 = `REGISTRY UNREACHABLE`.

**The origin:** on 2026-09-04 npm's audit endpoint
(`/-/npm/v1/security/audits`) timed out or answered 503 for about two hours and
failed seven consecutive CI runs of a branch whose lockfile had not changed.
Every one of those runs printed `ERR_SOCKET_TIMEOUT`; none printed a finding.

**What the loop deliberately does not do is pass when the registry is down.** An
unreachable audit is an audit that did not run, and this is a payment system.

Two things to correct if you read the prose around it: the recipe runs
`--audit-level=moderate`, not `high` (so moderates fail), and `audit-web` **is**
in the `ci:` recipe today despite its own comment and `AGENTS.md` saying it is
not.

Nothing in `ci.yml` itself retries anything.

## Testcontainers serialisation — `.config/nextest.toml`

This is the one real flake in the Rust suite and it was engineered out rather
than retried. Read the file before changing test parallelism.

**The symptom:** `vpay-tests-integration` starts a fresh `postgres:16-alpine`
container per `#[tokio::test]`. Under nextest's default parallelism (one thread
per CPU) 13+ of them race to start and port-map a container at once, which
intermittently fails with testcontainers' own "container port" error. Measured:
the identical test passes 3/3 in isolation at ~1 s each, and fails roughly 1
run in several full-workspace runs. On rootless Docker the same race shows up
as `RootlessKit PortManager.AddPort(): bind: address already in use`.

**The configuration:**

```toml
[test-groups]
postgres-containers = { max-threads = 1 }
```

applied by an override whose filter is
`package(vpay-tests-integration) | package(vpay-tests-conformance) |
package(vpay-db) | package(vpay-server) | package(vpay-testkit)`.

The group is "our tests that ask Docker for a port", **not** "our tests that
use Postgres" — `vpay-tests-conformance` starts a WireMock container per rail
per test and contends for the same random host ports.

**Two rejected alternatives, both with the measurement that rejected them:**

- **`max-threads = 3`** still produced a "container port" failure roughly 1 run
  in 14 across ~30 full-workspace runs. A real improvement over unbounded, but
  not the zero-failures bar this needs. The binding constraint is not the
  host's CPU count but the Docker VM's own allocation, which is what
  testcontainers' port mapping goes through.
- **A shared container in a `once_cell`/`OnceLock` `static`** leaks the
  underlying Docker container forever, because **Rust does not drop statics at
  process exit** — so `ContainerAsync`'s `Drop` never runs. Proven, not
  assumed: 8+ consecutive runs left hundreds of orphaned `postgres:16-alpine`
  containers still `Up`, which drove the Docker VM into memory pressure and
  OOM-killed an unrelated container.

The cost of `max-threads = 1` is wall-clock, not correctness: each test keeps
its own container and its already-correct `Drop` cleanup; only the _starts_ are
serialised.

A `package(...)` naming a package that does not exist is a **hard nextest
config error**, not a no-op, so that filter list cannot silently rot when a
crate is renamed.

`nextest-version = { required = "0.9.59" }` is a bare **minimum**, not a
comparator — a future 0.9.x or 0.10.x is not rejected by it.

## `docs.yml`

`pull_request` on `docs/**`, `README.md`, `AGENTS.md`, `CLAUDE.md`. One job,
`status-is-current` / "status.md matches the code": resolve the channel from
`rust-toolchain.toml`, run `cargo xtask verify-status`. `permissions: contents:
read` — it never comments, labels or pushes.

## `release.yml`

`namespace` (derive the registry namespace) → `build` (a matrix over images ×
platforms, amd64 on `ubuntu-latest` and arm64 on `ubuntu-24.04-arm`, uploading
digests as artifacts) → `merge` (`docker buildx imagetools create` to make the
manifest list, then keyless `cosign` signing via GitHub OIDC).

It groups concurrency by `github.ref` with **`cancel-in-progress: false`** —
deliberately the opposite of `ci.yml`. A half-cancelled `imagetools create`
leaves a tag pointing at whichever manifest list won, so every run there is
allowed to finish.

`backends/Dockerfile` builds the **builder's own host triple**, which is why
`.cargo/config.toml` spells out `+crt-static` for both musl triples: a
static-linking decision that applied on only one of two published
architectures would not be a decision (ADR-0014).

`just release-dry-run` is the local rehearsal — it builds all three images for
the host arch only and then runs `just helm-check`.
