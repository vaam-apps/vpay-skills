---
name: vpay-tooling
description: How to build, test, lint and gate vpay — the just recipes that matter, the twelve verify gates and how contributors trip each one, what just ci actually runs, the toolchain pins that must move in lockstep, the compose stacks and ports, and the CI workflows. Load this before running any command in vpay, before adding a test binary or a migration, before bumping a toolchain pin, and whenever a gate fails and the message is not self-explanatory.
---

# vpay tooling and gates

> **Verified against vpay `991e9825` (2026-09-17).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`just` is the entry point for everything. `justfile` is ~4700 lines and about
two-thirds comment (2026-09-16) — the recipes are buried in prose. Do not read
it top to bottom; `just --list` counts them for you.

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

A live example, measured 2026-09-16 on `master`:

| The prose says                                                                   | The recipe does                                  |
| -------------------------------------------------------------------------------- | ------------------------------------------------ |
| the `expected_suites` comment block discusses 42 test binaries in several places | `just --evaluate` says `expected_suites := "48"` |

**Two more sat here until 2026-09-16 and have been fixed** — `audit-web`'s
comment and `AGENTS.md` said it was not in `just ci` and that `just ci` ran
offline, and `audit-web`'s comment and `package.json` said `--audit-level=high`.
Both had been wrong since 2026-09-11, when issue #103 changed the recipe and
nobody re-read the prose around it. They are named here rather than deleted
because they are what this rule was worth: reading the recipe found two false
statements about what CI runs, five days after the change that falsified them.

What is true now: `audit-web` **is** in `just ci`, it runs at
`--audit-level=moderate` so moderates fail, and **`just ci` therefore needs the
network.**

A stale comment is not a bug to fix in passing — these are separate branches'
comments that never got re-read. Treat them as a warning about the next one
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

| Gate                  | Refuses                                                                                                                                                                                                         |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `verify-no-mocks`     | a test double reachable from a shipping binary through non-dev edges of the resolved cargo graph                                                                                                                |
| `verify-status`       | a `NotImplemented` token not declared in `docs/status.md`, **or** a declaration whose token is gone, **or** (since 2026-09-16) a `<rail>::…` token carried outside that rail's adapter crate — three directions |
| `verify-errors`       | a `pub` error type in `backends/crates` not implementing `Classify`; `anyhow` outside `backends/apps` (ADR-0011)                                                                                                |
| `verify-sdk-parity`   | a parity-matrix claim naming a test that does not exist, a gap without a date and owner, or a row/method mismatch in **either** direction (ADR-0015)                                                            |
| `verify-links`        | a relative link in a tracked `*.md` that does not resolve to a **git-tracked** path                                                                                                                             |
| `verify-npm-scope`    | a publishable SDK package misnamed, unpublishable, or a retired `@vpay/*` name outside the docs allowlist                                                                                                       |
| `check-schema`        | a missing `cratestack` CLI (**fails, never skips**), a schema with no `datasource`, or fewer than 15 model/enum declarations                                                                                    |
| `verify-serde`        | a serialisable type under `backends/crates` without `rename_all`, and a **stale exemption row** (ADR-0016 §3)                                                                                                   |
| `verify-repositories` | anything outside `vpay-db` naming a concrete repository implementation. No exemption mechanism (ADR-0016 §5)                                                                                                    |
| `verify-toolchain`    | `backends/Dockerfile`'s `FROM rust:` disagreeing with `rust-toolchain.toml`                                                                                                                                     |
| `verify-ui`           | eleven numbered greps: palette colours, daisyUI-4 classes, `!important`, `cva` in an app, computed `className`, and more                                                                                        |
| `verify-migrations`   | a `backends/migrations/*.sql` whose SHA-256 no longer matches `MANIFEST.sha256`                                                                                                                                 |

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

