---
name: vpay-sdks
description: "The six packages under sdks/ and the parity rule that binds the two merchant SDKs — add a capability to one and you add it to all, or you write a dated gap row. Covers what each SDK is and which are published, how a parity row is written and all three directions of cargo xtask verify-sdk-parity with the measured holes that motivated each, the Stripe SDK compatibility story, the five refund methods and the destination[<rail>][msisdn] payee whose leading + no SDK may add for you, the three payer surfaces — browser, Flutter and the Tauri v2 plugin whose Rust half no root gate compiles — and the generated code that exists (pigeon, ZenStack) versus the OpenAPI codegen that does not. Load before adding, renaming or removing any SDK method or test, and before touching anything under sdks/tauri."
---

# vpay SDKs

> **Verified against vpay `999a23f9` (2026-09-22).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

The Tauri payer plugin arrived with
[vpay#238](https://github.com/vaam-apps/vpay/pull/238), merged 2026-09-22 as
`999a23f9`; it is absent from any `master` checkout older than that.
~~This page carried a note that #238 was unmerged and that
`node tools/verify-coverage.mjs` therefore failed against `master`.~~
**Corrected 2026-09-22:** it merged, the stamp above moved onto the merge
commit, and the gate passes against `master`.

`sdks/` holds **six** packages as of 2026-09-22 (five until then). Two are
merchant SDKs, **three** are payer surfaces, and one is evidence rather than an
SDK.

## The rule, before anything else

> **Add a capability to one SDK and you add it to all of them — or you write a
> dated `⛔` row in `docs/sdks/parity.md` for the ones that do not have it.**

That is ADR-0015, and `cargo xtask verify-sdk-parity` enforces it on every
`just verify`, which is in `just ci`. It is not advisory and it is not a
reviewer's job.

Three things follow that catch people out:

- **Renaming a test breaks the build.** A `✅` cell names the test(s) that prove
  the capability; the gate looks each one up by exact name under that column's
  directory. Rename `a_cached_token_is_reused_across_calls` and `just verify`
  fails naming the cell.
- **A method with no row breaks the build**, naming the `file:line` it is
  declared on.
- **An `#[ignore]`d, `it.skip`ped or `skip: true` test does not count.** Nor
  does a commented-out declaration, nor a title that only appears inside a
  string.

> **A `✅` proves a test NAME exists. It never proves anything RUNS it.**
> `verify-sdk-parity` reads the name and finds it in that SDK's sources; it
> cannot tell whether any job executes it. Measured 2026-09-16: the Rust
> column's `live_refund_lifecycle` and `live_refund_destination_refusals` were
> `✅` on **three rows for a full day** while `.github/workflows/ci.yml` named
> only `--test live_invoices`, so nothing compiled that binary. The workflow
> names `--test live_refunds` too now. **Read a `✅` as "a case exists that
> would fail", not "something ran it."** Adding a row whose test lives in a
> new binary means adding that binary to the workflow in the same change —
> the gate will not remind you.

Full mechanics, including how to write a row and what each direction of the
gate refuses: `references/parity.md`. Read it before editing
`docs/sdks/parity.md`.

## The packages

| Path                                    | Name                                                                          | What it is                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Published                                                                                                                                            |
| --------------------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sdks/nodejs`                           | `@vaam-apps/vpay-sdk`                                                         | **Merchant** SDK for `/v1`. `private_key_jwt` auth, the single-401 re-auth, the form encoder, `verifyWebhook`, and nine resources. Second entry point `./stripe` exports `createStripeAuthenticator`, with `stripe` as an **optional** peer                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | yes, public                                                                                                                                          |
| `sdks/rust`                             | `vpay-sdk`                                                                    | The Rust twin. Same nine resources, same wire                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | **`publish = false`, deliberately** — "publishing a client for an API nobody can reach would be actively misleading". Flip it once `/v1` is deployed |
| `sdks/stripe-js`                        | `@vaam-apps/vpay-stripe-js`                                                   | The browser **payer** surface, Stripe.js-shaped. `loadStripe`, `initEmbeddedCheckout`, `openCheckoutPopup`, `notifyCheckoutOpener`. **Zero runtime dependencies**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | yes, public                                                                                                                                          |
| `sdks/flutter/vpay_checkout_flutter`    | `vpay_checkout_flutter`                                                       | The mobile **payer** surface. **Renders the checkout as a native Flutter sheet** (`VpayCheckoutSheet`, driven by the session's server-sent `rails`) since 2026-09-17 (#189 lane 2). The payer's own browser (Custom Tab / `SFSafariViewController`, since 2026-09-16) is no longer the checkout: the **only** thing it opens is a `redirect` rail's leg, and since 2026-09-18 (#195/#200) that is vpay's own `/c/{id}/redirect` page on the checkout origin — never the hosted page, and never the rail's own URL. **Material 3 since 2026-09-17** ([vpay#198](https://github.com/vaam-apps/vpay/pull/198)), themed through `VpayCheckoutTheme` — the sheet still paints no palette of its own, but a deployment's `primary_color` may now seed its `ColorScheme`. Reports a typed result once the intent actually settles | **`publish_to: none`** (2026-09-13)                                                                                                                  |
| `sdks/tauri/tauri-plugin-vpay-checkout` | `tauri-plugin-vpay-checkout` (crate) + `@vaam-apps/vpay-tauri-checkout` (npm) | The **Tauri v2 payer** surface, new 2026-09-22 (ADR-0023, T1–T7; it inherits ADR-0021's D1–D9 unchanged). One guest-JS state machine, three deliberately stupid hosts: a partial Custom Tab on Android, an `SFSafariViewController` on iOS, the default browser on desktop — and the **same package** is a `window.open` popup host outside Tauri, which is what "android + ios + web" meant. **The crate is its own Cargo workspace (T2), so no root gate compiles it** — see the section below before touching anything here                                                                                                                                                                                                                                                                                             | crate **`publish = false`** with a stated reason; npm package publish-ready and **published by nothing** (2026-09-22)                                |
| `sdks/stripe-compat`                    | `@vaam-apps/vpay-stripe-compat`                                               | **Evidence, not an SDK.** Drives the real `stripe@22.6.1` package against a live compose stack. `private: true`, ships nothing, and **gets no rows in the parity matrix** — "the compat suite proves claims rather than making them"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | never                                                                                                                                                |

The **three** payer surfaces are **not** additional merchant SDKs. They
authenticate a payer's _device_ with a publishable key and a per-session
`client_secret`, speak `/v1/browser`, and share **no capability row** with the
merchant tables. Each has its own table in `docs/sdks/parity.md`, and the
Tauri plugin's is a **single-column** one for the same reason the Flutter
plugin's is: there is no second implementation to diverge from, so the table's
job is the other direction — a `✅` names a test that exists, and everything
without one is a dated `⛔`.

**Nine** merchant resources as of 2026-09-16, in both languages:
`payment_intents`, `checkout.sessions`, `customers`, `invoices`,
`invoice_items`, `refunds`, `events`, `account_holders`, `balance`. (~~Eight.~~
This page said eight beside a list of nine; corrected 2026-09-16 by counting
`client.ts`. Nothing gates a count in prose.)

## Refunds: five methods each since 2026-09-16, and one trap that will bite

Both SDKs ship **create, retrieve, update, list and cancel** on `refunds`, and
both event unions carry `charge.refunded` and `charge.refund.updated`.
`refunds.create` takes a `destination`. `balance.retrieve` is now the only SDK
method in either package with no route behind it.

**The wire shape is `destination[<rail_code>][msisdn]`** — a rail-agnostic
envelope with a rail-specific interior, and **the rail's own code is the outer
key**, not a constant. The rail is not a preference: a refund goes back on the
rail the charge was made on, the server reads that off the charge, and a
`destination` naming any other rail is a `400` naming `destination`.

> **The trap.** The server's validator — `vpay_provider::RefundTarget::
mobile_money`, which is fallible and canonicalising — **requires a leading
> `+`**. `+237600000200` is a payee; the bare national `600000200` is a `400`
> naming `destination`, even though `GET /v1/account_holders` accepts the bare
> form on the same server. The asymmetry is deliberate and runs in the safe
> direction: a lookup that guesses the country wrong returns the wrong name, a
> transfer that guesses wrong sends the money.
>
> **Neither SDK normalises the number, and neither may start.** They send the
> string the merchant wrote, byte for byte — no `+` added, none stripped, no
> rewriting. An SDK that "helpfully" canonicalised would pass every one of its
> own fixtures and fail against the real server the moment the server's rule
> widened, because the fixtures would be asserting the SDK's rule back to
> itself. `the_sdk_never_normalises_a_payees_number` and `refunds.create never
normalises a payee's number on its way to the wire` are the cases; deleting
> either is how this regresses.

`destination` is optional in both param types **only** because the port allows
a rail that refunds to the instrument that paid, and such a rail refuses one.
Neither rail vpay carries is one — both declare `RefundDestination::Required`
— so a create with no `destination` is a `400` against every deployment this
repository can build.

**And the sentence none of this changes: no rail has ever returned money to
anyone.** A `201` from `refunds.create` means a row exists and its amount is
reserved against the intent. It does not mean the payee has been paid: the
refund is `pending` and **nothing settles a pending refund** (RFC-0003 open
question 8). Both SDKs' module docs open with that; keep it there.

One asymmetry recorded rather than faked: Rust's `CreateRefundParams` has a
hand-written `Debug` that redacts the payee and the Node package **cannot** —
its params are an object literal the merchant allocated, so `util.inspect` and
`JSON.stringify` print the number in full. A `⛔ 2026-09-16` row, written
because the row above it was `✅/✅` on a title naming `inspect` while neither
Node test checked it.

## Step A: two new parameters, one source break in Rust

Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged <pending>). **No method
was added**; two existing ones take more:

- **`customer` on `payment_intents.list`, `checkout.sessions.list` and
  `refunds.list`**, in both SDKs, validated by neither. An older server ignores
  it and returns the **unfiltered** list; both SDKs' doc comments say so.
- **`invoices.pay` takes one optional out-of-band object** (ADR-0024 D19) —
  Rust `out_of_band: Option<OutOfBandParams>`, Node `outOfBand` — whose
  presence puts `paid_out_of_band=true` and `out_of_band[…]` on the wire and
  no URL. A URL beside it is refused before any request.

> **BREAKING in `sdks/rust` — source, not wire.** `ListPaymentIntentsParams`,
> `ListCheckoutSessionsParams`, `ListRefundsParams` and `PayInvoiceParams`
> each gained a public field, so a struct literal naming every field with no
> `..Default::default()` stops compiling (`E0063`). A three-field
> `ListPaymentIntentsParams { … }` literal is the likeliest casualty.
> Constructors, `Default::default()` and `..Default::default()` are
> unaffected, and the bytes on the wire are the same. **`sdks/nodejs` is not
> breaking.** Recorded in `sdks/rust/README.md` § Status.

**Node's first camelCase request fields are deliberate:** `outOfBand` and
`receivedAt`, by the maintainer's decision of 2026-09-23. Do not "fix" either
SDK to match the other. Shapes, counts and the corrected parity note:
[references/parity.md](references/parity.md) § "Step A".

