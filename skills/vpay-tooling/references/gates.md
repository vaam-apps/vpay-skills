# The twelve gates

`just verify` runs twelve gates and then one report:

```
verify: verify-no-mocks verify-status verify-errors verify-sdk-parity \
        verify-links verify-npm-scope check-schema verify-serde \
        verify-repositories verify-toolchain verify-ui verify-migrations \
        verify-docs
```

Ten are `cargo xtask` subcommands implemented in `.xtask/src/main.rs`;
`check-schema` and `verify-ui` are shell in the `justfile`. CI's `self-checks`
job runs exactly this list in exactly this order, and that job has no
`changes` gating — **it runs on every push and every pull request**.

The order is chronological by the date each gate landed, deliberately, so that
ordinals written down in other files ("check-schema is the seventh gate") stay
true when a gate is appended. Do not re-sort it by subject.

`cargo xtask verify-all` chains the ten xtask gates. It excludes
`verify-citations` (network) and cannot run `check-schema` or `verify-ui`,
which are shell. `just verify` is the real list; `verify-all` is a convenience.

## Both-directions gates

Four gates fail in **both** directions, and that is the property to remember,
because the intuitive half is the one that never bites you:

- **`verify-status`** — a token in code with no declaration fails, _and_ a
  declaration whose token no longer exists fails.
- **`verify-sdk-parity`** — a claim naming a missing test fails, _and_ a method
  either SDK declares with no matrix row fails, _and_ a row whose method is
  gone fails. Two-directional since 2026-09-06; before that, deleting a whole
  row passed.
- **`verify-serde`** — a non-compliant type fails, _and_ an exemption row
  naming a type that now complies, or no longer exists, fails. A stale
  exemption is a decision the code already reversed, described as current.
- **`verify-migrations`** — an edited file fails, _and_ a manifest line whose
  file is gone fails.

## 1. `verify-no-mocks`

**Refuses:** any test-only crate reachable from a shipping binary through
non-dev edges of the **resolved** dependency graph.

**Implements:** `cargo xtask verify-no-mocks`, `verify_no_mocks` in
`.xtask/src/main.rs`. It walks `cargo metadata` rather than grepping manifests
— the question "which crates are reachable from a shipping binary" is about the
resolved graph and no amount of manifest grepping answers it.

**How you trip it:** adding `vpay-testkit` (or any mock adapter) as a normal
dependency instead of a `[dev-dependencies]` one, to make a binary easier to
run locally. This is ADR-0006's rule: a stub rail is a **separate process
reached over HTTP via configuration**, never a code path inside the app.

## 2. `verify-status`

**Refuses:** a `ProviderError::NotImplemented("…")` token in shipping code that
is not declared in `docs/status.md`, and a token declared there that shipping
code no longer carries.

**Implements:** `cargo xtask verify-status`, `verify_status` /
`declared_tokens` / `scan_not_implemented` in `.xtask/src/main.rs`.

**How you trip it:** leaving a `NotImplemented` without adding the bullet; or
retiring one and forgetting to remove the bullet.

**The subtlety worth knowing:** since 2026-09-05 the scanner **lexes** rather
than greps. A token mentioned in a `//`, `///`, `//!` or `/* */` comment, in a
`#[doc = "…"]` attribute, or inside a string, raw-string or char literal is
prose and counts for nothing in either direction. Before that, a trailing
comment carrying the token forced a phantom bullet into `docs/status.md` — and
the cheapest way to clear one was to delete the honest sentence from the
adapter's doc comment that explained the gap. If you are writing about a token
in prose, you no longer have to contort around this.

`.github/workflows/docs.yml` runs this gate alone on any PR touching `docs/**`,
`README.md`, `AGENTS.md` or `CLAUDE.md`.

## 3. `verify-errors`

**Refuses:** a `pub` error type in `backends/crates` that does not implement
`vpay_core::error::Classify`; `anyhow` outside `backends/apps` (ADR-0011).

**Implements:** `cargo xtask verify-errors`, `verify_errors` in
`.xtask/src/main.rs`.

**How you trip it:** adding an error enum to a library crate and not
classifying it; reaching for `anyhow` in a crate under `backends/crates`. A
composite error should `#[from]` its leaves and **delegate** classification
rather than re-deciding it.

## 4. `verify-sdk-parity`

**Refuses:** in `docs/sdks/parity.md`, a claimed capability naming a test that
does not exist in that SDK's sources; a gap without a date and an owner; and —
both directions — a `<resource>.<method>` either SDK declares with no row, or a
row whose method no longer exists.

