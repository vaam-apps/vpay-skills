---
name: vpay-dashboard
description: The staff operator console at frontends/apps/dashboard — it is the OAuth client and runs the code leg in its own process, the /dash/v1 read seam over two transports, the BFF the payments screens are wired to, the shell mounted once in a layout so navigation does not reload the document, navigation enforced as a gate rather than a comment, the Overview's exact counts, and the honest-absence rules that decide what a screen may and may not show. Load before adding a page, a column, a nav entry, a filter, a CrateStack procedure or any read, before touching the shell, Refine's providers or a loading state, and before assuming a row you see is backed by a column something writes.
---

# vpay dashboard

> **Verified against vpay `73c3a7cf` (2026-10-21).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`frontends/apps/dashboard` (`@vpay/dashboard`). Next ~~15.5.25~~ **`^15.5.27`** (since 2026-10-09, vaam-apps/vpay#272) App Router, React
19, Refine 5, `@vaam-apps/ui` `^0.5.0` (Material 3 Expressive) since
2026-10-21 (vaam-apps/vpay#318, merged as `73c3a7cf`; it was `^0.4.0`). Its
own `README.md` is the best in-repo companion, but ~~seven~~ ~~five~~ four of
its passages are stale at `e7406dda` (listed under "Known-stale prose" in
`references/read-seam-and-bff.md`; #318 fixed the pages table and the
Sign-out sentence, and vaam-apps/vpay#333, 2026-10-22, the "no current
password field" one). The flow docs are
`docs/flows/dashboard.md` plus the four pages under `docs/flows/dashboard/`
and `docs/flows/dashboard-auth.md` (plus `dashboard-auth/sso.md` on a vpay that has merged ADR-0030). Decisions: ADR-0008, ADR-0017, and ADR-0030 for single sign-on.

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

**Two screens are wired to the BFF, and since 2026-10-21 a server-seeded
read no longer reaches it.** The payments list and the payment detail are
Refine screens (since 2026-09-12, vaam-apps/vpay#138). Every render, the first
and every page after it, is read on the server and handed to Refine as
`initialData`. ~~Their _later_ reads, paging and refetches, go through
`dashDataProvider("/api/dash")` in `app/(dash)/layout.tsx`.~~ **Corrected
2026-10-21 (vaam-apps/vpay#318):** with TanStack's default `staleTime: 0`, a
next page was read twice, once by the Server Component and again by the
browser through `/api/dash`. `DashProviders` (`src/dash/providers.tsx`, where
`<Refine>` moved out of the layout) now sets `staleTime: Infinity` and
`gcTime: 0`, and #318's measured click-through made **zero** BFF fetches over
twelve steps. The provider is still `dashDataProvider("/api/dash")`, routed to
`/api/dash/payment_intents` and `/api/dash/payment_intents/{id}`, so a read
Refine makes without a server-seeded answer would still go there. #318
measured one click-through against a fake of `/dash/v1`, not every path. There are six
handlers under `app/api/dash/`; the four added 2026-09-14 (`refunds`,
`deliveries`, `customers`, `checkouts`) have no caller, because those four
screens render on the server and use no client data layer. _(vpay's README
has dated the first three 2026-09-13 since vaam-apps/vpay#255, after
`f9c7d2cd`, the branch commit that wrote them. None of the four was on
`master` before 2026-09-14, and this page dates a claim by `master`.)_ ~~So
**deleting the route files and `src/server/bff.ts` now breaks the payments
screens' paging.**~~ **Corrected 2026-10-21:** after #318 the paging is read
by the server, so that sentence no longer follows. But nobody has shown that
nothing reaches the BFF (a refetch, a key without `initialData`), and the
provider still points at it, so **do not delete the route files or
`src/server/bff.ts` on the strength of #318's measurement.**
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

| Directory                  | What lives there                                                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `app/`                     | Routes only. Composition, a redirect, a fetch — no logic worth testing alone                                                    |
| `app/api/dash/`            | The BFF's route handlers. Four lines each; `src/server/bff.ts` is the substance                                                 |
| `middleware.ts`            | Next reads a middleware **only** from the project root                                                                          |
| `src/components/`          | Every rendered component. Pure props in, markup out — no `fetch`, no `next/headers`                                             |
| `src/server/`              | Everything touching vpay, cookies or PKCE. Imported only by `app/` and itself                                                   |
| `src/dash/`                | The `/dash/v1` read seam — `getList`/`getOne` over `readDash` — and, since 2026-10-21, `providers.tsx` (the `<Refine>` wrapper) |
| `src/config/`              | `settings.ts` decides what a configuration means; `runtime.ts` reads the env once                                               |
| `src/format.ts`            | Money, instants, the em dash                                                                                                    |
| `src/payments-query.ts`    | The URL's filter vocabulary ↔ the API's, and the two paging links                                                               |
| `src/client-navigation.ts` | Which plain-`<a>` clicks the shell routes client-side (since 2026-10-21)                                                        |
| `src/status-systems.ts`    | Refund, delivery and checkout status systems (since 2026-10-21)                                                                 |
| `src/testing/`             | Fixtures. Imported by tests and by nothing under `app/`                                                                         |

The split that matters is `src/server/` vs `src/components/`: a component that
fetched would be a component no test could render, and a `fetch` inside a
component is how a page ends up unable to say _why_ it is empty.

`src/server/actions.ts` is a `'use server'` file and may export **only async
functions** — a constant exported from there is a build error, not a lint
warning. That is why `FormState`/`NO_ERROR` live in `src/form-state.ts` and the
route paths in `src/server/session.ts`. Since 2026-10-21 it exports one
function fewer: `checkSession`, the Server Action Refine's
`authProvider.check()` called, is deleted (see "Navigation does not reload
the document" below).

Every page is `export const dynamic = "force-dynamic"`, and has to be: they all
read cookies.

## Pages

| Route                                                 | What it does                                                                                                                                                                                                                                                                                                                                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `/`                                                   | Redirects to `/overview` or `/login`. ~~Redirects to `/payments` or `/login`~~: `HOME_PATH` (`src/server/session.ts`) was `/payments` until 2026-10-21 (vaam-apps/vpay#318), and the TOTP, password-change and SSO success paths send a browser to the same constant                                                                                                                 |
| `/login`, `/login/totp`, `/login/password`            | Password → TOTP (a first sign-in also renders the enrolment QR and secret) → the forced password change. **With ADR-0030 merged**, `/login` also offers a "Sign in with Keycloak" button when `GET /dash/v1/staff/sso` says `enabled`; see [references/sso.md](references/sso.md). Since 2026-10-21 each signed-out page is one card with the mark and a theme shortcut, and no rail |
| `/overview`                                           | Home since 2026-10-21. Twelve exact counts read from the search procedures, each a link to the list filtered by the same value, and the latest six payments. See "The Overview" below                                                                                                                                                                                                |
| `/payments`, `/payments/{id}`                         | REST-backed. Status filter chips (links) and a date range, cursor paging; the detail page adds charge, refunds, last error, timeline. ~~Status and date filters~~: the status `<select>` and its Apply button are gone since 2026-10-21                                                                                                                                              |
| `/refunds`, `/deliveries`, `/customers`, `/checkouts` | Procedure-backed lists, offset paged, no detail route yet. ~~No filters~~: since 2026-10-21 refunds and checkouts filter by `status` and deliveries by `state`, as chips, and refunds and checkouts link each row to its payment. Customers has no filter                                                                                                                            |
| `/signed-out`, `/healthz`                             | Route handlers, not pages: clear a dead session cookie (since 2026-09-07); "Next is answering" (since 2026-09-16, ADR-0022, vpay#183)                                                                                                                                                                                                                                                |

All four procedure-backed lists are Server Components reading through
`readProcedurePage`, and all four share `src/components/procedure-table.tsx`
and, since 2026-10-21, one server component,
`src/components/procedure-list-screen.tsx`, with their words and filters in
`src/dash/list-screens.ts`.

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
vpay#168, 2026-09-13). ~~ADR-0017 decision 1 refuses **every** authenticated
route to a session carrying it, `/oauth/authorize` included.~~ **Corrected
2026-09-25:** that is how ADR-0017 decision 1 words it, not what the code
does. At `f68fda09` the refusal is per route. `GET /dash/v1/oauth/authorize`
refuses such a session, so no `/dash/v1` token can exist until the change is
done. `GET /dash/v1/staff/session` and `POST /dash/v1/staff/totp` answer
`password_change_required: true` instead of refusing, and the dashboard's
`gateFor` (`src/server/gate.ts`) sends the session to `/login/password`. All
three ask `vpay_api::staff::password_change_required`, which is deliberately
not folded into `load_session`, so **a new authenticated staff route refuses
nothing until it asks too.** ~~There is
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
has no payments" from "I am looking at the wrong merchant". ~~From 640px it is
on screen. Below 640px it has been one tap away since 2026-09-14: in the Menu
drawer until vaam-apps/vpay#258, and in `SideNav`'s More sheet since that
merge (2026-09-25).~~ **Corrected 2026-10-21 (vaam-apps/vpay#318):** it is on
screen at every width, as a `Badge` in the sticky top app bar `AppShell`
renders above every signed-in screen. The struck sentence was true from
2026-09-25 until that merge. The account block (`SignedInBar`) still names the
merchant too.

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

**The rail has an Overview and sections since 2026-10-21 (vaam-apps/vpay#318).**
`OVERVIEW_NAV` (`/overview`) is `SideNav`'s `topItem` and the one destination
that is **not** a Refine resource: it lists nothing, it counts. The resources
are grouped by a `group` field on each `DASH_RESOURCES` entry, rendered as
`NAV_GROUPS` in `NAV_GROUP_NAMES` order: Money (Payments, Refunds,
Checkouts), Customers (Customers), Integrations (Deliveries). A field on the
entry rather than a second list, so a resource cannot be routed and missing
from the rail. `layout.test.tsx` gained two cases with it: the Overview's
href resolves to a page, and the sections hold every resource exactly once
and nothing else. A new destination that is not a resource has to pass the
same page-exists check the Overview does.

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
Since 2026-10-21 it is also in the top app bar from 640 to 1279px, and it is
an outlined (`variant="secondary"`) button with an avatar beside the address,
where it was a filled `destructive` one: signing out destroys nothing, and the
red button was the loudest thing on every screen.
`references/what-a-screen-may-show.md` says where the block's copies are and
how a test finds the visible one.

## Navigation does not reload the document (since 2026-10-21)

Until vaam-apps/vpay#318 every rail click was a **full document load**:
re-download, re-hydrate, Refine's cache thrown away. #318 measured one fixed
click-through at 1440px on a production build against a fake of the
`/dash/v1` contract (not a real vpay): **6 document loads → 1**, **2 session
reads per navigation → 1**, a next page of payments read **twice → once**, and
a rail click **~1.0–1.2 s → ~0.37 s** to full content. Four causes, each fixed
where it lives, and each easy to put back:

1. **`SideNav` renders plain `<a href>`.** `@vaam-apps/ui` has no Next
   dependency, so it cannot render `next/link`. `AppShell` wraps the rail in
   one `onClick` that asks `inAppNavigationTarget` (`src/client-navigation.ts`)
   whether the click is an ordinary same-tab navigation and, if so, calls
   `router.push`. A modified click, a middle click, a link with `target` or
   `download`, another origin, or a click something already handled goes to
   the browser. React events bubble through the portalled rails and the More
   sheet, so one handler covers them. `app-shell.test.tsx`'s "routes a rail
   click through the app router, not a document load" and
   `client-navigation.test.ts` fail if you drop the handler or a guard. **A new
   plain `<a>` inside the rail is client-routed for free; a plain `<a>`
   anywhere else in the app is a full document load.** Use `next/link` there.
2. **Refine's `authProvider.check()` makes no request.** Refine's internal
   `RouteChangeHandler` calls it on every pathname change and discards the
   answer. It called the `checkSession` Server Action, so every navigation paid
   a POST and a second `GET /dash/v1/staff/session`. `check` now returns
   `{ authenticated: true }` with no network, and `checkSession` is deleted.
   This is safe only because nothing acts on `check`: every page under
   `app/(dash)` runs `requireStaff()` on the server, and a `401` from a read
   still reaches `onError`. **Do not add `<Authenticated>` or
   `useIsAuthenticated`**: `auth-provider.test.ts` fails if either appears,
   because either would act on an answer that no longer checks anything.
3. **A server-seeded query is never re-read by the browser.** `DashProviders`
   sets `staleTime: Infinity` and `gcTime: 0` (see the BFF note above). Do not
   lower `staleTime` to "keep data fresh": the server is what reads, on every
   navigation.
4. **The shell is mounted once.** Each page rendered its own `AppShell`, so
   every navigation unmounted and remounted the rail. `app/(dash)/layout.tsx`
   is now a Server Component that calls `requireStaff()` and renders
   `DashProviders` and `AppShell` around the page, and Next keeps a layout
   mounted. `requireStaff` is wrapped in React's `cache()`, so the layout and
   the page share one session read per request. **Every page still calls
   `requireStaff()` itself**: on a client navigation Next re-renders the page
   and not the layout, so the page's call is the gate. Do not remove it on the
   grounds that the layout already gated. A page that meets an outage renders
   it without a shell, because there is no session to say who is signed in.

Every route under `app/(dash)` has a `loading.tsx`, so `<main>` shows the
screen's skeleton the moment a link is followed. #318 records the cost: a row
link reaches full content ~0.38 s where it was ~0.25 s, because React holds a
revealed skeleton for about 300 ms. `dashboard.cy.ts` was updated for the new
landing route and the chip filter and **was not run**.

## The Overview (since 2026-10-21)

`/overview`, `HOME_PATH`, read by `readOverview` in `src/dash/overview.ts`:
**twelve procedure calls in parallel on the server**, each with `limit: 1`
and the procedure's own filter, and each number is the procedure's
`totalCount` (`count(*) OVER ()`, exact for that statement). The twelve: every
payment (with the latest six rows), one per `PAYMENT_STATUS` (five), failed and
pending refunds, failed and exhausted deliveries, open checkouts, and
customers. Each tile links to the list filtered by the same value
(`/payments?status=succeeded`, `/refunds?status=failed`,
`/deliveries?state=exhausted`, …), so a number always opens the rows it
counted.

Three rules, the first two tested by `overview.test.ts` ("reports each count
exactly as vpay's totalCount, never summed" and the case below):

- **Nothing is summed or estimated.** "All payments" is its own unfiltered
  call, never the sum of the five statuses, which could disagree under
  concurrent writes and would then be a number vpay never said.
- **A refused read stays refused, per number.** Each count is its own
  `ApiResult`, and a refused one is drawn as a dash beside the refusal, never
  as `0`. Replacing the failure branch with `{ ok: true, value: 0 }` fails
  "keeps a refused count refused — never a zero".
- **These are counts, not sums.** The payment detail's "vpay exposes no
  aggregates" caption is still true of amounts, and nothing on the Overview
  adds money up.

The payment counts come from `searchPaymentIntents`, the procedure. `/payments`
itself still reads the cursor-paged REST route, which has no total, so the
list still shows no count. See `references/read-seam-and-bff.md`.

## The demo stack's dashboard (from `docs/flows/dashboard/status-styling-and-demo.md`)

_Added 2026-09-29; the justfile variables were checked at `a33aac61`, the
behaviour was not re-run._ `demo_dashboard_port` (default `3000`, since
2026-09-10, issue #78) is threaded through `compose.demo.yml`'s publication (still
`!override` and bound to `127.0.0.1`, because what is behind it is a real
staff sign-in form), the app's `VPAY_DASHBOARD_REDIRECT_URI`, the generated
overlay's registered `redirect_uris` and Cypress's `baseUrl`, so two demo stacks
can each serve a dashboard; the two OAuth legs must still spell one string
identically, and `gen-demo-keys` regenerating the overlay is what keeps them
doing so. **Which tenant the demo dashboard reads** is `demo_dashboard_merchant`,
`shop-merchant-tenant` by default (and `demo_staff_merchant` is the same
variable, so the staff member cannot end up in the other tenant). It was
`demo-merchant-tenant` until 2026-09-11 (exp51), so a payment a person made by
hand through the shop was absent from the list and its id answered the uniform
cross-tenant `404`: both answers were correct, and the binding named the tenant
nobody clicks in. Point it back and `dashboard.cy.ts` fails on both the list and
the by-id read, which is asserted twice on purpose (once through the row link,
once by visiting `/payments/{id}` directly, because only the second can fail on
its own).

**Offset paging is not stable while payments are being created** (exp54 review,
2026-09-11), and it applies to the procedure-backed lists above: `OFFSET` is
counted afresh per call, so a payment created between page 1 and page 2 shifts
the window by one, repeating the last row of a page at the top of the next and
missing one at the tail — worst exactly where the list matters. The cursor list
`GET /dash/v1/payment_intents` does not have the property
(`starting_after` names a row, which an insert cannot move). Neither the
procedure nor anything else in vpay compensates. (The flow page, written when no
screen read through a procedure, called moving the frontend onto one "a
maintainer's decision"; four screens have been built on them since, per the
route table above, and this is what that costs them.)

## Flow pages, and where each is covered

| Page                                                 | Covered in                                                                                                                                                                      |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `docs/flows/dashboard/slice-1-gaps.md`               | this page's refund and `customer` sections; what slice 1 did not build, corrections since                                                                                       |
| `docs/flows/dashboard/status-built-and-not-built.md` | `references/what-a-screen-may-show.md` (the shell, the 0.3.0 and 0.4.0 chrome, the a11y gate)                                                                                   |
| `docs/flows/dashboard/status-read-seam-and-bff.md`   | `references/read-seam-and-bff.md`                                                                                                                                               |
| `docs/flows/dashboard/status-styling-and-demo.md`    | the demo section above; the 2026-10-21 redesign in the reload and Overview sections above and in `references/what-a-screen-may-show.md`; the styling stack is `vpay-frontend`'s |
| `docs/flows/dashboard-auth/rate-limiting.md`         | `references/staff-auth.md`                                                                                                                                                      |
| `docs/flows/dashboard-auth/sessions-and-refusals.md` | `references/staff-auth.md`; the forced password change is above                                                                                                                 |
| `docs/flows/dashboard-auth/scope-and-tokens.md`      | `references/staff-auth.md`, and the re-mint in `references/read-seam-and-bff.md`                                                                                                |
| `docs/flows/dashboard-auth/sso.md`                   | `references/sso.md` (only on a vpay that has merged ADR-0030)                                                                                                                   |

## More

- `references/staff-auth.md` — the sign-in limiter and which address it counts,
  why every refusal is one `401` and what that forbids a client from doing, the
  stage route, scope and token lifetimes, and where the JWKS is (not on
  `/dash/v1`). **Read this before touching `vpay_api::staff` or a `401`
  handler.**
- `references/sso.md` — single sign-on through an OpenID Connect provider
  (ADR-0030, **only on a vpay that has merged it**): the pending-flow table and
  why it is not a session, the order of the callback's checks, the one
  `prompt=login` retry, what Keycloak really issues and the two realm settings
  that make `amr` exist. **Read this before touching `vpay_api::staff::sso` or
  the dashboard's `/api/auth/sso/*` routes.**
- `references/read-seam-and-bff.md` — the two transports and the five mounted
  procedures, `PROCEDURE_OF`, the token lifecycle, the BFF's properties and its
  method policy. **Read this before adding a procedure or a read.**
- `references/what-a-screen-may-show.md` — the honest-absence rules, status
  colour, which `@vaam-apps/ui` components the app imports, the 0.5.0
  chrome (the top app bar, the More sheet and `accountSlot`, the loading
  skeletons, the filter chips and the date filter), the a11y gate, and what
  this app still cannot do. **Read this before
  touching the shell, a loading state or a filter.**
