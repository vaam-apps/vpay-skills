# Testing the checkout, and the gates that guard the gates

_Verified against vpay `7a79684e` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

The theme through this whole page: **every one of these checks has, at some
point, gone green while measuring nothing.** Each fix is a test that asserts the
_gate_ is still a gate. When you add a check here, ask what a vacuous version
of it would report.

## The suites

| Command                                       | What it is                                                                                                                                                                                                                                   |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm --filter @vpay/checkout test`           | The jsdom/node suite. `environment: "node"` by default so tests talking to `src/testing/browser-stub.ts` use the platform `fetch`; rendering tests opt in with a `// @vitest-environment jsdom` docblock. Run by `just test-web` → `just ci` |
| `pnpm --filter @vpay/checkout test-storybook` | 22 stories in a real headless Chromium with axe over each. **Not** in `just ci` — it needs a ~115 MB Playwright Chromium and `just ci` must pass offline. CI's `web` job runs it                                                             |
| `just build-storybook`                        | Builds both apps' Storybooks **and asserts the built stylesheet defines `--color-base-100`**                                                                                                                                                 |
| `just test-e2e`                               | Cypress against the real compose stack                                                                                                                                                                                                       |

`vitest.config.ts` sets `esbuild.jsx: "automatic"` for the jsdom suite because
the app's `tsconfig.json` says `"jsx": "preserve"` for Next.

## `src/a11y-gate.test.ts` — the gate on the gate

A plain jsdom test, in the suite `just ci` does run, guarding a browser suite
`just ci` cannot run. Six cases:

1. `.storybook/preview.ts` still sets `a11y.test: "error"` and not
   `"off"`/`"todo"`.
2. `preview.ts` still paints the document shell the real page paints — it must
   contain `bg-base-100`, `data-theme`, and `import "../app/globals.css"`.
3. `.storybook/main.ts` still loads `@storybook/addon-a11y` and
   `@storybook/addon-vitest`.
4. **`globals.css` imports the theme BEFORE any other at-rule.**
5. No story switches the addon off and nothing disables an axe rule. **This app
   pins the set at zero** — the expected arrays are empty, so the first
   suppression added fails `just ci` and has to be argued for in review.
6. The **built** Storybook stylesheet actually defines `--color-base-100`.

### Case 2 is not cosmetic

`bg-base-100` is what **paints** the background; the theme only defines the
variable. Without it a story renders on the browser's default white while the
real page is `#0a0b0d` — and axe then measures every foreground against the
wrong ground and returns a verdict that is confidently wrong. Measured: the
first run of this suite passed 22 stories against white, and a deliberately
dark-grey `#3a3a3a` probe (unreadable on this theme) passed with them.

`THEME` is imported from `src/config/theme.ts` rather than spelled in
`preview.ts`, so the story shell cannot drift from what the layout sets.

### Case 4 is the theme-ordering defect, as a position check

`app/globals.css` must read `@import "tailwindcss"` → `@import
"@vaam-apps/ui/styles/theme.css"` → `@plugin "daisyui" { themes: false }` →
`@source "../node_modules/@vaam-apps/ui/dist"`.

The theme `@import` sat **after** `@plugin "daisyui"` until 2026-09-12. CSS
drops an `@import` that follows another at-rule. Tailwind's own parser is
lenient, so `next build` and `src/styling-gate.test.ts` — which compiles the
same file through `@tailwindcss/postcss` directly — both inlined the theme and
both **passed**. Storybook's vite build did not: it emitted a stylesheet with
every `var(--color-base-100)` present and `--color-base-100` defined nowhere.
Every component rendered unstyled on the browser's default white, axe measured
every foreground against the wrong ground, and the suite reported **22 passing
stories for six consecutive runs** — with the `#3a3a3a` probe passing
alongside them.

> A green a11y run against the wrong background is worse than no run: it is a
> claim nobody will re-check.

The test strips comments before checking, because this file's own header
explains the defect in prose and names `@plugin` — the first draft failed on
its own documentation.

### Case 6 — `ctx.skip()`, not a bare `return`

The case reads `storybook-static/assets/*.css`. When that directory does not
exist it calls **`ctx.skip()`**. A bare `return` reported the case as **PASSED
with nothing measured** — and CI is exactly where that happens: the `web` job
runs `pnpm -r test` **before** `just build-storybook`, so `storybook-static/`
never exists when this file runs there. The one check standing behind this
app's whole styled-or-not claim was green-and-blind on every CI run. Found in
the dashboard's copy of this file and fixed in both, 2026-09-13.

Its failure message also names the _other_ cause: a stale `storybook-static/`.
A `just build-storybook` that fails its own check leaves the theme-less
artefact on disk on purpose, so it can be inspected — and every later run of
this test then reads that instead of the source. Re-run
`just build-storybook`: if it exits 0, the tree was fine and the artefact was
stale.