**Implements:** `cargo xtask verify-sdk-parity`.

**How you trip it:** adding a method to `sdks/nodejs` or `sdks/rust` and not
adding the row. The gate reads the SDK sources, so it finds the method before
you remember the matrix exists.

## 5. `verify-links`

**Refuses:** a relative link in any tracked `*.md` that does not resolve to a
path `git ls-files` knows about.

**Implements:** `cargo xtask verify-links`.

**How you trip it:** linking to a file you created and never `git add`ed. It
resolves on your machine and nowhere else — which is exactly why the gate uses
the index rather than a directory walk.

**What it does not check,** so nobody reads a green run as more than it is:
`#anchor` fragments (agreeing with GitHub's heading-slug algorithm is a guess,
and a wrong guess fails correct documents), `http(s)` URLs, and `mailto:`.
Fenced code blocks, inline code spans and HTML comments are masked out first.

## 6. `verify-npm-scope`

**Refuses:** a publishable package under `sdks/` that is not named
`@vaam-apps/vpay-*`, or does not declare `publishConfig.access: "public"`, name
this repository, carry a license, and ship a `files` allowlist with an entry
point under `dist/`. A **private** package that declares any `publishConfig` at
all. Any retired `@vpay/*` package name outside `docs/plans`, `docs/adr`,
`docs/status.md` and `docs/status/`.

**Implements:** `cargo xtask verify-npm-scope`.

**How you trip it:** dropping `publishConfig.access` — the one line between
`npm publish` and a scoped package defaulting to `restricted`. That deletion
was measured to be caught by nothing else: not the lockfile, not
`pnpm -r typecheck`, not `lint-web`, not `test-web`. Also: writing `@vpay/foo`
in a doc outside the four allowlisted prefixes.

**What it does not check:** that `dist/` exists (gitignored — a gate needing a
build would fail on a clean checkout for a reason that is not its subject), and
the registry (that needs the network).

## 7. `check-schema`

**Refuses, in this order:**

1. the `cratestack` CLI not being on PATH. **It fails; it does not skip.** The
   message tells you to
   `cargo install cratestack-cli --locked --version <cratestack_version>`.
2. `schemas/vpay.cstack` declaring no `datasource` block — without one,
   cratestack treats it as client-only and stops applying every
   database-backed-model rule, while still printing `schema OK`.
3. fewer than `cratestack_min_declarations` (15, as of 2026-09-16) top-level
   `model`/`enum` declarations — an emptied or truncated file type-checks
   vacuously.

Then it runs `cratestack check --schema schemas/vpay.cstack`.

**Implements:** shell in the `justfile`.

**How you trip it:** not having the CLI. A _version_ mismatch only WARNs and
the check still runs, against whatever grammar is on PATH.

**The connected trap:** `schemas/vpay.cstack` **is** wired into the build since
2026-09-06 — `vpay-db` has a private `mod schema` that compiles it, so a syntax
error is a `cargo build` failure, not just a gate failure. Adding a `model` is
not free: it must match the live table, and `postgres_smoke.rs`'s drift test
pins the exact gap. `CLAUDE.md` said the opposite ("not wired into the build,
do not try to make it compile") and was wrong in two stages.

## 8. `verify-serde`

**Refuses:** ADR-0016 standard 3 — a type deriving `Serialize`/`Deserialize`
under `backends/crates/*/src` that does not carry
`#[serde(rename_all = "snake_case")]`, does not rename every member itself, and
is not listed in the ADR's exemption table with a **non-blank** reason. Plus
the reverse: an exemption row naming a type that now complies or no longer
exists.

**Implements:** `cargo xtask verify-serde`, `verify_serde` in
`.xtask/src/main.rs`.

**How you trip it:** adding a wire type without `rename_all`; or fixing a type
and leaving its exemption row. Visibility is deliberately not part of the rule
— both adapters' wire modules are `pub(crate)`, and a wire does not care what
Rust thinks of a type's visibility.

**What it does not check:** whether a reason is a _good_ one. "models MTN's
camelCase Collections wire" and "too many to fix" are both non-empty strings.
The table exists to put the sentence where a reviewer sees it.

## 9. `verify-repositories`

**Refuses:** ADR-0016 standard 5 — anything outside `vpay-db` naming a concrete
repository implementation. The set of concrete implementations is **derived**
from `vpay-db`'s own source (a declaration holding a `PgPool`/`Transaction`
field, or a type on the right of `impl <a vpay-db trait> for …`), so a store
nobody has written yet is covered the day it is added.

