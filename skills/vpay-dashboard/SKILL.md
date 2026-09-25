---
name: vpay-dashboard
description: The staff operator console at frontends/apps/dashboard — it is the OAuth client and runs the code leg in its own process, the /dash/v1 read seam over two transports, the BFF that only the payments screens call, and only after their first render, navigation enforced as a gate rather than a comment, and the honest-absence rules that decide what a screen may and may not show. Load before adding a page, a column, a nav entry, a CrateStack procedure or any read, and before assuming a row you see is backed by a column something writes.
---

# vpay dashboard

> **Verified against vpay `f68fda09` (2026-09-25).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`frontends/apps/dashboard` (`@vpay/dashboard`). Next 15.5.25 App Router, React
19, Refine 5. Its own `README.md` is the best in-repo companion; the flow docs
are `docs/flows/dashboard.md` plus the four pages under `docs/flows/dashboard/`
and `docs/flows/dashboard-auth.md`. Decisions: ADR-0008, ADR-0017.

Load `vpay-frontend` first for the workspace, the styling stack and
`verify-ui`.

## What it is, in one paragraph

**The OAuth client** (ADR-0017 decision 4). It holds the staff session in an
httpOnly, Secure, SameSite=Lax cookie **on its own origin**, presents it to
vpay as `X-Vpay-Staff-Session` server-side, and runs the authorization-code leg
— request, follow the `302`, exchange — inside its own process. A browser never
sees a code, a verifier or a token. The `/dash/v1` access token is read back
out of the `staff_sessions` row on every render rather than kept here, which is
what makes signing out a revocation.

## Three things that look like bugs and are not

**There is no route at `redirect_uri`, and nothing is missing.**
`dashboard_client.redirect_uris` names `http://localhost:3000/dash/v1/callback`
in `config/application-sandbox.yml` (the base `config/application.yml` spells
`http://localhost:8080/dash/v1/callback`; checked 2026-09-23), and **no
browser ever visits it**. This app's own server follows the `302`, so
that string is an identifier the two OAuth legs must spell identically —
matched byte for byte by `ClientRegistration::allows_redirect_uri`, no prefix,
no wildcard — and not a page. A route there would be a page nobody can reach.