## Parity is per capability, not per method name or per shape

ADR-0015 decision 1. Two deliberate asymmetries you will meet:

- **One spelling divergence exists and it is a table, not a rule.**
  `PARITY_COLUMN_SPELLINGS` in `.xtask/src/main.rs` carries exactly one entry:
  Node's `invoices.markUncollectible` is the row's `invoices.mark_uncollectible`.
  Rust cannot spell the first (`non_snake_case`) and the Node package should
  not spell the second (Stripe's own Node SDK says `markUncollectible`).
  **An entry in that table that exempts nothing fails the build**, the way
  ADR-0016's serde exemption table does. A _rule_ ("TypeScript may camel-case
  any row") was rejected: it would make every future divergence silent.
- **Types diverge where the languages do.** A patch field is three-state in
  both (`Option<Option<String>>` / `string | null | undefined`) because
  `description=` clears; an _invoice item_ patch field is two-state in both,
  because its columns are `NOT NULL` and `description=` is a `400`. `refund.fee`
  is `Option<i64>` in Rust and `fee?: number | null` in TypeScript. Each cell's
  proving test asserts the resulting **body**, never the type.

## The live suites fail rather than skip

`sdks/rust/tests/live_invoices.rs` and `live_refunds.rs` (both behind the
`live-stack` cargo feature) and `sdks/nodejs/src/{invoices,refunds}.live.test.ts`,
all run by `just sdk-live` and CI's `e2e` job.

**The two languages are wired differently, and only one of them is safe.**
Node's live cases are one vitest project matching a **glob** —
`vitest.live.config.ts`, `src/**/*.live.test.ts`, with `passWithNoTests: false`
— so a new `*.live.test.ts` is picked up with no wiring. A Rust live binary
must be **named explicitly** in `.github/workflows/ci.yml`: the step is `cargo
test -p vpay-sdk --features live-stack --test live_invoices --test
live_refunds`, and `live_refunds` was missing from it for a day after wave 3
added the file, which is the measurement behind the parity-gate warning above.
Add a Rust live binary and add it to the workflow in the same change.

`sdks/stripe-compat` works the same way:
`src/preflight.ts` is a vitest `globalSetup` that fails the run when no stack
answers `/healthz` or the merchant handshake does not complete.

Run with no `VPAY_BASE_URL` they **fail, naming the variable**. They never
skip, because `AGENTS.md` forbids `#[ignore]` for exactly this reason: a suite
that skipped itself would print `ok` with zero cases and be indistinguishable,
in a CI summary, from one that passed.

It earned its keep immediately: the first live run found that `currency` is
**required** on `POST /v1/invoices`, which both SDKs had documented as optional
with a deployment default that does not exist.

## There is no OpenAPI spec and no OpenAPI codegen

Both merchant SDKs are **hand-written** against the wire contract in
`docs/flows/merchant-auth.md` and the flow docs. `verify-sdk-parity` is what
keeps them in step — there is no generator to re-run and no spec to edit.
Generated code that _does_ exist (pigeon, ZenStack) and how to regenerate it is
in `references/generated-code.md`.

## The Tauri plugin: what an agent must know before touching it

All 2026-09-22, and all caveat rather than tour — the tour is
`references/tauri-plugin.md`, which you should read before editing anything
under `sdks/tauri/`.

- **No root gate compiles the Rust, and none compiles the Kotlin or the Swift
  either.** The crate carries its own empty `[workspace]` table (T2), so
  `cargo nextest run --workspace`, `just clippy`, `just deny`,
  `verify-no-mocks`'s `cargo metadata` sweep and `verify-serde` (which scans
  `backends/crates` only) all walk past it. The only commands in the
  repository that build it are `just test-tauri-rust`,
  `just clippy-tauri-rust` and `just check-tauri-mobile` — each driving cargo
  through `--manifest-path`, and **none of the three in `just ci` or
  `just verify`** (T7: the CI image has no `libwebkit2gtk-4.1-dev`, no Android
  SDK/NDK and no Xcode). `just test-tauri-js` is a scoped convenience. The
  **TypeScript** half is the one genuinely gated part: `pnpm-workspace.yaml`
  names `sdks/tauri/*`, so `pnpm -r typecheck|lint|test` reaches it through
  `just lint-web`/`just test-web` with no recipe change. The Kotlin and the
  Swift **do** now compile and link — but only inside `examples/tauri-checkout`
  (`tauri android build` / `tauri ios build`, both exit 0 on 2026-09-22), run
  by a human, never by a gate — and the example's own `src-tauri/` is the
  ungated slice (its TypeScript does run in `pnpm -r`; its Rust, lockfile and
  `dev` script are in 0 recipes, 0 workflows and 0 xtasks, which is how
  vpay#241's two bugs stayed green — `references/tauri-plugin.md`).
- **The state machine is guest-JS (T1)**, so **a change to D1 or D4 behaviour
  is a TypeScript change.** The Rust, Kotlin and Swift hosts hold no rule
  about money; their whole vocabulary is
  `{ outcome: "dismissed" | "stopUrlReached", reachedUrl }`, once per `show`,
  with no `succeeded`/`canceled`/`failed` on that wire at all.
- **iOS cannot produce `stopUrlReached` (T6)** — not "unverified",
  unreachable. tauri-v2.11.6's Swift `Plugin` base class exposes no
  app-delegate hook, and Tauri's own `deep-link` plugin ships no `ios/` at
  all. **Every iOS checkout ends `dismissed`**, which D1/D4 make correct;
  `matchesStopUrl`/`handleUniversalLink` exist and nothing calls them. Do not
  "fix" this by deleting them or by inventing a hook.
- **Desktop has no dismissal signal at all (T4)**, by decision. `dismiss()` is
  the only trigger. The Flutter macOS host shipped a focus-regained signal
  once and removed it on 2026-09-16; do not re-invent it here.
- **`verify-sdk-parity` cannot cite a test whose title contains `|`.** Its
  table reader splits a row on every raw `|` with no escape handling, and two
  live cases in `guest-js/host-tauri.test.ts` assert the invoke names
  `plugin:vpay-checkout|show` / `|dismiss`. Both pass; neither can appear in a
  cell, and the parity table says so in prose instead. **Do not rename a test
  to suit the parser.**
- **`permissions/autogenerated/` and `permissions/schemas/` are tracked build
  output.** `tauri_plugin::Builder` rewrites them on **every** `cargo build`,
  and they are in `.prettierignore` because a formatted copy is un-formatted
  again by the next build. Commit what the generator wrote.
- **swift-rs compiles the Swift for iOS 13.0** unless
  `IPHONEOS_DEPLOYMENT_TARGET` or a consuming app's
  `bundle.iOS.minimumSystemVersion` says otherwise, and the host needs 15.0
  for `sheetPresentationController`: `cargo check --target aarch64-apple-ios`
  failed **exit 101** until an `if #available(iOS 15.0, *)` guard landed in
  `VpayCheckoutExternalBrowserSession.swift`. **That guard is load-bearing.**
- **The guest-JS depends on `@vaam-apps/vpay-stripe-js`** (`workspace:*`, T3)
  and every npm script chains a `deps` script that builds it first — its types
  resolve through `exports` to `dist/`. A bare `tsc` there fails for that
  reason and no other.
- **Quote `VITE_VPAY_SESSION_URL` in any dotenv file.** A session URL's
  fragment **is** the session secret, and vite reads an unquoted `#` as a
  comment and silently drops it — the symptom is `unresolved` with
  `invalid_request` and nothing reaching the server, which reads as a broken
  plugin and is a broken `.env` line (2026-09-22).

## More

- `references/parity.md` — how to write a row, all three directions of the
  gate, and the measured holes each direction closed. **Read this before
  editing the matrix.**
- `references/stripe-compat.md` — using the official Stripe SDKs against vpay:
  the one seam, and the divergences an integration actually hits.
- `references/flutter-plugin.md` — the payer plugin's three architectures
  (WebView → payer's browser → native sheet), which one is current, the
  screen machine and what still has no native analogue.
- `references/tauri-plugin.md` — the Tauri v2 payer plugin: the shape, the
  wire between guest-JS and the four hosts, T1–T7 with the reason for each,
  the dated gaps, and which `just` recipe proves what. **Read it before
  editing anything under `sdks/tauri/`.**
- `references/generated-code.md` — pigeon and ZenStack, and the hand-added line
  that regeneration will silently drop.
