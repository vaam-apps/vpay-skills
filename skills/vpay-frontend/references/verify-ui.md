# `just verify-ui` — the class-string gate

`justfile`'s `verify-ui` recipe. It is part of `just verify`, which is part of
`just ci`. It is the only gate on `docs/status/gates.md` that is a `just`
recipe of `git grep`s rather than a `cargo xtask` — deliberately: "a `git grep`
is the honest tool here, which is more ceremony than a dozen greps deserve".

Every check carries a recorded decisive mutation: add the offending line,
confirm the recipe exits non-zero, remove it. A check nobody has mutated is a
claim, not a check.

Facts below are as of 2026-09-16. Where the recipe's own comment and this page
disagree, **the recipe is right** — that instruction is in the recipe's header.

## The checks

| # | What it refuses | Pathspec | Refusal message |
| --- | --- | --- | --- |
| 1 | A raw Tailwind **palette colour**: `(bg\|text\|border\|ring\|fill\|stroke\|from\|via\|to\|decoration\|outline\|shadow\|accent\|caret\|divide\|placeholder)-(<hue>-NNN\|black\|white)(/NN)?` | `frontends/apps`, `examples/shop` | a palette colour outside a theme token — use a daisyUI theme token |
| 1b | An **arbitrary colour value**: the same prefixes followed by `-[#`, `-[rgb`, `-[hsl`, `-[oklch`, `-[color-mix` | same | a hard-coded colour value — use a daisyUI theme token |
| 2 | **daisyUI 4 classes daisyUI 5 removed**: `form-control`, `label-text`, `label-text-alt`, `btn-group`, `input-group`, `card-compact`, `input-bordered`, `select-bordered`, `textarea-bordered`, `tabs-bordered`, `tabs-lifted`, `tabs-boxed` | `frontends`, `examples`, minus `docs` | a daisyUI 4 class removed in daisyUI 5 |
| 3 | `!important`, with three named exemptions | `frontends`, `examples` | !important outside the documented exemptions |
| 4 | `cva(` — one variant map, not one per app | `frontends/apps`, `examples` | a cva variant map in an app |
| 5 | Any **import of the deleted `@vpay/ui`**, four spellings | `frontends`, `examples`, `sdks`, minus `*.md` | @vpay/ui was deleted on 2026-09-12 — nothing may import it |
| 5b | Any **filesystem path into `frontends/packages/ui`** | `frontends`, `examples`, `sdks`, `justfile`, `.github`, minus `*.md` | a path into the deleted frontends/packages/ui |
| 7a-i | A **computed `className`** — `className=` followed by `{` | `frontends/apps`, minus tests and stories | a computed className in an app — a class string assembled at runtime is a variant map; put the variants in a component |
| 7a-ii | A raw **status-colour theme token** — `…-state-<hue>-(fg\|bg\|border)` or `…-destructive(-foreground)?` | `frontends/apps`, minus tests and stories | a status colour written in an app — use a status system or an InlineBanner variant |
| 7a-iii | A **class attribute over 60 characters** | `frontends/apps`, minus tests and stories | a className over 60 characters in an app — compose a @vaam-apps/ui component instead of a longer string |
| 7b | A **daisyUI component class** in an app, in any quoting | `frontends/apps` | a daisyUI component class in an app — compose a @vaam-apps/ui primitive instead |
| 7b-ii | `table` / `select` / `mask` when they **are** the daisyUI component | `frontends/apps`, minus one file | same message |

### Check 1 — why the `className=` prefix is gone

The first version required `className=` earlier on the **same line**, so a
class held in a lookup object was invisible. That is not hypothetical:
`TONE_CLASS` in the checkout's `screens.tsx` was exactly that shape. The
utility name is now the whole signal. Two other measured holes were closed at
the same time: `black`/`white` were missing from the palette list (so
`bg-black/40` passed, and one was live in the old `Drawer`), and an arbitrary
value was not matched at all (`text-[#ff0000]` passed).

A daisyUI **theme** token — `bg-base-100`, `text-error` — is not a palette
colour and never matches. That is what 7a-ii exists for.

### Check 2 — these do not error, they stop styling

A removed daisyUI 4 class is not a build failure. It silently styles nothing.
The list is the subset this repository was measured to actually use, **not the
complete daisyUI 4→5 delta** — assume it is incomplete.

`input-bordered`, `select-bordered` and `textarea-bordered` were added
2026-09-12, found by dogfood rather than inspection: ESLint's
`better-tailwindcss/no-unknown-classes` reported `select-bordered` the moment
it compiled the real `@vaam-apps/ui` theme. Reading `daisyui@5.7.28` directly
confirmed it — **no `-bordered` class exists** in `select.css`, `input.css` or
`textarea.css`; "bordered" became each form control's default appearance.