**Two screens call the BFF, and only after their first render (since
2026-09-12, vaam-apps/vpay#138).** The payments list and the payment detail are
Refine screens. Their first render is read on the server and handed to Refine
as `initialData`. Their _later_ reads, paging and refetches, go through
`dashDataProvider("/api/dash")` in `app/(dash)/layout.tsx`, to
`/api/dash/payment_intents` and `/api/dash/payment_intents/{id}`. There are six
handlers under `app/api/dash/`; the four later ones (`refunds`, `deliveries`
and `customers` since 2026-09-13, `checkouts` since 2026-09-14) have no
caller, because those four screens render on the server and use no client
data layer. _(This dated all four 2026-09-14 until 2026-09-25, as vpay's
README did until vaam-apps/vpay#255.)_ So **deleting the
route files and `src/server/bff.ts` now breaks the payments screens' paging.**
Whether this app should have an authenticated browser-reachable surface at all
still **reverses a stated property of its security model** and is RD5 in the
Refine plan — a maintainer's call, not this code's. _(This said "Nothing in
this app calls the BFF … Deleting the route files … breaks nothing else" until
2026-09-23, and had been wrong since 2026-09-12. vpay's own README said the
same until vaam-apps/vpay#247.)_

**The `$procs` transport mounts five procedures, and three places in the Rust
say it mounts one.** Code wins — five are mounted, and only
`searchPaymentIntents` is probed by a test at the transport layer. A sixth
procedure is routed automatically and tested by nobody. Detail and the exact
stale sentences: `references/read-seam-and-bff.md`.

## The layout

| Directory               | What lives there                                                                    |
| ----------------------- | ----------------------------------------------------------------------------------- |
| `app/`                  | Routes only. Composition, a redirect, a fetch — no logic worth testing alone        |
| `app/api/dash/`         | The BFF's route handlers. Four lines each; `src/server/bff.ts` is the substance     |
| `middleware.ts`         | Next reads a middleware **only** from the project root                              |
| `src/components/`       | Every rendered component. Pure props in, markup out — no `fetch`, no `next/headers` |
| `src/server/`           | Everything touching vpay, cookies or PKCE. Imported only by `app/` and itself       |
| `src/dash/`             | The `/dash/v1` read seam — `getList`/`getOne` over `readDash`. No framework         |
| `src/config/`           | `settings.ts` decides what a configuration means; `runtime.ts` reads the env once   |
| `src/format.ts`         | Money, instants, the em dash                                                        |
| `src/payments-query.ts` | The URL's filter vocabulary ↔ the API's, and the two paging links                   |
| `src/testing/`          | Fixtures. Imported by tests and by nothing under `app/`                             |

The split that matters is `src/server/` vs `src/components/`: a component that
fetched would be a component no test could render, and a `fetch` inside a
component is how a page ends up unable to say _why_ it is empty.

`src/server/actions.ts` is a `'use server'` file and may export **only async
functions** — a constant exported from there is a build error, not a lint
warning. That is why `FormState`/`NO_ERROR` live in `src/form-state.ts` and the
route paths in `src/server/session.ts`.

Every page is `export const dynamic = "force-dynamic"`, and has to be: they all
read cookies.

## Pages

| Route                                                 | What it does                                                                                                                          |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `/`                                                   | Redirects to `/payments` or `/login`                                                                                                  |
| `/login`, `/login/totp`, `/login/password`            | Password → TOTP (a first sign-in also renders the enrolment QR and secret) → the forced password change                               |
| `/payments`, `/payments/{id}`                         | REST-backed. Status and date filters, cursor paging; the detail page adds charge, refunds, last error, timeline                       |
| `/refunds`, `/deliveries`, `/customers`, `/checkouts` | Procedure-backed lists. Offset paged, no filters, no detail route yet                                                                 |
| `/signed-out`, `/healthz`                             | Route handlers, not pages: clear a dead session cookie (since 2026-09-07); "Next is answering" (since 2026-09-16, ADR-0022, vpay#183) |

All four procedure-backed lists are Server Components reading through
`readProcedurePage`, and all four share `src/components/procedure-table.tsx`.

`/healthz` takes no dependency and probes nothing. Probing `/dash/v1` from it
would take the dashboard pod's readiness down with every `vpay-server`
rollout. Do not "improve" it into a deep health check.

**A refund's events do not reach the payment detail's timeline (read from
the code on 2026-09-23; no test covers it).** The refund writers store the
refund's own `re_…` id as `object_id` (`vpay_api::v1::refunds::refund_event`).
The detail read asks `Events::list_for_objects` only for the intent's id and
the charge's (`vpay_api::dash::payment_intents`). So a refunded payment's
timeline shows no `charge.refunded`, even though its refunds section lists the
refund. The integration test that seems to prove otherwise seeds its
`charge.refunded` row by hand with the charge's id, and no code path writes
that shape (`docs/flows/dashboard/slice-1-gaps.md`, 2026-09-23).

**`GET /dash/v1/payment_intents` refuses `customer` with a `400` naming it**,
since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251;
`refuse_customer` in `vpay_api::dash::payment_intents`). `/v1` gained that
filter in the same change; before it, the dashboard's `ListParams` dropped the
unknown key and **answered every customer's intents** — which an operator, or
the BFF copying a `/v1` query, would read as the filtered answer. It is a
refusal and not a filter because the console has no customer view to filter
from, and a parameter no screen sends is untested surface. **Only `customer`
is refused**: `ListParams` still has no `deny_unknown_fields`, so any other
key `/dash/v1` does not know is still silently ignored. If you give this list
a customer filter, it is a new capability with its own test, not the removal
of the refusal.

**No screen shows or records an out-of-band invoice payment.** Step A built
one (`POST /v1/invoices/{id}/pay` with `paid_out_of_band=true`); only the
merchant `/v1` surface writes it, because `/dash/v1` refuses every non-`GET`
and ADR-0008's dashboard writes and their audit log do not exist.

**`/refunds` can show rows from 2026-09-16** and never could before: nothing
wrote a `refunds` row until `vpay_db::Refunds::create` landed (RFC-0003,
vpay#178). If you are reasoning about that screen, it is no longer safe to
assume the empty state is the only state — but every row it shows will read
`pending`, because **nothing settles a refund** (no poll ladder, RFC-0003 open
question 8) and **no rail has ever returned money to anyone.** A row on this
screen is an instruction recorded, never a payout. The dashboard still cannot
_create_ one: `/dash/v1` refuses every non-`GET` at the boundary.

`/login/password` is a step, not a nag: `vpay-server staff add` writes the
new staff member's password credential with `must_change: true` (a
`staff_members.password_change_required` column until migration `0044`,
vpay#168, 2026-09-13), and ADR-0017 decision 1 refuses **every**
authenticated route to a session whose password still carries it,
`/oauth/authorize` included (`vpay_api::staff::password_change_required` is
the check). No `/dash/v1` token can exist until it is done. ~~There is
deliberately no "current password" field — the session has already presented
both factors.~~ **Corrected 2026-09-25, wrong since 2026-09-10 (vpay#97,
issue #79 item 3):** the form asks for the **current password**, and the
server requires `current_password`. The two factors were presented once, up to
twelve hours earlier, so without it the session cookie alone could change the
password. On a first sign-in the current password is the one-time password
`staff add` printed, and the field's hint says so.

## Configuration fails closed

Four environment variables — `VPAY_DASH_API`, `VPAY_DASHBOARD_CLIENT_ID`,
`VPAY_DASHBOARD_REDIRECT_URI`, `VPAY_DASHBOARD_SCOPE` — read **once at start
through bracket notation**, because Next inlines a statically-written
`process.env.FOO` at build time and a value baked into the image is not a value
an operator can change.

A missing one is **not defaulted**. `/login` renders the variable names and
offers no form. That is the opposite of the checkout's runtime config, and
deliberately: a payment page with no branding still takes a payment; a
dashboard with no client registration is a login form that cannot log anybody
in, and rendering one invites somebody to retype a password.

A fifth variable, `VPAY_DASHBOARD_PUBLIC_ORIGIN` (since 2026-09-10), is
**optional**: the origin a browser reaches the dashboard on, read by
`csrf.ts`. It is not defaulted either. Unset or unusable, the origin check
falls back to comparing with `Host`, never to allowing everything.

The **merchant** is not configured here at all — it comes from
`GET /dash/v1/staff/session` and is rendered beside the staff address on every
signed-in page, so an operator looking at an empty list can tell "this merchant
has no payments" from "I am looking at the wrong merchant". From 640px it is
on screen. Below 640px it has been one tap away since 2026-09-14: in the Menu
drawer until vaam-apps/vpay#258, and in `SideNav`'s More sheet since that
merge (2026-09-25).

## Navigation is a gate, not a comment

> "The navigation is only ever allowed to link to slices that exist. A menu
> entry for a page nobody wrote is the same lie as an empty table."

`NAV_LINKS` and `src/nav.tsx` are **deleted**. `DASH_RESOURCES` in
`src/dash/resources.ts` is the **one** declaration: Refine routes from it and
the rail renders from `NAV_ENTRIES`, derived from the same array.
`src/layout.test.tsx` checks it three ways — every rail route resolves to a
`page.tsx` on disk, every resource Refine routes on does too, and the rail's
hrefs are exactly the registry's list routes. Adding a `resources` entry for
`/webhooks` fails two of the three.

`pageExists` resolves **route groups**: `app/(dash)/payments` serves
`/payments`, so the helper tries the direct path and then each top-level
`(group)`. Without that it reported every signed-in route as dangling the
moment the routes moved into `(dash)`.

No entry declares a `create` or `edit` route and `meta.canDelete` is `false`,
so Refine renders no create button and builds no edit route. The refusal is
expressed in the routing rather than discovered at submit time.

~~There is no "Sign out" in the nav, and that is the same rule — this layout
renders on `/login` too.~~ Identity and the way out are in `SignedInBar`,
which only the pages behind the gate render. **Corrected 2026-09-25, wrong
since 2026-09-14:** there is a Sign out in the nav. The rule is about the
**root layout**, which `/login` shares: it renders no nav and no signed-in
control at all (`layout.test.tsx`: "renders no signed-in control — this layout
is on /login too"). The rail is `AppShell`'s, which only the pages behind the
gate render, and its `SideNav` carries the account block (`SignedInBar`, Sign
out included, and `ThemeSwitcher`) as `accountSlot`. That was in the ≥1280px
sidebar only from 2026-09-14, and has been at every width since
vaam-apps/vpay#258 (2026-09-25), behind the toolbar's More control below
1280px. Sign out stays a form that POSTs to the `signOut` Server Action.
`references/what-a-screen-may-show.md` says where the block's copies are and
how a test finds the visible one.

## More

- `references/read-seam-and-bff.md` — the two transports and the five mounted
  procedures, `PROCEDURE_OF`, the token lifecycle, the BFF's properties and its
  method policy. **Read this before adding a procedure or a read.**
- `references/what-a-screen-may-show.md` — the honest-absence rules, status
  colour, which `@vaam-apps/ui` components the app imports, the 0.4.0 chrome
  (the More sheet and `accountSlot`, the loading skeletons, the date filter),
  the a11y gate, and what this app still cannot do. **Read this before
  touching the shell, a loading state or a filter.**
