# What a dashboard screen may and may not show

_Verified against vpay `f885170c` (2026-10-27), the merge of
vaam-apps/vpay#383, for which files and tests still exist; the measurements
below were not re-run, and each keeps the date it was made. Version-sensitive
claims carry the date they became true — see
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

> **Since 2026-10-27 no screen renders any data.** #383 deleted every page,
> the shell, the navigation and the loading routes. `frontends/apps/dashboard`
> renders `ConsoleUnavailable` for every path (`SKILL.md`). The
> **presentational components survive with their tests and stories**, because
> VP-6b will render them again. The rules below are the rules those
> components already obey and any restored screen must keep. Where a rule was
> about a deleted module, it is struck and dated.

vpay's cardinal rule (nothing may look more finished than it is) has a
specific shape on an operator console, because the failure mode is not a lie in
prose: it is a **populated-looking row**. Every rule below is a case where the
honest answer is an absence, and where a plausible substitute was available and
refused. The notice page is the same rule applied to the whole app: no form
that cannot work and no page that cannot load.

## The two named absences

**No "Rail" column on the list.** `GET /dash/v1/payment_intents` returns no
charge, so the only rail-shaped value in that response is
`payment_method_types`, the rails an intent _may_ be confirmed against. The
column (`src/components/payments-table.tsx`) is therefore headed **Methods**.
A "Rail" heading over it would be wrong for every intent that offers two and
was taken by one, and wrong **invisibly**. The detail component has a real
`Rail`, from `charge.provider_code`.

**The masked payer is an em dash, and it is a real `null`.**
`charges.payer_ref_masked` is never written by anything (the module doc of
`vpay_api::dash::payment_intents`), so `payment-detail.tsx` renders it **from
the column** and never derives it. The only other value that could produce a
mask is the payer's unmasked phone number, and reading that into a staff
surface to make a row look populated is the trade this refuses. The row
exists rather than being omitted so the value appears the day the column is
written. `payment-detail.test.tsx` pins both directions. ~~`dashboard.cy.ts`
walks the list until it finds a payment that **has** a charge before
asserting it.~~ _(Deleted 2026-10-27 with the spec; it is case 9 of the ten
`docs/status/frontend.md` says VP-6b must restore.)_

The **list** has no payer column at all, which is the stronger form of the
same point: there is no field there to be null.

## The general rules these are instances of

