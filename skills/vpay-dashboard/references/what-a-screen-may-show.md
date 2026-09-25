# What a dashboard screen may and may not show

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
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
`charges.payer_ref_masked` is never written by anything (`docs/status.md`), so
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

| Situation                       | The honest shape                                                                                                                                                                                      |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A column nothing writes         | Render it from the column, show `ABSENT` (`—`), never derive a substitute                                                                                                                             |
| A read that failed              | Return the refusal to the screen. `readProcedurePage` returns the refusal rather than an empty page: an empty list and a refused read look identical on screen unless the screen is told which it got |
| A status this build cannot name | Render it as **text**, not as a coloured pill — "a green badge on an unfamiliar value is a claim"                                                                                                     |
| A value the wire cannot parse   | Return it verbatim rather than replacing it (`formatIsoInstant`)                                                                                                                                      |
| No `total` from the API         | No page count, no page numbers. Two cursors and nothing else                                                                                                                                          |
| A field too large for a list    | Omit it and say so. `/refunds` does not show `failure_raw` — up to 2 000 characters of a rail's own prose, belonging to a detail read that does not exist                                             |
| A slice nobody built            | No nav entry. `layout.test.tsx` fails if one appears                                                                                                                                                  |

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

`data-theme` is `dark` and `src/layout.test.tsx` pins it: `@vaam-apps/ui`
registers its one theme under daisyUI's built-in name `dark`, so a `data-theme`
that says anything else renders the page **completely unthemed** in a real
browser, with no error anywhere.

## Which components come from `@vaam-apps/ui`

~~`Table`, `FormField` (was `Field`), `Input`, `Button`, `InlineBanner` (was
`Alert`), `InlineEmptyState`, `Code`, `ScreenStack`, `SideNav`,
`MoreDetailDrawer`, `ThemeSwitcher`, `Pagination`, `Timeline`,
`DataList`/`DataListRow`, `createStatusPill`/`defineStatusSystem`.~~
**Corrected 2026-09-24** (read off the imports under `frontends/apps/dashboard/`
at vaam-apps/vpay#258's head, `@vaam-apps/ui` 0.4.0; not on `master` before
that merge): `Table` and its parts, `FormField` (was `Field`), `Input`,
`Button`, `InlineBanner` (was `Alert`), `InlineEmptyState`, `Code`,
`ScreenStack`, `SideNav`, `Skeleton`, `DetailList`/`DetailRow`,
`InstrumentPanel`, `DateRangePicker` and its parts (with `IsoDateRange`),
`ThemeSwitcher`, `createStatusPill`/`defineStatusSystem`. The dashboard
imports no `Pagination`, `Timeline`, `DataList`/`DataListRow`,
`RouteSkeleton`, `MoreDetailDrawer` or `DrawerClose`. The last three went with
vaam-apps/vpay#258: on `master` before it, both payments screens rendered
`RouteSkeleton`, and the Menu pill's drawer was a `MoreDetailDrawer`.

Two things that changed shape rather than name: `InlineBanner` renders **no
`role`** where `Alert` defaulted to `role="alert"`, so every site relying on
that default now wraps it and names the role explicitly. And
`payments-filters.tsx` uses a **native `<select>`** — ~~`@vaam-apps/ui`'s
`Select` cannot be named by a `<label>`.~~ **Corrected 2026-09-24:** it can.
`SelectTrigger` takes an `id` (on `0.2.4`, `0.3.0` and `0.4.0`), so a
`FormField` label names it. The native `<select>` stays for the reason the file
itself gives: the
package's `Select` wraps a Headless UI `Listbox` and forwards no `name`, and no
ref a plain `onChange` can read.

**The date filter, since vaam-apps/vpay#258** (`@vaam-apps/ui` 0.3.0's picker,
unchanged in the 0.4.0 that merge ships; not on `master` before it).
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
picker's first click was a one-day range. While the picker is open the page is
inert, so the first press on Apply only closes it.

The trigger has a fixed width, `w-[calc(23ch+16px+3.5rem)]`, sized to the
longest value `YYYY-MM-DD → YYYY-MM-DD`. The `16px` and `3.5rem` are kept apart
on purpose. The rem chrome grows with the browser's default font size and the
14px value does not, so a single pixel figure cut the range short at Chrome's
"Large" setting. `font-mono text-prose` on it only sets the font that `ch` is
measured in; keep it. `max-w-full` on the trigger and on both filters'
`FormField`s caps them at the row, so below a 314px window the field truncates
rather than push the page sideways. The Status `<select>` is `h-10`, so the two
labels and controls line up. The row is `flex-wrap` at every width and never
scrolls sideways: one line wherever it fits, more lines where it doesn't.
Don't add a breakpoint or `overflow-x-auto` to it. The `FiltersWithRange`
(1280×800) and `FiltersWithRangePhone` (375×812) stories measure all of this in
a real browser, including at a 20px root.

