# The recipe inventory

_Verified against vpay `67c90ea5` (2026-09-20). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`justfile` is 4 704 lines, two-thirds of them comment (2026-09-16).
`just --list` is the index,
`just --show <recipe>` is the authority, `just --evaluate` prints every
variable (ports, pinned versions, expected counts). Recipe bodies win over
their own comments — see the skill's authority rule.

Counts and versions below were re-measured 2026-09-16.

## Setup

| Recipe              | Does                                                                                                           | Needs                                                                                                               |
| ------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `just install`      | `install-rust` then `install-node`                                                                             | network                                                                                                             |
| `just install-rust` | `rustup show`; `cargo install cargo-nextest --locked \|\| true`; `cargo install cargo-deny --locked \|\| true` | network. The `\|\| true` means a **failed install is silent** — check the tools are actually there                  |
| `just install-node` | `corepack enable`; `pnpm install`                                                                              | network. `.npmrc` sets `engine-strict=true`, so Node below `engines.node` fails here rather than mysteriously later |

`.npmrc` also sets `node-linker=isolated` and `shamefully-hoist=false`:
undeclared imports fail loudly rather than working by accident through
hoisting. That is why a package that forgets a dependency breaks in this
workspace and not in a hoisted one.

## Build

| Recipe                 | Does                                                                                                                                                                                                       | Needs                                          |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `just build`           | `build-rust` + `build-web`                                                                                                                                                                                 |                                                |
| `just build-rust`      | `cargo build --workspace`                                                                                                                                                                                  |                                                |
| `just build-web`       | `pnpm -r build`                                                                                                                                                                                            | `node_modules`                                 |
| `just build-dist`      | `cargo build --profile dist --target x86_64-unknown-linux-musl -p vpay-server`                                                                                                                             | `rustup target add x86_64-unknown-linux-musl`  |
| `just build-storybook` | builds both apps' Storybooks, then greps the built stylesheet asserting `--color-base-100` is **defined**, not merely referenced                                                                           | `node_modules`                                 |
| `just release-dry-run` | builds the three release images for the **host arch only** (`vpay-server`, `vpay-dashboard`, `vpay-checkout`) with `--push=false`, packages (does not push or sign) the Helm chart, then `just helm-check` | docker + buildx + helm + kubeconform + network |

The `build-storybook` grep is not tidiness. PR #135 produced a build that
referenced `--color-base-100` 22 times and defined it 0 — the `@vaam-apps/ui`
theme had been dropped from the bundle, so every story rendered on the
browser's default white and every colour-contrast verdict was about a ground no
user ever sees.

`.cargo/config.toml` sets `-C target-feature=+crt-static` for **both** musl
triples, x86_64 and aarch64, spelled out rather than collapsed into a `cfg()`
so a grep for the triple finds the flag that applies to it (ADR-0014).

## Test

