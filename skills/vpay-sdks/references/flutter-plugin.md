# `sdks/flutter/vpay_checkout_flutter`

A **payer** surface, like `@vaam-apps/vpay-stripe-js` — not a third merchant
SDK. It authenticates a payer's _device_ with a publishable key and a
per-session / per-intent `client_secret`, speaks the same `/v1/browser` routes,
and shares no capability row with the merchant tables. It has its own
single-column table in `docs/sdks/parity.md`.

Design: `docs/plans/2026-09-13-flutter-plugin.md` (D1–D9). Decisions:
ADR-0021. Process: `docs/flows/mobile-checkout.md`. `publish_to: none`.

## What it does, in five steps

1. Parse the session URL the merchant's backend handed the device.
2. **Pre-flight** — read the session, derive the stop URLs, buy the intent's
   own `client_secret`. An `embedded` session is **refused here, before any
   window opens**, with a typed error.
3. **Show** — a native window (Android `WebView`, iOS/macOS `WKWebView`, a
   `window.open` popup on web) loading `session.url`. The window's only job is
   to report "reached one of these stop URLs" or "the payer left".
4. **Resolve** — poll the payment intent, on a budget.
5. **Answer** — one of `VpayCheckoutSucceeded`, `VpayCheckoutFailed`,
   `VpayCheckoutCanceled`, `VpayCheckoutPending` (still processing — not a
   failure and not lied about as one) or `VpayCheckoutUnresolved` (a typed
   `VpayError`, never a silent guess).

## The design decisions the tests actually pin

- **D1 — the outcome is never read off a URL.** The proving test names the
  proof itself: _`resolveAfterStopUrlReached` takes no URL argument at all —
  the signature itself is the proof._ Reaching `success_url` with an intent
  that is still `processing` yields **pending**, not succeeded.
- **D4 — a dismissal polls before it reports.** A dismissed sheet fifteen
  seconds after the payer approved an MTN push is not a cancellation; it is a
  payer who has not been asked yet. A dismissal with a succeeded intent reports
  `succeeded`.
- **D2 — stop-URL matching is scheme + host + port + path.** Query and fragment
  are **ignored**, so they are not carried across the platform channel at all —
  there is nothing there for a host to compare against by mistake.
  `{CHECKOUT_SESSION_ID}` is substituted before a URL becomes a stop rule.
- **D3 — this package has no `confirm` method, by design.** vpay's own hosted
  _page_ submits the confirm, and no JavaScript bridge exists for the package
  to do otherwise.
- **D6 — the `client_secret` and the session URL stay out of every error,
  diagnostic and `toString`**, _including the generated channel types_
  ("generated code is how this regresses"). A hosted session's URL carries the
  session secret in its fragment, so it is redacted too, and the redaction is
  asserted to survive string interpolation — which is how it would actually be
  logged.
- **A bad or expired link is the uniform 404**, mapped to the same typed error
  the six causes `browser::authenticate` does not distinguish between all
  render. The Dart client must not try to tell them apart either.

## The two e2e suites (commit `107eff12`, 2026-09-15)

Both refuse **loudly, never skip**, when what they need is absent. Neither is
in `just ci` (D-M3), so every count this repository quotes for this package is
a human running the recipe by hand.

### `just test-flutter-e2e` — the Dart core against a real stack

`test_e2e/real_stack_e2e_test.dart`. **Kept out of `test/` on purpose**:
`flutter test` with no arguments only discovers `test/`, so the plain unit
suite (`just test-flutter`, 82 passed / 0 skipped) stays stack-independent and
must keep passing with the stack down.

The decisive check is first: the recipe curls `/healthz` on both vpay and
`examples/shop` and prints _"FAIL — nothing answers … this is a REAL end-to-end
test and refuses to fake one. bring a stack up first: just demo-up"_.

The fixture is two real Checkout Sessions minted through **`examples/shop`'s
own running server** — a real `POST /v1/payment_intents` + `POST
/v1/checkout/sessions` under a real `private_key_jwt` exchange, the same two
calls `examples/shop/src/server/orders.ts` makes for a paying customer — with
the second expired via `POST /v1/checkout/sessions/{id}/expire` under a
merchant token the recipe mints itself. It reads whichever private key the
running shop container already has (`docker cp`, never generated or committed)
and signs with `sdks/nodejs/scripts/mint-assertion.mjs`. **No merchant
credential this package would ever hold appears in the test** — the file only
ever receives a session `url`, exactly what a merchant's backend hands a
device. The recipe does not bring the stack up or tear it down.

**It found a real bug no `MockClient` fixture could catch**, which is the whole
argument for it existing: `GET /v1/browser/checkout/sessions/{id}` never echoes
the **session's** own `client_secret` back — only the intent's — yet
`CheckoutSession.fromJson` _required_ that field. Every real pre-flight against
a real server failed with `unexpected_response(200)`, and the suite had been
green only because every `test/` fixture fabricated the key. `clientSecret` is
now a **caller-supplied parameter** (the value the caller already authenticated
the read with), and every mock fixture was updated to drop the key it was
wrongly asserting, without weakening any assertion.

