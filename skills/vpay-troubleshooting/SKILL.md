---
name: vpay-troubleshooting
description: A symptom index for vpay — the exact error text or observed behaviour, its real cause, and the fix. Covers clippy and toolchain failures, rootless Docker and testcontainers flakes, pnpm/Cypress/prettier traps, the migration checksum rule that bricks every database, demo and compose port collisions, boot exit codes, Helm and deployment defects, and the repository's own known-wrong documentation. Load this the moment a command fails, a test is red, or a document contradicts what you just measured.
---

# vpay troubleshooting

> **Verified against vpay `f063ee96` (2026-09-15).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

Scan the index. Most entries are a lookup, not a read.

Two habits before anything else:

1. **A failure on a loaded machine is usually the machine.** Diagnose it,
   record it with evidence, and re-run — do not count it as a result and do not
   "fix" the tree. See [references/docker-testcontainers.md](references/docker-testcontainers.md).
2. **When prose and a machine-checked source disagree, the machine-checked
   source wins**, and you fix the prose in the same commit. See
   [references/known-wrong-docs.md](references/known-wrong-docs.md).

## Symptom index

| Symptom                                                                 | Page                                                         |
| ----------------------------------------------------------------------- | ------------------------------------------------------------ |
| clippy rejects `unwrap`/`expect`/`panic`/`dbg!`                         | inline, below                                                |
| `verify-toolchain` fails; `FROM rust:` vs `rust-toolchain.toml`         | [build-toolchain](references/build-toolchain.md)             |
| `cannot produce proc-macro … target does not support these crate types` | [build-toolchain](references/build-toolchain.md)             |
| `check-schema: FAIL — needs the 'cratestack' CLI on PATH`               | [build-toolchain](references/build-toolchain.md)             |
| a `schemas/vpay.cstack` edit breaks `cargo build`                       | [build-toolchain](references/build-toolchain.md)             |
| `cargo deny` advisory on `rsa` / RUSTSEC-2023-0071                      | [build-toolchain](references/build-toolchain.md)             |
| testcontainers cannot reach a Docker daemon                             | inline, below                                                |
| `failed to create a container: Timeout error` at ~120 s                 | inline, below                                                |
| `RootlessKit PortManager.AddPort(): bind: address already in use`       | [docker-testcontainers](references/docker-testcontainers.md) |
| `OCI runtime exec failed … outside of container mount namespace root`   | [docker-testcontainers](references/docker-testcontainers.md) |
| container reports `unhealthy` while serving correctly                   | [docker-testcontainers](references/docker-testcontainers.md) |
| `pnpm install` fails on Node version                                    | [node-web](references/node-web.md)                           |
| `sh: 1: tsc: not found` inside a **Rust** test                          | [node-web](references/node-web.md)                           |
| `ERR_PNPM_RECURSIVE_EXEC_FIRST_FAIL Command "prettier" not found`       | [node-web](references/node-web.md)                           |
| prettier rewrote pasted evidence inside a markdown code fence           | [node-web](references/node-web.md)                           |
| Cypress has no binary / `CYPRESS_INSTALL_BINARY`                        | [node-web](references/node-web.md)                           |
| `EADDRINUSE 127.0.0.1:4181`, or `ECONNREFUSED` on a port no spec chose  | [node-web](references/node-web.md)                           |
| every Storybook story renders unstyled and every test passes            | [node-web](references/node-web.md)                           |
| `migration <n> was previously applied but has been modified`            | inline, below                                                |
| `vpay-shop` dies in `zen migrate deploy`                                | [migrations-database](references/migrations-database.md)     |
| `invalid_client: Client authentication failed` from a demo stack        | [demo-compose](references/demo-compose.md)                   |
| two demo stacks collide on a port                                       | [demo-compose](references/demo-compose.md)                   |
| `500` / `write_matched_no_row` during a confirm                         | [demo-compose](references/demo-compose.md)                   |
| `ClientBuilder::build()` panics on a rustls `CryptoProvider`            | [demo-compose](references/demo-compose.md)                   |
| a binary exits `78` at boot and you do not know which of six causes     | [config-boot](references/config-boot.md)                     |
| an unresolved `${VAR}`, or a flag that is silently ignored              | [config-boot](references/config-boot.md)                     |
| `checkout_not_configured`, a dead `session.url`, "invalid link"         | [config-boot](references/config-boot.md)                     |
| the embedded iframe is an empty box / "This page will not load here"    | [config-boot](references/config-boot.md)                     |
| the pod exits 78 naming a file it can see and cannot open               | [deployment](references/deployment.md)                       |
| every rail callback 404s while every object reports healthy             | [deployment](references/deployment.md)                       |
| a profile typo boots happily on placeholder credentials                 | [deployment](references/deployment.md)                       |
| a server crash-loops after a signing-key rollback                       | [deployment](references/deployment.md)                       |
| the dashboard shows an em dash, no rail column, no page count           | [dashboard](references/dashboard.md)                         |
| a document says X and the code/recipe says Y                            | [known-wrong-docs](references/known-wrong-docs.md)           |