| Situation                       | The honest shape                                                                                                                                                                                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| A column nothing writes         | Render it from the column, show `ABSENT` (`—`, `src/format.ts`), never derive a substitute                                                                                                                                                             |
| A read that failed              | Hand the refusal to the screen as an `ApiResult` failure (`src/server/api.ts`, `read-failure.tsx`, `src/dash/failure.ts`), never an empty page: an empty list and a refused read look identical on screen unless the screen is told which it got       |
| A status this build cannot name | Render it as **text**, not as a coloured pill: "a green badge on an unfamiliar value is a claim"                                                                                                                                                       |
| A value the wire cannot parse   | Return it verbatim rather than replacing it (`formatIsoInstant`)                                                                                                                                                                                       |
| No `total` from the API         | No page count, no page numbers. Two cursors and nothing else                                                                                                                                                                                           |
| A field too large for a list    | Omit it and say so. The refunds list did not show `failure_raw` (up to 2 000 characters of a rail's own prose, belonging to a detail read that does not exist)                                                                                         |
| A page nobody can feed          | No page. `src/layout.test.tsx` fails if any `page.*` but the notice's appears under `app/` (since 2026-10-27). ~~A slice nobody built: no nav entry, `layout.test.tsx` fails if one appears.~~ The nav is deleted; VP-6b's first page changes the test |

## Status colour has exactly one source

`src/payment-status.ts`'s `PAYMENT_STATUS_SYSTEM` is a `defineStatusSystem`
table and is **the one place** a `PaymentIntent` status is drawn. Its `label`
field reads `@vpay/tokens`'s `statusLabel`, so the operator-facing copy has
one source. `PaymentStatusPill` is `createStatusPill(PAYMENT_STATUS_SYSTEM)`
and takes its tone from that table, never from a local map, so a status cannot
read one colour in the list and another on the detail page.

`statusTone` was **deleted** on 2026-09-12. Do not reintroduce a per-screen
tone map; `verify-ui`'s check 7a-ii refuses a raw status-colour token written
in an app.

**Three more status systems since 2026-10-21 (vaam-apps/vpay#318), still in
`src/status-systems.ts` at `f885170c`:** `REFUND_STATUS_SYSTEM`,
`DELIVERY_STATE_SYSTEM` and `CHECKOUT_STATUS_SYSTEM` (and `RefundStatusPill`,
`DeliveryStatePill`, `CheckoutStatusPill`). The words come from
`@vpay/tokens` (`refundStatusLabel`, `deliveryStateLabel`,
`checkoutStatusLabel`), and each vocabulary is copied from its table's
`CHECK` constraint. So a refund's `succeeded` reads "Refunded" and a
delivery's `pending` reads "Attempt owed". Only a failed refund and an
exhausted delivery are `attention: "loud"`. A value outside the list is still
**text**: `asRefundStatus` and its siblings answer `null`. Add a state to the
`CHECK` constraint and to its token list together, or the new state renders as
plain text.

## The theme

`data-theme` is `dark` on `<html>` and `src/layout.test.tsx` pins it, reading
the registered names (`dark`, `light`) off `@vaam-apps/ui`'s own `theme.css`.
`app/layout.tsx` runs the package's `themeInitScript` as the **first child of
`<body>`** and renders nothing in `<head>`. **Do not move it back into
`<head>`, and do not add a React-rendered child there**: on 2026-10-21 that
broke hydration under Cypress (React #418) and `dashboard.cy.ts` went 17 of
18 red on `master` at `0706cd5` (fixed by vaam-apps/vpay#323).
`suppressHydrationWarning` covers an element's attributes, not its children.
The same file self-hosts Roboto Flex and JetBrains Mono
(`@fontsource-variable/*`).

Whether an unregistered `data-theme` still renders dark on `@vaam-apps/ui`
0.5.0 was **not re-measured** (it did on 0.4.0, measured 2026-09-25 in
headless Chromium). `layout.test.tsx`'s header still says a `data-theme`
naming neither theme "compiles no styling at all", which was measured false on
0.4.0. Pin `dark` and do not rely on the fallback. As of 2026-10-27 no
rendered component offers a theme control: `src/components/theme-toggle.tsx`
survives, and nothing renders it.

## Which components come from `@vaam-apps/ui` (`^0.5.0`)

**Imported by the app as of vpay `f885170c` (2026-10-27)**, read off every
`import … from "@vaam-apps/ui"` under `frontends/apps/dashboard/` outside
stories and tests: `Table` and its parts
(`TableHeader`/`TableBody`/`TableRow`/`TableHead`/`TableCell`), `FormField`,
`Button`, `Badge`, `Card`/`CardHeader`/`CardBody`, `InlineBanner`,
`InlineEmptyState`, `Code`, `IdDisplay`, `CopyButton`, `ScreenStack`,
`Skeleton`, `StatTile`, `DetailList`/`DetailRow`, `InstrumentPanel`,
`DateRangePicker` and its parts, `useTheme`, `themeInitScript`,
`createStatusPill`/`defineStatusSystem`.

~~… `Input`, `Separator`, `SideNav`, `ThemeSwitcher` …~~ **Gone 2026-10-27**
with the sign-in forms and the shell: none of the four is imported now
(`SideNav` and `ThemeSwitcher` survive only in comments). The list was right
at `73c3a7cf` (2026-10-21). Earlier lists, from `f68fda09` and `b747e5d5`,
were corrected on this page before and are in its git history.

`InlineBanner` renders **no `role`** where the old `Alert` defaulted to
`role="alert"`, so every site that needs one wraps it and names the role
(`ConsoleUnavailable` wraps its banner in `role="status"`).

## Filters, the date picker and the tables

**Filters are chips that are links (since 2026-10-21).** `FilterChips`
(`src/components/filter-chips.tsx`) renders a single-choice filter as a
`<nav aria-label=…>` of M3 filter chips, each a `next/link` to the filtered
URL, the selected one `aria-current="true"`. The URL stays the only filter
state: a chip works before hydration and a middle click opens the filtered
list in a new tab. Every payments href drops the cursor (`paymentsHref`). The
selected chip and the unselected one are two literal branches, because
`verify-ui` refuses a computed `className`; the selected one is
`primary-container`, the one hue in the theme that is never a status.
~~`ProcedureListScreen` passes the URL's value to the procedure unchanged, so
a value vpay does not know is a `400` naming the field; the procedure lists
take their filters from `src/dash/list-screens.ts`.~~ _(Both deleted
2026-10-27. The `400` naming an unknown filter value is vpay's, and a restored
list should still render it as a refusal, not as an empty list.)_

**The date filter** (`payments-filters.tsx`, since vaam-apps/vpay#258,
2026-09-25) is `DateRangePicker`'s compound parts inside a `FormField`
labelled "Created between". The label's `htmlFor` is the trigger's `id`
(`payments-filter-created`) and the trigger has no `aria-label`, so tests find
it with `getByRole("button", { name: "Created between" })`. A pick is staged:
it reaches the filter only on **Save**, and Cancel, Escape or a press outside
discards it. A start saved alone sends `created_from` only. The trigger is
`w-[calc(25ch+16px+3.5rem)]` (it was `23ch` until 2026-10-21, before the
self-hosted JetBrains Mono): the `16px` and `3.5rem` are kept apart on
purpose, because the rem chrome grows with the browser's default font size
and the 14px value does not. Keep `font-mono` on it, since that sets the font
`ch` is measured in. `max-w-full` caps it at the row. Don't add
`overflow-x-auto` to the filter row. The `FiltersWithRange` (1280×800) and
`FiltersWithRangePhone` (375×812) stories measure it in a real browser.

**`ProcedureTable`** (`src/components/procedure-table.tsx`) is the app's own
shell for the four procedure lists, which differ only in their columns. A
row's id is `IdDisplay`. Its `secondary` flag hides a column below `sm`
through `Table`'s own `hideBelow` prop (`@vaam-apps/ui` 0.5.0); before that it
was two literal branches, because `verify-ui` refuses a computed `className`.
`Table` already wraps itself in `w-full overflow-x-auto`; the flag exists
because an operator who does not know to swipe reads the first two columns as
the whole answer.

**Skeletons survive; the routes that showed them do not.** `ListSkeleton`,
`PaymentsSkeleton` and `PaymentSkeleton` are still in `src/components/`, and
the `PaymentsLoading`, `PaymentsLoadingPhone`, `PaymentLoading` and
`PaymentLoadingPhone` stories still compare their claimed boxes with the real
screen's within 1px at 1280 and 375px and at a 20px root. Text that wraps and
values that decide where a row wraps are invisible copies of the real thing.
An unselected chip's placeholder keeps its transparent border: without it the
`PaymentsLoading` stories fail by 2px a chip. So **a change to a component's
header, description, filter row or panel changes its skeleton in the same
commit.** No skeleton renders an `<h2>`, and the payment's renders no
`data-testid="detail-id"`. ~~Every route under `app/(dash)` has a
`loading.tsx`; `OverviewSkeleton`; `screens.test.tsx` fails if a loading
branch renders `RouteSkeleton`.~~ _(All deleted 2026-10-27.)_

## The shell, the rail and the account block (deleted 2026-10-27)

Kept, struck, because these were the sections an agent most often acted on:

- ~~**The top app bar** (since 2026-10-21): `AppShell` renders a sticky bar
  with the mark, the merchant as a `Badge`, the theme shortcut and, from 640
  to 1279px, `SignedInBar` with Sign out.~~
- ~~**Floating chrome** (since 2026-09-25): below 1280px `SideNav`'s toolbar
  ends in a **More** control opening a modal sheet; the account block
  (`SignedInBar`, a "Theme" caption, `ThemeSwitcher`) is `accountSlot`, mounted
  up to three times, so tests scope to `main` or to the dialog.~~
- ~~**Navigation is a gate**: `DASH_RESOURCES` in `src/dash/resources.ts` is
  the one declaration, and the rail's hrefs are exactly the registry's list
  routes.~~

**Deleted 2026-10-27** with `app/(dash)/layout.tsx`, `app-shell.tsx`,
`signed-in-bar.tsx`, `src/dash/resources.ts` and their tests. What survives:
`src/client-navigation.ts` and its test, the guard that decides which plain
`<a>` clicks inside the rail become `router.push` (modified clicks, middle
clicks, `target`, `download` and other origins go to the browser). Nothing
calls it, because there is no rail. A restored rail that renders `SideNav`'s
plain `<a href>` without it is back to a full document load per click, which
#318 measured at about 1.0 to 1.2 s apiece on 2026-10-21.

## What this app cannot do (as of 2026-10-27)

_(This section said "As of vpay `73c3a7cf`" until 2026-10-27, when every
entry in it became a subset of the first.)_

- **Anything at all.** It renders one notice and calls nothing. It cannot
  sign anyone in, list a payment or show a count until VP-6b.
- **Anything to a payment, even after VP-6b.** `/dash/v1` refuses every write
  method in the layer, before the router matches: no re-poll, no replay, no
  refund, no annotation, and therefore **no `audit_log`, because there is
  nothing yet to audit**. ADR-0008 gates writes behind that log. If you are
  asked to add a write, that is the conversation, not a route.
- **Any other slice.** Webhook configuration and retries, balances, settings
  and rail health are not built.
- **Contrast checking in the unit suite.** jsdom computes no paint.
  `src/a11y.test.tsx` runs axe-core's **structural** rules over the root
  layout around the notice and over the payments list and filters, the
  payment detail (with and without a charge) and the pager. Colour is answered only by
  `pnpm --filter @vpay/dashboard test-storybook`, in a real Chromium.
- **Show a refund's events on the payment timeline.** Read from the code on
  2026-09-23 and untested: refund events are keyed on the `re_…` id, and the
  detail read fetches events for the intent and the charge only (`SKILL.md`).

## Testing notes

`vitest.config.ts` sets `globals: false`, so Testing Library does **not**
register its own cleanup; `vitest.setup.ts` calls `afterEach(cleanup)`
explicitly. Without it every render in a file accumulates in one
`document.body` and `getByRole` finds the previous test's copy of a control.

`src/a11y-gate.test.ts` is the twin of the checkout's. It pins both
suppression lists (`SUPPRESSED_STORIES`, `SUPPRESSED_RULES`) as **empty**
(still true at `f885170c`): no story may disable an axe rule or carry a
`parameters` override, so a story's viewport or theme goes in story-level
`globals`, and a reviewed exception has to be written into the test's two
lists. The history of the one suppression it once carried (`landmark-unique`,
for vaam-apps/ui#16, a `SideNav` defect fixed in `@vaam-apps/ui` 0.4.0) is in
this page's git history; the `Shell*` stories that proved the fix were deleted
with the shell on 2026-10-27.

The dashboard's Storybook has **20 stories as of vpay `f885170c`**
(2026-10-27), 3 of them `light`, all over surviving components (payments
table, filters, chips, pagers, detail, skeletons, a read failure, the timeline
gap). It had 36 at `73c3a7cf`. The stories are bound to the same
`src/testing/fixtures.ts` `a11y.test.tsx` renders, so a component cannot gain
a story without a test already covering it.

## The decisive mutations

Each was applied to the tree, the suite run, and the mutation reverted, when
the row was written. None has been re-run for this restamp.

| Mutation                                                               | Fails                                                                  |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| A second `page.*` under `app/` (since 2026-10-27)                      | `src/layout.test.tsx`, "serves `/` and nothing else" (not re-run here) |
| `pageCursors` reads `has_more` the same way in both directions         | `src/payments-query.test.ts`, twice                                    |
| `inAppNavigationTarget` drops its origin or modifier-key guards (#318) | `src/client-navigation.test.ts`                                        |

Struck 2026-10-27, because the code and the test are both deleted. A restored
equivalent needs a new mutation and a new row:

| Mutation                                                                         | Failed                                           |
| -------------------------------------------------------------------------------- | ------------------------------------------------ |
| ~~`COOKIE_ATTRIBUTES.httpOnly` → `false`~~                                       | ~~`src/server/cookies.test.ts`~~                 |
| ~~Exchange a _fresh_ PKCE verifier rather than the one the challenge came from~~ | ~~`src/server/oauth.test.ts`~~                   |
| ~~A nav entry for a page nobody wrote~~                                          | ~~`src/layout.test.tsx`, twice~~                 |
| ~~The BFF reads the session cookie **after** the upstream read~~                 | ~~`src/server/bff.test.ts`, twice~~              |
| ~~`apiIsSameOrigin` drops its `originIsAllowed` call~~                           | ~~`src/server/bff.test.ts`, twice~~              |
| ~~The BFF serves the parsed upstream document instead of the named fields~~      | ~~`src/server/bff.test.ts`~~                     |
| ~~`middleware` returns `NextResponse.next()` unconditionally~~                   | ~~`middleware.test.ts`, three times~~            |
| ~~`apiIsSameOrigin` drops its `Sec-Fetch-Site` comparison~~                      | ~~`dashboard.cy.ts`'s frame case, in a browser~~ |
| ~~`AppShell` loses the rail's `onClick` (#318)~~                                 | ~~`src/components/app-shell.test.tsx`~~          |
| ~~`readOverview`'s failure branch becomes `{ ok: true, value: 0 }` (#318)~~      | ~~`src/dash/overview.test.ts`~~                  |

The last row's rule outlives its test: **a refused count stays refused, drawn
as a dash, never a `0`**, and nothing is summed or estimated. A restored
Overview must keep both, and needs a new test that fails when either breaks.