Second finding in the same commit: `package:http`'s Map-body helper
percent-encodes `payment_method_data[type]`'s brackets, which vpay's
Stripe-style form decoder (`backends/crates/vpay-api/src/form.rs`) does not
recognise — it splits on the raw, unescaped bracket by design. The e2e builds
that one request body by hand.

**One HTTP call in that file is not on `BrowserClient`**: `_rawConfirm`, which
stands in for what a payer's browser submits on vpay's page (D3, above).
Everything else goes through the package's own client.

### `just test-flutter-emulator` — the real page in a real emulator

Refuses the moment no Android device answers `adb`. **Does not boot an
emulator** — booting is slow, the host is shared, and a recipe that silently
starts one silently leaves one running. It selects by `VPAY_EMULATOR_SERIAL`,
or by finding **exactly one** attached device whose own AVD name is
`vpay_e2e_avd` (never some other device already on the host), and refuses on
zero or more than one.

It calls `adb reverse` for two ports rather than rewriting URLs to `10.0.2.2`,
and the reason is worth carrying: **the checkout page's own client-side JS
calls `NEXT_PUBLIC_VPAY_API_URL`, baked into the container as
`localhost:8080`**, so `adb reverse` is the one fix that makes that string
resolve correctly for both the recipe's HTTP calls and the WebView's.

Its fixtures are two **unconfirmed** hosted sessions minted through
`orders.create` — unlike `test-flutter-e2e`'s, because this suite drives the
real page's own confirm UI. Specs live in `example/integration_test/`.

## D8 — the external browser

`VpayCheckoutMode.externalBrowser` is the mitigation for the design's named
biggest risk: **Orange's page has never been driven inside a WebView** — vpay
has only ever talked to a WireMock stub serving two links.

As of 2026-09-14 it is **wired, not merely designed**:
`pigeons/checkout.dart`'s `ShowCheckoutRequest` carries a `mode` field
(`CheckoutWindowMode`), threaded end to end. It previously threw
`UnimplementedError`, and the ⛔ row saying so is struck through and dated.

- **Android** — `VpayCheckoutExternalBrowserSession` launches Custom Tabs
  (`androidx.browser`) and reports the payer's return over
  `Application.ActivityLifecycleCallbacks`. **No `startActivityForResult`, no
  scheme to call back on**, because the outcome is always polled (D1). It
  degrades correctly with **no merchant deployment work at all**. Proven by
  _running_, not just compiling: a dedicated headless AVD showed a real
  `CustomTabsIntent` open Chrome and a real hardware back press return control
  to the host `Activity`, reporting a real dismissal over the real channel.
- **iOS** wraps `SFSafariViewController`; **macOS** opens the default browser
  via `NSWorkspace`. **Both are compiled by nobody** — this repository runs on
  Linux and has no `xcodebuild` (`swiftc`/`swift` confirmed absent). Reviewed
  by reading, and that is all. A dated ⛔.
- **Web** — `inApp` and `externalBrowser` collapse to the identical
  `window.open` popup, since there is no in-app WebView on Flutter web. Proven
  by `flutter build web`. **No browser has driven it.**
- **NOT built: D8's "tier 1"** — Android App Links / iOS 17.4+ Associated
  Domains, so an external window can close itself instead of the payer
  switching back manually. It needs a merchant-hosted
  `assetlinks.json` / `apple-app-site-association` deployment this repository
  cannot provide. `ASWebAuthenticationSession` is **deliberately not** an iOS
  dependency, for the same reason `androidx.browser` was not one before D8: _a
  dependency with no code path that could ever run is its own kind of false
  claim._

Custom URL scheme returns are recorded as a **decision, not an oversight**:
`checked_forward_url` accepts only `http(s)` for `success_url`/`cancel_url`, so
a bounce page is the only route to `myapp://…`, and a custom scheme is
first-come-first-served on Android — any installed app may claim it and receive
the return.

**A safety note the README carries and you should not soften:**
`externalBrowser` does **not** move a digital-goods app out of App Store / Play
scope. It can itself be the violation.

## Recipes

| Recipe                       | What it does                                                                                                 |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `just install-flutter`       | `flutter pub get`                                                                                            |
| `just analyze-flutter`       | `dart analyze --fatal-infos` — the strictest setting, because this package has no `verify-*` gate of its own |
| `just test-flutter`          | Unit only. No emulator, no `adb` — the controller is pure by design                                          |
| `just test-flutter-e2e`      | Above                                                                                                        |
| `just test-flutter-emulator` | Above                                                                                                        |

All five go through `_flutter-preflight`, which refuses clearly when the
package directory or the Flutter SDK is missing and **warns** (does not fail)
when the version on PATH differs from `flutter-toolchain.toml`'s pin.

## What has never happened

No real rail. The stack `just test-flutter-e2e` drives is real; the rail behind
it is WireMock, as everywhere else in this repository. No device, no App Store
or Play review, and the Android 21 / iOS 12 floor is a claim nobody has tested.
