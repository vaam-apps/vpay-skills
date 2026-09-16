---
name: vpay-checkout
description: The payer-facing checkout page at frontends/apps/checkout — the three surfaces (hosted, embedded, popup) and how a payer reaches each, why the client_secret rides in the URL fragment and a query-string one is ignored rather than used, the entry decision that refuses before it reads a credential, the postMessage protocols and the popup's deliberate departures from the iframe one, the twelve-state pure reducer, and the a11y and styling gates that have already gone green while measuring nothing. Load before changing anything a payer sees or any credential handling on that page.
---

# vpay checkout page

> **Verified against vpay `f063ee96` (2026-09-15).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See VERSIONING.md.

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

| Route                                 | Mode                                                                                                                | CSP                                                   |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `/c/{cs_id}?key={pk}#{client_secret}` | hosted — a full navigation, or a popup the merchant opened                                                          | `frame-ancestors 'none'`                              |
| `/e/{cs_id}?key={pk}#{client_secret}` | embedded — framed by `initEmbeddedCheckout`                                                                         | `frame-ancestors <the merchant's registered origins>` |
| `/c/{cs_id}/return?t={return_token}`  | where a redirect rail sends the payer back                                                                          | `frame-ancestors 'none'`                              |
| `/config/v1`                          | an **operator** verification surface. Not an API a browser uses — every page already receives its branding as props | —                                                     |

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
anywhere** — it is whatever was in the address bar. `sessionStorage` holds only
the publishable key (`vpay.checkout.key.{id}`), never a secret, because
`sessionStorage` outlives the payment and the fragment does not.

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
no channel**.

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

## More

- `references/state-machine.md` — the twelve states, eleven events, what the
  reducer refuses, and why there is no `failed` intent status.
- `references/testing-and-gates.md` — `a11y-gate.test.ts`, the theme-ordering
  defect it guards, the Storybook gotchas, and the Cypress suite.
