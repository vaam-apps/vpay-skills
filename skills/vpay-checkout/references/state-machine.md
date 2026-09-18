# The checkout state machine

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`frontends/apps/checkout/src/lib/machine.ts` — a **pure reducer**. No `fetch`,
no timer, no DOM. `src/lib/controller.ts` is the impure half: it turns network
answers into the events below and does nothing else.

Pure on purpose: everything that goes wrong on a payment page goes wrong in the
transitions — a status rendered before it was read, a "succeeded" screen shown
for an intent that only reached `processing`, a forward fired twice — and a
reducer with no I/O in it is the only version of those transitions a test can
enumerate.

## The one rule the shape enforces

> **No state carries a status this page invented.**

`outcome` is reachable only from an `intent_updated` event whose intent
`intentOutcome` judged terminal, and `intentOutcome` reads the same two fields
`@vaam-apps/vpay-stripe-js`'s poll ladder does. If you find yourself deriving a
status in a component, you are in the wrong layer.

## The twelve states

`CheckoutState`, discriminated on `name`:

| State            | Carries                             | Means                                                                                                           |
| ---------------- | ----------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `loading`        | —                                   | the initial state                                                                                               |
| `error`          | `error: CheckoutError`              | a closed-vocabulary refusal, e.g. a missing key                                                                 |
| `refused`        | `reason`, `context \| null`         | `embed_not_allowed` (D4) or `no_supported_rail` (D9)                                                            |
| `expired`        | `context`                           | the session's own `expired`, not a failure                                                                      |
| `select_rail`    | `context`, `rails`                  | two or more supported rails on offer                                                                            |
| `collect_msisdn` | + `rail`, `problem`                 | a push rail's number form                                                                                       |
| `ready_redirect` | + `rail`, `problem`                 | a redirect rail, before the payer presses go                                                                    |
| `confirming`     | `context`, `rail`                   | a confirm is in flight                                                                                          |
| `waiting`        | `context`, `rail \| null`, `notice` | "check your phone". `rail` is `null` after a reload — a confirmed intent does not name the rail it was taken by |
| `redirecting`    | + `rail`, `url`                     | recorded _before_ the navigation happens                                                                        |
| `outcome`        | `kind`, `failure`, `reason`         | terminal: `succeeded` / `failed` / `canceled`                                                                   |
| `forwarding`     | `kind`, `url`                       | on the way to the merchant's `success_url` / `cancel_url`                                                       |

`INITIAL_STATE` is `{ name: "loading" }`.

`waiting.notice` is a poll that could not be answered. **The payer stays on the
waiting screen** — the payment is in flight and saying otherwise would be a
claim this page cannot support — with the reason shown beside it.

## The eleven events

`loaded`, `load_failed`, `refuse`, `choose_rail`, `back`, `problem`,
`confirm_started`, `intent_updated`, `redirect_required`, `session_refreshed`,
`forward`.

## `contextOf` strips the session secret

`contextOf` converts the wire's one object (the session with `payment_intent`
expanded and a `merchant` beside it) into three named parts — and **drops
`session.client_secret` on the way in**. The controller already holds it as a
credential; a second copy on the object every screen renders from would put it
inside anything that ever serialises a state: a devtools snapshot, an error
report, a `postMessage` written in a hurry. `secrets.test.ts` and
`machine.test.ts` both pin its absence. Do not re-add it "for convenience".

