# `sdks/flutter/vpay_checkout_flutter`

_Verified against vpay `84143e1d` (2026-09-18). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

A **payer** surface, like `@vaam-apps/vpay-stripe-js` — not a third merchant
SDK. It authenticates a payer's _device_ with a publishable key and a
per-session / per-intent `client_secret`, speaks the same `/v1/browser` routes,
and shares no capability row with the merchant tables. It has its own
single-column table in `docs/sdks/parity.md`.

Design: `docs/plans/2026-09-13-flutter-plugin.md` (D1–D9). Decisions:
ADR-0021. Process: `docs/flows/mobile-checkout.md`. `publish_to: none`.

## Three architectures — know which one you are reading about

This package has been rebuilt twice since it shipped. A sentence written for
an earlier build reads as current and is not — check which of these three it
describes before you trust it:

1. **A `WebView`/`WKWebView`** (original design, through 2026-09-15) —
   rendered vpay's hosted checkout page inside the merchant app's own
   process.
2. **The payer's own browser, with a selectable `inApp`/`externalBrowser`
   mode** (2026-09-14, D8) — **cutover 2026-09-16 (D5, revised)**: the mode
   toggle and the `WebView` are both **deleted outright**, not deprecated. A
   partial Custom Tab on Android, `SFSafariViewController` on iOS,
   `NSWorkspace` on macOS, `window.open` on web — the only surface, because a
   `WebView` puts `evaluateJavascript`, the cookie store and a navigation
   delegate inside a process the merchant controls, where a compromised app
   could read a payer's PAN and OTP undetectably.
