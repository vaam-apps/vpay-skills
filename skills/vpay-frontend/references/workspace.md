# The workspace, package by package

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Everything below verified 2026-09-16 against `frontends/`, `examples/shop` and
the root `package.json` / `justfile`.

## Scripts per package

| Package                                            | Scripts worth knowing                                                                                                                                                                                             |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@vpay/checkout`                                   | `deps` (builds `@vaam-apps/vpay-stripe-js` first — `dev`, `build`, `typecheck`, `test` and `lint` all chain it), `dev -p 3001`, `storybook -p 6006`, `build-storybook`, `test-storybook` (separate vitest config) |
| `@vpay/dashboard`                                  | same shape, `dev -p 3000`, `storybook -p 6007`. `test` is `vitest run --passWithNoTests`                                                                                                                          |
| `@vpay/tokens`, `@vpay/config`, `@vpay/api-client` | `typecheck` / `test` / `lint`. `build` where present is `tsc --noEmit` — these ship raw source                                                                                                                    |
| `@vpay/e2e`                                        | `e2e` = `e2e:default` then `e2e:framed` (`VPAY_E2E_FRAMED=1`). `deps` builds `@vaam-apps/vpay-sdk`                                                                                                                |
| `@vpay-examples/shop`                              | `generate` (= `DO_NOT_TRACK=1 zen generate`, also `postinstall`), `build:deps` (builds **both** SDKs), `lint` runs eslint **and** `prettier --check` over its own tree                                            |

Root `package.json`: `pnpm -r build|lint|test|typecheck`, `test:e2e` →
`@vpay/e2e`, `storybook`/`test-storybook` → `@vpay/checkout` only.

## Which gate runs where

| Gate                                                                       | Command                            | In `just ci`?                                                                |
| -------------------------------------------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------- |
| Prettier, repo-wide, working tree                                          | `just fmt-check-web`               | yes, **first**                                                               |
| Typecheck + ESLint, 15 packages / 214 files                                | `just lint-web`                    | yes                                                                          |
| jsdom/node suites (`a11y-gate`, `styling-gate`, `layout.test`, `bff.test`) | `just test-web` (= `pnpm -r test`) | yes                                                                          |
| Class-string rules                                                         | `just verify-ui`                   | yes, via `just verify`                                                       |
| Storybook build **and** the theme-defined assertion                        | `just build-storybook`             | **no** — CI `web` job                                                        |
| Storybook + axe in real Chromium                                           | `just test-storybook`              | **no** — needs a ~115 MB Playwright Chromium and `just ci` must pass offline |
| Cypress against the compose stack                                          | `just test-e2e` / `just e2e-specs` | no — CI `e2e (compose)`                                                      |

CI's `web` job calls **these recipes**, not copies of their commands, so the
gate and the local check cannot drift.

## `just build-storybook` carries an assertion, not just a build

It builds both apps' Storybooks and then, for each, counts occurrences of
`--color-base-100:` and of `var(--color-base-100` in the emitted stylesheet and
fails when the first is zero. `grep -o | wc -l` counts occurrences; `grep -c`
counts matching _lines_, and a minified stylesheet is one line — it would
answer 1 whether the variable were defined once or thirty times.

This lives in the recipe rather than only in the test because CI's `web` job
runs `pnpm -r test` **before** `just build-storybook`, so `storybook-static/`
never exists when the test runs there. It matters most for the dashboard, whose
`.storybook/main.ts` deliberately carries **no** theme-resolve alias — measured
unnecessary on today's dependency tree, and this is what would catch a
dependency bump making it necessary again.

## Storybook wiring gotchas (both apps)

Each was a measured false green. None of them fails loudly.

**Set `esbuild.jsx` in `.storybook/main.ts`'s `viteFinal`, never in
`vitest.storybook.config.ts`.** Both apps' `tsconfig.json` say
`"jsx": "preserve"` because Next requires it, and Vite's esbuild reads the
nearest tsconfig — so it emits classic `React.createElement` for stories that
import no React, and every one dies with `ReferenceError: React is not
defined`. `main.ts` is read by `build-storybook` _and_ by `addon-vitest`
through `configDir`, so one setting covers both. It was duplicated in the
vitest config for one revision: the vitest suite passed all 22 stories while
`just build-storybook` produced a Storybook where every story rendered the red
"React is not defined" panel. The build exits 0 either way, because compiling
is not rendering.

**`test.css: true` in `vitest.storybook.config.ts`.** Vitest stubs CSS imports
by default, so `preview.ts`'s `import "../app/globals.css"` was a no-op:
measured, `--color-base-100` resolved to the empty string and `document.body`'s
background came back `rgba(0, 0, 0, 0)`. The suite passed 22 stories that way,
with an unreadable `#3a3a3a` probe alongside them. `build-storybook` does not
share this failure, so a green build says nothing about whether the vitest run
is styled — which is why `a11y-gate.test.ts` asserts the setting rather than
trusting it.

**Declare every dependency the stories reach in `optimizeDeps.include`.**
Without it vite discovers a package's entries mid-run, re-bundles, and modules
loaded before the re-bundle keep a `null` React. Stories then die with
`Cannot read properties of null (reading 'useContext')` and **vitest reports
every story PASSING** alongside a separate count of unhandled errors — a story
that throws while rendering still counts as a passing test, and axe never ran
on it. Measured: 24 such errors on a CI runner against 0 locally, because it
reproduces from a **cold** dep cache only. `just test-storybook` clears the
cache before every run so a local green means what a runner's green means.
`resolve.dedupe` does not fix it; `optimizeDeps.exclude` makes it worse.

**No `setupFiles` calling `setProjectAnnotations`.** `@storybook/addon-vitest`
has applied `.storybook/preview.ts` itself since Storybook 10.3 and the manual
call **suppresses** the automatic one — measured on the predecessor of this
config, which reported 93 passing stories while checking none of them.

**The checkout's `.storybook/main.ts` aliases
`@vaam-apps/ui/styles/theme.css` by hand**, computed from the package's own
`exports` via `createRequire(...).resolve` rather than written as a path.
`@tailwindcss/postcss` honours the `exports` map; Storybook's vite pipeline
resolves CSS `@import` itself and silently emitted a stylesheet with no theme
in it at all.

**The dashboard's `.storybook/preview.ts` writes `@vaam-apps/ui`'s own
storage key and dispatches a synthetic `StorageEvent`** before the decorator
runs, because `ThemeSwitcher` overwrites `data-theme` on mount from its own
store. Three stories rendered in whatever theme the _runner's_
`prefers-color-scheme` said until this landed.

## `pnpm.overrides` — read the prose first

The root `package.json` carries a `"//pnpm"` array of prose explaining every
override, because JSON has no comments. Each entry is **dated**, names the
advisory, and states the condition under which it should be removed — "an
override that outlives its advisory is a pin nobody meant to keep".

Live as of 2026-09-16: `next>postcss`, `@prisma/config>deepmerge-ts`, and three
scoped `*>lodash` entries under `chevrotain`. The prose array also keeps the
**removed** entries with the measurement that justified removing them
(`@storybook/addon-actions>uuid`, `vite`, and `packageExtensions['daisyui@4']`).

If you add an override, add a dated paragraph. If you remove one, leave the
paragraph and append what you measured. `just audit-web` is the gate these keep
green.

## `examples/shop` is a third party

It is in the workspace but it is **not** a vpay frontend. It depends on
`@vpay/config` (dev only) and on the two published SDKs, and on **no** vpay
design-system package — by design, so that what it demonstrates is reproducible
by a merchant. It pins Next 16.3.4 while the two vpay apps are on 15.5.25; that
difference is real and intentional, and the dashboard's BFF review explicitly
notes it read Next 16.3.4's behaviour while the dashboard resolves 15.5.25.

Its persistence is ZenStack over Postgres: `examples/shop/zenstack/schema.zmodel`
plus migrations under `examples/shop/zenstack/migrations/`. Regenerate with
`pnpm --filter @vpay-examples/shop generate`.

Three rules the shop exists to demonstrate, from its own README: the **server**
prices the order (no price on the wire); the `Idempotency-Key` is **derived
from the order id** (`shop-order-{id}-intent`, `-session-hosted`,
`-session-embedded`), never random; and **only the signature-verified webhook
marks an order paid** — the return page displays the `session_id` vpay
substituted into `success_url` and takes no decision from it.