**Implements:** `cargo xtask verify-repositories`, `verify_repositories` in
`.xtask/src/main.rs`.

**There is no exemption mechanism, deliberately:** there is no exception today,
and an escape hatch nobody needs is the one that gets used.

**How you trip it:** `pub use`ing a store type from `vpay-db`; making
`vpay-db`'s generated `mod schema` `pub` or re-exporting it. That module exists
in no source file — the macro creates it — so nothing else in the repository
would object.

This gate found a real one on the day it landed: `vpay-api` named
`vpay_db::SqlClientAssertionStore`, a concrete implementation that had been
`pub` since Step 6 and that no gate, lint or compiler error had objected to.

## 10. `verify-toolchain`

**Refuses:** `backends/Dockerfile`'s `FROM rust:<version>-alpine…` naming a
compiler that `rust-toolchain.toml`'s `channel` does not pin.

**Implements:** `cargo xtask verify-toolchain`, `verify_toolchain` in
`.xtask/src/main.rs`.

**How you trip it:** bumping `rust-toolchain.toml` alone. That `FROM` line is
the one place in the repository that names a compiler version and cannot read
the toolchain file, so it is the one place the pin can drift.

**Why it exists, measured rather than assumed:** during the 1.95.0 → 1.98.0
bump, with the toolchain file moved and the Dockerfile left behind, `just
verify` and `just fmt-check` both exited 0 and **the whole of `just ci` was
green**. The first symptom would have been a release binary built by a compiler
no local run and no CI job had ever used.

The Alpine base suffix is deliberately **not** checked — it moves on its own
evidence.

## 11. `verify-ui`

Shell, not a `cargo xtask`, and that is a decision rather than laziness: the
highest-risk check is that a daisyUI 4 class daisyUI 5 removed **still parses
and still renders** — it just silently stops styling anything, and every test
keeps passing. Nothing else in `just ci` would notice.

Eleven numbered checks over twelve greps. Each accumulates into `fail` and the
recipe `exit $fail`s at the end, so it reports **every** violation, not just
the first. The numbering is non-contiguous — check 6 was deleted, 7a was split
three ways, 5b and 7b-ii were added. Read the recipe body, not its header
paragraph.