## The four you will hit first

### clippy complains about `expect` in a test

**Cause:** you are outside a `#[cfg(test)]` module. `clippy.toml`'s
`allow-*-in-tests` exemptions are scoped to clippy's own notion of test code, so
a helper in the main module body is production code.

**Fix:** move it inside `#[cfg(test)]`, or handle the error. Note
`allow-dbg-in-tests = false` — **`dbg!` is denied even in tests.**

### testcontainers cannot reach a Docker daemon

**Symptom:** `Error: postgres:16-alpine container starts … Caused by: … (Connect)`,
usually on the first Postgres-backed test.

**Cause:** `testcontainers` talks to `/var/run/docker.sock` by default and your
daemon is rootless.

**Fix:**

```bash
DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock cargo nextest run --workspace
```

These suites **fail loudly and never skip** — that is deliberate, so a green
run is a real one. There is no "Docker not available, skipping" path to find.

### `failed to create a container: Timeout error` at ~120 s

**This is almost never your change.** It is testcontainers' 120 s create
deadline losing to host contention, and it is the most-recorded flake in the
repository — it shows up in a dozen `docs/plans/*/opus-review.md` files, always
on a _different, untouched_ test, never as an assertion failure.

**Causes, in the order they actually occur:** another agent or worktree driving
the same daemon (load average 9–25); dead `Created`-state containers piling up
(250 of them were once found, accumulating over days); `fs.inotify.max_user_instances`
saturated.

**Fix:**

```bash
docker ps -a --filter label=org.testcontainers.managed-by=testcontainers
# remove Created-state debris older than ~30 min, then re-run
```

One measured pair, from `docs/plans/exp53-ui-package-notes/opus-review.md`: five
consecutive failures at load average 25 with another worktree's e2e stack up;
the same suite then passed **1696/1696 at load average 3 with nothing about the
Rust changed**.

**Report it as an environment failure with the evidence, not as a result.** The
house phrasing is "Not counted as a result".

### `migration <n> was previously applied but has been modified`

**Symptom:** every binary exits 78 at boot. Every database that applied the
original is stuck.

**Cause:** a migration file that had already shipped was edited. `sqlx::migrate!`
stores a SHA-384 of each file's **whole bytes, comments included**.

**This really happened, and it is why the gate exists.** PR #39 reflowed **one
line of comment** inside `backends/migrations/0028_create-checkout-sessions.sql`
after it shipped — an old package name in a note. No SQL changed. Every stack
brought up in that window stopped booting, **with every job in CI green.**

**Fix:** never edit a shipped migration. To correct one, write a **new**
migration. To repair a database already in the broken state, follow
`docs/runbooks/migrations.md` — and read
[references/migrations-database.md](references/migrations-database.md) first,
because the first draft of that runbook told operators to write the **original**
checksum, which changes nothing and reports success.

## The rest

- [references/build-toolchain.md](references/build-toolchain.md) — rustc, musl, cargo, CrateStack, `deny.toml`.
- [references/docker-testcontainers.md](references/docker-testcontainers.md) — rootless Docker, container-start races, nextest groups.
- [references/node-web.md](references/node-web.md) — pnpm, Node, tsc, ESLint, prettier, Cypress, Storybook.
- [references/migrations-database.md](references/migrations-database.md) — the immutability rule, the manifest, stale volumes.
- [references/demo-compose.md](references/demo-compose.md) — `just demo`, two stacks on one machine, the confirm/poll race.
- [references/config-boot.md](references/config-boot.md) — exit codes, `${VAR}`, checkout and origin refusals.
- [references/deployment.md](references/deployment.md) — Helm, Secrets, overlays, ingress, probes.
- [references/dashboard.md](references/dashboard.md) — what looks wrong and is not.
- [references/known-wrong-docs.md](references/known-wrong-docs.md) — the correction shapes, the census, and which source wins.
