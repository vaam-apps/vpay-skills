# What a dashboard screen may and may not show

_Verified against vpay `73c3a7cf` (2026-10-21); the sections about the
shell, the filters, loading states, the imported components and status colour
were re-read there, and the rest last at `f68fda09` (2026-09-25). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

vpay's cardinal rule — nothing may look more finished than it is — has a
specific shape on an operator console, because the failure mode is not a lie in
prose: it is a **populated-looking row**. Every rule below is a case where the
honest answer is an absence, and where a plausible substitute was available and
refused.

## The two named absences

**No "Rail" column on the list.** `GET /dash/v1/payment_intents` returns no
charge, so the only rail-shaped value in that response is
`payment_method_types` — the rails an intent _may_ be confirmed against. The
column is therefore headed **Methods**. A "Rail" heading over it would be wrong
for every intent that offers two and was taken by one, and wrong **invisibly**.
The detail page has a real `Rail`, from `charge.provider_code`.

**The masked payer is an em dash, and it is a real `null`.**
`charges.payer_ref_masked` is never written by anything (the module doc of
`vpay_api::dash::payment_intents`; this cited `docs/status.md` until
2026-09-23, which does not mention the column), so
the detail page renders it **from the column** and never derives it. The only
other value that could produce a mask is the payer's unmasked phone number, and
reading that into a staff surface to make a row look populated is the trade
this refuses. The row exists rather than being omitted so the value appears the
day the column is written. `payment-detail.test.tsx` pins both directions, and
`dashboard.cy.ts` walks the list until it finds a payment that **has** a charge
before asserting it — a dash on an intent nobody confirmed proves nothing.

The **list** has no payer column at all, which is the stronger form of the same
point: there is no field there to be null.

## The general rules these are instances of

