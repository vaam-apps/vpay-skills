---
name: vpay-frontend
description: The vpay pnpm workspace under frontends/ — the seven packages and what each really is, the Tailwind v4 + daisyUI 5 setup whose three load-bearing lines all fail silently, the verify-ui class-string gate that trips almost every contributor, prettier's repo-root config and the one option that must stay off, and just fmt-check-web running over the working tree. Load this before touching any file under frontends/ or examples/shop, before adding a dependency, and before writing a single className.
---

# vpay frontends

> **Verified against vpay `93c6dfd0` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

Node 22.23.2 (`.nvmrc`), pnpm 9.15.0, TypeScript strict everywhere
(`tsconfig.base.json` — `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
`verbatimModuleSyntax`). Workspace globs live in `pnpm-workspace.yaml`:
`frontends/packages/*`, `frontends/apps/*`, `frontends/tests/*`, `examples/*`,
`sdks/*` — fifteen TS/JS packages, all linted.

`.npmrc` pins `node-linker=isolated` and `strict-peer-dependencies=true`, on
purpose: "undeclared imports fail loudly rather than working by accident
through hoisting". An import that works locally because pnpm happened to hoist
something is the failure mode that file exists to prevent.

## The three things that will bite you first

1. **`verify-ui` refuses a computed `className`.** `className={cn(...)}`,
   a template literal, a ternary, or a bare identifier all fail `just verify`.
   So does a class attribute over 60 characters, a raw status-colour token, and
   any daisyUI component class in an app. See `references/verify-ui.md`.
2. **Three lines in `globals.css` fail SILENTLY** if you touch them. No error,
   no warning — just a page that renders unstyled, and an axe suite that
   confidently reports it as accessible. See below.
3. **`just fmt-check-web` walks the working tree, not the index.** An untracked
   scratch `.ts`, `.md` or `.json` fails it.

## The packages

| Package               | Path                            | What it is                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------------- | ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@vpay/checkout`      | `frontends/apps/checkout`       | The payer page, hosted + embedded + popup. Next 15.5.25, React 19. **REAL.** Load `vpay-checkout`                                                                                                                                                                                                                                                                                                                       |
| `@vpay/dashboard`     | `frontends/apps/dashboard`      | Staff console. Next 15.5.25, React 19, Refine 5. **REAL.** Load `vpay-dashboard`                                                                                                                                                                                                                                                                                                                                        |
| `@vpay/tokens`        | `frontends/packages/tokens`     | `PAYMENT_STATUS`, `CHECKOUT_OUTCOME`, `statusLabel`, `checkoutOutcomeTone`. Ships raw `.ts` (`main: ./src/index.ts`), so both apps list it in `transpilePackages`                                                                                                                                                                                                                                                       |
| `@vpay/config`        | `frontends/packages/config`     | The shared ESLint flat-config factory (`./eslint` export) and `DASH_API_BASE`                                                                                                                                                                                                                                                                                                                                           |
| `@vpay/api-client`    | `frontends/packages/api-client` | **TYPES ONLY — its own header says "No request is issued yet".** Its stub functions throw `NotImplementedError`. Declared as a dependency of `@vpay/dashboard` and named in its `transpilePackages`, and **no file in either app imports it** (verified 2026-09-16). The dashboard's real formatting lives in its own `src/format.ts`. Do not build on this package; do not delete it without checking `docs/status.md` |
| `@vpay/e2e`           | `frontends/tests/e2e`           | Cypress 15. `pnpm e2e` runs the suite twice — plain, then `VPAY_E2E_FRAMED=1`                                                                                                                                                                                                                                                                                                                                           |
| `@vpay-examples/shop` | `examples/shop`                 | The demo merchant storefront. **Next 16.3.4** (deliberately newer than the two vpay apps' 15.5.25), tRPC, ZenStack. It is a _third party_ — it depends on no vpay design-system package                                                                                                                                                                                                                                 |
| ~~`@vpay/ui`~~        | ~~`frontends/packages/ui`~~     | **DELETED 2026-09-12.** Both apps now compose the published `@vaam-apps/ui`. Two permanent `verify-ui` guards stop it coming back: nothing may import it by any spelling, and nothing may reach into its old directory by path                                                                                                                                                                                          |

`frontends/Dockerfile` builds **from the repository root** and has two named
targets: `--target runner` → `vpay-dashboard`, `--target checkout` →
`vpay-checkout`. Every consumer names its target explicitly; nothing relies on
"the last stage wins".

## Tailwind v4 + daisyUI 5

**There is no `tailwind.config.ts` anywhere in the repository** — Tailwind 4 is
CSS-first. `postcss.config.js` is one line in every app:
`export default { plugins: { "@tailwindcss/postcss": {} } };`

Both vpay apps pin `tailwindcss 4.3.3`, `@tailwindcss/postcss 4.3.3`,
`daisyui 5.7.28`, `@vaam-apps/ui ^0.1.2`. Their `app/globals.css` must open in
**exactly this order**:

```css
@import "tailwindcss";
@import "@vaam-apps/ui/styles/theme.css";
@plugin "daisyui" {
  themes: false;
}
@source "../node_modules/@vaam-apps/ui/dist";
```

`examples/shop` is different and that is correct — it imports `tailwindcss`
and `@plugin "daisyui" { themes: bumblebee --default; }` with no
`@vaam-apps/ui` at all, because a merchant integrating vpay would not have
vpay's design system.

### The three lines that fail silently

**`@import` must come before `@plugin`.** CSS drops an `@import` that follows
another at-rule. The theme import sat _after_ `@plugin "daisyui"` until
2026-09-12. Tailwind's own parser is lenient, so `next build` inlined the theme
and passed, and `src/styling-gate.test.ts` — which compiles the same file
through `@tailwindcss/postcss` directly — inlined it and passed too. Any
pipeline with a spec-compliant CSS parser did not: Storybook's vite build
emitted a stylesheet with every `var(--color-base-100)` present and
`--color-base-100` defined **nowhere**. Every story rendered unstyled on the
browser's default white while the shipped page is `#0a0b0d`, and axe passed all
22 stories for **six consecutive runs** — including a deliberately unreadable
`#3a3a3a` probe. Measured delta from the reordering alone: 136 360 → 145 903
bytes. `src/a11y-gate.test.ts` in both apps now asserts the position.

**`themes: false`** stops daisyUI's built-in `dark` out-specificity-ing
`@vaam-apps/ui`'s theme, which registers under the _same_ name. Remove it and
`--color-base-100` resolves to three different values in one sheet, the last
being daisyUI's stock `oklch(25.33% 0.016 252.42)`.

**`@source "../node_modules/@vaam-apps/ui/dist"`** is what makes Tailwind scan
inside `node_modules` at all. Delete it and Tailwind generates none of the
package's utilities; measured, the compiled sheet went 201 041 → 97 446 bytes
and the marker-class count 2 → 0.

Each app carries its own `src/styling-gate.test.ts` asserting (a) and (b),
because the two apps' PostCSS pipelines are separate processes over separate
entries and share nothing. Both assert a **non-zero match count first** — a
contrast helper ported unchanged once parsed zero colours and still passed.

## Formatting and lint

`.prettierrc.json` at the repository root is the authority — config resolution
finds it from every package, so `examples/shop`'s own `prettier --check` cannot
disagree with `just fmt`. Every option in it is prettier's own default written
out, so an upgrade cannot silently reformat the repository, **except**:

> `"embeddedLanguageFormatting": "off"` — and it must stay off.

With the default `"auto"`, prettier rewrites the _contents_ of fenced code
blocks in markdown. This repository's markdown is largely transcripts. Measured
2026-09-10: it rewrote **40 fences across 22 files**, pretty-printing a one-line
`tracing` log line in `docs/runbooks/demo.md` into ten lines of output the
server does not emit — inside the runbook section telling an operator what to
grep for. A formatter that edits pasted evidence is this repository's first
failure mode with a config key.

`just fmt-check-web` is literally `pnpm exec prettier --check .` — **one
process from the root, over the whole repository, over the working tree**. Two
consequences:

- `just ci` runs it first, so on a tree where `pnpm install` has not been run
  CI dies in its first seconds rather than several minutes in at `lint-web`.
- An untracked scratch file fails it. `.gitignore` it or delete it. Do not
  reach for `--write` on a tree you have not looked at.

`just lint-web` is `build-sdk-node` → `pnpm -r typecheck` → `pnpm -r lint`. It
depends on the SDK build because `sdks/stripe-compat` imports
`@vaam-apps/vpay-sdk/stripe`, whose types resolve to gitignored `dist/`.
`frontends/packages/config/src/eslint.test.js` asserts the lint gate is still a
gate — every package still declares `eslint . --max-warnings 0` and still
reaches the shared factory. Every assertion in it was written against a
mutation measured to leave `pnpm -r lint` at exit 0.

## More

- `references/verify-ui.md` — every numbered check, its pathspec, its exact
  refusal, and the list of ways contributors trip it.
- `references/workspace.md` — per-package scripts, the Storybook wiring
  gotchas, `pnpm.overrides`, and which gate runs where.