| Recipe                    | Does                                                                                                                      | Needs                                               |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `just test`               | `test-rust` + `test-doc` + `test-web`                                                                                     |                                                     |
| `just test-rust`          | `cargo nextest run --workspace`                                                                                           | **Docker**; on macOS also `just loopback-aliases`   |
| `just loopback-aliases`   | `sudo ifconfig lo0 alias` for the ten `127.0.0.x` addresses `staff_sign_in.rs` sends from; no-op on Linux                 | macOS, `sudo`                                       |
| `just test-rust-all`      | `cargo nextest run --workspace --run-ignored all` — "expect failures; this is for seeing what is NOT covered, not for CI" | Docker                                              |
| `just test-doc`           | `cargo test --doc --workspace`                                                                                            |                                                     |
| `just test-web`           | `pnpm -r test`                                                                                                            | `node_modules`                                      |
| `just verify-ignored`     | `cargo nextest list` + `jq`, three assertions                                                                             | `jq`                                                |
| `just test-storybook`     | clears three Vite caches, then both apps' `vitest run --config vitest.storybook.config.ts`                                | Playwright Chromium (≈115 MB, network on first run) |
| `just playwright-browser` | `playwright install chromium` (no `--with-deps`; it needs root and CI's image already has the libraries)                  | network                                             |

**Docker is required for the Rust suite**, not optional.
`vpay-tests-integration`, `vpay-tests-conformance`, `vpay-db`, `vpay-server`
and `vpay-testkit` all start real `postgres:16-alpine` and
`wiremock/wiremock` testcontainers — one per test, with `Drop`-based cleanup.
You never need to provide a Postgres yourself.

**`just test-doc` is a second runner, not a flag.** `cargo nextest` runs no
doctests at all, so from the first commit until 2026-09-03 nothing in this
repository ever compiled one. Both `just ci` and CI's `rust` job run it as its
own step. When you report test results here, report the doctest count
separately — `CLAUDE.md` calls a result without one "half an answer".

**`#[ignore]` is effectively banned.** `expected_ignored` is `0` and
`verify-ignored` enforces it. The sanctioned way to write a test that needs a
running stack is a **Cargo feature**, not `#[ignore]`:
`sdks/rust/tests/live_invoices.rs` declares `required-features =
["live-stack"]`, so `cargo nextest list --workspace` neither builds nor lists
it, and `just sdk-live` turns the feature on. `sdks/stripe-compat` is the same
shape on the TypeScript side — it declares a `compat` script and no `test`
script, so `pnpm -r test` skips it.

**The census `verify-ignored` enforces** (variables in `justfile`, read with
`just --evaluate`): `expected_suites` = 48 test binaries, `expected_ignored` =
0, `min_tests` = 1080 (a floor, deliberately set under the measured count —
1772 as of the last measurement — so it is not a number people bump
reflexively). On failure the recipe prints the binary listing, which is how "a
binary was added" and "a binary vanished" are told apart. **Adding a test
binary means bumping `expected_suites` in the same commit.**

Subsets: `cargo nextest run -p <crate>`, `cargo nextest run -E 'test(name)'`,
`just test-sdk-rust` (`-p vpay-sdk`), `pnpm --filter <pkg> test`.

## Verify, lint and format

| Recipe                | Does                                                                                                                                     |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `just verify`         | the fifteen gates (2026-09-20; fourteen until 2026-09-18, twelve until 2026-09-17) + the `verify-docs` report — see [gates.md](gates.md) |
| `just lint`           | `fmt-check` `clippy` `lint-web`                                                                                                          |
| `just fmt`            | `cargo fmt --all`; `pnpm exec prettier --write .`                                                                                        |
| `just fmt-check`      | `fmt-check-rust` + `fmt-check-web`                                                                                                       |
| `just fmt-check-rust` | `cargo fmt --all -- --check`                                                                                                             |
| `just fmt-check-web`  | `pnpm exec prettier --check .`                                                                                                           |
| `just clippy`         | `cargo clippy --workspace --all-targets -- -D warnings`                                                                                  |
| `just lint-web`       | depends on `build-sdk-node`; `pnpm -r typecheck` then `pnpm -r lint`                                                                     |
| `just deny`           | `cargo deny check`                                                                                                                       |
| `just audit-web`      | two `pnpm audit` runs, `--prod` first then the whole workspace                                                                           |

Two recipes joined that list in 2026-09-17/18 and each is runnable on its own,
offline, in seconds — reach for them directly rather than the whole of
`just verify`:

| Recipe                          | Does                                                                                                                                                                                   |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `just verify-versions`          | `cargo xtask verify-versions` — the 13th gate (2026-09-17, #201)                                                                                                                       |
| `just verify-privacy-inventory` | `cargo xtask verify-privacy-inventory` — the 14th gate (2026-09-18, #187), needs no Postgres                                                                                           |
| `just verify-doc-counts`        | `cargo xtask verify-doc-counts` — the 15th gate (2026-09-20, #233); checks every `<!-- count:KIND ARG… -->`-marked number against what the tree measures, including its own gate tally |

**`fmt-check-web` walks the working tree, not the index.** An untracked scratch
`.ts`, `.md` or `.json` left lying about fails it. `.gitignore` it or delete it
— do not reach for `--write` on a tree you have not looked at. Because it is
step 2 of `just ci`, this is how `just ci` fails in its first seconds.

`.prettierrc.json` writes out every prettier default explicitly so an upgrade
cannot silently reformat the repository, with one exception:
`embeddedLanguageFormatting: "off"`, which must stay off. With the default
`"auto"`, prettier rewrites the **contents** of fenced code blocks, and this
repository's markdown is largely pasted transcripts — measured 2026-09-10, it
rewrote 40 fences in 22 files, pretty-printing a one-line log line into ten
lines the server does not emit.

`lint-web` depends on `build-sdk-node` because the typecheck does:
`sdks/stripe-compat` imports `@vaam-apps/vpay-sdk/stripe`, whose types resolve
through `dist/`, and `dist/` is gitignored. Without the build you get
`TS2307: Cannot find module` — a missing artefact reported as a broken import.

`clippy.toml` denies `unwrap`/`expect`/`panic`/`dbg` in production code and
exempts tests. **If clippy complains about `expect` in a test, you are outside
a `#[cfg(test)]` module.**

## Web and dev servers

`just dev-dashboard` → `pnpm --filter @vpay/dashboard dev` on `:3000`.

Direct package scripts, for the rest: checkout `next dev -p 3001`, checkout
Storybook `:6006`, dashboard Storybook `:6007`, `examples/shop` `next dev -p
3000`.

Neither app's `.storybook/**` is type-checked by anything in `just ci`. That is
measured, not assumed: `tsc --listFiles` lists 76 files for the checkout and
1713 for the dashboard, and **0** of either under `.storybook/` — TypeScript's
include-glob expansion skips dot-directories. Both `eslint.config.js` name
`.storybook/**` in `outsideTsconfig`, so those files get syntax and style rules
with no program behind them, by design. `just build-storybook` and `just
test-storybook` are what fail when the config breaks.

## End to end

| Recipe                       | Does                                                                                                                                            | Needs                                                |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `just test-e2e`              | tears the stack down, `up -d --build --wait`, polls `/healthz` for 120 s, runs the specs, tears down                                            | Docker, network, two image builds                    |
| `just e2e-specs`             | the Cypress half only, against a stack that is **already up**                                                                                   | a running demo stack                                 |
| `just stripe-compat`         | brings up `compat_services` only, runs the stripe-node conformance suite                                                                        | Docker                                               |
| `just sdk-live`              | brings up `compat_services`, runs the Rust `live_invoices` suite **and** `pnpm --filter @vaam-apps/vpay-sdk test:live`; **leaves the stack up** | Docker                                               |
| `just test-flutter-e2e`      | drives the plugin's own `BrowserClient`/`SheetController` against a real stack, real `private_key_jwt`, real `cs_…` sessions                    | `curl docker jq node pnpm`, and `just demo-up` first |
| `just test-flutter-emulator` | drives the real native sheet's confirm UI on an Android emulator                                                                                | `adb`, an AVD named `vpay_e2e_avd`, a running stack  |

Cypress lives in `frontends/tests/e2e` (`@vpay/e2e`). Four specs:
`checkout.cy.ts`, `dashboard.cy.ts`, `shop-embedded.cy.ts`,
`shop-hosted.cy.ts`. The `e2e` script runs **two passes** — `cypress run`, then
`VPAY_E2E_FRAMED=1 cypress run`. Interactive: `pnpm --filter @vpay/e2e open`.

The Cypress binary needs `pnpm exec cypress install` on a network that reaches
its CDN. `CYPRESS_INSTALL_BINARY=0` skips it, and the `justfile` re-exports the
variable, so a value you export locally propagates into every recipe.

**~~Flutter is not built on this branch~~ — corrected 2026-09-17: it is, and
has been since 2026-09-13's Lane A/B/C.** `sdks/flutter/vpay_checkout_flutter/`
exists; the package is `vpay_checkout_flutter`, the mobile **payer** surface
(`vpay-sdks`'s `references/flutter-plugin.md` has the architecture — it has
been rebuilt twice, most recently into a native Flutter sheet, #189).

Six `*flutter*` recipes, all gated by a shared `_flutter-preflight` that
refuses clearly (names the missing directory) if
`sdks/flutter/vpay_checkout_flutter/` does not exist or Flutter is not on
`PATH`, and **warns without failing** when the version on `PATH` differs
from `flutter-toolchain.toml`'s pin:

| Recipe                       | Proves                                                                                                                                                                                                                                                                                                                                                                                       | Needs                                                                                                          |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `just install-flutter`       | `flutter pub get` for the plugin and its `example/` app                                                                                                                                                                                                                                                                                                                                      | Flutter on `PATH`, network                                                                                     |
| `just analyze-flutter`       | `dart analyze --fatal-infos` — this package's strictest setting, since it has no `verify-*` gate of its own                                                                                                                                                                                                                                                                                  | Flutter on `PATH`                                                                                              |
| `just test-flutter`          | Unit tests only (`flutter test`) — the state machines are pure by design (D1/D4), no emulator, no `adb`, no stack. **Stack-independent and must keep passing with nothing running** — 258 passed / 0 skipped, 2026-09-17                                                                                                                                                                     | Flutter on `PATH`                                                                                              |
| `just test-flutter-web`      | The **only** recipe that runs `test/web/` at all — `@TestOn('chrome')` keeps it out of a bare `flutter test` entirely (proven: `flutter test` reports the unit count with those files present, neither a pass nor a skip — not in the run). Drives `window.open`, the origin-**and**-`event.source`-pinned `vpay:complete` listener, and the popup `closed` poll in a **real** Google Chrome | `CHROME_EXECUTABLE`, or `google-chrome` on `PATH` — refuses loudly, never fakes a browser, if neither resolves |
| `just test-flutter-e2e`      | The plugin's own `BrowserClient`/`SheetController` against a **real, running** stack — no `MockClient` anywhere in the path. Curls `/healthz` on vpay and `examples/shop` first and refuses loudly (_"bring a stack up first: just demo-up"_) rather than failing three tools deep. Does not bring the stack up or tear it down                                                              | `curl docker jq node pnpm`, and `just demo-up` already running                                                 |
| `just test-flutter-emulator` | The real native sheet's confirm UI, driven on a real (not booted-by-this-recipe) Android emulator/device — selects by `VPAY_EMULATOR_SERIAL`, or the one attached device whose AVD is named `vpay_e2e_avd`, and refuses on zero or more than one. Calls `adb reverse` (not a `10.0.2.2` rewrite) because the checkout page's own client-side JS bakes in `localhost:8080`                    | `adb curl docker jq`, a running `vpay_e2e_avd` device, a running stack                                         |

**None of the six is in `just ci`** (D-M3, unchanged since 2026-09-13): no
CI job and no `vpay-ci` VM installs a Flutter SDK, so every count quoted for
this package anywhere in this repository is a human running the recipe by
hand. `flutter-toolchain.toml` pins Flutter `3.47.2` / Dart `3.13.2` the way
`rust-toolchain.toml` pins the Rust compiler — but its `channel` field reads
`[user-branch]`, which is **not** a clean channel pin (unlike
`rust-toolchain.toml`'s `channel = "1.98.0"`) and the file says so itself:
it is the version installed on the authoring host, not a value computed from
a `pubspec.yaml` constraint, and nothing enforces agreement between the two
the way `verify-toolchain` enforces the Rust pin.

### The four `*-tauri-*` recipes (2026-09-22, #238)

New with [vpay#238](https://github.com/vaam-apps/vpay/pull/238), merged
2026-09-22 as `999a23f9` — absent from any `master` checkout older than
that. They build and test `tauri-plugin-vpay-checkout`, the Tauri v2 payer
surface (`vpay-sdks`'s `references/tauri-plugin.md` has the architecture and
every caveat). Added here 2026-09-23, because this page was already cited as
the depth for them and did not mention Tauri at all.

All four share one `_tauri-preflight`, which refuses by name when
`sdks/tauri/tauri-plugin-vpay-checkout/` or its `Cargo.toml` is missing —
the same shape as `_flutter-preflight` above. All four drive cargo or pnpm
through `--manifest-path` / `--filter` rather than `-p`, because **the crate
is its own Cargo workspace** (ADR-0023 T2): `cargo clippy --workspace`,
`cargo nextest run --workspace` and `cargo deny` at the root cannot see it,
so these are the only commands in the repository that compile it.

| Recipe                    | Proves                                                                                                                                                       | In `just ci`?                     |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------- |
| `just test-tauri-rust`    | `cargo test --manifest-path …` — the unit suite **and** the `init()` doctest in `src/lib.rs` in one invocation (`cargo test`, not nextest, deliberately)     | **no** (T7)                       |
| `just clippy-tauri-rust`  | `cargo clippy --all-targets -- -D warnings` — the only thing in the repository that lints this crate                                                         | **no** (T7)                       |
| `just check-tauri-mobile` | `cargo check` for `aarch64-linux-android` and `aarch64-apple-ios` — that the `#[cfg(mobile)]` half still compiles. **Nothing about the Kotlin or the Swift** | **no** (T7)                       |
| `just test-tauri-js`      | `pnpm --filter @vaam-apps/vpay-tauri-checkout test` — scoping only, for a single-package iteration loop                                                      | not by name — `pnpm -r test` does |

**None of the three Rust ones is in `just ci` or `just verify`, and the
justfile says why in its own header (T7):** the `tauri` crate does not build
on Linux without `libwebkit2gtk-4.1-dev`, which the CI image does not carry,
and `check-tauri-mobile` needs an Android and an Apple target the image has
no toolchain for. Adding either is a runner-image change — the maintainer's
call, not a recipe's. So **every Rust count this repository quotes for this
crate is a human running a recipe by hand**, exactly as with Flutter above.
The justfile's gate tally is unchanged at fifteen: none of these is a gate.

`rustup target add aarch64-linux-android aarch64-apple-ios` before
`check-tauri-mobile`. It sets no `IPHONEOS_DEPLOYMENT_TARGET` on purpose —
swift-rs defaults to iOS 13.0 and the Swift host needs 15.0, handled by an
`if #available(iOS 15.0, *)` guard in the source. A Swift availability
failure here is that guard regressing, not your environment.

The **TypeScript** half needs no recipe to be gated: `pnpm-workspace.yaml`
names `sdks/tauri/*`, so `just lint-web`, `just test-web` and
`just fmt-check-web` already reach `@vaam-apps/vpay-tauri-checkout`, and
CI's `web` job filter already names `**/*.ts`.

**What no recipe covers at all:** `examples/tauri-checkout/src-tauri/` — the
example app's Rust, and with it the only route in this repository that
compiles the plugin's Kotlin and Swift. It is in **0** justfile recipes, 0
workflow files and 0 xtasks (measured 2026-09-23). The example's TypeScript
does run, through `pnpm -r`, because `pnpm-workspace.yaml` globs
`examples/*`.

### WireMock steering MSISDNs — the hex convention is dead (2026-09-17, #191)

`worker_e2e`/`worker_kill9` (the chaos suites) used to steer WireMock's own
mock rail by hiding the scenario as **hex digits inside the MSISDN**
(`…0ce9` slow status, `…0cf9` slow submit, `…c15` SIGTERM) — a string that
was never a real phone number and, worse, used prefix `60`, not a real
Cameroon mobile prefix. Once server-side phone validation landed (#186), a
confirm carrying one of those numbers gets vpay's own `400` before the rail
is ever asked, which defeats the point of a steering number.

**Fixed 2026-09-17 (#191).** Steering MSISDNs are now real,
`phonenumber`-valid Cameroon mobile numbers matching `2376[579]\d{7}` — the
real prefix set — continuing the old numeric _suffix_ convention
(`237670000900/909/915/919` for the four chaos scenarios). `examples/shop`'s
own demo-number table moved the same way, onto real `67x` (MTN) / `69x`
(Orange) prefixes, keeping each number's old suffix (`101`, `102`, `103`,
`400`, `503`) so the table reads the same as before. **Both tests now assert
the new invariant** (`/^2376[579]\d{7}$/` for the shop table) rather than
merely matching new literals, so reverting either move fails its own suite,
not just a fixture comparison.

Not touched, deliberately: the Rust fixtures under `confirm_rails.rs` and
`adapter_conformance.rs` that never pass through the validating API route
still use the old `2376000000xx` block — that redesign is recorded there as
deferred, not silently different from this fix.

## Docs

| Recipe                      | Does                                                                                                                     | Needs                                                              |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| `just docs-check`           | `verify-status` + `verify-links` — the short loop while editing docs                                                     | nothing                                                            |
| `just docs-check-citations` | `cargo xtask verify-citations`                                                                                           | network + authenticated `gh`. **Fails**, never skips, without them |
| `just verify-doc-counts`    | `cargo xtask verify-doc-counts` — every `<!-- count:KIND ARG… -->`-marked number, checked against what the tree measures | nothing                                                            |
| `just verify-docs`          | `cargo xtask verify-docs` — a report, exits 0 always                                                                     |                                                                    |

## Database and migrations

There is **no** `just migrate`. `sqlx::migrate!` runs at server boot.

| Recipe                     | Does                                                                                               |
| -------------------------- | -------------------------------------------------------------------------------------------------- |
| `just verify-migrations`   | `cargo xtask verify-migrations`                                                                    |
| `just migrations-manifest` | **appends** SHA-256 lines for `backends/migrations/*.sql` to `backends/migrations/MANIFEST.sha256` |

`migrations-manifest` is append-only and refuses to rewrite an existing line
whose hash changed, or to drop a line whose file is gone. Adding a migration
means running it in the same commit. Editing a shipped migration is the failure
mode — see [gates.md](gates.md#12-verify-migrations).

## Docker and the compose stacks

The three compose files **layer**. `demo_compose` is
`-f compose.yml -f compose.e2e.yml -f compose.demo.yml`.

### `compose.yml` — the base (`just up` / `just down`)

| Service           | Image                     | Host port |
| ----------------- | ------------------------- | --------- |
| `postgres`        | `postgres:16-alpine`      | `5432`    |
| `wiremock-mtn`    | `wiremock/wiremock:3.9.2` | `8081`    |
| `wiremock-orange` | `wiremock/wiremock:3.9.2` | `8082`    |

Credentials are `vpay`/`vpay`/`vpay`. The WireMock stubs mount
`backends/tests/conformance/wiremock/{mtn,orange}` read-only.

**The stub rail is a separate process, on purpose** (ADR-0006). WireMock is a
_host_ the app reaches over HTTP through configuration — the same mechanism as
production. Nothing in the application knows it is talking to a stub, and
`verify-no-mocks` keeps it that way. Do not "simplify" this into an in-process
mock adapter.

Their healthcheck is `curl /__admin/health` rather than a TCP probe, because a
TCP probe cannot distinguish "the JVM bound the port" from "the stub tree has
loaded".

### `compose.e2e.yml` — the full stack

Adds, on top of the base:

| Service            | Host port | Note                                             |
| ------------------ | --------- | ------------------------------------------------ |
| `vpay-server`      | `8080`    | `backends/Dockerfile`                            |
| `vpay-worker`      | —         | same image, `command: ["worker"]`                |
| `wiremock-webhook` | `8083`    | the receiver; `/__admin/requests` is the journal |
| `dashboard`        | `3000`    | `frontends/Dockerfile`                           |
| `vpay-checkout`    | `3080`    | vpay's own payer page                            |
| `vpay-shop`        | `3001`    | `examples/shop`                                  |

It also mounts `deploy/dev/postgres-init/10-shop-database.sql` into
`/docker-entrypoint-initdb.d`. **Postgres runs init scripts exactly once, on an
empty data directory** — so a `pgdata` volume from before this file existed has
no `shop` database and `vpay-shop` dies in `zen migrate deploy`. The fix is
`docker compose ... down -v` (which is what `just demo-down` runs).

`vpay-server` carries **no healthcheck** and cannot: its runtime image is `FROM
scratch` (ADR-0004) — no shell, no curl, and the only executable in it is
`/vpay-server`. See [ci.md](ci.md) for how readiness is observed instead.

### `compose.demo.yml` — one registered merchant

Layers on the e2e stack and changes exactly two things per service: the profile
(`VPAY_PROFILE: demo`) and one read-only bind mount of the generated
`application-demo.yml` into `/config/`. It binds everything to `127.0.0.1` and
parameterises every port; `postgres` and `wiremock-mtn` get `ports: !reset []`
and are not exposed at all.

Ports (`just --evaluate`, all overridable):

| Variable              | Default                                                |
| --------------------- | ------------------------------------------------------ |
| `demo_port`           | 8080                                                   |
| `demo_receiver_port`  | 8083                                                   |
| `demo_orange_port`    | 8082                                                   |
| `demo_checkout_port`  | 3080                                                   |
| `demo_shop_port`      | 3001                                                   |
| `demo_dashboard_port` | 3000                                                   |
| `demo_fixture_port`   | 4180 (and 4181 for the frame fixture) — host-side only |

`demo_project` is `vpay-demo`. `demo_services` is the nine-service set;
`compat_services` is the six-service set without the web apps.

### Three demo-stack traps

**A missing overlay is not an error.** `Config::load_with_env` merges the
profile overlay only `if overlay_path.is_file()`, so a stack brought up with
`VPAY_PROFILE: demo` and no `application-demo.yml` boots happily on the base
file alone — with a placeholder modulus as the entire merchant registry, an
`invalid_client` at the first token request, and no hint as to why. That is
what makes `just gen-demo-keys` load-bearing rather than convenient, and why
`just demo` runs it first.

**The overlay replaces `merchant_clients` wholesale.** figment deep-merges maps
but a _list_ in an overlay replaces the base list entirely, so while the demo
stack is up the base file's `acme-cameroon` is gone. That is correct — nobody
holds its private key — but it means you cannot authenticate as it against a
demo stack. Everything the overlay does not mention (`providers`,
`currencies`, `deployment.public_base_url`, `dashboard_client`) still comes
from the base file.

**`just demo-staff` prints the password once.** It runs `vpay-server staff add`
in a one-shot container and captures the one-time password into
`.e2e/vpay-demo/staff-password.txt`. Only the hash is stored server-side, so if
the account exists and that file is gone the password **cannot be recovered** —
the recipe tells you so and the only route back is `just demo-down` (which
deletes the volumes) and starting again. `dashboard.cy.ts` signs in as this
account.

### The demo recipes

| Recipe                                                | Does                                                                                                                            |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `just demo`                                           | `demo-up` + `demo-walk`                                                                                                         |
| `just demo-up`                                        | `gen-demo-keys`, then `up -d --build --wait`, then polls `/healthz` for 120 s                                                   |
| `just demo-walk`                                      | `cargo run -q -p merchant-demo` against the running stack, then prints every useful URL                                         |
| `just demo-status`                                    | `docker compose ... ps` plus every `vpay`-named container on the machine                                                        |
| `just demo-down`                                      | `down -v` — containers **and volumes**                                                                                          |
| `just demo-shop` / `demo-checkout` / `demo-dashboard` | echo the URL                                                                                                                    |
| `just gen-demo-keys`                                  | `cargo xtask gen-signing-key` for `demo-merchant` and `shop-merchant`, extracts the public JWKs, writes the overlay. Idempotent |
| `just gen-e2e-signing-key`                            | `openssl genpkey` RSA 3072 PKCS#8 → `.e2e/oauth-signing-key.pem`. Idempotent                                                    |

When `/healthz` does not answer, `demo-up` dumps `ps` and the last 80 log
lines, and reminds you that **exit 78 in that log means a config or CLI
prerequisite is missing**.

## Environment files

`.env.example` → `.env`. Every `${VAR}` placeholder in `config/application.yml`
must resolve: **an unresolved placeholder is exit 78 at boot, never an empty
string.**

Required: `VPAY_CONFIG` (neither long-running mode starts without a validated
config file), `VPAY_BIND`, `VPAY_OBSERVABILITY_BIND` (the `/livez` + `/metrics`
listener — **never the same port as `VPAY_BIND`**, because `/metrics` must not
be reachable from whatever fronts 8080), `VPAY_OAUTH_SIGNING_KEY_FILE` (RS256
PKCS#8 PEM), the five rail credentials (`MTN_SUBSCRIPTION_KEY`, `MTN_API_USER`,
`MTN_API_KEY`, `ORANGE_CLIENT_ID`, `ORANGE_CLIENT_SECRET`, plus
`ORANGE_MERCHANT_KEY`), and `MERCHANT_WEBHOOK_SECRET`.

Two extra rules apply to `MERCHANT_WEBHOOK_SECRET` under `livemode: true`,
because whoever holds it can forge a `payment_intent.succeeded` the merchant's
handler will believe: it must stay a `${VAR}` and never a literal
(`ConfigError::LiteralSecret`, checked against the file's **text** before
placeholders resolve), and it must be at least 32 bytes
(`ConfigError::WeakWebhookSecret`, checked against the **resolved** value — so
a correctly written `${VAR}` holding `a` is still a refusal to boot).

The CLI is **one binary** since issue #77 (2026-09-07): no subcommand serves
the API, `worker` runs the job loop, `staff add` creates a dashboard account.
Anything you read describing `vpay-worker-bin` as a separate binary is stale.

`.env.live.example` → `.env.live` (git-ignored) is for the `live` profile test
against MTN's real sandbox: the three MTN values, `VPAY_STAFF_PEPPER` and
`VPAY_STAFF_TOTP_KEY` (both base64url), and the **public** JWK halves
`LIVE_MERCHANT_JWK_KID` / `LIVE_MERCHANT_JWK_N`. Source it with
`set -a; . ./.env.live; set +a`. Walkthrough:
`docs/runbooks/live-sandbox-test.md`.

## Helm

`just helm-check` needs `helm` and `kubeconform` on PATH and **HTTPS**
(kubeconform fetches the upstream schema mirror plus the CRD catalog for
ServiceMonitor/PrometheusRule). CI's `deploy` job runs this recipe, not a copy
of its commands.

What it proves: the chart lints under three value sets; all three render (the
Gateway API one needs `--api-versions gateway.networking.k8s.io/v1`, without
which the `HTTPRoute` templates render **nothing** and every check below would
pass over an empty file); the **24 named guards** under
`deploy/helm/vpay/ci/guards/` are exactly the 24 the recipe lists and each
fires _by name_ with a non-zero exit; the default render templates no checkout
page and `ci/values-full.yaml`'s does; the Ingress carries `limit-rps` and the
token Ingress is **tighter** than `/v1`'s; both mechanisms route `/provider`;
and every rendered object validates against upstream schemas.

_This said **22** until 2026-09-16, when ADR-0022 retired
`dashboard-not-templated` (the chart now templates the dashboard) and added
`connection-budget` and `networkpolicy-management-route`._ Note that **24
guards are covered by 25 fixture files**: the harness learned
`<guard>.<n>.yaml` variants so `connection-budget` could be gated in both
directions, and the recipe's own count is of guard _names_, not files.

The guard set is written out rather than counted, because "22 files found, 22
fired" is also what deleting a guard _and_ its values file looks like.

**What it proves about a cluster: nothing. Nothing here has ever been applied
to one.**

## SDK

| Recipe                                        | Does                                                                                                                      |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `just build-sdk-node` / `test-sdk-node`       | `@vaam-apps/vpay-sdk`                                                                                                     |
| `just build-sdk-browser` / `test-sdk-browser` | `@vaam-apps/vpay-stripe-js`                                                                                               |
| `just test-sdk-rust`                          | `cargo nextest run -p vpay-sdk` (scoping only — `test-rust` already covers it)                                            |
| `just build-checkout-browser`                 | vendors `sdks/stripe-js/dist/` into `examples/checkout-browser/dist/stripe-js/`                                           |
| `just sdk-conformance-node`                   | mints an assertion with `sdks/nodejs/scripts/mint-assertion.mjs` and verifies it with the Rust `verify_assertion` example |

Adding an SDK method means adding its row in `docs/sdks/parity.md` in the same
commit — `verify-sdk-parity` fails in both directions.