| Situation                       | The honest shape                                                                                                                                                                                                                                     |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A column nothing writes         | Render it from the column, show `ABSENT` (`—`), never derive a substitute                                                                                                                                                                            |
| A read that failed              | Return the refusal to the screen. `readProcedurePage` (`readProcedureList` since vaam-apps/vpay#371) returns the refusal rather than an empty page: an empty list and a refused read look identical on screen unless the screen is told which it got |
| A status this build cannot name | Render it as **text**, not as a coloured pill — "a green badge on an unfamiliar value is a claim"                                                                                                                                                    |
| A value the wire cannot parse   | Return it verbatim rather than replacing it (`formatIsoInstant`)                                                                                                                                                                                     |
| No `total` from the API         | No page count, no page numbers. Two cursors and nothing else                                                                                                                                                                                         |
| A field too large for a list    | Omit it and say so. `/refunds` does not show `failure_raw` — up to 2 000 characters of a rail's own prose, belonging to a detail read that does not exist                                                                                            |
| A slice nobody built            | No nav entry. `layout.test.tsx` fails if one appears                                                                                                                                                                                                 |

## Status colour has exactly one source

`src/payment-status.ts`'s `PAYMENT_STATUS_SYSTEM` is a `defineStatusSystem`
table and is **the one place** a `PaymentIntent` status is drawn. Its `label`
field reads `@vpay/tokens`'s `statusLabel`, so the operator-facing copy has one
source. `PaymentStatusPill` is `createStatusPill(PAYMENT_STATUS_SYSTEM)` and
takes its tone from that table, never from a local map — so a status cannot
read one colour in the list and another on the detail page.

`statusTone` was **deleted** on 2026-09-12 (zero consumers after the
`@vaam-apps/ui` cutover). Do not reintroduce a per-screen tone map;
`verify-ui`'s check 7a-ii refuses a raw status-colour token written in an app.

**Three more status systems since 2026-10-21 (vaam-apps/vpay#318).**
`src/status-systems.ts` declares `REFUND_STATUS_SYSTEM`,
`DELIVERY_STATE_SYSTEM` and `CHECKOUT_STATUS_SYSTEM` (and
`RefundStatusPill`, `DeliveryStatePill`, `CheckoutStatusPill`) the same way.
Until that merge `/refunds`, `/deliveries` and `/checkouts` printed their state
column as the raw wire string. The words come from `@vpay/tokens`
(`refundStatusLabel`, `deliveryStateLabel`, `checkoutStatusLabel`), and each
vocabulary (`REFUND_STATUS`, `DELIVERY_STATE`, `CHECKOUT_STATUS`) is copied
from its table's `CHECK` constraint. So a refund's `succeeded` reads
"Refunded" and a delivery's `pending` reads "Attempt owed". Only a failed
refund and an exhausted delivery are `attention: "loud"`. A value outside the
list is still rendered as **text**: `asRefundStatus` and its siblings answer
`null` and the column prints the wire value. `@vpay/tokens` also gained
`statusShortLabel`, the five payment statuses in two words or fewer, for the
Overview's tiles. Add a state to the `CHECK` constraint and to its token list
together, or the new state renders as plain text.

`data-theme` is `dark` and `src/layout.test.tsx` pins it. ~~`@vaam-apps/ui`
registers its one theme under daisyUI's built-in name `dark`, so a `data-theme`
that says anything else renders the page **completely unthemed** in a real
browser, with no error anywhere.~~ **Corrected 2026-09-23, in two parts.**

1. **There are two themes, not one.** `dark` is the default and `light` is
   opt-in, and that has been true since `0.1.2`, the version this page was
   written against (`app/layout.tsx`). `ThemeSwitcher` writes the attribute.
   The Storybook runs 29 stories `dark` and 4 `light` as of vpay `f68fda09`
   (2026-09-25); it ran 21 and 4 from 2026-09-13
   (`docs/flows/dashboard/status-built-and-not-built.md`) through `b747e5d5`.
2. **Since `^0.2.4` (vpay#249, 2026-09-23) the package also declares its own
   dark tokens on `:root`**, in a `:root, [data-theme="dark"]` block that
   `0.4.0` keeps unchanged. **An unregistered `data-theme` renders dark.**
   Measured 2026-09-25 in headless Chromium: `@vaam-apps/ui` 0.4.0's built
   `dist/styles/theme.css`, compiled the way `app/globals.css` does it
   (`tailwindcss`, then the theme, then `daisyui` with `themes: false`) with
   the tailwindcss 4.3.3 and daisyUI 5.7.28 vpay's lockfile resolves, on a
   bare test page rather than the app's own build. `data-theme="nonsense"`
   and `data-theme="corporate"` computed the same `--color-base-100`
   (`#0a0b0d`), `--surface-3` (`#1c1f25`) and body background as
   `data-theme="dark"` and as no attribute, under both
   `prefers-color-scheme` values. Only `light` differed (`#fcfcfd`). The same
   probe on `0.1.2`'s `theme.css` gave daisyUI's colours dark as well, with
   the package's own `--surface-3` unset: with this toolchain a wrong name
   lost the package's own tokens, not all styling. vpay's own
   `layout.test.tsx` header still says a `data-theme` naming neither theme
   "compiles no styling at all", and the dashboard README says it renders
   "completely unthemed". Both are wrong at `f68fda09`. Fix them in vpay.

**`@vaam-apps/ui` 0.5.0 changed the values above (2026-10-21,
vaam-apps/vpay#318), and nobody has re-run the probe.** 0.5.0 generates its
palette from a Material 3 seed. `--color-base-100` is now
`var(--md-sys-color-surface)`, so the literal `#0a0b0d` the measurement found
no longer exists. Both apps' `styling-gate.test.ts` asserted that literal and
were re-pinned to the chain and to the package's own dark `surface` value,
read from the package. Whether an unregistered `data-theme` still renders dark
on 0.5.0 was **not re-measured**. Pin `dark` and do not rely on the fallback.

**The stored theme is applied before the first paint since 2026-10-21.**
`app/layout.tsx` runs the package's `themeInitScript` (a fixed string,
nothing interpolated) as the **first child of `<body>`**, and renders nothing
in `<head>`. The script reads the operator's stored choice and sets
`data-theme` before anything paints. `<html>` still says `data-theme="dark"`
and carries `suppressHydrationWarning` for that one attribute. Without the
script a light-theme operator saw every document load paint dark and then
flip. ~~It ran in `<head>`.~~ **Corrected later on 2026-10-21
(vaam-apps/vpay#323):** in `<head>` it was the one node React rendered
there, so React hydrated `<head>` expecting it first. The document Cypress
serves puts a whitespace text node in front, so hydration failed (React
#418), and `dashboard.cy.ts` went 17 of 18 red on `master` at `0706cd5`.
**Do not move it back into `<head>`, and do not add a React-rendered child
there.** `suppressHydrationWarning` covers an element's attributes, not its
children's structure. `src/layout.test.tsx` fails if the script leaves the
start of `<body>`. The `vaam-ui` skill's setup page still says `<head>`: in
this app, that advice is what broke the spec. The same file self-hosts Roboto Flex and JetBrains Mono
(`@fontsource-variable/*`), which `theme.css` names and the app never loaded
before: until then every glyph fell back to the system fonts. A width written
in `ch` of the mono face changed with it (see the date filter below).

~~`layout.test.tsx`'s case is still titled "the only theme @vaam-apps/ui
actually compiles", which is stale.~~ **Corrected 2026-09-25:**
vaam-apps/vpay#255 (2026-09-24) retitled it "sets the dark theme —
@vaam-apps/ui's default, and one of the two it compiles". The case now reads
the registered theme names off the package's `theme.css`, asserts they are
exactly `dark` and `light`, and asserts the layout pins exactly one
`data-theme`, `dark`. Pin `dark` because it is what the app ships, not because
nothing else would render.

## Which components come from `@vaam-apps/ui`

~~`@vaam-apps/ui` is `^0.2.4` since 2026-09-23 (vpay#249); it was `^0.1.2`
before.~~ **Corrected 2026-09-25:** it is `^0.4.0` since vaam-apps/vpay#258
(merged 2026-09-25 as vpay `f68fda09`). The struck sentence was true at
`b747e5d5` and until that merge. `0.3.0` was on vaam-apps/vpay#258's branch
only and never on `master`. `0.2.0` shipped visual breaking changes with no
API change, among them the `SideNav` width below `xl` and the dark theme
applying without a `data-theme`. So a green typecheck does not show that a
screen still looks right.

**`^0.5.0` since vaam-apps/vpay#318 (merged 2026-10-21 as vpay `73c3a7cf`),
the Material 3 Expressive release.** It was `^0.4.0` from #258 until then.

**Imported by the app as of vpay `73c3a7cf` (2026-10-21, `@vaam-apps/ui`
0.5.0)**, read off every `import … from "@vaam-apps/ui"` under
`frontends/apps/dashboard/` outside stories and tests: `Table` and its parts
(`TableHeader`/`TableBody`/`TableRow`/`TableHead`/`TableCell`), `FormField`,
`Input`, `Button`, `Badge`, `Card`/`CardHeader`/`CardBody`, `Separator`,
`InlineBanner`, `InlineEmptyState`, `Code`, `IdDisplay`, `CopyButton`,
`ScreenStack`, `SideNav`, `Skeleton`, `StatTile`, `DetailList`/`DetailRow`,
`InstrumentPanel`, `DateRangePicker` and its parts (with `IsoDateRange`),
`ThemeSwitcher`, `useTheme`, `themeInitScript`,
`createStatusPill`/`defineStatusSystem` (with the `StatusSystem` type). New
since 0.4.0: `Badge`, the `Card` parts, `Separator`, `IdDisplay`,
`CopyButton`, `StatTile`, `useTheme` and `themeInitScript`. The dashboard
still imports no `Pagination`, `Timeline`, `DataList`/`DataListRow`,
`RouteSkeleton`, `MoreDetailDrawer` or `DrawerClose`. The timeline and the
paging links are the app's own markup.

The list has been corrected three times, and the earlier versions are kept
because an agent on an older tree will meet them:

- ~~Imported by the app as of vpay `f68fda09` (2026-09-25, `@vaam-apps/ui`
  0.4.0): `Table` and its parts, `FormField`, `Input`, `Button`,
  `InlineBanner`, `InlineEmptyState`, `Code`, `ScreenStack`, `SideNav`,
  `Skeleton`, `DetailList`/`DetailRow`, `InstrumentPanel`, `DateRangePicker`
  and its parts (with `IsoDateRange`), `ThemeSwitcher`,
  `createStatusPill`/`defineStatusSystem`.~~ True from `f68fda09` until
  #318.

- ~~`Table`, `FormField` (was `Field`), `Input`, `Button`, `InlineBanner` (was
  `Alert`), `InlineEmptyState`, `Code`, `ScreenStack`, `SideNav`,
  `MoreDetailDrawer`, `ThemeSwitcher`, `Pagination`, `Timeline`,
  `DataList`/`DataListRow`, `createStatusPill`/`defineStatusSystem`.~~ The
  first version. The app imported none of `Pagination`, `Timeline` and
  `DataList`/`DataListRow` at `d3a8810b`, `b747e5d5` or `f68fda09`, and the
  list missed components the app did import.
- ~~Imported by the app as of 2026-09-23: `Table` (and
  `TableHeader`/`TableBody`/`TableRow`/`TableHead`/`TableCell`), `FormField`
  (was `Field`), `Input`, `Button`, `InlineBanner` (was `Alert`),
  `InlineEmptyState`, `Code`, `ScreenStack`, `SideNav`,
  `MoreDetailDrawer`/`DrawerClose`, `ThemeSwitcher`, `DetailList`/`DetailRow`,
  `DateRangePicker`, `InstrumentPanel`, `RouteSkeleton`,
  `createStatusPill`/`defineStatusSystem`.~~ Corrected 2026-09-23 against
  `b747e5d5` (`^0.2.4`), and true there. vaam-apps/vpay#258 took out
  `MoreDetailDrawer`, `DrawerClose` and `RouteSkeleton`, and added
  `Skeleton`: on `master` before it, both payments screens rendered
  `RouteSkeleton`, and the Menu pill's drawer (`more-menu.tsx`, deleted) was a
  `MoreDetailDrawer` with a `DrawerClose`.

Two things that changed shape rather than name: `InlineBanner` renders **no
`role`** where `Alert` defaulted to `role="alert"`, so every site relying on
that default now wraps it and names the role explicitly. ~~And
`payments-filters.tsx` uses a **native `<select>`**.~~ **Corrected
2026-10-21:** the status `<select>` and its Apply button are gone
(vaam-apps/vpay#318). Until then the native `<select>` stayed because the
package's `Select` wraps a Headless UI `Listbox` and forwards no `name`, and no
ref a plain `onChange` can read.

**Filters are chips that are links, since 2026-10-21.** `FilterChips`
(`src/components/filter-chips.tsx`) renders a single-choice filter as a
`<nav aria-label=…>` of M3 filter chips, each a `next/link` to the filtered
URL, the selected one marked `aria-current="true"`. The URL stays the only
filter state: a chip works before hydration, a middle click opens the
filtered list in a new tab, and the Server Component reads the URL. `/payments`
offers "All" plus one chip per `PAYMENT_STATUS` (words from `statusLabel`);
every href drops the cursor (`paymentsHref`). The procedure lists take theirs
from `src/dash/list-screens.ts`: refunds and checkouts by `status`, deliveries
by `state`, customers none. `ProcedureListScreen` passes the URL's value to the
procedure unchanged, so a value vpay does not know is a `400` naming the field,
rendered as a refusal and not as an empty list. An empty filtered list says
`Nothing matches this filter. Choose "All" to see every row.` The selected chip
and the unselected one are two literal branches, because `verify-ui` refuses a
computed `className`; the selected one is `primary-container`, the one hue in
the theme that is never a status. Paging a filtered procedure list keeps the
filter on both pager links.

**The date filter, since vaam-apps/vpay#258** (merged 2026-09-25; it navigates
on its own since #318, see below;
`@vaam-apps/ui` 0.3.0's picker, unchanged in the 0.4.0 that merge ships; not
on `master` before it).
`DateRangePicker` is compound parts:
`DatePickerTrigger`, `DatePickerValue` (placeholder "Any time"),
`DatePickerClear` and `DatePickerContent`, inside a `FormField` labelled
"Created between". The label's `htmlFor` is the trigger's `id`
(`payments-filter-created`), and the trigger has no `aria-label`. Its name is
"Created between" in every state, and its description is the picked dates, or
"Any time" when empty. Tests find it with
`getByRole("button", { name: "Created between" })`. Clicking the label focuses
the field without opening the picker.

A pick is staged: it reaches the filter only on **Save**, and Cancel, Escape or
a press outside discards it. Taps follow M3's range rule: the first sets the
start, a tap on or after it sets the end, and any other tap starts again. A
start saved alone sends `created_from` only, "from that day on", where the old
picker's first click was a one-day range. ~~While the picker is open the page is
inert, so the first press on Apply only closes it.~~ Since 2026-10-21 there is
no Apply: the picker's Save (its `onValueChange`) or its Clear navigates to the
filtered URL, with the cursor dropped.

The trigger has a fixed width, ~~`w-[calc(23ch+16px+3.5rem)]`~~
`w-[calc(25ch+16px+3.5rem)]` since 2026-10-21, sized to the longest value
`YYYY-MM-DD → YYYY-MM-DD`. It grew because the value is now set in the
self-hosted JetBrains Mono, which needed `25ch` where the system monospace
fitted in `23ch`. #318's stories caught the value truncated. The `16px` and
`3.5rem` are kept apart on purpose. The rem chrome grows with the browser's
default font size and the 14px value does not, so a single pixel figure cut
the range short at Chrome's "Large" setting. `font-mono` on it sets the font
that `ch` is measured in; keep it (it was `font-mono text-prose` until
2026-10-21). `max-w-full` on the trigger and on its `FormField` caps them at
the row, so a narrow window truncates the field rather than push the page
sideways. ~~The Status `<select>` is `h-10`, so the two labels and controls
line up. The row is `flex-wrap` at every width and never scrolls sideways.
Don't add a breakpoint or `overflow-x-auto` to it.~~ **Corrected 2026-10-21:**
the filter row is a column below `lg` and a row from `lg` (chips, then the
date field), and the chips wrap inside it. Still don't add `overflow-x-auto`.
The `FiltersWithRange` (1280×800) and `FiltersWithRangePhone` (375×812) stories
measure the date field in a real browser, including at a 20px root; this
paragraph's numbers before #318 were measured on the old row.

**The top app bar, since 2026-10-21 (vaam-apps/vpay#318).** `AppShell`
renders a sticky bar at the top of `<main>` on every signed-in screen: the
mark (`Brand`, not a heading), the merchant as a `Badge`, the theme shortcut
(`ThemeToggle`), and from 640 to 1279px `SignedInBar` with Sign out. The
document's one `<h1>` is a `sr-only` "vpay dashboard" in a bare `<header>`, and
every screen's title is an `<h2>` (`PageHeader`) under it, which is what
`dashboard.cy.ts` finds. Each screen is `PageHeader`, then its filters, then a
`Card`.

**Floating chrome, since vaam-apps/vpay#258** (2026-09-25, `@vaam-apps/ui`
0.4.0; `toolbarVariant="vibrant"` since 2026-10-21, so the toolbar's edge stays
visible over the Overview's tiles). Below
1280px `SideNav`'s floating toolbar ends in a **More** control that opens a
modal sheet: a bottom sheet from the phone bar below 640px ~~(288px wide with
the dashboard's five destinations, as of 2026-09-24)~~, a drawer from the left
edge from 640 to 1279px. ~~The phone bar's sheet holds the destinations the bar
has no room for (Customers)~~. **Since 2026-10-21** there are six destinations
(the Overview plus five resources): at 375px the phone bar shows Overview,
Payments, Refunds and Checkouts, and Deliveries and Customers are behind its
More (#318's own note; not re-measured here). The drawer lists every
destination, labelled. Both end
with the account block, `SignedInBar` then a "Theme" caption and
`ThemeSwitcher`, which `app-shell.tsx` passes as `accountSlot` at every width.
There is no Menu pill any more (on `master` before this merge there was one, at
`bottom-3 left-3`), so don't add your own `fixed` account or menu button.
Anything else pinned near the bottom must clear the bar's top edge, 80px up
(its 16px offset plus 64px): `bottom-24` sits 16px above it. `main` carries
~~`pb-20`~~ `pb-24` below 640px (since 2026-10-21), `sm:pl-24` from 640px and
`xl:pl-0` from 1280px, where the sidebar is in flow.

The account block is in the page more than once, and tests have to allow for
it. The sidebar's copy is always in the DOM, hidden by CSS below 1280px and
**before** `<main>`; `<main>` has its own `SignedInBar` from 640 to 1279px
(hidden below 640px); the sheet mounts a third copy while it is open. So a
Cypress query that finds the email or Sign out by its text scopes to `main`
(`dashboard.cy.ts` does, three times) or to
`getByRole("dialog", { name: "More" })`; below 640px only the sheet's copy is
visible, so a phone-width test opens it first. In jsdom there is no CSS: every
copy counts (`app-shell.test.tsx` expects exactly two sign-outs with the sheet
shut), and both toolbars render, each with a "More" control, so the tests pick
the phone bar's through `[data-floating-rail-axis="horizontal"]`, which
upstream does not list among its stable hooks (`data-side-nav-more`,
`data-side-nav-sheet`, `data-side-nav-account` are). That control is named
"More" and opens a `dialog` only because an `accountSlot` was passed; without
one it is "More destinations" and opens a `menu`. Nothing inside `accountSlot`
may hard-code an `id`, since the block is mounted twice while the sheet is open
(use `useId`; `a11y.test.tsx`'s open-sheet case fails on `duplicate-id`). And if
Sign out ever asks for confirmation, use `InlineConfirm`, not a `Dialog`: a
dialog opened from the modal sheet opens under its scrim, out of a pointer's
reach.

**Loading states, since vaam-apps/vpay#258 (2026-09-25).** ~~Both payments
screens load in their own shape.~~ **Since 2026-10-21 every route under
`app/(dash)` has a `loading.tsx`**, so `<main>` shows the screen's skeleton the
moment a link is followed: `OverviewSkeleton` (`overview-skeleton.tsx`),
`ListSkeleton` (`list-skeleton.tsx`, built from the same `list-screens.ts`
copy the four procedure lists render, so a description wraps where the page's
does), and the two below. Both payments
screens load in
their own shape: `PaymentsSkeleton` (`payments-skeleton.tsx`) for the list and
`PaymentSkeleton` (`payment-skeleton.tsx`) for one payment. On `master` before
that merge both rendered `RouteSkeleton`, whose header, filter bar and rows
matched neither page, so everything moved when the read returned. Headings are
one-line `h-lh` placeholders. Text that wraps (the list's description, the "At
a glance" caption) and values whose width decides where a row wraps (the filter
row's ~~select and Apply~~ chips since 2026-10-21, a four-digit XAF amount, a
status pill) are invisible copies of the real thing, so they wrap where the
page does. An unselected chip's placeholder keeps its transparent border:
without it the `PaymentsLoading` stories fail by 2px a chip. The
`PaymentsLoading`, `PaymentsLoadingPhone`, `PaymentLoading` and
`PaymentLoadingPhone` stories compare the claimed boxes within 1px at 1280 and
375px, in the shell's narrower columns and at a 20px root, and
`screens.test.tsx` fails if either loading branch renders `RouteSkeleton` again.
So a change to a page's header, description, filter row or panel changes its
skeleton in the same commit. No skeleton renders an `<h2>`, and the
payment's renders no `data-testid="detail-id"`: `dashboard.cy.ts` waits on
those as the sign the page has loaded. The rows below the filter row or the
Summary heading are data, and nothing claims they line up.

`ProcedureTable` (`src/components/procedure-table.tsx`) is the app's own shell
shared by the four procedure lists, because they differ only in their columns.
Since 2026-10-21 the four pages also share one server component,
`ProcedureListScreen` (`procedure-list-screen.tsx`: gate, read, filter chips,
refusal or empty state or table, pager), and each page is its columns and its
words. A row's id is `IdDisplay` (`IdCell`), and refunds and checkouts link
each row's payment through `PaymentLinkCell`. Its `secondary` flag hides a column below `sm`, ~~as a **static class per column,
never a computed one**~~ since 2026-10-21 through `Table`'s own `hideBelow`
prop (`@vaam-apps/ui` 0.5.0). Before that it was two literal branches, because
`verify-ui` refuses a computed `className` anywhere in an app. `Table` already wraps itself in `w-full overflow-x-auto`, so a wide
table _can_ be swiped on a phone; the flag exists because an operator who does
not know to swipe reads the first two columns as the whole answer.

## What this app still cannot do

As of vpay `73c3a7cf` (2026-10-21). _(This said "As of 2026-09-16" until
2026-09-23, when the last entry was added, and "As of vpay `f68fda09`" until
2026-10-21, when the BFF entry changed.)_ #318 added no write and moved no read
to the browser.

- **Anything at all to a payment.** `/dash/v1` refuses every non-`GET` method
  **at the boundary, before the router matches** — no re-poll, no replay, no
  refund, no annotation, and therefore **no `audit_log`, because there is
  nothing yet to audit**. ADR-0008 gates writes behind that log. If you are
  asked to add a write, that is the conversation, not a route.
- **Any other slice.** Webhook configuration, balances, settings and rail
  health are not built, and the nav gate fails if it ever links to them.
- **Contrast checking in the unit suite.** jsdom computes no paint.
  `src/a11y.test.tsx` runs axe-core's **structural** rules over the real
  rendered `<body>` of the layout and every screen including its error and
  pending states — it exists because an earlier draft dropped the `<main>`
  landmark, `region` went 0 → 1 violation, and every other gate stayed green.
  Colour is answered only by `just test-storybook`, in a real Chromium.
- **Most reads through the BFF.** Only the payments list and the payment
  detail are wired to it. ~~and only for reads after their first render~~
  Since 2026-10-21 their server-seeded reads do not reach it, and #318's
  click-through made zero BFF fetches (this line said "No page and no
  component calls it" until 2026-09-23, and "only for reads after their first
  render" until 2026-10-21).
- **Show a refund's events on the payment timeline.** This was read from the
  code on 2026-09-23 and is untested. Refund events are keyed on the `re_…`
  id, and the detail read fetches events for the intent and the charge only.
  See the `SKILL.md` note.

## Testing notes

`vitest.config.ts` sets `globals: false`, so Testing Library does **not**
register its own cleanup — `vitest.setup.ts` calls `afterEach(cleanup)`
explicitly. Without it every render in a file accumulates in one
`document.body` and `getByRole` finds the previous test's copy of a control.

`src/a11y-gate.test.ts` is the twin of the checkout's. ~~With one difference:
this app **does** carry axe-rule suppressions, and the test pins exactly which
stories may carry one and which rule id.~~ **Corrected for vaam-apps/vpay#258**
(merged 2026-09-25, `@vaam-apps/ui` 0.4.0): it pins both lists as **empty**.
No story may disable an axe rule or carry a `parameters` override at all, so a
story's viewport or theme goes in story-level `globals`, and a reviewed
exception has to be written into the test's two lists. Until 0.4.0 the
`Shell` stories disabled `landmark-unique`, the rule below. They were first
written as `parameters.a11y = { test: "todo" }`, which switches the addon off
for the **whole story** — measured, a `#3a3a3a`-on-`#0a0b0d` probe placed inside
`Shell` PASSED under it, so the chrome around every screen in the app had no
colour-contrast verdict at all. With the suppression narrowed to one rule id
the same probe fails at 1.73.

That suppression covered a **real third-party defect**, not this app's markup:
at 1200×900 (below `xl`), `@vaam-apps/ui@0.1.2`'s `SideNav` rendered its
in-flow sidebar `<nav aria-label="Primary">` with only `xl:`-prefixed
utilities and no unprefixed `hidden`, so it fell back to `display: block` while
the 1024–1279px floating rail was also visible — two landmarks answering the
same name. Reproduced directly with axe-core at four viewports and traced to
the upstream line. ~~**It has not been reported upstream.**~~ **Corrected
2026-09-24:** it had been, as
[vaam-apps/ui#16](https://github.com/vaam-apps/ui/issues/16) (2026-09-13).

What happened to it next, in order. Two re-checks landed a day apart, and the
first one's advice is now spent:

1. **2026-09-23, `0.2.4`, read and not measured.** The dependency had moved
   to `^0.2.4`, and the suppression in `src/a11y-gate.test.ts` still named
   `@vaam-apps/ui@0.1.2`. In `side-nav.js`, the in-flow `<nav>`'s
   non-collapsed floating class string (`xl:flex …`, with no unprefixed
   `hidden`) was byte-identical in `0.1.2` and `0.2.4`, so the defect had
   probably not been fixed. `0.2.4` also rendered **three**
   `<nav aria-label="Primary">` shapes and relied on their breakpoint gates
   being complements. ~~Do not drop the suppression without re-running axe
   at 1200×900.~~ _(Right on `master` until vaam-apps/vpay#258 merged. That
   merge is the re-run: its `Shell` and `ShellLight` stories run axe at the
   suite's 1200×900 with nothing suppressed.)_
2. **2026-09-24, `0.3.0`, measured** by the vaam-apps/vpay#258 review: one
   violation at 375–1279px, none from 1280px.
3. **Fixed in `0.4.0`** (vaam-apps/ui#39, 2026-09-24): the in-flow `<nav>` is
   hidden below 1280px, so one `Primary` landmark is exposed at every width.
   vaam-apps/vpay#258 (merged 2026-09-25) dropped the suppression, and the
   `ShellPhone`, `ShellRail`, `ShellLaptop` and `ShellDesktop` stories (375,
   700, 1100 and 1280px) assert exactly one exposed `Primary` landmark.

The dashboard's Storybook has 36 stories as of vpay `73c3a7cf`
(vaam-apps/vpay#318, merged 2026-10-21), 31 rendered `dark` and 5 `light`. It
had 33 from vaam-apps/vpay#258 (merged 2026-09-25), and 25 from 2026-09-13,
when it was added, through `b747e5d5` (counted 2026-09-23). Both a11y lists in
`src/a11y-gate.test.ts` are still empty at `73c3a7cf`. They are bound to the same
`src/testing/fixtures.ts` `a11y.test.tsx` renders, so a screen cannot gain a
story without a test already covering it.

## The decisive mutations

Each was applied to the tree, the suite run, and the mutation reverted, when
the row was written. The three rows marked #318 are that change's own record
(its commit message and its verification page,
`docs/status/verification/2026-10-21-dashboard-md3e.md`), not re-run for this
page. None has been re-run for a restamp since. Re-read, not
re-run, against `f68fda09` on 2026-09-25: every file and identifier named is
still there, and between `b747e5d5` and `f68fda09` none of the files behind
these rows changed except `layout.test.tsx` (its theme case, not its nav
cases), `src/dash/resources.ts` (a helper only the deleted `more-menu.tsx`
used) and `dashboard.cy.ts` (three queries scoped to `<main>`, not the frame
case).

| Mutation                                                                     | Fails                                                                                             |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `COOKIE_ATTRIBUTES.httpOnly` → `false`                                       | `src/server/cookies.test.ts`                                                                      |
| Exchange a _fresh_ PKCE verifier rather than the one the challenge came from | `src/server/oauth.test.ts`                                                                        |
| A nav entry for a page nobody wrote                                          | `src/layout.test.tsx`, twice                                                                      |
| `pageCursors` reads `has_more` the same way in both directions               | `src/dash/provider.test.ts`, and `src/payments-query.test.ts` twice                               |
| The BFF reads the session cookie **after** the upstream read                 | `src/server/bff.test.ts`, twice — vpay is contacted for a caller with no cookie                   |
| `apiIsSameOrigin` drops its `originIsAllowed` call                           | `src/server/bff.test.ts`, twice                                                                   |
| The BFF serves the parsed upstream document instead of the named fields      | `src/server/bff.test.ts` — the **detail** case, plus a second case about the envelope's key names |
| `middleware` returns `NextResponse.next()` unconditionally                   | `middleware.test.ts`, three times                                                                 |
| `apiIsSameOrigin` drops its `Sec-Fetch-Site` comparison                      | `dashboard.cy.ts`'s frame case, in a browser                                                      |
| `AppShell` loses the rail's `onClick` (2026-10-21, #318)                     | `src/components/app-shell.test.tsx` — the href is followed instead of routed                      |
| `inAppNavigationTarget` drops its origin or modifier-key guards (#318)       | `src/client-navigation.test.ts`                                                                   |
| `readOverview`'s failure branch becomes `{ ok: true, value: 0 }` (#318)      | `src/dash/overview.test.ts`, "keeps a refused count refused — never a zero"                       |

`oauth.test.ts`'s stub echoes the challenge into the code it returns, so the
assertion is that the exchange presents the verifier whose `S256` **is** the
challenge this call sent — not merely that some verifier was present.