`allowedMethods` (the deployment's `checkout.allowed_methods`) is carried **on
the context** rather than threaded as a third argument, precisely because the
reducer is pure and cannot read a config file or a React context. It can only
narrow the rail list, never widen it.

`merchantOf` takes `unknown` on purpose: `isSessionEnvelope` deliberately does
not require a `merchant` member, so the value has only been through
`JSON.parse`. Absent, `null`, a bare string, or an object whose `name` is a
number are all the same answer — this page has no name to show — and none of
them is a reason to refuse a payment. A blank or whitespace-only name is also
`null`, because rendering `Pay ` reads like finished copy with the name
missing.

## `intentOutcome` — and there is no `failed` status

| Intent status                                              | Verdict                                                                |
| ---------------------------------------------------------- | ---------------------------------------------------------------------- |
| `succeeded`                                                | terminal, `succeeded`                                                  |
| `canceled`                                                 | terminal, `canceled`                                                   |
| `requires_payment_method` **with** `last_payment_error`    | terminal, `failed`, carrying the rail's `code` and a cleaned `message` |
| `requires_payment_method` with **no** `last_payment_error` | **still in flight**                                                    |
| `processing`                                               | still in flight — the rail is moving and the status changes on its own |
| `requires_action`                                          | **not in flight, and not the rail's turn** — see below (2026-09-17)    |

**vpay has no `failed` PaymentIntent status.** A refused charge returns the
intent to `requires_payment_method` with `last_payment_error` set. That is why
the fourth row exists and why it must not be treated as an outcome: it is also
the status of an intent nobody has confirmed, so judging it terminal would
render "payment not completed" on a page the payer just opened.

`failure` (a closed `FailureCode`) and `reason` (the rail's own sentence,
cleaned by `providerReason` in `src/lib/failures.ts`) are **not**
interchangeable. One is translated; the other is shown as data and never
renders on its own.

## `stateForContext` — who wins on a fresh read

1. `session.status === "complete"` → `outcome` / `succeeded`. The session's own
   status wins where it is terminal, because the worker writes it **in the
   settlement transaction** and it is what the merchant's `success_url`
   correlates against.
2. `session.status === "expired"` with `payment_status === "failed"` → the
   intent's outcome; otherwise → `expired`.
3. Session still `open` → the intent decides. That is what makes a reload
   during a push land back on "check your phone" rather than on an empty form.
4. `processing` → `waiting`, `rail: null`.
5. `requires_action` → `resume_redirect`, carrying the redirect URL, since
   2026-09-17 ([vpay#199](https://github.com/vaam-apps/vpay/pull/199) on the
   web, [vpay#208](https://github.com/vaam-apps/vpay/pull/208) for the Flutter
   sheet). **This page said `requires_action` → `waiting` until then, and that
   was the bug those PRs fixed**, so an agent acting on the old sentence would
   reintroduce it.

   `processing` means the rail is still working and the status will change on
   its own, so a spinner is honest. `requires_action` means the **payer** has a
   redirect to finish on the rail's own page — and if they abandoned it,
   nothing changes until they go back. Showing "check your phone" there is
   false twice over: a redirect rail never sees the payer's number, and there
   is nothing in flight to validate. The page then polled a status only the
   payer could move until the budget expired.

   `resume_redirect` offers the one thing that can move it: a primary action
   back to the rail's page, plus "choose another method" when more than one
   rail is on offer. The URL is `next_action.redirect_to_url`, which
   `payment_intents.rs`'s `rendered_intent` rebuilds from the stored charge row
   on **every** read of a `requires_action` intent and hard-errors when it
   cannot — so a polled intent carries it exactly as a freshly-confirmed one
   does. If it is somehow absent, both surfaces fall back to `waiting`.

   Both halves matter on the native side: splitting the reducer alone was not
   enough, because `_afterIntentUpdate` (how a _poll_ becomes a screen) and the
   browser-return path each built a `waiting` directly. vpay#208's own commit
   message records that the unit tests passed while a device still showed the
   old screen.

6. Zero supported rails → `refused` / `no_supported_rail`; exactly one →
   straight to that rail's screen; more → `select_rail`.

## Ordering the controller owns

From `controller.ts`'s own header — these are the three things that go wrong:

- **`confirm` is called once per attempt**, and only from a state that has a
  rail. A second press while `confirming` is dropped by the reducer.
- **The redirect is performed after the machine has recorded it**, so a payer
  who comes back to the tab mid-navigation sees "taking you to Orange Money"
  rather than the form again.
- **`vpay:complete` is posted after a re-read of the session**, so the status
  the parent receives is the session's own and not this page's guess from the
  intent.

## Page memory — and the PIN vault that is deliberately absent

`src/lib/memory.ts` / `memory-idb.ts`. Two values on the payer's own device:
the canonical MSISDN they last paid with, and the rail they last chose. One
IndexedDB database, one object store, one key. It never leaves the browser.

- Opt-in, written at **exactly one moment** — a payer ticks the box and submits
  an entry screen. Visiting, choosing a rail or reading an outcome stores
  nothing.
- The remembered rail is **marked, never chosen** — a "Last used" badge.
- Ninety days, enforced on read.
- A stored number is **re-validated with `normalizeCameroonMsisdn`** before it
  reaches the form.
- `page_memory: false` removes the offer entirely — no checkbox, no read, no
  write.
- The cost is stated on the checkbox, in the payer's language, not in a
  tooltip.

**Everything above describes the WEB page.** The Flutter sheet ports this to
`shared_preferences` and keeps the 90-day read-side TTL and the
write-on-deliberate-submit rule, but it is **not** the same behaviour in three
named ways as of 2026-09-18 — no "Last used" badge, no re-validation on read,
and a redirect rail's box that persists nothing. Do not read this section as a
description of the sheet: `vpay-sdks`' `references/flutter-plugin.md`
§ "Remember this number on this device" is the one that is true of it.

**There is no PIN vault, and that is a refusal rather than an omission.** The
requirement asked for one; this page has no PIN field and neither does anything
behind it. `POST /v1/browser/payment_intents/{id}/confirm` accepts
`payment_method_data[mtn_momo][msisdn]` and nothing else, and no
`ProviderAdapter` takes a PIN. Building it would have meant adding a PIN input
that goes nowhere — on a payment page, a field indistinguishable from phishing.
If a rail is ever integrated that takes a PIN through the API, the store, the
opt-in, the horizon and the clear control are all already here and a `pin`
member is the change.

## Runtime configuration

`src/config/settings.ts` holds pure parsers for the two YAML files an operator
mounts; `src/config/runtime.ts` reads them once at startup and memoises.
Nothing in `settings.ts` touches the filesystem or the environment, so every
way a file can be wrong is a unit test.

**A bad file never blanks the page.** Every parser answers a _complete_
settings object plus a list of problems: an unreadable file, a malformed
document, a key of the wrong type and a value that fails its rule each cost
exactly that key and leave the rest standing. The problems go to the container
log, not to the payer and not onto `/config/v1` — a payer cannot fix a mounted
file, and the paths are the operator's business.

This is the **opposite** of the dashboard, which refuses to render a login form
when its four environment variables are unset. Deliberately: a payment page
with no branding still takes a payment; a dashboard with no client registration
is a login form that cannot log anybody in.

`src/config/theme.ts` turns an operator's `#rrggbb` into a daisyUI override
scoped to `:root[data-theme="dark"]`. `THEME` is `"dark"` because
`globals.css` sets `themes: false`, which stops daisyUI's `bumblebee`
compiling at all — a `[data-theme="bumblebee"]` selector would match nothing.
The foreground is `color-mix(in oklch, <hex> 20%, white|black)`, daisyUI's own
rule, and `theme.test.ts` asserts every case at WCAG AA (4.5:1) by
re-implementing the maths independently rather than importing the function.