Gates outside `just verify` **and** outside `just ci`: `test-storybook`,
`helm-check`, `docs-check-citations`. CI runs all three on every PR, so a green
local run does not predict them. `audit-web` is outside `just verify` but
**inside `just ci`** (since 2026-09-11, issue #103) — which is why `just ci`
needs the network.

## Lockstep bumps

Change one of these and the other must move in the **same commit**.

| If you change                     | You must also change                                                                                                                                                                   | Enforced by                                 |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| `rust-toolchain.toml` `channel`   | `backends/Dockerfile`'s `FROM rust:<version>-alpine…`                                                                                                                                  | `verify-toolchain`                          |
| `justfile`'s `cratestack_version` | `Cargo.toml`'s `cratestack = { package = "cratestack-pg", version = "=…" }`                                                                                                            | nothing — read `CLAUDE.md`, then check both |
| `.nvmrc`                          | `package.json` `engines.node` (`.npmrc` sets `engine-strict=true`)                                                                                                                     | `pnpm install --frozen-lockfile` exits 1    |
| add or drop a **test binary**     | `expected_suites` in `justfile`                                                                                                                                                        | `verify-ignored`                            |
| add a **migration**               | run `just migrations-manifest`                                                                                                                                                         | `verify-migrations`                         |
| add a **helm guard**              | its name in `helm-check`'s `expected_guards`, its values file in `deploy/helm/vpay/ci/guards/`, and the `fail` in `templates/_validate.tpl`                                            | `helm-check`                                |
| add a `NotImplemented` token      | its declaration in `docs/status.md`                                                                                                                                                    | `verify-status`                             |
| add an SDK method                 | its row in `docs/sdks/parity.md`                                                                                                                                                       | `verify-sdk-parity`                         |
| `flutter-toolchain.toml`'s pin    | nothing — no `verify-*` gate reads it, unlike `rust-toolchain.toml`'s `verify-toolchain`. Its `channel` field is `[user-branch]`, not a clean channel pin, by the file's own admission | nothing                                     |

`verify-toolchain` exists because the drift was measured: with `channel` moved
to 1.98.0 and the Dockerfile left on 1.95.0, the whole of `just ci` was green.
CI reads the channel out of `rust-toolchain.toml` with an **anchored `sed`** in
five jobs, so reformatting that line breaks CI.

## Flutter, `.agents/skills/`, and a dead MSISDN convention

Three facts with no home above, kept short here on purpose — depth is in
[references/recipes.md](references/recipes.md).

- **Six `*flutter*` recipes** (`install-flutter`, `analyze-flutter`,
  `test-flutter`, `test-flutter-web`, `test-flutter-e2e`,
  `test-flutter-emulator`) build and test `vpay_checkout_flutter`
  (`vpay-sdks` owns what the package does). All six share one
  `_flutter-preflight` and **none is in `just ci`** (D-M3) — every count
  quoted for this package is a human running the recipe by hand.
  `flutter-toolchain.toml` pins Flutter 3.47.2 / Dart 3.13.2, unenforced
  (see the Lockstep table above).
- **`.prettierignore`'s `.agents/skills/` entry** excludes vpay's own
  vendored agent skills (installed by issue #188 from
  `cratestack/cratestack-skills`, hash-pinned in vpay's `skills-lock.json`)
  — a different, unrelated `skills/` from this repository. Reformatting a
  vendored copy would change the bytes the lock file hashes; do not "fix" a
  lint complaint under that path.
- **A hex-suffixed MSISDN no longer steers WireMock** (2026-09-17, #191):
  server-side phone validation (#186) rejects it before the mock rail is
  ever asked. `worker_e2e`/`worker_kill9` and `examples/shop`'s demo table
  now use real, `phonenumber`-valid Cameroon numbers matching
  `2376[579]\d{7}`.

## Further reading

- [references/gates.md](references/gates.md) — the twelve gates in depth,
  including every `verify-ui` check (its numbering is non-contiguous, so count
  the recipe's greps rather than trusting its header) and the both-directions gates.
- [references/recipes.md](references/recipes.md) — the full recipe inventory,
  the compose layering and ports, the demo-stack traps, and the `.env` rules.
- [references/ci.md](references/ci.md) — the three workflows, `changes` path
  gating, the readiness-poll lore, the `audit-web` retry loop, and the
  testcontainers serialisation in `.config/nextest.toml`.