3. **A native Flutter sheet, `VpayCheckoutSheet`** (#189 lane 2,
   2026-09-17 — **current**). The browser from step 2 does not go away —
   **it stops being the checkout and becomes only the `redirect`-rail
   handler.** MTN's push form and Orange's ready-to-redirect prompt now
   render as Flutter widgets, driven by the session's own server-sent
   `rails` array (#186). This is what the rest of this page describes.
   **Narrowed again on 2026-09-18 (#195, PR #200):** what that browser is
   handed is vpay's own `/c/{cs_id}/redirect` page on the checkout origin,
   never the rail's URL — see "The browser leg is a controlled surface"
   below.

There is no `mode` argument left anywhere in this package. If you see one,
you are reading about architecture 1 or 2.

## What it does now, in order

1. **Pre-flight.** `SheetController` reads the session
   (`GET /v1/browser/checkout/sessions/{cs_id}`), buys the intent's own
   `client_secret`, and gets back `rails` — each rail's `code`, `flow`
   (`push` or `redirect`), `label_key`, an optional `display_name`, and its
   typed fields. **No SDK release is needed for a new rail's fields, flow or
   validation** — only its label falls back to the raw code until the
   catalogue is updated. An unrecognised field type (`"card"` included)
   decodes to `RailFieldKindUnknown` and is never rendered — see "Cards are
   out of scope" below.
2. **Render.** `VpayCheckoutSheet` — reachable through `showVpayCheckoutSheet`
   (a large-detent, draggable `showModalBottomSheet`) or
   `showVpayCheckoutSheetRoute` (a full route) — walks the 14-screen state
   machine ported from `machine.ts`, skipping `select_rail` straight to the
   single rail's entry screen when only one rail is supported. Theming is
   **Material 3 through `VpayCheckoutTheme`** since 2026-09-17 — this said
   "nothing here reads a `VpayCheckoutTheme`" until then. See "Theming" below.
3. **Confirm.** For a `push` rail, `BrowserClient.confirmPaymentIntent`
   (new in this lane) POSTs form-encoded, bracket-nested
   `payment_method_data[type]` / `payment_method_data[<rail>][msisdn]` —
   directly, no rail code branched on in the code that builds the request.
4. **Poll.** `SheetController` polls with lane 1's jittered ladder
   (`poll_jitter.dart`, `[0.75, 1.25] × interval`, now actually wired into a
   controller for the first time) until the intent stops moving. The
   terminal rule is the ported `client.ts:673-679` rule: `succeeded`,
   `canceled`, or `requires_payment_method` **with a `last_payment_error`
   present**. A bare `requires_payment_method` is not terminal — it means
   "never confirmed," not "failed."
5. **Redirect rails hand off, they don't take over — and what they hand off
   is a vpay page, not the rail.** A `flow: "redirect"` rail (Orange) records
   `CheckoutRedirecting` **before** the platform host is ever shown, then
   opens the **same** `VpayCheckoutPlatform` browser host architecture 2
   built. When that resolves (`stopUrlReached` or a dismissal), control
   returns to the sheet's confirming/waiting state; a redirect confirm that
   answers with no `next_action` polls directly, without ever opening a
   window. **Since 2026-09-18 (issue #195, PR #200)** the URL handed to that
   browser is `{checkout_base}/c/{cs_id}/redirect?key=…#{client_secret}`,
   built by `SheetController.redirectLegUrlFor` — **never the rail's own
   URL.** See "The browser leg is a controlled surface" below.
6. **Answer.** One of `VpayCheckoutSucceeded`, `VpayCheckoutFailed`,
   `VpayCheckoutCanceled`, `VpayCheckoutPending`, or `VpayCheckoutUnresolved`
   — unchanged from architecture 2. A sheet dismissed **before any confirm**
   resolves `Unresolved`, never a synthesised `Canceled`; a sheet dismissed
   **mid-payment** (after confirm, intent still moving) polls first (D4) and
   resolves `Pending`, never `Canceled`.

## The design decisions the tests actually pin

- **D1 — the outcome is never read off a URL.** Still true, now proven at
  two layers: `resolveAfterStopUrlReached` takes no URL argument at all (the
  signature itself is the proof, architecture 2), and the sheet's own poll
  is what resolves a push confirm — nothing in `sheet_controller.dart`
  short-circuits on a confirm response.
- **D4 — a dismissal polls before it reports.** `mid-payment (after
confirm, still moving) resolves Pending — never canceled`, `before any
confirm, resolves Unresolved — never canceled`.
- **D2 — stop-URL matching is scheme + host + port + path**, used now only
  by the redirect hand-off (step 5 above). Query and fragment are ignored.
- **D3 — no JavaScript bridge, no native peer added to any page.** Extended,
  not just restated, by architecture 2: the checkout page itself now runs in
  a separate browser process this plugin's code cannot reach at all.
- **D6 — the `client_secret` and the session URL stay out of every error,
  diagnostic and `toString`**, including the generated channel types.
  `SheetController` reads a `client_secret` directly (never a whole session
  URL) the same way `CheckoutController.preflight` already does.
- **A bad or expired link is the uniform 404**, mapped to the same typed
  error the six causes `browser::authenticate` does not distinguish between.

## The browser leg is a controlled surface (#195, PR #200, 2026-09-18)

**The sheet never hands a redirect rail's own URL to the browser.** It opens
`{checkout_base}/c/{cs_id}/redirect?key=…#{session client_secret}`, built by
`SheetController.redirectLegUrlFor`. That vpay page reads the session, reads
the **intent** for the rail's URL, marks the tab as a sheet's redirect leg, and
navigates. `CheckoutRedirectRequired` still carries the rail's URL — that is
what the state means — but `_handOffToBrowser` is not given it, so a crafted
link to a payment origin can never become an open redirect.

> **`SheetController.sessionPageUrlFrom(sessionUrl)` is the only supported
> source of the checkout origin**, and it is `public` rather than
> `@visibleForTesting` for exactly that reason: "the alternative every caller
> reaches for first (`client.baseUrl`) is the wrong origin". `BrowserClient.
baseUrl` is `deployment.public_base_url` — the **API**, which mounts no
> `/c/` route at any prefix (`:8080` in `compose.demo.yml`, against the
> checkout app's `:3080`). The leg shipped built on it, green, because the
> Dart test passed one string as both origins and asserted a URL equally true
> of the right origin and the wrong one. `sessionPageUrlFrom` takes everything
> before the first `?` or `#` of the **server-minted** session URL (`vpay-db`'s
> `hosted_url`), so a deployment path prefix survives it.

Two details that look like sloppiness and are not: the `client_secret` goes
into the fragment **raw**, because `parsePageCredentials` calls
`decodeURIComponent` and percent-encoding would double-decode; and the base is
stripped of trailing slashes only, because `//redirect` is a path segment the
app does not route.

**D2 consequence, and it is a real narrowing.** Stop-URL matching still runs,
but a suppressed return page renders **no "return to the merchant" button**, so
the browser leg can no longer reach one of `stopUrls` by itself. The sheet
therefore resolves a redirect rail on the **dismissal** signal alone — which
D4's poll makes correct regardless, and which is the same signal already called
unverified on every platform. Do not "fix" this by adding a button to the
suppressed screen; that re-creates the duplicate outcome #195 exists to close.

**`sessionPageUrl` is a required parameter of `SheetController`** as of this
PR. Merging #197 and #200 in either order is conflict-free under `git` and the
result **does not compile**: #197 adds nine `SheetController(...)`
constructions to `test/sheet/sheet_controller_test.dart` and they fail
`dart analyze` with `missing_required_argument`. One line per site
(`sessionPageUrl: _sessionPageUrl,`), owed by whichever merges second.

## Money never touches a float, and the terminal rule is not the obvious one

Two of the bar `frontends/apps/checkout` set that a port loses first if
nobody watches for it, both pinned by name:

- **Money.** `money_format.dart` is built on lane 1's `money.dart` digit
  surgery (ported from `money.ts:46-56`) — `amount / 100.0` is exactly the
  bug this exists to prevent, and the zero-decimal currency table (XAF, XOF,
  JPY, KRW, CLP, VND) is honoured. `a zero-decimal currency (XAF) never
gains a decimal point`, `never touches a float — digit surgery only,
reusing money.dart`.
- **The poll's terminal rule.** `intent_updated on confirming with a
non-terminal intent becomes waiting, carrying the rail` is the case that
  fails first if someone "simplifies" `hasStoppedMoving` back to checking
  only `status`.

## Cards are out of scope — structurally, not by convention

`RailFieldKind.fromJson` cannot represent a card field: an unrecognised
field type decodes to `RailFieldKindUnknown`, which `rails.dart`'s
`railChoices` already refuses to render (lane 1, D9). A native PAN field
would move the integration from PCI **SAQ-A to SAQ-D**; nothing added in
lane 2 changes that boundary, and nothing in the sheet's own code path can
represent a card field even by mistake.

## What has no native analogue, and is dropped by an ADR line, not silently

`frame.ts`/`origins.ts`/`csp.ts`/`entry.ts` — the hosted page's D4/D8
iframe-embedding refusal and the parent `postMessage` protocol — are **not**
ported. A Flutter widget tree has no iframe to be embedded in, no parent
frame to police an origin against, and no `postMessage` channel to gate.
Recorded in ADR-0021's 2026-09-17 addition, not an oversight: those files
still gate a real security property for the hosted page (a different threat
model — an arbitrary embedding site, not a merchant's own app process), and
the sheet's own boundary is D6 above plus D3's continued refusal of a
native-to-page bridge.

## "Remember this number on this device" — page memory (2026-09-17/18)

`remember_msisdn.dart` is the hosted page's `src/lib/memory.ts` ported to
`shared_preferences` rather than IndexedDB, and the two rules that matter are
the **same**: a 90-day TTL **enforced on read** (`rememberedMsisdnTtl`, the
web's `MEMORY_MAX_AGE_MS`), and a write at **exactly one deliberate moment** —
the payer ticks the box and submits an entry screen. Visiting, choosing a rail
or reading an outcome stores nothing. No PIN, ever. The shared-phone warning
is inside `CheckboxListTile`'s own merged semantics label, not a tooltip.

`VpayRememberedMsisdn` is the class `SheetController` talks to. Two of its
predicates are **not** interchangeable and picking the wrong one is the defect
issue #194 was half made of: `hasRecord()` is true for an expired record too
(the "Forget" button still shows, because an expired record is still something
to forget); `hasActiveRecord()` is the **non-expired** half and is what seeds
`rememberChecked`, because an expired record is nothing the box can truthfully
say the device still remembers. The web agrees by construction —
`parseMemoryRecord` returns `null` for an expired record, so its box is
unticked for one too.

**The read was broken until 2026-09-17 (issue #194, PR #197)** — field empty
and box unticked after a cold relaunch, while "Forget" still rendered. Two
independent defects, neither in the store: the prefill was gated on a screen
transition while `defaultMsisdn` arrives **asynchronously** (so the value
always landed after the one chance to use it), and `rememberChecked` was only
ever set by the payer's own tap. Now the prefill runs on **every** controller
change, guarded by `_msisdnManuallyEdited`, empty text and a
`CheckoutCollectMsisdn` state — a payer editing a prefilled number is never
overridden — and the box seeds **once per sheet** from `hasActiveRecord()`.

**An unticked submit now CLEARS the record** (`submitMsisdn` calls `forget()`),
matching `checkout-client.tsx`'s `rememberOnSubmit -> pageMemory.clear()`.
Without it a payer who unticks and pays still got the number back on the next
relaunch — the very read-back #194 is about. `forgetRemembered` also unticks
the box, matching the web's `onForget -> setRemember(false)`.

> **The ordering bug the review found, and the reason to read this before
> touching `_loadRememberedMsisdn`.** That clear is guarded by
> `_rememberSeeded` so it cannot run before the async seed lands. As first
> written the flag was set **before** awaiting `hasActiveRecord()`, so the
> window it exists to close was open for the whole length of the read: a
> submit inside it saw `_rememberSeeded == true` with `rememberChecked` still
> `false`, took the unticked branch, and **destroyed a record the payer had
> never unticked.** The test that was meant to pin this gated the _first_ of
> the three reads and so proved a window that was never the dangerous one.
> Set the flag **after** the value it announces;
> `_SeedGatedRememberedMsisdnStore` gates the seed's own read and fails on the
> old ordering with `Expected: not null / Actual: <null>`.

### Three named divergences from the web — none of them accidental

1. **No "Last used" badge on the rail picker.** The web marks the remembered
   rail (`screens.tsx`, `data-testid="last-used"`). The sheet's picker is a
   plain `OutlinedButton` per rail. The string exists in **both** locales
   (`i18n.dart`, `memory.last_used`) and **nothing in `lib/` reads it** — so
   this is a gap, not a decision. (Not auto-selecting the remembered rail
   **is** a decision, and matches the web: `lastRail` is "a hint, never a
   preselection".)
2. **No `normalizeCameroonMsisdn` re-validation on read.** The web's
   `parseMemoryRecord` runs `isStoredMsisdn` — `normalizeCameroonMsisdn(v) === v`
   — before a stored number reaches the form. The Flutter read path only
   JSON-decodes and checks the TTL and rail; `normalizeCameroonMsisdn` is
   called at **submit** time only.
3. **`startRedirect` neither writes nor clears**, so the box on a redirect
   rail's entry screen **seeds and reads but persists nothing** — it is
   display-only. The web's `onStartRedirect` is not: it calls
   `rememberOnSubmit(null, rail)`, which writes a rail-only record or clears
   one. #197 made the redirect screen offer the box and the "Forget"
   affordance; it did not give that screen a write.

**And one open defect, left visible rather than claimed closed (2026-09-17).**
`chooseRail` calls `_setState` — and so `notifyListeners()` — **before**
`_loadRememberedMsisdn`, so the widget rebuilds once while `defaultMsisdn`
still holds the previous rail's number. The field is never actively cleared on
a rail switch, so a payer who clears it on rail A, goes back and picks rail B
is prefilled with **rail A's** number. No test covers it and none can today:
rails come from `CheckoutSession.rails` and no fixture or deployment offers two
`push` rails at once (`orange_money` is `redirect` and renders no field).
Reachable in principle — `rails.dart` branches on `RailFlow`, never on a rail
code, so a second push rail needs no client change.

## Theming — Material 3, and the one contract that was reversed

`VpayCheckoutTheme` (`lib/src/sheet/checkout_theme.dart`, exported from the
package root alongside `kVpayCheckoutSheetCornerRadius`) is the only way a
host app changes how the sheet looks. Pass it to `showVpayCheckoutSheet`,
`showVpayCheckoutSheetRoute` or `VpayCheckoutSheet`. Every field has a
default that produces the stock M3 sheet, so `const VpayCheckoutTheme()` and
passing nothing are the same thing.

**The reversal, because a skill that misses it teaches the old rule.** Until
2026-09-17 this package deliberately had _no_ theming surface: the sheet
"inherits `ThemeData`, never a fixed vpay palette", and
`CheckoutPageBranding.primaryColor` was parsed out of
`/.well-known/vpay-checkout` and pointedly **not** rendered. The maintainer
reversed that in vpay#198. What survived the reversal is the part worth
keeping straight: **the sheet still has no palette of its own** — there is no
vpay colour anywhere in it, and every colour it draws is a `ColorScheme`
role. What changed is only _whose_ colour wins.

`resolve(base, {deploymentBrandColor})` decides that, highest first:

| #   | Input                            | Becomes                                                         |
| --- | -------------------------------- | --------------------------------------------------------------- |
| 1   | `colorScheme`                    | used verbatim                                                   |
| 2   | `seedColor`                      | `ColorScheme.fromSeed`                                          |
| 3   | the deployment's `primary_color` | `ColorScheme.fromSeed`, unless `useDeploymentBrandColor: false` |
| 4   | nothing                          | `base.colorScheme` — the host app's, untouched                  |

A host that passes no theme and deploys no brand colour gets exactly the
appearance it had before. The ordering is the point: the deployment's colour
is the _operator's_ brand, and it loses to anything the app developer says
explicitly, because the app developer is the one looking at the screen.
`brightness` defaults to the host's, which is what lets a seeded sheet
respect a dark host without the caller re-deriving it.

The rest of the surface, with its defaults:

| Field                 | Default                               | What it is                                                                                                                                                                                                                                                   |
| --------------------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `sheetCornerRadius`   | `kVpayCheckoutSheetCornerRadius` = 28 | the modal's top corners. 28 and not Material's default because a _system browser sheet_ replaces this one on a redirect rail, and both iOS and Android draw those much rounder — a squarer vpay sheet handing over to a rounder system one reads as a glitch |
| `surfaceCornerRadius` | 16                                    | the amount card, notice and outcome blocks, rail tiles                                                                                                                                                                                                       |
| `fieldCornerRadius`   | 12                                    | the MSISDN field                                                                                                                                                                                                                                             |
| `buttonCornerRadius`  | 16                                    | squarer than M3's stadium on purpose, so the primary action reads as a button beside the rounded tiles                                                                                                                                                       |
| `minimumTapTarget`    | `Size.fromHeight(52)`                 | above both floors (Material 48dp, Apple 44pt) rather than at either — a payer uses this once, one-handed, anxious about money                                                                                                                                |
| `contentPadding`      | `fromLTRB(20, 12, 20, 24)`            | around the scrolling content                                                                                                                                                                                                                                 |
| `filledFields`        | `true`                                | M3-filled MSISDN field; `false` gives outlined, for a host whose every other field is outlined                                                                                                                                                               |
| `textTheme`           | `null`                                | keeps the host's typography — the sheet addresses type through M3 _roles_, never hardcoded sizes                                                                                                                                                             |

`showVpayCheckoutSheet` also still takes `borderRadius`, and it **wins over**
`theme.sheetCornerRadius`: a caller passing it is naming the exact geometry.

**What `resolve` actually installs, and why it is not just colours.**
`useMaterial3` is never set — it has defaulted to `true` since Flutter 3.16
and this package floors at `>=3.47.0`, so setting it would be noise. What the
class supplies is the component theming M3 wants and Flutter does not default
for you: a filled `InputDecorationTheme` with a real shape (without it the
MSISDN field falls back to M2's underline on a host with no theme), the four
button themes carrying `minimumTapTarget` as `minimumSize`, and
`cardTheme.elevation: 0` so a card inside an already-raised bottom sheet does
not read as two stacked elevations.

**A trap this cost a session.** Every private helper in `checkout_sheet.dart`
takes its `BuildContext` from a `Builder` _under_ the installed `Theme`, never
from `State.context`. `State.context` sits **above** that `Theme`, so a helper
reading `Theme.of(this.context)` silently gets the host's scheme and ignores
the seeded one — the sheet is themed in the widget tree and unthemed on
screen. Harmless while the override only carried button sizes; a real bug the
moment it carried a `ColorScheme`.

## i18n — French default, 74 keys, both locales complete

`lib/src/sheet/i18n.dart` carries 74 keys against the page's 75 —
`frontends/apps/checkout/src/i18n/{en,fr}.ts`. The single divergence is
deliberate and asserted by name in `test/sheet/i18n_test.dart`: the page's
`outcome.back_to` / `outcome.back_to_unnamed` collapse into one
`outcome.done`, because a native sheet drawn over the merchant's own app
never took the payer anywhere to come back from. (It was 72/72 until
2026-09-17, when `state.resume_redirect_*` added three to each side.) `VpayLocale.fallback` is
`fr` — defaulting to English would be a regression, because French is
Cameroon's and Orange's language. A rail's `label_key` resolves through the
catalogue; an unknown rail falls back to the deployment's own configured
`display_name`, then its raw code.

## A real defect this lane's own gate caught — not a unit test (2026-09-17)

Every confirm the sheet sent was refused by the real server:
`payment_method_data[type]` never arrived, because `BrowserClient`'s form
encoder percent-encoded the bracket syntax's own structural characters
(`payment_method_data%5Btype%5D`), and `vpay-api`'s form parser
(`backends/crates/vpay-api/src/form.rs`) splits a raw key on the **literal**
`[` before decoding anything. Every assertion in
`test/browser_client_test.dart` had passed regardless, because they read the
request back through `Uri.splitQueryString`, which decodes the whole key
before an assertion ever sees it — green tests, broken product, found only
because the hand-driven walk actually tried to pay. Fixed
(`BrowserClient._bracketKey`); the test now asserts the **literal wire
bytes**, and reverting the fix is confirmed to fail it.

**A related, non-SDK finding from the same walk**: `examples/shop`'s
previously advertised MTN demo number `237600000000` is not a valid
Cameroon mobile number under the server-side `phonenumber` validation the
sheet's confirm now goes through (#186) — `curl` reproduces the same `400`
independent of this SDK. `237671234567` (a real MTN prefix) confirms
cleanly. See "WireMock steering MSISDNs" in `vpay-tooling`'s recipes
reference for the parallel, already-fixed problem in vpay's own chaos-test
fixtures.

## Gates, 2026-09-17 (`docs/status/verification/2026-09-17-flutter-native-sheet.md`)

| Gate                                  | Exit                                                                                                                                                                                             |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `flutter test`                        | 0 — **258 passed / 0 skipped** (82 through architecture 2, 200 after lane 1)                                                                                                                     |
| `dart analyze --fatal-infos .`        | 0                                                                                                                                                                                                |
| `dart format --set-exit-if-changed .` | 0 — 57 files, 0 changed                                                                                                                                                                          |
| `cargo xtask verify-sdk-parity`       | 0 — 661 proving tests, 37 dated gaps, 39 rows                                                                                                                                                    |
| `just test-flutter-e2e`               | 0 — 3 passed, against a real, rebuilt `vpay-demo` stack                                                                                                                                          |
| `just test-flutter-emulator`          | 0 — **two** suites (`checkout_dismiss_test.dart`, `checkout_external_browser_test.dart`) — the third, the old in-app-`WebView` suite, was deleted with the `WebView` in architecture 2's cutover |

`just ci` was **not** run for this lane, per its own brief — none of the six
`*flutter*` recipes is in it (D-M3). See `vpay-tooling`'s recipes reference
for what each recipe proves and needs.

**`flutter test`'s count after the two follow-ups, and why there is no single
number.** Each branch measured **its own tree**, so these are not a series:
#197 (`2026-09-17-flutter-remember-msisdn-read.md`) reports **305 / 0** on its
base and **306 / 0** after the review pass added the seed-window case, on
Flutter 3.48.0-1.0.pre / Dart 3.14 rather than the pin; #200
(`2026-09-17-redirect-leg-review.md`) reports **302 / 0** on its base, 297
before. A throwaway merge of the two, which **no branch carries**, is **314
passed / 0 failed** once the nine `sessionPageUrl:` arguments are added. Quote
the branch, not a total.

## Driven by hand, Android only, 2026-09-17

Three cold MTN launches to `paid` in `examples/shop`'s own database (via a
real signed webhook, never a value the app computed itself), one Orange
redirect hand-off and back through the same browser host architecture 2
built, and one deliberate mid-payment dismissal that stayed on the sheet's
waiting screen (D4) and later reached the rail's own real terminal state
(`failed`, `payer_timeout`) rather than a fabricated `canceled` at the
moment of dismissal. Full detail, screenshots and transaction ids:
`docs/status/verification/2026-09-17-flutter-native-sheet.md`.

## What is still not done, honestly

- **iOS and macOS were not touched by this lane** — Android only, on the
  maintainer's own `emulator-5554`. They remain compiled by nobody (no
  `xcodebuild` on this repository's Linux host); reviewed by reading only.
- **The redirect hand-off's stop-URL matching is still unverified on every
  platform** — the same deep-link signal architecture 2's cutover
  documented. Every Orange walk in this lane also ended in `dismissed`,
  never `stopUrlReached`; D4's poll is what made that correct anyway. **Since
  2026-09-18 the leg can no longer reach a stop URL at all** — see the D2
  consequence above.
- **The redirect leg has never been driven end to end** (2026-09-18) — not on
  a device, not against `just demo-up`. Every number behind #200 is unit- or
  component-level, and whether `SFSafariViewController` and Chrome Custom Tabs
  carry the `sessionStorage` marker across the rail's cross-origin redirect is
  reasoned about and asserted in jsdom, **never measured**. It fails to the
  duplicate screen, never to a wrong outcome.
- **The "remember Orange Money" checkbox writes nothing** — see above. #197
  gave that screen the box's _state_ and the "Forget" affordance; it did not
  give it a write.
- **No "Last used" badge on the rail picker** (2026-09-17) — the string is in
  both locales and nothing reads it.
- **No real rail, anywhere.** WireMock behind every walk in this lane, as
  everywhere else in this repository.
- **No CI gate runs any of the six `*flutter*` recipes** (D-M3, unchanged).
  Every count on this page is a human running the recipe by hand.
- **The Android/web window proof from architecture 2 is retired, not
  current**, for the one suite it depended on: `checkout_window_test.dart`
  and its debug-only JS-injection hook (`VpayCheckoutActivityTestHarness.kt`)
  were deleted in the 2026-09-16 browser cutover along with the `WebView`
  they drove. No suite in this repository drives a full MTN push through
  vpay's real hosted checkout page end to end on Android any more — the
  native sheet replaces what that suite proved, on a different code path.
- **D8's "tier 1"** (Android App Links / iOS 17.4+ Associated Domains, so
  the redirect hand-off's browser can close itself) is still not built —
  needs a merchant-hosted `assetlinks.json`/`apple-app-site-association`
  deployment this repository cannot provide.

## More

- `docs/flows/mobile-checkout.md` — the process, current as of 2026-09-17.
- ADR-0021 — every decision, including the 2026-09-17 addition recording
  what has no native analogue and reaffirming cards are out of scope.
- `docs/status/mobile-flutter-plugin.md` and
  `docs/status/verification/2026-09-17-flutter-native-sheet.md` — the area
  page and this lane's dated verification evidence.
- `docs/status/verification/2026-09-17-flutter-remember-msisdn-read.md` (#194
  / PR #197) and
  `docs/status/verification/2026-09-17-redirect-leg-review.md` (#195 / PR
  #200) — the two follow-ups above, each with its own mutation runs and its
  own "what this does not prove".
- `vpay-checkout`'s `SKILL.md` — the `/c/{id}/redirect` page and the return
  page it suppresses, from the web side.
- `vpay-tooling`'s recipes reference — the six `*flutter*` recipes, what
  each needs, and the `.agents/skills/` prettierignore rule (a different,
  unrelated vendored-skills directory inside vpay itself).
