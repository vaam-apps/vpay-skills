# Node, pnpm, prettier, Cypress, Storybook

_Verified against vpay `7a79684e` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

## `pnpm install` fails outright on Node version

**Cause:** `.npmrc` sets `engine-strict=true`. The baseline is `.nvmrc`
(`22.23.2` as of 2026-09-16). It **fails rather than warns**.

**Related pin:** ESLint **9**, not 10, deliberately — ESLint 10 requires Node
`^22.13.0` and `.nvmrc` (which CI reads) is below it. Re-check when `.nvmrc`
moves.

## A **Rust** test fails with ``"`pnpm --filter @vpay/sdk build` failed:\nsh: 1: tsc: not found"``

**Symptom:** the first `just test-rust` in a fresh worktree fails
`webhooks::the_delivered_signature_verifies_with_the_shipping_node_sdk`. It
looks like a signature defect and is a missing toolchain.

**Cause:** that test shells out to the **shipping Node SDK** to verify a
signature vpay produced. `just test-rust` does not depend on `install-node`, and
a worktree with no `node_modules` has no `tsc`. CI's `rust` job installs Node
and builds the SDK first, so CI never sees it.

**Fix:** `pnpm install --frozen-lockfile`, then re-run.

The test is behaving as designed — it fails rather than skipping, which is the
point. The same missing `node_modules` shows up as `lint-web` dying on
`tsc: not found`.

## `ERR_PNPM_RECURSIVE_EXEC_FIRST_FAIL Command "prettier" not found`

**Cause:** `just fmt`'s prettier half, in a worktree with no `node_modules`.

**Fix:** `pnpm install`. Note `just fmt-check` (the Rust gate, and the one `just
ci` runs) passes either way — so a green `fmt-check` does not mean the markdown
was checked.

## Prettier rewrote pasted evidence inside a markdown code fence

**This is the one to be careful about, because it makes documents false and no
gate notices.**

**Cause:** prettier's default `embeddedLanguageFormatting: "auto"` reformats the
code _inside_ fenced blocks for every language it can parse. This repository's
markdown is largely transcripts, quoted logs and historical notes — "that
default is not a style choice here — **it edits records**."

**Measured damage when it was briefly on:** 40 of 611 fences changed across 22
files (`json` 15, `markdown` 9, `ts` 6, `yaml` 3, `tsx` 2, `css` 2, `jsonc` 1,
`console` 1). Two that made a document false:

- `docs/runbooks/demo.md` — a single-line `tracing` JSON log line, pasted from a
  real run, pretty-printed across ten lines. `tracing`'s JSON layer emits one
  line, so the runbook then showed an operator output the server does not
  produce, **in the section that tells them what to grep for.**
- `docs/plans/exp9-notes/opus.md` — a workflow fragment indented to show
  structure, reindented away.

**Fix:** it is configured off. Do not re-enable embedded formatting. If you are
reformatting markdown, diff the `code` node values, not just the file.

Two related findings from the same review, worth carrying:

- Reformatting `pnpm-lock.yaml` (+5444/−2837) was pulled out as unacceptable
  blast radius in a PR that lands last.
- A claim that a formatting change was "verified format-only via
  `prettier --parser markdown` before/after" is **circular and proves nothing.**

## Cypress has no binary

**Cause:** the Cypress binary is **not** fetched by a plain `pnpm install`.

**Fix:** `pnpm exec cypress install`, on a machine that can reach Cypress's CDN.
In restricted networks, `CYPRESS_INSTALL_BINARY=0` lets the rest of the install
proceed without it.

`pnpm -r test` no longer touches Cypress at all — `@vpay/e2e`'s own test script
is `e2e`, not `test` — so the ordinary unit sweep works either way.

## `EADDRINUSE: address already in use 127.0.0.1:4181`, or `cy.visit()` failing with `ECONNREFUSED` on a port the spec never chose

**Cause, and this is open on `master` as of 2026-09-16:** the Cypress side of
`just test-e2e` starts two fixture servers on **fixed** ports —
`checkoutBrowserServer.ts` on **4180** and `frameFixtureServer.ts` on **4181**.
Each is overridable by an environment variable (`CHECKOUT_BROWSER_PORT`,
`VPAY_E2E_FRAME_FIXTURE_PORT`) that `just test-e2e` does **not** set from a
`just` variable, unlike every one of the seven compose ports beside it. A
concurrent `just test-e2e` in another worktree takes both.

**Fix:** set the two environment variables by hand before the run. The proper
fix — wiring them to `just` variables alongside `demo_project` and its siblings
— has not been done.

**A second cause of the same message:** an orphaned `Cypress: Config Manager`
process and the static server it spawned, both still listening from a
`before:run` phase that never reached `after:run` cleanup. Kill by PID, confirm
the port free, retry.

## Every Storybook story renders unstyled and every test passes

**Cause:** `frontends/apps/checkout/app/globals.css` stopped importing the
theme **before its other at-rules**. That defect "made this suite render every
story unstyled for six runs while passing all of them."

**The guard:** `frontends/apps/checkout/src/a11y-gate.test.ts` runs in
`just test-web`, so in `just ci`, and fails if the Storybook a11y addon goes, if
a violation stops failing, if the preview stops painting the document shell the
real page paints, **or if `globals.css` stops importing the theme before its
other at-rules.**

**Why it matters more than it looks:** `just test-storybook` renders every
checkout story in a real Chromium and is "the only thing in this repository
that returns a colour-contrast **verdict** for the screens a payer sees". The
jsdom axe suites compute no colour, and an earlier Cypress attempt only ever
got `incomplete` out of the real page.

`just test-storybook` runs in CI's `web` job. It needs the network the first
time — Playwright fetches a ~115 MB Chromium. **Run it before opening a PR that
touches a checkout screen, a story or the theme.**

## `just verify-ui` refuses something that looks fine

`frontends/packages/ui` (`@vpay/ui`) was **deleted 2026-09-12**; both apps
compose the published `@vaam-apps/ui`. `verify-ui` refuses, among others: an
import of `@vpay/ui` or a path into the deleted directory; a computed
`className` assembled at runtime ("a class string assembled at runtime is a
variant map; put the variants in a component"); a raw status-colour token
written in an app; a `className` over 60 characters; a daisyUI component class
in an app; a daisyUI-4 class removed in daisyUI 5; `!important` outside the
documented exemptions; a `cva` variant map in an app.

The status-token rule is the one most likely to surprise: use
`defineStatusSystem` (`StatusPill`/`StateChip`) or an `InlineBanner` variant
rather than `text-state-<hue>-fg` and friends. The repo ships a local skill for
this library at `.agents/skills/vaam-ui/`, including a `pitfalls.md`.

## `AGENTS.md`'s TypeScript section is not trustworthy on libraries

It has been corrected twice. It once named Headless UI, framer-motion and vaul,
"none of which is a dependency of any `package.json` in this repository". The
2026-09-12 correction records that `@vpay/ui` was deleted, `@base-ui/react` left
with it, and the theme registers under daisyUI's built-in name `dark`, not
`bumblebee`. Trust `package.json` and `just verify-ui`.
