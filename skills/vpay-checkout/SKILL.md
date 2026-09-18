---
name: vpay-checkout
description: The payer-facing checkout page at frontends/apps/checkout — the three surfaces (hosted, embedded, popup) and how a payer reaches each, why the client_secret rides in the URL fragment and a query-string one is ignored rather than used, the entry decision that refuses before it reads a credential, the postMessage protocols and the popup's deliberate departures from the iframe one, the twelve-state pure reducer, the redirect leg that is a vpay page rather than the rail's own URL and the return page it suppresses, and the a11y and styling gates that have already gone green while measuring nothing. Load before changing anything a payer sees or any credential handling on that page.
---

# vpay checkout page

> **Verified against vpay `d3a8810b` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`frontends/apps/checkout` (`@vpay/checkout`). Next 15.5.25 App Router, React 19. Process doc: `docs/flows/hosted-checkout.md` plus the four pages under
`docs/flows/hosted-checkout/`. The surface underneath it is
`docs/flows/browser-checkout.md`.

Load `vpay-frontend` first for the workspace, the Tailwind/daisyUI setup and
`verify-ui`. The Flutter payer surface is a different package — load
`vpay-sdks`.

**The invariant the whole page is built around:** a payer's browser holds
credentials for **one** checkout and nothing else, and every one of them
expires. No bearer token, no cookie, no server-side session beyond the row.

## Routes

