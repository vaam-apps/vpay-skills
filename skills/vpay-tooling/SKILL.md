---
name: vpay-tooling
description: How to build, test, lint and gate vpay — the just recipes that matter, the twelve verify gates and how contributors trip each one, what just ci actually runs, the toolchain pins that must move in lockstep, the compose stacks and ports, and the CI workflows. Load this before running any command in vpay, before adding a test binary or a migration, before bumping a toolchain pin, and whenever a gate fails and the message is not self-explanatory.
---

# vpay tooling and gates

> **Verified against vpay `f063ee96` (2026-09-15).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See VERSIONING.md.

`just` is the entry point for everything. `justfile` is ~4700 lines and about
95% comment — roughly 90 recipes buried in prose. Do not read it top to bottom.

```bash
just --list                 # every recipe, with its last comment line
just --show ci              # one recipe, body and dependencies, authoritative
just --evaluate             # every variable: ports, versions, expected counts
```

## The authority rule — read this first

**The recipe body wins. Over its own comment, over `AGENTS.md`, over
`CLAUDE.md`, over `package.json` prose.** This repository documents itself
exhaustively and the prose drifts faster than anything else in it. Always
confirm with `just --show <recipe>` or `just --evaluate` before you act on a
sentence.

Three live examples, measured 2026-09-16 on `master`:

| The prose says                                                                                                    | The recipe does                                                                                       |
| ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `audit-web`'s own comment and `AGENTS.md` both say it is **not** in `just ci`, and that `just ci` runs offline    | the `ci:` line lists `audit-web`. `just ci` hits the npm registry today and does **not** work offline |
| `audit-web`'s comment and `package.json`'s `//pnpm` block both say `--audit-level=high`, "moderate does not fail" | the body runs `pnpm audit --audit-level=moderate`. **Moderates fail**                                 |
| the `expected_suites` comment block discusses 42 test binaries in several places                                  | `just --evaluate` says `expected_suites := "48"`                                                      |

None of these is a bug to fix in passing — they are three separate branches'
comments that never got re-read. Treat them as a warning about the fourth one
you have not found yet.

The inverse habit is also house style here and worth copying: when you correct
a claim, say what it said before and date it. A silent correction tells a later
reader nothing about which sentences have been checked.

## `just ci` — the exact chain

`ci: fmt-check clippy verify test-rust test-doc verify-ignored lint-web
test-web audit-web deny`

In execution order:

1. `cargo fmt --all -- --check`
2. `pnpm exec prettier --check .`
3. `cargo clippy --workspace --all-targets -- -D warnings`
4. `just verify` — the twelve gates plus the `verify-docs` report
5. `cargo nextest run --workspace`
6. `cargo test --doc --workspace` — a **second runner**; nextest runs no doctests
7. `just verify-ignored` — the suite census
8. `just lint-web` — `pnpm -r typecheck`, then `pnpm -r lint`
9. `pnpm -r test`
10. `just audit-web` — two `pnpm audit` runs, needs the network
11. `cargo deny check`

Step 2 is prettier over the **working tree, not the index**, so an untracked
scratch `.ts`/`.md`/`.json` fails `just ci` in its first seconds. So does a
tree where `pnpm install` has not been run.

`AGENTS.md` requires `just ci` to pass locally before review.

## `just verify` — the twelve gates

Ten are `cargo xtask` subcommands; `check-schema` and `verify-ui` are shell in
the justfile. The order is chronological by landing date on purpose, so that
"check-schema is the seventh gate" — which four other files say — stays true
when a gate is added.

| Gate                  | Refuses                                                                                                                                              |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `verify-no-mocks`     | a test double reachable from a shipping binary through non-dev edges of the resolved cargo graph                                                     |
| `verify-status`       | a `NotImplemented` token not declared in `docs/status.md`, **or** a declaration whose token is gone — both directions                                |
| `verify-errors`       | a `pub` error type in `backends/crates` not implementing `Classify`; `anyhow` outside `backends/apps` (ADR-0011)                                     |
| `verify-sdk-parity`   | a parity-matrix claim naming a test that does not exist, a gap without a date and owner, or a row/method mismatch in **either** direction (ADR-0015) |
| `verify-links`        | a relative link in a tracked `*.md` that does not resolve to a **git-tracked** path                                                                  |
| `verify-npm-scope`    | a publishable SDK package misnamed, unpublishable, or a retired `@vpay/*` name outside the docs allowlist                                            |
| `check-schema`        | a missing `cratestack` CLI (**fails, never skips**), a schema with no `datasource`, or fewer than 15 model/enum declarations                         |
| `verify-serde`        | a serialisable type under `backends/crates` without `rename_all`, and a **stale exemption row** (ADR-0016 §3)                                        |
| `verify-repositories` | anything outside `vpay-db` naming a concrete repository implementation. No exemption mechanism (ADR-0016 §5)                                         |
| `verify-toolchain`    | `backends/Dockerfile`'s `FROM rust:` disagreeing with `rust-toolchain.toml`                                                                          |
| `verify-ui`           | eleven numbered greps: palette colours, daisyUI-4 classes, `!important`, `cva` in an app, computed `className`, and more                             |
| `verify-migrations`   | a `backends/migrations/*.sql` whose SHA-256 no longer matches `MANIFEST.sha256`                                                                      |