### Check 3 — the three `!important` exemptions

- `frontends/apps/checkout/app/globals.css` — the `prefers-reduced-motion`
  block. Load-bearing: daisyUI's `.loading` sets `animation` in the same layer
  and this rule has to beat it whatever order the layers land in.
- `frontends/apps/checkout/src/config/theme.ts` — a doc comment that *uses* the
  word to explain the code avoids needing one. Prose, not CSS.
- `examples/checkout-browser/index.html` — `[hidden]{display:none !important}`
  in a plain-HTML demo with no framework and no Tailwind.

### Checks 5 and 5b — the two permanent guards on the deleted package

5 covers four spellings because the package exported `.`, `./testing`,
`./testing/contrast` and `./styles.css`, and the last was reached by `@import`
from a stylesheet and by `createRequire(...).resolve` from a test — neither of
which is a `from "…"`. It also matches a `"@vpay/ui": …` manifest entry.

**5b is not redundant with 5.** `frontends/packages/config/src/eslint.js`
resolved a `new URL(…)` whose first argument was the deleted package's
`styles.css`, two directories up by a relative path, as the Tailwind
class-universe entry point for three packages — including `examples/shop`,
which never depended on `@vpay/ui` at all. **No grep for the package name would
ever have found it.** That file now resolves the entry point from the
*consumer's* own `tsconfigRootDir` (`resolveTailwindEntryPoint`, trying
`app/globals.css` then `src/app/globals.css`).

5b requires the path to sit **right after a quote** — a real string literal.
Measured before shipping: a naive pattern with no quote requirement matched ten
lines of legitimate **prose** across nine files (a backticked path inside a doc
comment), which would have made the check impossible to ship.
`frontends/packages/config/src/eslint.test.js` is exempted by path for a
measured false positive: it names a file in the deleted package as a
double-quoted string in a table asserting which ESLint rules apply where, and
ESLint's `calculateConfigForFile` matches the path *string* against each
block's glob — it never reads the file from disk.

### 7a — why the blanket rule had to go

The rule was "no `className` in an app at all", and its remedy was "add the
missing primitive to `@vpay/ui`". That package is gone, and `@vaam-apps/ui`
ships **no layout or typography primitive** — no `PageShell`, `Stack`,
`Heading`, `Text`, `List`, `Link`, `Section` or `VisuallyHidden`. `className`
appears on 39 of its 56 declaration files: the package's own API says "extend
me with a class". Under the old rule both apps would have shipped structurally
faithful, visually unstyled screens.

So layout classes are now legal and these three are not:

| 7a was written to ban | now covered by |
| --- | --- |
| a raw palette colour | checks 1 + 1b |
| a status colour as a theme token | 7a-ii |
| a hand-rolled variant system | check 4 (`cva`) + 7a-i |
| a re-implemented primitive | 7b + 7a-iii's budget |
| a layout class | **not banned, on purpose** |

7a-i's known limit, stated openly: it permits `className="flex items-center"`
and forbids `className={"flex items-center"}`, which are the same thing.
Acceptable — prettier normalises the second to the first.

7a-iii's 60 is not a number chosen to keep the check green: it is
`enforce-consistent-line-wrapping`'s own `printWidth` (100) minus
`className=""`. The decisive negative control is that a real primitive the apps
replaced fits under it — `@vpay/ui`'s deleted `PageShell` string,
`"mx-auto flex max-w-md flex-col gap-6 p-6"`, is 46 characters.

### 7b — two greps, and one token dropped outright

7b's alternation used to contain four tokens that are also real Tailwind
utilities, and the collision was invisible while no app wrote a class. `table`
matches inside `table-fixed`/`table-cell`; `select` matches inside
`select-none`/`select-text`; `mask` matches Tailwind 4's `mask-*` family. Those
three moved to 7b-ii, which only fires on the bare token or a **daisyUI**
modifier suffix — so `select-none` and `table-fixed` pass while
`select select-bordered` and `table table-zebra` fail.

`collapse` is **dropped outright, not special-cased**: `<div class="collapse">`
is daisyUI's accordion and `visibility: collapse`'s cousin, and they are the
identical string. Undecidable, not a modifier collision. Recorded here so
nobody re-adds it as a "simple" fix.

`examples/shop` is deliberately **not** in 7b's scope: it is a merchant's
storefront — a third party who has vpay's SDK and not vpay's design system.
Holding it to this rule would ship the demo a look no real merchant would have.