| #      | Refuses                                                                                                                                                                                                                                                                                                                                | Scope                        |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| 1      | a hard-coded palette colour: `(bg\|text\|border\|ring\|fill\|stroke\|from\|via\|to\|decoration\|outline\|shadow\|accent\|caret\|divide\|placeholder)-(red\|green\|blue\|amber\|slate\|…\|black\|white)`                                                                                                                                | `frontends`, `examples`      |
| 1b     | an arbitrary colour value: the same prefixes followed by `-[#`, `-[rgb`, `-[hsl`, `-[oklch`, `-[color-mix`                                                                                                                                                                                                                             | same                         |
| 2      | a daisyUI 4 class daisyUI 5 removed: `form-control`, `label-text`, `label-text-alt`, `btn-group`, `input-group`, `card-compact`, `input-bordered`, `select-bordered`, `textarea-bordered`, `tabs-bordered`, `tabs-lifted`, `tabs-boxed`                                                                                                | same                         |
| 3      | `!important`, outside three documented exemptions                                                                                                                                                                                                                                                                                      | same                         |
| 4      | a `cva(` call outside a component package                                                                                                                                                                                                                                                                                              | `frontends/apps`, `examples` |
| 5      | importing `@vpay/ui` — the package was **deleted 2026-09-12**. Covers `from`/`import`/`require(`/`resolve(`, `@import`, and a `"@vpay/ui":` dependency key                                                                                                                                                                             | tree                         |
| 5b     | path-reaching into it: `"../ui/src"`, `"frontends/packages/ui"`                                                                                                                                                                                                                                                                        | tree                         |
| 6      | **deleted.** It was a 200-line ceiling on `@vpay/ui` files; after the package went, its pathspec matched nothing, the loop body never ran and `fail` stayed 0. A check that cannot fail reads as coverage it no longer provides                                                                                                        | —                            |
| 7a-i   | a computed class name: `className\s*=\s*{`                                                                                                                                                                                                                                                                                             | `frontends/apps`             |
| 7a-ii  | a status-colour theme token: `(bg\|text\|border\|ring\|fill\|stroke\|divide\|outline\|shadow)-(state-[a-z]+-(fg\|bg\|border)\|destructive(-foreground)?)`                                                                                                                                                                              | `frontends/apps`             |
| 7a-iii | a class attribute over 60 characters: `className="[^"]{61,}"`                                                                                                                                                                                                                                                                          | `frontends/apps`             |
| 7b     | a daisyUI **component** class. The alternation deliberately omits `table`, `select` and `mask` (real Tailwind utilities — `table-fixed`, `select-none`, Tailwind 4's `mask-*`) and omits `collapse` outright, because daisyUI's `.collapse` and Tailwind's `visibility: collapse` are the same token and a grep cannot tell them apart | `frontends/apps`             |
| 7b-ii  | `table`/`select`/`mask` when daisyUI-modifier-suffixed or bare. One exemption: `frontends/apps/checkout/src/components/locale-switch.tsx`                                                                                                                                                                                              | `frontends/apps`             |

**How you trip it, in rough order of frequency:** writing `bg-red-500` instead
of a theme token; copying a daisyUI 4 tutorial and using `form-control` or
`label-text`; a Tailwind class string in an app that runs past 60 characters;
`className={cn(…)}` in an app.

Check 1 has been widened twice against measured holes: it no longer requires
`className=` earlier on the same line (a class held in a lookup object — the
checkout's `TONE_CLASS` is exactly that shape — used to be invisible), and it
no longer requires a numeric suffix (`bg-black/40` used to pass).

## 12. `verify-migrations`

**Refuses:** any `backends/migrations/*.sql` whose SHA-256 differs from its
line in `backends/migrations/MANIFEST.sha256`, and any manifest line whose file
is no longer on disk. Only `*.sql` — that is what `sqlx::migrate!` reads;
`backends/migrations/README.md` is not a migration and no database ever hashed
it.

**Implements:** `cargo xtask verify-migrations`, `verify_migrations` in
`.xtask/src/main.rs`.

**Why it exists:** `sqlx::migrate!` stores a SHA-384 of each file's whole bytes
in `_sqlx_migrations.checksum` and **refuses to boot** when a file no longer
hashes to what the database recorded. A reflowed comment bricks every database
that applied the original. That is not hypothetical — PR #39 did it to
migration `0028` (issue #76), which is why this gate exists.

**How you trip it:** editing a shipped migration at all. Reformatting,
reflowing a comment, fixing a typo in a comment — all of it.

**The fix is never to regenerate.** `just migrations-manifest` is
**append-only**: it refuses to rewrite a line whose hash changed and refuses to
drop a line whose file is gone. If it rewrote lines, this gate would be one
anybody could silence with one command, and the command's name would make that
look like the fix rather than the mistake. Revert the file and write a **new**
migration that corrects it. See `docs/runbooks/migrations.md`.

What the gate does not stop is someone who edits a migration _and_ hand-edits
its manifest line — nothing can, since whoever can edit two files can edit
three. What it buys is that the edit becomes a one-line diff on a file whose
only purpose is to be reviewed.

## The report: `verify-docs`

`cargo xtask verify-docs` runs last in `just verify` and **exits 0 whatever it
finds**. It reports doc-comment lines against code lines per crate, every
production function of 80 lines or more, every ` ```ignore ` doctest fence and
every `#[allow]`/`#[expect]` in production code.

It is a report and not a gate on purpose (Step 7 decision 4; ADR-0016 standard
6 keeps it that way): **the cheapest way to pass a doc-ratio gate is to delete
the `# Errors` and `# Panics` sections** that ADR-0011 and rustdoc depend on. A
number that is read is worth more here than a number that is enforced.

It is last so the report a human reads is the final thing on the terminal. The
`verify: ok` line the recipe prints afterwards means the twelve gates passed
and says nothing about the numbers `verify-docs` printed.

## The thirteenth gate: `verify-citations`

`just docs-check-citations` → `cargo xtask verify-citations`. **Not** in `just
verify` and **not** in `just ci`, because it needs the network and an
authenticated `gh`. It is deliberately excluded from `cargo xtask verify-all`
too.

It resolves every workflow-run id, pull request and issue that a tracked `*.md`
cites as evidence — `run 33929374661`, `PR #31`, `Issue #11` — against this
repository, and fails on one that does not exist.

**It FAILS when `gh` is missing or unauthenticated.** It does not print
"skipped" and exit 0, because a check that downgrades itself reports success
for a run in which nothing was checked, in a log indistinguishable from a run
in which everything passed.

Run it when you add or edit a document that cites an id. Fix a dead citation
with a struck-through, dated correction — never by substituting an id you have
not checked.
