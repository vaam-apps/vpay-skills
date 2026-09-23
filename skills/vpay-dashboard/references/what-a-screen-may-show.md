# What a dashboard screen may and may not show

_Verified against vpay `b747e5d5` (2026-09-23). Version-sensitive claims
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

`data-theme` is `dark` and `src/layout.test.tsx` pins it. ~~`@vaam-apps/ui`
registers its one theme under daisyUI's built-in name `dark`, so a `data-theme`
that says anything else renders the page **completely unthemed** in a real
browser, with no error anywhere.~~ **Corrected 2026-09-23, in two parts.**

1. **There are two themes, not one.** `dark` is the default and `light` is
   opt-in, and that has been true since `0.1.2`, the version this page was
   written against (`app/layout.tsx`). `ThemeSwitcher` writes the attribute,
   and the Storybook runs 21 stories `dark` and 4 `light`
   (`docs/flows/dashboard/status-built-and-not-built.md`, 2026-09-13).
2. **Since `^0.2.4` (vpay#249, 2026-09-23) the package also declares its
   dark tokens on `:root`.** So an unregistered `data-theme` should no longer
   render unthemed. This comes from reading the package's
   `dist/styles/theme.css`. Nobody has measured it in a browser.

`layout.test.tsx`'s case is still titled "the only theme @vaam-apps/ui
actually compiles", which is stale. Pin `dark` because it is what the app
ships, not because nothing else would render.

## Which components come from `@vaam-apps/ui`

`@vaam-apps/ui` is `^0.2.4` since 2026-09-23 (vpay#249); it was `^0.1.2`
before. `0.2.0` shipped visual breaking changes with no API change, among them
the `SideNav` width below `xl` and the dark theme applying without a
`data-theme`. So a green typecheck does not show that a screen still looks
right.

Imported by the app as of 2026-09-23: `Table` (and `TableHeader`/`TableBody`/
`TableRow`/`TableHead`/`TableCell`), `FormField` (was `Field`), `Input`,
`Button`, `InlineBanner` (was `Alert`), `InlineEmptyState`, `Code`,
`ScreenStack`, `SideNav`, `MoreDetailDrawer`/`DrawerClose`, `ThemeSwitcher`,
`DetailList`/`DetailRow`, `DateRangePicker`, `InstrumentPanel`,
`RouteSkeleton`, `createStatusPill`/`defineStatusSystem`. _(Until 2026-09-23
this list also named `Pagination`, `Timeline` and `DataList`/`DataListRow`.
The app imported none of the three, at `d3a8810b` or since. The timeline and
the paging links are the app's own markup.)_

Two things that changed shape rather than name: `InlineBanner` renders **no
`role`** where `Alert` defaulted to `role="alert"`, so every site relying on
that default now wraps it and names the role explicitly. And
`payments-filters.tsx` uses a **native `<select>`** — `@vaam-apps/ui`'s
`Select` cannot be named by a `<label>`.

`ProcedureTable` (`src/components/procedure-table.tsx`) is the app's own shell
shared by the four procedure lists, because they differ only in their columns.
Its `secondary` flag hides a column below `sm` as a **static class per column,
never a computed one** — `verify-ui` refuses a computed `className` anywhere in
an app. `Table` already wraps itself in `w-full overflow-x-auto`, so a wide
table _can_ be swiped on a phone; the flag exists because an operator who does
not know to swipe reads the first two columns as the whole answer.

## What this app still cannot do

As of 2026-09-23 (this said 2026-09-16; the list is unchanged except the last
entry):

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
- **Show a refund's events on the payment timeline.** This was read from the
  code on 2026-09-23 and is untested. Refund events are keyed on the `re_…`
  id, and the detail read fetches events for the intent and the charge only.
  See the `SKILL.md` note.

## Testing notes

`vitest.config.ts` sets `globals: false`, so Testing Library does **not**
register its own cleanup — `vitest.setup.ts` calls `afterEach(cleanup)`
explicitly. Without it every render in a file accumulates in one
`document.body` and `getByRole` finds the previous test's copy of a control.

`src/a11y-gate.test.ts` is the twin of the checkout's, with one difference:
this app **does** carry axe-rule suppressions, and the test pins exactly which
stories may carry one and which rule id. They were first written as
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
line. **It has not been reported upstream.**

**Re-checked against `0.2.4` on 2026-09-23 by reading the code, not by
measuring.** The dependency moved to `^0.2.4`, and the suppression in
`src/a11y-gate.test.ts` still names `@vaam-apps/ui@0.1.2`. In
`side-nav.js`, the in-flow `<nav>`'s non-collapsed floating class string
(`xl:flex …`, with no unprefixed `hidden`) is byte-identical in `0.1.2` and
`0.2.4`, so the defect has probably not been fixed. `0.2.4` also renders
**three** `<nav aria-label="Primary">` shapes and relies on their breakpoint
gates being complements. Do not drop the suppression without re-running axe at
1200×900.

The dashboard's Storybook has 25 stories (counted 2026-09-23) bound to the same
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