`verify-docs` runs last and is **not** a gate — it exits 0 whatever it finds.
That is deliberate: the cheapest way to pass a doc-ratio gate is to delete the
`# Errors` and `# Panics` sections.

Full detail, and how contributors trip each one, in
[references/gates.md](references/gates.md).

## The day-to-day loop

```bash
just fmt                 # cargo fmt --all + prettier --write .
just test-rust           # cargo nextest run --workspace       (needs Docker)
just test-doc            # cargo test --doc --workspace
just test-web            # pnpm -r test
just docs-check          # verify-status + verify-links, offline, seconds
just verify-ui           # the UI greps alone, no build
just ci                  # before you open anything
```

Subsets: `cargo nextest run -p <crate>`, `cargo nextest run -E 'test(name)'`,
`pnpm --filter <pkg> test`. Every recipe inventory detail is in
[references/recipes.md](references/recipes.md).

## What each thing needs

| Need                           | Which commands                                                                                                                                                                                                                                                                            |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Docker**                     | `just test-rust` and anything running the workspace suite — `vpay-tests-integration`, `vpay-tests-conformance`, `vpay-db`, `vpay-server`, `vpay-testkit` all start real `postgres:16-alpine` / WireMock testcontainers. Also `just up`, `demo-*`, `test-e2e`, `stripe-compat`, `sdk-live` |
| **Network**                    | `just audit-web` (so `just ci`), `docs-check-citations`, `helm-check` (kubeconform fetches schemas), `test-storybook` (≈115 MB Chromium, first run), image builds                                                                                                                         |
| **`gh`, authenticated**        | `just docs-check-citations` only. It **fails** rather than skipping when `gh` is missing                                                                                                                                                                                                  |
| **A real Postgres you manage** | nothing. Tests bring their own container; `just up` brings up the dev one on `:5432`                                                                                                                                                                                                      |
| **Extra binaries on PATH**     | `cratestack` (`check-schema`), `helm` + `kubeconform` (`helm-check`), `jq` (`verify-ignored`), `openssl` (`gen-e2e-signing-key`)                                                                                                                                                          |

Gates outside `just verify` and (on paper) outside `just ci`: `audit-web`,
`test-storybook`, `helm-check`, `docs-check-citations`. CI runs all four on
every PR, so a green local run does not predict them.

## Lockstep bumps

Change one of these and the other must move in the **same commit**.

| If you change                     | You must also change                                                                                                                        | Enforced by                                 |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| `rust-toolchain.toml` `channel`   | `backends/Dockerfile`'s `FROM rust:<version>-alpine…`                                                                                       | `verify-toolchain`                          |
| `justfile`'s `cratestack_version` | `Cargo.toml`'s `cratestack = { package = "cratestack-pg", version = "=…" }`                                                                 | nothing — read `CLAUDE.md`, then check both |
| `.nvmrc`                          | `package.json` `engines.node` (`.npmrc` sets `engine-strict=true`)                                                                          | `pnpm install --frozen-lockfile` exits 1    |
| add or drop a **test binary**     | `expected_suites` in `justfile`                                                                                                             | `verify-ignored`                            |
| add a **migration**               | run `just migrations-manifest`                                                                                                              | `verify-migrations`                         |
| add a **helm guard**              | its name in `helm-check`'s `expected_guards`, its values file in `deploy/helm/vpay/ci/guards/`, and the `fail` in `templates/_validate.tpl` | `helm-check`                                |
| add a `NotImplemented` token      | its declaration in `docs/status.md`                                                                                                         | `verify-status`                             |
| add an SDK method                 | its row in `docs/sdks/parity.md`                                                                                                            | `verify-sdk-parity`                         |

`verify-toolchain` exists because the drift was measured: with `channel` moved
to 1.98.0 and the Dockerfile left on 1.95.0, the whole of `just ci` was green.
CI reads the channel out of `rust-toolchain.toml` with an **anchored `sed`** in
five jobs, so reformatting that line breaks CI.

## Further reading

- [references/gates.md](references/gates.md) — the twelve gates in depth,
  including `verify-ui`'s eleven numbered checks and the both-directions gates.
- [references/recipes.md](references/recipes.md) — the full recipe inventory,
  the compose layering and ports, the demo-stack traps, and the `.env` rules.
- [references/ci.md](references/ci.md) — the three workflows, `changes` path
  gating, the readiness-poll lore, the `audit-web` retry loop, and the
  testcontainers serialisation in `.config/nextest.toml`.