## `src/styling-gate.test.ts` — what `just ci` cannot build

`just ci` never builds this app. This test compiles `app/globals.css` through
`postcss([tailwind()])` in Node, which is possible because Tailwind v4's
`@source` scan runs under a plain `.process(css, { from, to })` call.

Two cases, each with a hand-run, reverted mutation:

- **`@source` reached `@vaam-apps/ui/dist`** — a utility only the package's
  compiled `dist` writes must be present. Deleting the `@source` line took the
  count 2 → 0 and the file 201 041 → 97 446 bytes.
- **Every `--color-base-100` the build emits is one the package declares**.
  Removing `themes: false` produced **three** values — `#0a0b0d`,
  `oklch(100% 0 0)` and daisyUI's stock dark `oklch(25.33% 0.016 252.42)`, the
  last at higher specificity.

Both assert a **non-zero match count first**. The discipline exists because a
contrast helper ported unchanged once parsed zero colours and still passed.

Note the split of responsibilities: `styling-gate.test.ts` uses the _lenient_
parser, so it **cannot** catch the `@import` ordering defect and did not.
`a11y-gate.test.ts` reads the _built_ Storybook's stylesheet instead.

## The 22 stories

`src/components/checkout-screens.stories.tsx`. Seventeen checkout states, two
of them repeated in English, and three return-page screens. The states come
from `src/testing/screen-states.ts` — **the same literals
`checkout-view.test.tsx` asserts against** — so a screen a designer reviews is a
screen a test covers, and a screen added to one without the other fails the
"covers each state the machine can be in" assertion.

This is the **only** thing in the repository that can answer `color-contrast`
for these screens. The jsdom axe suites next door (`screens.axe.test.tsx`,
`outcome-contrast.test.ts`) compute no colour and answer `color-contrast`
"incomplete". Issue #73's Cypress attempt could not get a verdict either:
daisyUI's `:root` scroll-lock rule carries an unconditional `background-image`
and axe-core abandons the rule under any such ancestor. A story renders inside
`#storybook-root` with no such ancestor, which is why this route works where
Cypress could not.

The stories were deleted along with `@vpay/ui` and **restored verbatim** on
2026-09-12 — the props of `CheckoutView` and `ReturnView` did not change in the
cutover, so the file is the one that was deleted rather than a reconstruction.

## Storybook wiring gotchas

All four are in `vpay-frontend`'s `references/workspace.md` in full, because
both apps share them. The short list, each a measured false green:

- `esbuild.jsx` belongs in `.storybook/main.ts`'s `viteFinal`, **not** in
  `vitest.storybook.config.ts` — one setting, read by both `build-storybook`
  and `addon-vitest` through `configDir`.
- `test.css: true` in `vitest.storybook.config.ts`, or every story renders
  unstyled and passes.
- Declare every dependency the stories reach in `optimizeDeps.include`, or a
  cold dep cache gives a `null` React and vitest reports every story passing
  beside a count of unhandled errors.
- **No** `setupFiles` calling `setProjectAnnotations` — the manual call
  suppresses the automatic one, measured reporting 93 passing stories while
  checking none.
- `main.ts` aliases `@vaam-apps/ui/styles/theme.css` by hand, computed from the
  package's own `exports`, because vite drops the `exports`-mapped specifier
  silently.

## End to end

`frontends/tests/e2e/cypress/e2e/` — `checkout.cy.ts`, `shop-hosted.cy.ts`,
`shop-embedded.cy.ts`, `dashboard.cy.ts`. `pnpm --filter @vpay/e2e e2e` runs
the suite twice: plain, then with `VPAY_E2E_FRAMED=1`. `just e2e-specs` is what
supplies every URL and port; the staff password is passed as a **path to a
file** read in Node, so it never reaches `Cypress.env`, a browser, or a
`cypress run` argument list.

Two Node-side fixture servers are started by `cypress.config.ts` itself:
`checkoutBrowserServer.ts` and `frameFixtureServer.ts`.

## What is still not proven

- No Cypress case has ever opened a **real popup** — every window in
  `sdks/stripe-js/src/popup.test.ts` is a stub, because jsdom implements
  neither `window.open` nor cross-window `postMessage`. `examples/shop`'s popup
  mode was exercised by hand. This is a dated ⛔ in `docs/sdks/parity.md`.
- The `@vaam-apps/vpay-stripe-js` package has never run against a live stack;
  its server is always `src/testing/browser-stub.ts`. The `/v1/browser` routes
  themselves are proven server-side by
  `backends/tests/integration/tests/browser_checkout.rs`.
- The rail behind every one of these runs is a WireMock host. See
  `docs/status.md`'s banner.