| Route                                 | Mode                                                                                                                                                                                                                                                                                                                                   | CSP                                                   |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `/c/{cs_id}?key={pk}#{client_secret}` | hosted — a full navigation, or a popup the merchant opened                                                                                                                                                                                                                                                                             | `frame-ancestors 'none'`                              |
| `/e/{cs_id}?key={pk}#{client_secret}` | embedded — framed by `initEmbeddedCheckout`                                                                                                                                                                                                                                                                                            | `frame-ancestors <the merchant's registered origins>` |
| `/c/{cs_id}/return?t={return_token}`  | where a redirect rail sends the payer back                                                                                                                                                                                                                                                                                             | `frame-ancestors 'none'`                              |
| `/c/{cs_id}/redirect?key={pk}#{client_secret}` | **since 2026-09-18 (#195/#200)** — the Flutter sheet's redirect leg. `app/c/[id]/redirect`. Renders nothing a payer acts on; it navigates | `frame-ancestors 'none'`                              |
| `/config/v1`                          | an **operator** verification surface. Not an API a browser uses — every page already receives its branding as props                                                                                                                                                                                                                    | —                                                     |
| `/.well-known/vpay-checkout`          | mounted 2026-09-17 ([vpay#196](https://github.com/vaam-apps/vpay/pull/196)). The **client** contract: what an SDK needs to know about this deployment before a payer taps anything — branding, the operator's `allowed_methods` floor, `support_contact`. Same document as `/config/v1`, different audience, so its shape is a promise | —                                                     |

Both modes render the same screens from the same state machine. The mode is a
property of the session and the page refuses the wrong one: `/c/{id}` refuses
if it **is** framed, `/e/{id}` refuses if it is **not**.

## Credentials in the URL — the rule that is easy to break

> **The secret rides in the fragment. The publishable key rides in the query.**

A fragment is not sent to a server, not written to an access log, and does not
reach a `Referer`. `src/lib/link.ts` enforces two consequences:

1. **A `client_secret` in the query string is ignored, not used.** Reading one
   would make the safe shape and the unsafe shape work equally well — and the
   unsafe one is what a copy-pasted URL turns into.
2. The publishable key _is_ read from the query, because it is not a secret: it
   is rendered into the merchant's public page by construction.

A fragment value without `_secret_` in it is refused and **never echoed
anywhere** — it is whatever was in the address bar.

~~`sessionStorage` holds only the publishable key
(`vpay.checkout.key.{id}`).~~ **Corrected 2026-09-18 (#195/#200):** it holds
two things now — that key, and the redirect-leg marker
`vpay.checkout.redirect_leg.{id}` (`src/lib/redirect-leg.ts`). **Neither is a
secret, and that is the rule that did not change**: `sessionStorage` outlives
the payment and the fragment does not, so nothing that would still be a
credential tomorrow may go in it. Adding a third key means asking that
question again, not copying the two that are there.

## `decideEntry` refuses before it reads the credential

`src/lib/entry.ts`. Pure function, so every refusal is a test rather than a
branch inside a `useEffect` nobody can reach twice. **The order is the point:**
the embedding check runs first, before the credential is even looked at, so a
page framed by an origin the merchant never registered refuses **without
touching the API** — and therefore without a hostile framer learning whether
the session id in the URL exists.

Do not reorder it. Do not add an early `fetch`.

## Popup is not iframe

A merchant may `window.open(session.url)` instead of framing. That broke the
protocol silently once: **inside a popup `window.parent === window`**, so
`createFrameChannel` returned `null` and vpay said nothing at all to the
merchant's page when the payment finished. The channel now takes a `peer`,
`parent` or `opener`.

|                      | Framed `/e/{id}` | Popup `/c/{id}`               | Top-level `/c/{id}` |
| -------------------- | ---------------- | ----------------------------- | ------------------- |
| Peer                 | `window.parent`  | `window.opener`               | none                |
| An unresolvable peer | **refused**      | renders, no channel           | normal              |
| `vpay:resize`        | yes              | **no** — a popup sizes itself | n/a                 |
| `vpay:redirect`      | yes              | **no**                        | n/a                 |
| `vpay:complete`      | yes              | yes                           | n/a                 |

Three deliberate departures, each a decision and not an omission:

- **No `vpay:redirect` to an opener.** A popup _is_ a top-level browsing
  context and may navigate itself. Asking the opener to navigate would send
  **the merchant's own page** to Orange Money out from under the payer, losing
  the page they expect to come back to.
- **`vpay:complete` is posted at most once per page.** Two moments can reach it
  — the outcome first appearing, and the payer pressing the button. A second
  copy fires a merchant's `onComplete` twice, and a merchant who treats that as
  a cue to create an order creates two.
- **An unresolvable opener is not a refusal.** A hosted page is complete on its
  own; it loses only the ability to _tell_ the opener. The embedded case still
  refuses, because a framed page with no parent has no way to finish at all.

`'*'` appears **nowhere** as a `postMessage` target, and the parent posts
nothing into the frame at all — the child learns its framer's origin from the
CSP vpay served it, not from a message. The strongest form of "never
`postMessage(…, '*')`" is "never `postMessage`".

The **return** page resolves its opener by a different rule and has to:
a payer arriving there came from the _rail_, so `document.referrer` names
Orange, not the merchant. It uses `soleOrigin` — exactly one registered
`checkout_origins` entry is the target; with none or with several, **there is
no channel**. And ~~the return page always reports the outcome~~ — **since
2026-09-18 (#195/#200) it is conditional**; see the next section.

## The redirect leg is a controlled surface, not the rail's URL

**Since 2026-09-18 (issue #195, PR #200).** The Flutter native sheet no longer
hands a redirect rail's own URL to the payer's browser. It opens
`/c/{cs_id}/redirect` on the **checkout** origin, and `redirect-client.tsx`
reads the session for the **intent's** `client_secret` (`src/lib/api.ts`), then
reads `GET /v1/browser/payment_intents/{id}` for `next_action.redirect_to_url`
through `@vaam-apps/vpay-stripe-js`'s `retrievePaymentIntent` — the division of
clients the section below describes, not an exception to it — and navigates.
**The rail's URL is never a parameter of that page**, which is the whole reason
a crafted link to a payment origin cannot be turned into an open redirect; and
`redirectUrlOf` (`src/lib/controller.ts`) still gates the one navigation it
does make on `http:` or `https:`.

> **Two reads, not one, and the trap is that one read type-checks.**
> `GET /v1/browser/checkout/sessions/{id}` expands the intent and **never
> renders a `next_action`**: `PaymentIntentObject::try_from(&row)` sets it
> `None` unconditionally and only `with_next_action` — which the browser
> session routes never call — attaches one. The page as first written read
> `next_action` off the session and redirected **nobody**, on every real
> deployment, while a hand-written `fetch` stub carrying that shape kept the
> suite green. `src/testing/browser-stub.ts` now answers `next_action: null`
> on both session routes so no later test can certify that shape again.

Before navigating, the page writes the `sessionStorage` marker
`vpay.checkout.redirect_leg.{id}` = `"1"` (`rememberRedirectLeg`,
`src/lib/redirect-leg.ts`; the prefix constant is
`REDIRECT_LEG_STORAGE_PREFIX`). On the way back, `return-client.tsx` calls
`recallRedirectLeg` **before `decideReturnEntry`** — so a malformed return URL
cannot re-surface a foreign error screen on top of the sheet's outcome — then
renders the `redirect_leg` screen ("returning to the app") and **returns
without ever constructing a `ReturnController`**. No poll, no outcome, no
channel. A query parameter could not do this job: the rail controls the
redirect back and will not echo a vpay-added parameter.

**Four limits vpay discloses rather than hides, all as of 2026-09-18:**

- **The marker is consumed, not kept.** `recallRedirectLeg` is followed
  immediately by `clearRedirectLeg`, so a later normal web checkout in the same
  tab is not suppressed — and so a **manual reload of the return page shows the
  full outcome**. The suppression is once per trip, by design, and there is no
  TTL doing that work.
- **It is unmeasured in an in-app browser.** Whether
  `SFSafariViewController` and Chrome Custom Tabs carry `sessionStorage` across
  the rail's cross-origin redirect is reasoned about in `redirect-leg.ts` and
  asserted in jsdom, **never measured**. It fails to the duplicate screen,
  never to a wrong outcome: the rail's `return_url` carries both `t` and `key`,
  so a return page that finds no marker still has every credential it needs.
- **A suppressed return page renders no "return to the merchant" button**, so
  the browser leg reaches no `stopUrl` by itself; the sheet resolves the rail
  on the **dismissal** signal alone, which D4's poll makes correct regardless.
- **Nobody has driven the leg end to end** — not on a device, not against
  `just demo-up`. Every number behind it is unit- or component-level.

## The browser surface

`src/lib/api.ts` speaks exactly three routes: `GET
/v1/browser/checkout/sessions/{id}`, the same with `/return?t=…`, and `GET
/v1/browser/checkout/origins?key=…`. It is **deliberately not**
`@vpay/api-client` — that is the dashboard's `/dash/v1` client under an OIDC
session (ADR-0008), and this app never holds a merchant credential of any kind.
The confirm and poll routes belong to `@vaam-apps/vpay-stripe-js`;
re-implementing them here would be a second client for one wire contract.

**Nothing in that module throws or rejects.** Every call answers a `Result`
with a closed `CheckoutErrorCode`, because a payment page that renders a thrown
value renders whatever the network stack put in a message — which on this
surface is a URL carrying a credential.

## `middleware.ts` fails closed, four ways

No `key` in the URL, no `VPAY_API_URL` configured, a lookup that failed, and a
lookup that returned an empty list all produce `frame-ancestors 'none'`. There
is no branch in which an unknown answer widens the policy. The lookup carries a
2 s timeout (`ORIGINS_TIMEOUT_MS`) whose abort lands in the same empty list.

It resolves the origin list for `/e/{id}`, `/c/{id}` **and** `/c/{id}/return` —
the latter two for the popup channel, not for framing — and the two uses of the
list are **separate expressions** so widening one cannot widen the other.
`middleware.test.ts` has a case named for that: the hosted page never lets that
list reach its CSP, measured failing with the guard removed.

`/c/{id}/redirect` is the fourth page on this origin and **deliberately gets no
lookup** (2026-09-18, #195) — it never frames and never `postMessage`s, so the
list would answer a question nothing asks. It still gets `no-referrer` (which
is what keeps the session secret in its fragment out of the rail's `Referer`),
`no-store`, `nosniff` and `frame-ancestors 'none'`, because the **matcher is
every path** and those are the constant half. The omission is written down
beside `EMBEDDED_PATH`/`HOSTED_PATH`/`RETURN_PATH` in `middleware.ts` on
purpose: absent from that list, it read as forgotten rather than decided.

## More

- `references/state-machine.md` — the twelve states, eleven events, what the
  reducer refuses, and why there is no `failed` intent status.
- `references/testing-and-gates.md` — `a11y-gate.test.ts`, the theme-ordering
  defect it guards, the Storybook gotchas, and the Cypress suite.
