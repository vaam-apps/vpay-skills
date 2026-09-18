# `sdks/flutter/vpay_checkout_flutter`

_Verified against vpay `9d83ff0e` (2026-09-17). Version-sensitive claims
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
   `showVpayCheckoutSheetRoute` (a full route) — walks the 13-screen state
   machine ported from `machine.ts`, skipping `select_rail` straight to the
   single rail's entry screen when only one rail is supported. It inherits
   the host app's own `ThemeData` — nothing here paints a fixed vpay palette
   or reads a `VpayCheckoutTheme`.
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
5. **Redirect rails hand off, they don't take over.** A `flow: "redirect"`
   rail (Orange) records `CheckoutRedirecting` **before** the platform host
   is ever shown, then opens the **same** `VpayCheckoutPlatform` browser
   host architecture 2 built. When that resolves (`stopUrlReached` or a
   dismissal), control returns to the sheet's confirming/waiting state; a
   redirect confirm that answers with no `next_action` polls directly,
   without ever opening a window.
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

## "Remember this number on this device"

`remember_msisdn.dart` — `shared_preferences`-backed (not the hosted page's
IndexedDB, but the same shape: opt-in, 90-day TTL **enforced on read**,
written only after a real accepted confirm, no PIN ever, a "Forget"
affordance). The shared-phone warning is inside `CheckboxListTile`'s own
merged semantics label, not a tooltip. `a record exactly at the 90-day
boundary is still readable`, `a record one microsecond past the 90-day TTL
is no longer readable`.

**Named gap, not silently dropped**: the redirect entry screen's own
"remember Orange Money on this device" checkbox renders and its label reads
correctly, but only the MSISDN half of page memory has a persistence layer —
no record is written for a redirect rail's own "remember" checkbox.

## i18n — French default, 72 keys, both locales complete

`lib/src/sheet/i18n.dart` carries the same 72 keys as
`frontends/apps/checkout/src/i18n/{en,fr}.ts`. `VpayLocale.fallback` is
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
  never `stopUrlReached`; D4's poll is what made that correct anyway.
- **The "remember Orange Money" checkbox writes nothing** — see above.
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
- `vpay-tooling`'s recipes reference — the six `*flutter*` recipes, what
  each needs, and the `.agents/skills/` prettierignore rule (a different,
  unrelated vendored-skills directory inside vpay itself).
