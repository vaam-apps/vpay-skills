# `sdks/tauri/tauri-plugin-vpay-checkout`

_Verified against vpay `b747e5d5` (2026-09-23). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Arrived with [vpay#238](https://github.com/vaam-apps/vpay/pull/238), merged
2026-09-22 as `999a23f9` — nothing described below exists on a `master`
checkout older than that.

A **payer** surface, like `@vaam-apps/vpay-stripe-js` and
`sdks/flutter/vpay_checkout_flutter` — not a third merchant SDK. It
authenticates a payer's _device_ with a publishable key and a
per-session / per-intent `client_secret`, speaks the same `/v1/browser`
routes, and shares no capability row with the merchant tables. It has its own
single-column table in `docs/sdks/parity.md`.

Two artefacts from one directory: the Rust crate `tauri-plugin-vpay-checkout`
(`publish = false`) and the npm package `@vaam-apps/vpay-tauri-checkout`
(publish-ready, published by nothing). ~~Both `0.4.0` as of 2026-09-22.~~
**Corrected 2026-09-23: both are `0.4.1`**, and have been since
[vpay#240](https://github.com/vaam-apps/vpay/pull/240) (`078fa3d`). #238's
branch forked at `7134ecb`, before release-please's 0.4.0 → 0.4.1 bump
([#236](https://github.com/vaam-apps/vpay/pull/236)) landed on `master`, so
the branch was internally consistent with itself and wrong about the tree it
merged into — the merge was clean and the version was stale. This page
faithfully mirrored the branch. **Both are `0.5.0` since 2026-09-23**, when
the `v0.5.0` release ([vpay#239](https://github.com/vaam-apps/vpay/pull/239),
`d98fdaf`) moved them with every other annotated line — on `b747e5d5`
`cargo xtask verify-versions` prints `24 version references all say 0.5.0`.
The plugin has no version of its own: it moves with every vpay release.

Design: `docs/plans/2026-09-22-tauri-plugin.md` (T1–T7). Contract the parallel
lanes built against: `docs/plans/2026-09-22-tauri-plugin-brief.md`. Decisions:
ADR-0023, **Proposed — needs maintainer acceptance**, so do not read T1–T7 as
settled house rules the way D1–D9 are. Process:
`docs/flows/tauri-checkout.md`. Area page: `docs/status/mobile-tauri-plugin.md`.
Evidence: `docs/status/verification/2026-09-22-tauri-plugin.md`.

**It inherits ADR-0021's D1–D9 unchanged** and re-argues none of them. Where
this page says "D1", "D4" or "D6" it means that ADR's rule.
`references/flutter-plugin.md` is the fuller document on those; where the two
agree, that page has the reasoning.

## The shape — one state machine, four hosts

```text
 guest-js/  @vaam-apps/vpay-tauri-checkout   ← every decision lives here (T1)
   VpayCheckout.start(sessionUrl)
     parse → pre-flight → show → one event → poll → one of five kinds
        │
        ├── tauriHost()  invoke("plugin:vpay-checkout|show", {…, onEvent: Channel})
        │     └── the Rust crate → Android (Kotlin) │ iOS (Swift) │ desktop (`open`)
        └── webHost()    window.open popup, outside Tauri entirely
```

`defaultHost()` is `isTauri() ? tauriHost() : webHost()`, so the merchant's
code is the same three lines in a Tauri app and in a plain web build of the
same front end. **The hosts are deliberately stupid**: show this URL, and say
either that the payer left or that a deep link matched. None of them formats
money, knows a status word, or holds the URL longer than the window is open.

## The wire contract, and it is the whole of what a host may say

`show` takes **four flat camelCase arguments** — `url`, `stopUrls`,
`allowInsecureUrl`, `onEvent` — not one `request` object. That is a recorded
deviation from the brief with a reason: `tauri::ipc::Channel` implements
`CommandArg` and **not** `Deserialize`, so it cannot sit inside a deserialised
struct. The JS wire is exactly what the brief fixed; only the Rust signature
moved.

`dismiss` takes no arguments and closes the window if one is open.

The **one event**, delivered on the per-`show` `Channel` **exactly once**:

```jsonc
{ "outcome": "dismissed" | "stopUrlReached", "reachedUrl": "https://…" | null }
```

There is no `succeeded`, `canceled` or `failed` outcome anywhere on that wire,
because no window may decide what a payment did (D1). `reachedUrl` is
diagnostics; guest-JS reads nothing off it — `resolveAfterStopUrlReached` takes
no URL argument at all, and the signature is the proof.

A per-`show` `Channel` rather than `addPluginListener` is deliberate: a
plugin-wide event stream would make "exactly one event per show" something
guest-JS enforced by correlation instead of something the wire guarantees.

**Every host rejection becomes the fixed `platform_window_failed` error**, and
guest-JS never interpolates the thrown value — a host's message can quote the
URL, and the URL's fragment _is_ the session secret (D6). The native refusal
tokens are `already_open`, `invalid_url`, `insecure_url`, `invalid_arguments`,
`no_presenter` and `channel_send_failed`. Two of those are not in the brief
(`insecure_url`, `invalid_arguments`) and there is no `no_activity` branch on
Android, because Tauri always supplies the Activity.

## Per host, with the fact that will surprise you first

| Host                          | Window                                                                                   | Dismissal signal                    | `stopUrlReached`                                          |
| ----------------------------- | ---------------------------------------------------------------------------------------- | ----------------------------------- | --------------------------------------------------------- |
| Android                       | partial Custom Tab **requested** (90 %, adjustable, close at START) — Chrome declined it | yes — the tab closing               | wired via an exported forwarding Activity; never observed |
| iOS                           | `SFSafariViewController`, `.pageSheet` + `.large()`                                      | yes — Done **and** swipe-away       | **unreachable** (T6)                                      |
| Desktop (macOS/Windows/Linux) | the default browser, `open::that_detached`                                               | **none at all** — `dismiss()` only  | not implemented                                           |
| Plain browser (no Tauri)      | `window.open` popup, address bar visible                                                 | yes — `closed`, polled every 500 ms | n/a — a cross-origin popup's location is unreadable       |

_(Corrected 2026-09-23: the Android row said "partial Custom Tab, 90 %
height" flat. The height is a request a browser may decline, and Chrome
declined it on 2026-09-22 — see the gap list below. vpay's
`docs/flows/tauri-checkout.md` table made the same correction the same day,
vpay#242.)_

- **Android** declares **no `<intent-filter>` of its own.** A hostless `https`
  filter would claim every https URL on the device (harmful on API 21–30,
  inert on 31+), so the merchant declares one for their own verified host
  against `dev.vpay.tauri.checkout.VpayCheckoutAppLinkActivity` with
  `tools:node="merge"` and serves an `assetlinks.json`. The manifest carries
  that snippet verbatim. `minSdk` 21. ~~`jvmTarget = 17` is flagged as a risk
  in `build.gradle.kts` — `tauri-android` itself compiles at 1.8.~~
  **Resolved 2026-09-22:** a real `tauri android build` succeeded with the
  plugin at JVM 17 while `:tauri-android` and the app compiled at 1.8, and
  the documented fallback was **not** applied. The comment stays; the risk
  did not materialise. One Kotlin deprecation warning remains, at
  `VpayCheckoutActivity.kt:349` (`getParcelableArrayListExtra`).
- **iOS** confines UIKit work to `DispatchQueue.main.async` rather than
  `@MainActor`, because Tauri dispatches commands from a background queue
  through the Objective-C runtime, which bypasses actor isolation rather than
  hopping for it. swift-tools 5.9, `.iOS(.v15)`. `allowInsecureUrl` is decoded
  and unused: D6's refusal happens in guest-JS before a host is called.
- **Desktop** is a state machine over a crate-private `BrowserOpener` seam.
  `open::that_detached` — the one shipping implementation — is covered by no
  test and has never executed.
- **The plain-browser host** accepts a `{type:"vpay:complete"}` message only
  when `event.origin` is the page's **own** origin **and** `event.source` is
  the exact popup it opened. Origin alone is not enough; the Flutter web suite
  found a same-origin window could spoof completion on 2026-09-15. `stopUrls`
  and `allowInsecureUrl` are documented as **inapplicable** here rather than
  silently ignored.

## T1–T7, one line and one reason each

| #      | Decision                                                                                   | Because                                                                                                                                              |
| ------ | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **T1** | The state machine is guest-JS; Rust, Kotlin and Swift decide nothing                       | The request included **web**, and on the web there is no Rust — one implementation of D1/D4 on Node beats two, or a wasm build                       |
| **T2** | The crate carries its own empty `[workspace]` and belongs to no workspace                  | `tauri`'s graph (wry, tao, objc2, webkit2gtk) must not enter the root `Cargo.lock`, `cargo deny`, `verify-no-mocks` or `nextest run --workspace`     |
| **T3** | The two reads reuse `@vaam-apps/vpay-stripe-js` rather than a ported client                | One wire, one client: a divergence between two TypeScript clients on the same two routes is invisible until a payer hits it                          |
| **T4** | Desktop opens the default browser and has **no** dismissal signal                          | The Flutter macOS host shipped a focus-regained fake once and removed it on 2026-09-16; "the app regained focus" is not "the payer finished"         |
| **T5** | Outside Tauri the same package is the popup host, origin **and** source pinned             | "web" is a third host, not a second package — and origin alone let a same-origin window spoof completion                                             |
| **T6** | iOS cannot report `stopUrlReached`; the matching code stays, unreachable and labelled      | tauri-v2.11.6's Swift `Plugin` exposes no app-delegate hook, and Tauri's own `deep-link` plugin ships no `ios/`, handling iOS via `RunEvent::Opened` |
| **T7** | No Rust/Kotlin/Swift half is in `just ci` or `just verify`; the TypeScript half already is | The CI image has no `libwebkit2gtk-4.1-dev`, no Android SDK/NDK and no Xcode; adding them is a runner-image change — the maintainer's call           |

## Two traps inherited from the browser client (T3)

- **`CheckoutSession.client_secret` is typed `string` and the server never
  sends it back** (found by the Flutter e2e suite, 2026-09-14). The polling
  credential is `session.payment_intent.client_secret`. Touching
  `session.client_secret` compiles, typechecks, and fails every real
  pre-flight. `the polling credential is the intent's client_secret, never the
session's` is the case that pins it.
- **`loadStripe` accepts an `http://` base URL without complaint.** So
  `VpayCheckout`'s **constructor** refuses a non-`https` `baseUrl` before
  `loadStripe` is ever called, unless `allowInsecureBaseUrl: true` is passed
  by name (D6). It is forwarded to the host as `allowInsecureUrl` rather than
  re-derived from the scheme anywhere.

## Gate and tooling facts that will cost you time

- **`verify-sdk-parity` cannot cite a test title containing `|`** (found
  2026-09-22). `table_row` in `.xtask/src/main.rs` splits a row on every raw
  `|` with no escape handling. Two live vitest cases in
  `guest-js/host-tauri.test.ts` assert the invoke names —
  `"invokes plugin:vpay-checkout|show with the four camelCase keys the wire contract names"`
  and `"invokes plugin:vpay-checkout|dismiss with no arguments"`. Both pass;
  neither is cited in a cell, and the table says so in prose. **Do not rename
  a test to suit the parser.**
- **Do not spell a `…Resource` type anywhere under `sdks/tauri/`.**
  `verify-sdk-parity` enumerates `impl …Resource` blocks and `class …Resource`
  as merchant capabilities and would demand rows for them.
- **The `permissions/autogenerated/` and `permissions/schemas/` trees are
  tracked build output**, rewritten by `tauri_plugin::Builder` on every
  `cargo build`, the way every plugin in `tauri-apps/plugins-workspace` commits
  them. They are prettier-ignored because a formatted copy is un-formatted
  again by the next build. Commit the generator's bytes.
- **`links = "tauri-plugin-vpay-checkout"` is mandatory**, not decorative:
  `tauri_plugin::Builder::try_build` refuses to run without
  `CARGO_MANIFEST_LINKS`.
- **The crate restates the root workspace's `[lints.clippy]` block in its own
  manifest**, because `lints.workspace = true` needs a parent workspace and
  T2's `[workspace]` table is precisely what severs that link. The denies were
  proven live by mutation: an `unwrap` added to the crate made clippy exit 101.
- **swift-rs compiles the Swift for iOS 13.0** unless
  `IPHONEOS_DEPLOYMENT_TARGET` or a consuming app's
  `bundle.iOS.minimumSystemVersion` says otherwise. The host needs 15.0 for
  `sheetPresentationController`, and a bare
  `cargo check --target aarch64-apple-ios` failed **exit 101** until an
  `if #available(iOS 15.0, *)` guard landed in
  `VpayCheckoutExternalBrowserSession.swift`. **That guard is load-bearing** —
  if `just check-tauri-mobile` starts failing with a Swift availability error,
  it is what regressed.
- **Every npm script chains a `deps` script** that builds
  `@vaam-apps/vpay-stripe-js` first; its types resolve through `exports` to
  `dist/`. A bare `tsc` in this directory fails for that reason alone.
- **`release-please-config.json` carries two `extra-files` entries** for this
  package — a `generic` one for `Cargo.toml` (its `version` line carries
  `# x-release-please-version`) and a `json` one at `$.version` for
  `package.json`. Neither is a bare string, for the reason #203/#204 wrote
  down. `cargo xtask verify-versions` counts **24** references as of
  2026-09-22; it was 22 before. ~~All `0.4.0`.~~ **Corrected 2026-09-23:**
  the gate printed `24 version references all say 0.4.1` — and, after the
  `v0.5.0` release the same day, `… all say 0.5.0` on `b747e5d5`. The count was right
  and the version was not, for the fork-point reason at the top of this page
  — [vpay#240](https://github.com/vaam-apps/vpay/pull/240) moved all 24
  together, which is what a single wrong version reference would have
  failed on.
- **The example's `Cargo.lock` records the plugin's version even though the
  dependency is by path, and nothing gates it (2026-09-23).** A path
  dependency still gets a `version` line in
  `examples/tauri-checkout/src-tauri/Cargo.lock`, and **no gate in this
  repository builds that crate**: `--locked` appears in vpay only on
  `cargo install` lines (`release-please.yml`'s own comment says so),
  `verify-versions` reads manifests and config, never a lockfile, and the
  example's `src-tauri/` carries its **own `[workspace]`** the way the plugin
  does (T2), so no root cargo command resolves it. This is not hypothetical:
  [vpay#240](https://github.com/vaam-apps/vpay/pull/240) moved the manifest
  to `0.4.1` and **stranded that lockfile at `0.4.0`**, green the whole way,
  until [vpay#241](https://github.com/vaam-apps/vpay/pull/241) repaired it by
  hand. The **release** path self-heals — `release-please.yml`'s "Refresh
  `Cargo.lock`" step globs `git ls-files '*Cargo.lock'` and runs
  `cargo metadata` in each directory, so a release-please bump picks this
  file up — **so the exposure is the manual bump only**, which is exactly
  what #240 was. _(Observed 2026-09-23: the `v0.5.0` release left
  `examples/tauri-checkout/src-tauri/Cargo.lock` at `0.5.0` with no hand
  edit, the self-heal working as described. The example's own README still
  quotes `0.4.1` in its lockfile snippet.)_ If you bump this crate's version by hand: run
  `cargo metadata --format-version 1` in
  `examples/tauri-checkout/src-tauri/` and commit the lockfile in the same
  change.
- **A merchant must grant `vpay-checkout:default`** in
  `src-tauri/capabilities/default.json`, or Tauri's capability system denies
  both commands and `start` answers `unresolved` with `platform_window_failed`
  — the correct answer, and an opaque one.

## The recipes, and what each actually proves

`just --show` any of them; all four share a `_tauri-preflight` that refuses
with a named reason when the plugin directory or its manifest is missing, and
all four drive cargo through `--manifest-path` because of T2.

| Recipe                    | Proves                                                                                                                                  | In `just ci`?                     |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `just test-tauri-rust`    | `cargo test` — the crate's unit suite **and** its one doctest in the same invocation                                                    | **no** (T7)                       |
| `just clippy-tauri-rust`  | the only thing in the repository that lints this crate, at `-D warnings`                                                                | **no** (T7)                       |
| `just check-tauri-mobile` | the `#[cfg(mobile)]` half still compiles for `aarch64-linux-android` and `aarch64-apple-ios`. **Nothing about the Kotlin or the Swift** | **no** (T7)                       |
| `just test-tauri-js`      | the vitest suite, scoped to one package                                                                                                 | not by name — `pnpm -r test` does |

`rustup target add aarch64-linux-android aarch64-apple-ios` before the third.
The justfile's own gate tally is **unchanged at fifteen**: none of these is a
gate.

## What has and has not been verified

Full accounting: `docs/status/mobile-tauri-plugin.md`. Every command with its
exit code and who ran it:
`docs/status/verification/2026-09-22-tauri-plugin.md`. Both dated 2026-09-22.

**Measured 2026-09-22 by the lane that built it** (and, for the first four
rows, re-run in the docs lane's own pass through the `just` recipes):

| What                            | Result                                                                                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| The Rust crate                  | `cargo build`, `clippy --all-targets -D warnings`, `cargo doc` (0 warnings) and `cargo fmt --check` all exit 0                             |
| Its tests                       | **19 passed, 0 failed, 0 ignored**, plus **1 doctest** — eleven of the nineteen are the desktop host                                       |
| The mobile targets              | `cargo check` exits 0 for `aarch64-linux-android` (no NDK needed) and `aarch64-apple-ios` (no env var needed)                              |
| The guest-JS package            | typecheck, lint at `--max-warnings 0`, build, and **71 vitest cases across 8 files, 0 skipped** — all exit 0                               |
| The Swift                       | `swift build --sdk iphonesimulator -target arm64-apple-ios15.0-simulator` exit 0; `xcodebuild` BUILD SUCCEEDED                             |
| `cargo xtask verify-sdk-parity` | 0 — **750 proving tests, 45 dated gaps, 39 rows** (was 662 / 37) — ~~44~~, see below. **767 / 45 / 40 on `b747e5d5`** (step A, 2026-09-23) |

~~**44** dated gaps.~~ **Corrected 2026-09-23: the gate prints
`45 dated gap(s)`**, and the 44 was wrong on the day it was written rather
than overtaken by anything. The Tauri pass added **8** `⛔` rows — the eight
listed immediately below, and `docs/sdks/parity.md`'s Tauri table carries
exactly eight — so 37 + 8 = 45, not 7 and 44. This page mirrored the figure
out of vpay's own
`docs/status/verification/2026-09-22-tauri-plugin.md`, which still said 44
when this correction was written; a companion vpay PR is fixing that page in
the same breath. `750 proving test(s)`, `35 SDK method(s)` and `39 row(s)`
were and are correct — only the gap count was mis-transcribed on arrival.

**The dated `⛔` gaps, all 2026-09-22, each a row in `docs/sdks/parity.md`:**

- **No gate compiles the Rust, the Kotlin or the Swift** (T7). Every Rust count
  quoted for this crate is a human running a recipe by hand.
- ~~The Android Kotlin host was compiled by none of the lanes that wrote
  it — a reading is not a compile.~~ **Corrected 2026-09-22, same day:** the
  example app compiled and linked it. That sentence was true of the plugin
  tree alone and is the belief this correction exists to remove — nothing
  _inside_ `sdks/tauri/` can compile the Kotlin, and no gate does; a
  consuming app's Gradle is what can.
- **`stopUrlReached` has never fired on any platform.** Unreachable on iOS
  (T6), unimplemented on desktop, and on Android it needs a merchant-verified
  host serving `assetlinks.json` this repository cannot deploy — the same wall
  the Flutter plugin's equivalent hit on 2026-09-16.
- **Desktop has no dismissal signal, and `open::that_detached` has never
  executed** — no Windows or Linux desktop build either, and no release
  build on any platform.
- **Chrome rendered the Android Custom Tab FULL-HEIGHT**, not the partial
  90 % sheet the host requests (2026-09-22). `setInitialActivityHeightPx` is
  a hint a browser may decline, and Chrome declined it. The payment worked;
  the presentation is not what the design says, and no test can catch it.
- **A host that resolves `show` and then never reports anything hangs
  `start`.** There is no stream-done signal on that seam. Named by the lane
  that built it, not fixed; a host whose `show` _rejects_ resolves
  `unresolved` correctly.
- **No real rail.** The stack behind the runs below was WireMock, as
  everywhere else in this repository, and it ran **GHCR `edge` images** of
  `vpay-server`/`vpay-checkout` — **not a build of this tree**. So the
  end-to-end runs prove the plugin against a published vpay, not against the
  branch they shipped on.
- **No merchant webhook was verified.** `examples/shop` was not run, so
  nothing checked the `payment_intent.succeeded` delivery a merchant is
  supposed to fulfil on. The `succeeded` the app rendered is a UI fact plus
  one token-authenticated read.
- **No physical device**, no release build, and no App Store or Play review.
- **Nothing is published.**

## The example app, the real builds, and the two runs (2026-09-22)

`examples/tauri-checkout/` is a Vite + vanilla-TS Tauri app with its own
`[workspace]` and a **path** dependency on the plugin. It exists because
**nothing inside `sdks/tauri/` can compile the Kotlin or the Swift** — those
need a consuming app's Gradle and Xcode — and no gate anywhere does.

### What a gate does and does not reach in the example (2026-09-23)

Not "nothing builds it", which is the easy thing to assume and is wrong.
`pnpm-workspace.yaml` globs `examples/*`, so the example is a workspace
package and its `typecheck`, `lint` and `test` scripts **do** run in
`just ci` through `pnpm -r` — and each of those scripts chains the same
`deps` step that builds the guest-JS first, so the TypeScript half is
genuinely covered.

**A bare `cargo build` in `src-tauri/` fails on a clean checkout** (written
down 2026-09-23, vpay#242, in that crate's `Cargo.toml` header):
`tauri.conf.json`'s `frontendDist = "../dist"` is read by `tauri-codegen` at
compile time and `/dist` is gitignored. `beforeBuildCommand` does not help —
the Tauri CLI runs that hook, cargo does not. Build the front end first
(`pnpm --filter @vpay/example-tauri-checkout build`) or drive the build
through `pnpm exec tauri build`. `src-tauri/gen/` (the Gradle and Xcode
projects `tauri … init` writes) is gitignored and regenerated by `init`;
`src-tauri/Cargo.lock` is tracked.

**The ungated slice is exactly `src-tauri/` — the Rust, and with it every
route to the Kotlin and the Swift — plus any script `pnpm -r` never calls,
which today is `dev`.** It appears in **0** justfile recipes, **0** files
under `.github/workflows/`, and **0** places in `.xtask/src/` (measured
2026-09-23 against `dd1a48b`). Those two are precisely what
[vpay#241](https://github.com/vaam-apps/vpay/pull/241) had to repair: a
`Cargo.lock` under `src-tauri/` left at `0.4.0` by #240, and a `dev` script
that was a bare `"vite"` with no `pnpm run deps &&`, so a clean checkout
could not run the example by following its own README. Neither could go red
anywhere.

Why that particular hole matters more than its size suggests: this example
is the **only** thing in the vpay repository that can compile the Kotlin and
the Swift at all. The artefact the whole Android and iOS evidence story
rests on is the one nothing protects. Treat a change under
`examples/tauri-checkout/src-tauri/` the way you would treat a change with
no test — because that is what it is.

**The builds, and they retired the largest caveat this plugin had.** All on
`docs/status/verification/2026-09-22-tauri-plugin.md` § "Lane D":

- `pnpm exec tauri android build --debug --target aarch64 --apk` **exit 0**.
  `dexdump` finds the `dev/vpay/tauri/checkout/…` classes in the APK and
  `aapt2 dump xmltree` shows both Activities merged into the APK's own
  manifest with `VpayCheckoutActivity` `exported="false"` and
  `VpayCheckoutAppLinkActivity` `exported="true"` — **read out of the built
  APK, not off the source manifest.**
- `pnpm exec tauri ios build --debug --target aarch64-sim` **BUILD
  SUCCEEDED**; `nm` on the app binary shows `_init_plugin_vpay_checkout`.
- **Zero edits to the plugin were needed by any real build.** Every deviation
  recorded above survived first contact with a real Gradle, Xcode and cargo
  link, unchanged.
- **An app can raise the iOS floor; the plugin cannot.** With
  `bundle.iOS.minimumSystemVersion: "15.0"`, the generated `pbxproj` carries
  `IPHONEOS_DEPLOYMENT_TARGET = 15.0` and the Podfile `platform :ios, '15.0'`.
  The plugin's own `cargo check` still needs the `#available` guard.

> **The trap that will cost you an afternoon, and it is not in the plugin.**
> The example's `VITE_VPAY_SESSION_URL` **must be quoted** in a dotenv file.
> A session URL's fragment **is** the session secret, and vite treats an
> unquoted `#` as a start-of-comment and silently drops everything after it.
> The symptom is `unresolved` with `invalid_request` and **nothing reaching
> the server** — which looks like a broken plugin and is a broken `.env`
> line. Quote the value.

### It has taken a payment — Lane D2, 2026-09-22

Recorded in full on that verification page's § "Lane D2 — the plugin driven
for real on both simulators against a running vpay", which is where the
session ids, the intent ids and the verbatim screen text live. What an agent
needs from it:

`show` and `dismiss` **crossed the IPC boundary into the Swift and the Kotlin
at runtime for the first time**, on an **iPhone 17 simulator (iOS 26.3.1)**
and an **Android 36 arm64 emulator**, against a stack of
`ghcr.io/vaam-apps/vpay-*:edge` images with WireMock rails. On both, Pay
opened vpay's **real hosted checkout page** — an `SFSafariViewController`
sheet, a Chrome Custom Tab — an MTN push with `examples/shop`'s README
"pays" number `237670000000` reached `Payment received`, and closing the
window ran **D4**'s poll to **`succeeded`**. A sheet **swiped away without
paying** answered **`pending`**, never `canceled` — D4 observed on a real
intent for the first time.

The cross-check used a **real merchant credential**
(`GET /v1/payment_intents/{id}`, a different authority from the
publishable-key `/v1/browser` read the plugin polls): both paid intents
`succeeded`, and the dismissed one read `requires_payment_method` — still
moving, exactly as `pending` claimed.

Five things that run establishes or leaves standing, each of which changes
what you should do:

- **D6's insecure-base opt-in was exercised, not just unit-tested.** The
  stack is plain HTTP, so `allowInsecureBaseUrl` had to be ticked by name.
  That is the designed behaviour; a demo stack does not get it inferred.
- **Android needed `10.0.2.2`, and the checkout container recreated with
  `NEXT_PUBLIC_VPAY_API_URL` set** — `frontends/apps/checkout/src/lib/env.ts`
  reads it at **runtime**, so rebuilding the image is not what changes it.
  **No cleartext workaround was needed:** Tauri's generated
  `app/build.gradle.kts` already sets
  `manifestPlaceholders["usesCleartextTraffic"] = "true"` for debug builds.
- **Chrome's first-run screen intercepted the first Pay**, and force-stopping
  from it resolved **`pending`** — correct (a window closed, nothing paid,
  the poll ran), and a trap worth recognising as a Chrome state rather than a
  plugin bug.
- **The Custom Tab rendered full-height.** The plugin's requested bounds
  **are** in the task record, so the request is genuinely made; Chrome simply
  did not honour it. See the gap list above.
- **`dismiss()` while a sheet is open could not be driven on iOS at all** —
  the `.large()` detent covers the app's own UI, and dragging the grabber
  dismisses the sheet outright rather than reaching a Dismiss button. So that
  path stays unexercised on iOS as well as on desktop.

> **Do not read this as the Flutter plugin's e2e evidence, and the page says
> so itself.** That suite asks `examples/shop`'s own database whether the
> order row says `paid` — a field written only by a signature-verified
> webhook delivery. **No shop ran here, the webhook receiver's journal was
> empty, and no merchant webhook was verified.** This is a
> token-authenticated read, not an independent observer acting on a
> delivery. Untouched entirely: Orange Money, the failure MSISDNs, desktop,
> and any physical device.

## More

- `docs/flows/tauri-checkout.md` — the process, and a "what can go wrong"
  table that is the fastest way to learn what each result kind means.
- `references/flutter-plugin.md` — the other mobile payer surface, and the
  fuller treatment of D1–D9 that this plugin inherits.
- `references/parity.md` — how a row is written and all three directions of
  `verify-sdk-parity`, before you edit the Tauri table.
- `vpay-tooling`'s `SKILL.md` and `references/recipes.md` — the four
  `*-tauri-*` recipes beside the six `*flutter*` ones, and why neither set is
  in `just ci`.
- `vpay-checkout`'s `SKILL.md` — the hosted page these hosts open, from the
  web side.