**Floating chrome, since vaam-apps/vpay#258** (`@vaam-apps/ui` 0.4.0). Below
1280px `SideNav`'s floating toolbar ends in a **More** control that opens a
modal sheet: a bottom sheet from the phone bar below 640px (288px wide with the
dashboard's five destinations, as of 2026-09-24), a drawer from the left edge
from 640 to 1279px. The phone bar's sheet holds the destinations the bar has no
room for (Customers); the drawer lists every destination, labelled. Both end
with the account block, `SignedInBar` then a "Theme" caption and
`ThemeSwitcher`, which `app-shell.tsx` passes as `accountSlot` at every width.
There is no Menu pill any more (on `master` before this merge there was one, at
`bottom-3 left-3`), so don't add your own `fixed` account or menu button.
Anything else pinned near the bottom must clear the bar's top edge, 80px up
(its 16px offset plus 64px): `bottom-24` sits 16px above it. `main` carries
`pb-20` below 640px, `sm:pl-24` from 640px and `xl:pl-0` from 1280px, where the
sidebar is in flow.

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

**Loading states, since vaam-apps/vpay#258.** Both payments screens load in
their own shape: `PaymentsSkeleton` (`payments-skeleton.tsx`) for the list and
`PaymentSkeleton` (`payment-skeleton.tsx`) for one payment. On `master` before
that merge both rendered `RouteSkeleton`, whose header, filter bar and rows
matched neither page, so everything moved when the read returned. Headings are
one-line `h-lh` placeholders. Text that wraps (the list's description, the "At
a glance" caption) and values whose width decides where a row wraps (the filter
row's select and Apply, a four-digit XAF amount, a status pill) are invisible
copies of the real thing, so they wrap where the page does. The
`PaymentsLoading`, `PaymentsLoadingPhone`, `PaymentLoading` and
`PaymentLoadingPhone` stories compare the claimed boxes within 1px at 1280 and
375px, in the shell's narrower columns and at a 20px root, and
`screens.test.tsx` fails if either loading branch renders `RouteSkeleton` again.
So a change to a page's header, description, filter row or panel changes its
skeleton in the same commit. Neither skeleton renders an `<h2>`, and the
payment's renders no `data-testid="detail-id"`: `dashboard.cy.ts` waits on
those as the sign the page has loaded. The rows below the filter row or the
Summary heading are data, and nothing claims they line up.

`ProcedureTable` (`src/components/procedure-table.tsx`) is the app's own shell
shared by the four procedure lists, because they differ only in their columns.
Its `secondary` flag hides a column below `sm` as a **static class per column,
never a computed one** — `verify-ui` refuses a computed `className` anywhere in
an app. `Table` already wraps itself in `w-full overflow-x-auto`, so a wide
table _can_ be swiped on a phone; the flag exists because an operator who does
not know to swipe reads the first two columns as the whole answer.

## What this app still cannot do

As of 2026-09-16:

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
  detail call it, and only for reads after their first render (since 2026-09-12;
  this line said "No page and no component calls it" until 2026-09-23).

## Testing notes

`vitest.config.ts` sets `globals: false`, so Testing Library does **not**
register its own cleanup — `vitest.setup.ts` calls `afterEach(cleanup)`
explicitly. Without it every render in a file accumulates in one
`document.body` and `getByRole` finds the previous test's copy of a control.

`src/a11y-gate.test.ts` is the twin of the checkout's. ~~With one difference:
this app **does** carry axe-rule suppressions, and the test pins exactly which
stories may carry one and which rule id.~~ **Corrected for vaam-apps/vpay#258**
(`@vaam-apps/ui` 0.4.0): it pins both lists as **empty**. No story may disable
an axe rule or carry a `parameters` override at all, so a story's viewport or
theme goes in story-level `globals`, and a reviewed exception has to be written
into the test's two lists. Until 0.4.0 the `Shell` stories disabled
`landmark-unique`, the rule below. They were first written as
`parameters.a11y = { test: "todo" }`, which switches the addon off for the
**whole story** — measured, a `#3a3a3a`-on-`#0a0b0d` probe placed inside
`Shell` PASSED under it, so the chrome around every screen in the app had no
colour-contrast verdict at all. With the suppression narrowed to one rule id
the same probe fails at 1.73.

That suppression covers a **real third-party defect**, not this app's markup:
at 1200×900 (below `xl`), `@vaam-apps/ui@0.1.2`'s `SideNav` renders its in-flow
sidebar `<nav aria-label="Primary">` with only `xl:`-prefixed utilities and no
unprefixed `hidden`, so it falls back to `display: block` while the 1024–1279px
floating rail is also visible — two landmarks answering the same name.
Reproduced directly with axe-core at four viewports and traced to the upstream
line. ~~**It has not been reported upstream.**~~ **Corrected 2026-09-24:** it
had been, as [vaam-apps/ui#16](https://github.com/vaam-apps/ui/issues/16)
(2026-09-13). The vaam-apps/vpay#258 review re-measured it on `@vaam-apps/ui`
`0.3.0`: one violation at 375–1279px, none from 1280px. **Fixed in 0.4.0**
(vaam-apps/ui#39, 2026-09-24): the in-flow `<nav>` is hidden below 1280px, so
one `Primary` landmark is exposed at every width.
vaam-apps/vpay#258 dropped the suppression, and the `ShellPhone`, `ShellRail`,
`ShellLaptop` and `ShellDesktop` stories (375, 700, 1100 and 1280px) assert
exactly one exposed `Primary` landmark.

The dashboard's Storybook has 33 stories as of vaam-apps/vpay#258 (2026-09-24;
25 when it was added on 2026-09-13), bound to the same
`src/testing/fixtures.ts` `a11y.test.tsx` renders, so a screen cannot gain a
story without a test already covering it.

## The decisive mutations

Each was applied to the tree, the suite run, and the mutation reverted:

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

`oauth.test.ts`'s stub echoes the challenge into the code it returns, so the
assertion is that the exchange presents the verifier whose `S256` **is** the
challenge this call sent — not merely that some verifier was present.