7b-ii has **one** exemption:
`frontends/apps/checkout/src/components/locale-switch.tsx`. A native `<select>`
is forced there because `@vaam-apps/ui`'s `SelectTrigger` destructures only
`{id, className, children}` and spreads nothing else, so `aria-labelledby`
never reaches the rendered element — and `checkout-view.test.tsx` asserts the
control's accessible name comes from a visible French label. A genuine library
gap, narrowed to one file so a wider exemption cannot swallow a real
regression.

## Check 6 was DELETED, not narrowed

Check 6 was a 200-line ceiling on every file in `@vpay/ui`. Its pathspec was
`git ls-files -- <that package's src>`, which after the deletion **matched no
tracked file**. `git ls-files` on a dead pathspec exits 0 with empty output, so
the loop body never ran and `fail` stayed 0 — the check would have gone on
printing "verify: ok" having measured **zero files against a limit**. That is
worse than no gate, because it reads as coverage it no longer provides.

(The same reasoning retired the old check 5, a `.js`-relative-import guard
scoped to the same directory. Its slot was spent on the permanent
`@vpay/ui` guards above rather than left vacuous.)

Re-pointing the ceiling at app components was **considered and rejected** for
that change: three shipping files already exceeded 200 lines while the
migration was still rewriting them — `frontends/apps/checkout/src/components/
screens.tsx`, `checkout-client.tsx` and `checkout-view.tsx`, measured at 718,
336 and 279 lines the day the package was deleted.

**The 200-line directive is therefore recorded as ungated, not dropped.**
Whether a component-size limit transfers from a shared primitive package to app
screens is undecided. Do not treat a 400-line component as blessed because
nothing greps for it.

## The residual hole, stated rather than hidden

> A re-implemented primitive assembled from **several short literal
> `className` attributes across one file** passes 7a-i, 7a-ii, 7a-iii *and* 7b.

There is no grep for that. The mitigation is review and the 60-character
budget, not a claim that the gate is complete. The recipe says so itself.

## How contributors trip it

| You wrote | Trips | Do instead |
| --- | --- | --- |
| `className={cn("card", loud && "text-state-danger-fg")}` | 7a-i, 7a-ii, 7b | Compose a `@vaam-apps/ui` component and pass a variant prop |
| `className={busy ? "opacity-50" : "opacity-100"}` | 7a-i | The component takes the boolean |
| `className={STYLE.row}` or a template literal | 7a-i | Inline the literal, or move the variants into a component |
| `className="btn btn-primary"` copied from daisyUI docs | 7b | `<Button>` from `@vaam-apps/ui` |
| `className="select select-bordered"` | 7b-ii **and** 2 | `-bordered` no longer exists in daisyUI 5 |
| `className="text-state-danger-fg"` for a failed status | 7a-ii | `defineStatusSystem` / `createStatusPill`, or an `InlineBanner` variant |
| `className="bg-red-500"` | 1 | A daisyUI theme token |
| `className="bg-[#0a0b0d]"` | 1b | A theme token |
| A layout string that grew past 60 chars | 7a-iii | Split it, or compose a component |
| A one-off `!important` | 3 | Raise specificity, or argue for a fourth exemption in review |
| Anything still naming `@vpay/ui`, including a CSS `@import` | 5 / 5b | It is gone; use `@vaam-apps/ui` |

The underlying rule, from `AGENTS.md`: **never inline a status colour in a
component — a status must not be green in one view and grey in another.**
`verify-ui` is that rule with a grep behind it.

## ESLint does the other half

`frontends/packages/config/src/eslint.js`, on `tailwind: true`:
`enforce-consistent-line-wrapping` (`printWidth: 100`),
`enforce-consistent-class-order`, `no-unknown-classes` (the rule that caught
`select-bordered`), `enforce-consistent-variable-syntax` (caught
`w-[--anchor-width]`, Tailwind 3 syntax that Tailwind 4 compiles into an
invalid declaration rather than rejecting), `no-conflicting-classes`,
`no-duplicate-classes`.

Run `pnpm --filter <app> lint` before `just verify-ui`: ESLint compiles the
real theme and names an unknown class precisely, where the gate answers with a
grep hit.

## Known-stale prose

The recipe's check 5 comment says `frontends/packages/ui` "is still on disk
waiting for Group F to `git rm` it". As of 2026-09-16 it is **not on disk** —
`frontends/packages/` holds `api-client`, `config` and `tokens` only. The
`:!frontends/packages/ui` pathspec exclusions are therefore inert, which is
harmless but means the comment describes a tree that no longer exists.
