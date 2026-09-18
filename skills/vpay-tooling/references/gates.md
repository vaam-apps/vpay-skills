# The fourteen gates

_Verified against vpay `d3a8810b` (2026-09-16); §§ 13-14 read from `pr-187`
(at `932df356`) and `master` (`aeb9e242`) on 2026-09-18, before either merged.
The `pr-187` branch moved twice more the same day; gate names, recipe wiring
and refusal logic were unchanged across all three commits, so treat those as
stable and the line numbers and narrative wording as a snapshot.
Version-sensitive claims carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

**Fourteen as of 2026-09-18.** ~~Twelve.~~ It was twelve until 2026-09-17,
when [#201](https://github.com/vaam-apps/vpay/pull/201) appended
`verify-versions`; [#187](https://github.com/vaam-apps/vpay/pull/187) appends
`verify-privacy-inventory` and makes it fourteen.

> **Read the recipe, not the prose — and this is the best live example of the
> authority rule there has ever been.** Measured on `master` `aeb9e242`
> (2026-09-18), a full day after #201 merged: the `verify` recipe runs
> **thirteen** gates and echoes _"the thirteen gates above passed"_, while
> `AGENTS.md` says _"on this commit the gates are twelve"_ and lists twelve,
> `docs/status.md` says _"twelve gates and one advisory report"_ over a
> twelve-row table with no `verify-versions` row, and the `justfile`'s own
> header block says _"Twelve invariants"_. One change moved the recipe, the
> echo, `verify_all` and the CI step and left **every prose count in the tree**
> behind it. `AGENTS.md` predicted this about itself — "it has gone stale at
> nearly every count it has carried".

`just verify` runs fourteen gates and then one report:

```
verify: verify-no-mocks verify-status verify-errors verify-sdk-parity \
        verify-links verify-npm-scope check-schema verify-serde \
        verify-repositories verify-toolchain verify-ui verify-migrations \
        verify-versions verify-privacy-inventory verify-docs
```

Twelve are `cargo xtask` subcommands implemented in `.xtask/src/main.rs`;
`check-schema` and `verify-ui` are shell in the `justfile`. CI's `self-checks`
job runs exactly this list in exactly this order, and that job has no
`changes` gating — **it runs on every push and every pull request**.

The order is chronological by the date each gate landed, deliberately, so that
ordinals written down in other files ("check-schema is the seventh gate") stay
true when a gate is appended. Do not re-sort it by subject. Two gates landing
on the same day from branches that had not seen each other is a **normal**
event here, not a mistake — `verify-npm-scope`/`check-schema` collided that way
on 2026-09-05 and `verify-versions`/`verify-privacy-inventory` did on
2026-09-17. It is resolved the same way both times: both, in the order they
landed, and whichever lands second renumbers itself.

`cargo xtask verify-all` chains the twelve xtask gates. It excludes
`verify-citations` (network) and cannot run `check-schema` or `verify-ui`,
which are shell. `just verify` is the real list; `verify-all` is a convenience.

## Both-directions gates

Five gates fail in **both** directions (2026-09-18; four before
`verify-privacy-inventory`) — `verify-status`, `verify-sdk-parity`,
`verify-serde`, `verify-migrations` and `verify-privacy-inventory`. That is the
property to remember, because the intuitive half is the one that never bites
you:

- **`verify-status`** — a token in code with no declaration fails, _and_ a
  declaration whose token no longer exists fails, _and_ (since 2026-09-16) a
  `<rail>::…` token carried outside that rail's adapter crate fails. **Three**
  directions now, not two — see [§ 2](#2-verify-status) for the measured hole
  the third one closes.
- **`verify-sdk-parity`** — a claim naming a missing test fails, _and_ a method
  either SDK declares with no matrix row fails, _and_ a row whose method is
  gone fails. Two-directional since 2026-09-06; before that, deleting a whole
  row passed.
- **`verify-serde`** — a non-compliant type fails, _and_ an exemption row
  naming a type that now complies, or no longer exists, fails. A stale
  exemption is a decision the code already reversed, described as current.
- **`verify-migrations`** — an edited file fails, _and_ a manifest line whose
  file is gone fails.
- **`verify-privacy-inventory`** — a migrated column no inventory element
  classifies fails, _and_ an inventory copy naming a column no migration
  creates fails. See [§ 14](#14-verify-privacy-inventory).

**`verify-links` is not one of them**, though it is the one people add to this
list from memory: it fails a link that resolves to no tracked path and has no
second direction — nothing fails a tracked file that no link points at, and
nothing could, because most files are not link targets.

And the property has a floor, which [§ 14](#14-verify-privacy-inventory) is the
proof of: **both directions of a derive-then-compare gate read the same derived
set**, so a deriver that answers "nothing" for input it does not understand
makes both directions agree about something neither can see. Two directions
are only as good as the oracle they share.

## What the gates last printed

vpay's own `docs/status.md` carries this table and re-runs it; these are its
figures **as re-run on 2026-09-16**, on `888b00c3` — the last commit before
the table was filled in, which is **not** the merge commit `7a79684e`, so a
number that counts documents or links can legitimately be a few higher on the
merged tree. Re-run the gate rather than quoting this page if a count is
load-bearing for you.

| Gate                  | Last printed (2026-09-16)                             |
| --------------------- | ----------------------------------------------------- |
| `verify-status`       | 1 unimplemented item                                  |
| `verify-errors`       | 20 error types, 17 `#[from]` variants                 |
| `verify-sdk-parity`   | 603 proving tests, 36 dated gaps, 35 methods, 39 rows |
| `verify-links`        | 1 731 links in 375 files                              |
| `check-schema`        | 27 declarations                                       |
| `verify-serde`        | 96 types, 17 exemptions                               |
| `verify-repositories` | 4 implementations, 83 source files outside            |
| `verify-migrations`   | 48 files                                              |

Six of these moved between 2026-09-11 and 2026-09-16 — `verify-errors` 19→20,
`verify-sdk-parity` 550/35/32→603/36/35, `verify-links` 1 600/352→1 731/375,
`check-schema` 26→27, `verify-serde` 90→96, `verify-migrations` 42→48. That
rate is the reason every count on these pages carries a date.

**`docs/status.md` re-ran all fourteen on 2026-09-17** on `pr-187`'s merge of
`master` at `eb078020` — the newest full column, and the only one that includes
the two new gates. Its own note says merging `master` "moved nine of the
numbers and added a gate"; these are the rows that differ from the column
above:

| Gate                       | 2026-09-16 (`888b00c3`)        | 2026-09-17 (`eb078020`, 14 gates)           |
| -------------------------- | ------------------------------ | ------------------------------------------- |
| `verify-sdk-parity`        | 603 tests, 36 gaps, 35 methods | **661 tests, 37 gaps, 35 methods, 39 rows** |
| `verify-links`             | 1 731 links in 375 files       | **1 719 links in 399 files**                |
| `verify-serde`             | 96 types, 17 exemptions        | **102 types, 17 exemptions**                |
| `verify-repositories`      | 4 impls, 83 files outside      | **4 impls, 84 files outside**               |
| `verify-versions`          | did not exist                  | **red** — 2 unannotated `extra-files`       |
| `verify-privacy-inventory` | did not exist                  | **295 columns / 25 elements / 10 surfaces** |

`verify-privacy-inventory`'s own line breaks down further, re-measured on
`pr-187` head `4e9fe73e` (2026-09-18): **295 columns, 25 elements** of which
16 are personal-data and 17 `necessary`, **10 non-database surfaces** of which
6 are not yet statically enumerable. `cargo test -p xtask` on that head is
**276 passed**, 26 of them in `privacy_inventory_tests`.

`verify-status` is still **1**; `verify-migrations` still **48**;
`check-schema` still **27**, under cratestack 0.12.0. `verify-ui` prints
nothing at all on success — exit 0 is its whole output, so there is no number
to quote and a green run is the only evidence there is.

A **red** row in a gate table is a thing this repository writes down rather
than omits ("a gate table with a green row for a red gate is worse than no
table"). If `verify-versions` is red on a branch, check whether that branch
predates #204 before looking for a cause in your own change.

`verify-links` counted **fewer** links across **more** files — that is what a
documentation split does, and a count moving down is not evidence of deletion.
PR #200's branch measured 1 711 in 397 on the same day, on a different tree;
neither is wrong.

**`verify-status` moved twice in one day and came back.** It printed **2**
partway through 2026-09-15, when RFC-0003 § 5 gave `orange_money` a `refund`
token, and **1** again by 2026-09-16, because the same day's MTN work retired
`NotImplemented("mtn_momo::refund")` by writing the Disbursements `transfer`
call. Same number, different token, different reason.

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
is not declared in `docs/status.md`; a token declared there that shipping code
no longer carries; and — **the third direction, since 2026-09-16** — a token
whose prefix names a rail this workspace ships an adapter for, carried by any
file outside that rail's crate.

**It reports ONE token as of 2026-09-16** (`orange_money::refund`) — down from
eight on 2026-09-03 and from two partway through 2026-09-15, when
`mtn_momo::refund` was retired by a real Disbursements `transfer` call. The
list has moved in **both** directions, which is the distinction it exists to
keep visible: `orange_money::refund` left on 2026-09-03 and came back on
2026-09-15 meaning something different (not "Orange has no refund API" but
"vpay has not written Orange's transfer").

**Implements:** `cargo xtask verify-status`, `verify_status` /
`declared_tokens` / `scan_not_implemented` in `.xtask/src/main.rs`.

**How you trip it:** leaving a `NotImplemented` without adding the bullet; or
retiring one and forgetting to remove the bullet; or copy-pasting a `refund`
body between adapters and shipping `NotImplemented("mtn_momo::refund")` inside
`vpay-adapter-orange-money`.

**Why the third direction exists, measured.** The first two directions compare
_sets of strings_ and neither knows which file a token came from, so one
adapter answering another rail's token is invisible to both as long as the two
sets still match. Measured on 2026-09-15: with Orange's token replaced by
MTN's, the gate first failed with _"`docs/status.md` declares
`orange_money::refund` and no shipping code carries it"_ — a message that
invites exactly the wrong repair. Delete that bullet as invited and
`verify-status` printed **"ok — 1 unimplemented item(s)"**, with a whole rail's
gap gone from the status page and an adapter blaming MTN for it.
`a_rail_without_the_refund_capability_answers_unsupported` also catches that
mutation and was written for it, but it needs Docker, covers only `refund` on
the two configured rails, and is not what `AGENTS.md` points at. This check
covers every token on every rail and runs in `just verify`.

**What it deliberately does not constrain:** only prefixes that _are_ a
shipping rail code. A token named `worker::poll` or `ledger::post` is
unconstrained — the repository has no convention about where such a token may
live, and inventing one in a gate would be a rule about characters rather than
about a rail.

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

The recipe's own header says "Eleven numbered checks over twelve greps"; count
the body and you get twelve, because the numbering is non-contiguous — 6 was
deleted, 7a split three ways, and 5b and 7b-ii were added. **Count the greps in
the recipe, not the header**, which is the authority rule applied to the very
file it is about. Each check accumulates into `fail` and the
recipe `exit $fail`s at the end, so it reports **every** violation, not just
the first. The numbering is non-contiguous — check 6 was deleted, 7a was split
three ways, 5b and 7b-ii were added. Read the recipe body, not its header
paragraph.

| #      | Refuses                                                                                                                                                                                                                                                                                                                                | Scope                             |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| 1      | a hard-coded palette colour: `(bg\|text\|border\|ring\|fill\|stroke\|from\|via\|to\|decoration\|outline\|shadow\|accent\|caret\|divide\|placeholder)-(red\|green\|blue\|amber\|slate\|…\|black\|white)`                                                                                                                                | `frontends/apps`, `examples/shop` |
| 1b     | an arbitrary colour value: the same prefixes followed by `-[#`, `-[rgb`, `-[hsl`, `-[oklch`, `-[color-mix`                                                                                                                                                                                                                             | same                              |
| 2      | a daisyUI 4 class daisyUI 5 removed: `form-control`, `label-text`, `label-text-alt`, `btn-group`, `input-group`, `card-compact`, `input-bordered`, `select-bordered`, `textarea-bordered`, `tabs-bordered`, `tabs-lifted`, `tabs-boxed`                                                                                                | same                              |
| 3      | `!important`, outside three documented exemptions                                                                                                                                                                                                                                                                                      | same                              |
| 4      | a `cva(` call outside a component package                                                                                                                                                                                                                                                                                              | `frontends/apps`, `examples`      |
| 5      | importing `@vpay/ui` — the package was **deleted 2026-09-12**. Covers `from`/`import`/`require(`/`resolve(`, `@import`, and a `"@vpay/ui":` dependency key                                                                                                                                                                             | tree                              |
| 5b     | path-reaching into it: `"../ui/src"`, `"frontends/packages/ui"`                                                                                                                                                                                                                                                                        | tree                              |
| 6      | **deleted.** It was a 200-line ceiling on `@vpay/ui` files; after the package went, its pathspec matched nothing, the loop body never ran and `fail` stayed 0. A check that cannot fail reads as coverage it no longer provides                                                                                                        | —                                 |
| 7a-i   | a computed class name: `className\s*=\s*{`                                                                                                                                                                                                                                                                                             | `frontends/apps`                  |
| 7a-ii  | a status-colour theme token: `(bg\|text\|border\|ring\|fill\|stroke\|divide\|outline\|shadow)-(state-[a-z]+-(fg\|bg\|border)\|destructive(-foreground)?)`                                                                                                                                                                              | `frontends/apps`                  |
| 7a-iii | a class attribute over 60 characters: `className="[^"]{61,}"`                                                                                                                                                                                                                                                                          | `frontends/apps`                  |
| 7b     | a daisyUI **component** class. The alternation deliberately omits `table`, `select` and `mask` (real Tailwind utilities — `table-fixed`, `select-none`, Tailwind 4's `mask-*`) and omits `collapse` outright, because daisyUI's `.collapse` and Tailwind's `visibility: collapse` are the same token and a grep cannot tell them apart | `frontends/apps`                  |
| 7b-ii  | `table`/`select`/`mask` when daisyUI-modifier-suffixed or bare. One exemption: `frontends/apps/checkout/src/components/locale-switch.tsx`                                                                                                                                                                                              | `frontends/apps`                  |

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

## 13. `verify-versions`

_The thirteenth gate, 2026-09-17 ([#201](https://github.com/vaam-apps/vpay/pull/201))._

**Refuses:** a release-please-owned version reference that disagrees with the
others (`.release-please-manifest.json`'s `"."`, each `json` extra-file's
`$.version`, and every `x-release-please-version`-annotated line in a `generic`
extra-file); a file listed in `release-please-config.json`'s `extra-files` that
carries **no** `x-release-please-version` line at all, because the `generic`
updater rewrites only annotated lines, so an unannotated entry is a version
that has silently stopped being bumped; an annotated line with no semver on it
to replace; and — the direction that actually bites — **an internal Cargo
`version = "…"` pin with no annotation**, in _any_ `Cargo.toml` in the tree,
not only the listed ones.

**Implements:** `cargo xtask verify-versions`, `verify_versions` and
`release_please_extra_files` in `.xtask/src/main.rs`.

**Why the Cargo half exists, in one sentence you should not have to rediscover:**
`deny.toml`'s `[bans] wildcards = "deny"` forces every internal dependency to
carry `version = "X.Y.Z"` beside its `path`; a bare `"0.1.0"` is `^0.1.0`, and
a 0.x caret range does not cross a minor boundary. There are **fourteen such
pins as of 2026-09-17** — eleven in the root manifest and three in member
manifests, the latter findable only by running `cargo metadata`, not by
reading. Miss one and the release pull request does not look untidy, it **fails
to resolve**: `failed to select a version for the requirement vpay-core =
"^0.1.0"`.

> **The `extra-files` entry shape is load-bearing, and the obvious spelling is
> the trap.** A **bare string** is refused outright. release-please's `base.ts`
> does not give a bare string the annotation-only `Generic` updater — it infers
> one from the extension: `.json`/`.yaml`/`.yml`/`.toml`/`.xml` each get a
> _typed_ updater composed with `Generic`, and a typed updater **reparses and
> re-serialises the document**, destroying every comment in it. Measured on
> vpay's own v0.1.1 release (2026-09-17/18): `deploy/helm/vpay/Chart.yaml` went
> from 48 lines to 13, its `version:` was **downgraded** 0.2.0 → 0.1.1 because
> `$.version` is the top-level key, and `appVersion` — the field actually
> annotated — was left alone, because the annotation had just been serialised
> away. `pubspec.yaml` lost its comments the same way. The only form this
> repository allows is `{"type": "generic", "path": …}`
> ([#204](https://github.com/vaam-apps/vpay/pull/204)).

**How you trip it:** adding a fifteenth internal dependency and annotating
nothing — an ordinary thing to do that gives no hint it has armed the next
release. Or adding a `.yaml`/`.toml`/`.json` extra-file as a bare string.

**What it deliberately does not check:** an annotated line in a file the config
does **not** list. That needs a whole-tree walk, and writing an annotation
while never touching the config is not a mistake anyone has made.

**A gate is allowed to be right about a broken tree.** `verify-versions` went
red one commit after it landed, on `master`, for exactly the regression it was
built for — and vpay recorded that rather than working around it. If you find
it red on a branch, check whether the branch predates #204's repair before
looking for a cause in your own change.

The bare-string refusal shipped with #204 and its **tests** arrive in
[#206](https://github.com/vaam-apps/vpay/pull/206), which also moves
`AGENTS.md`'s own gate count to thirteen; it reaches fourteen when #187 lands.
So on a tree between those two merges, `AGENTS.md` and the recipe disagree —
see the box at the top of this page for how far that went.

## 14. `verify-privacy-inventory`

_The fourteenth gate, from [#187](https://github.com/vaam-apps/vpay/pull/187)
(issue #144, ADR-0020, RFC-0002). Written on its branch as the thirteenth and
renumbered when `verify-versions` landed first. **Read from `pr-187` head
`4e9fe73e` on 2026-09-18, after its review and before it merged** — confirm
against the recipe._

**Refuses, completely** — this is the whole list:

- a column any `backends/migrations` file creates that **no element in
  `schemas/privacy-inventory.yaml` classifies**;
- an inventory `column` copy naming a table/column **no migration creates**
  (stale or misspelled);
- an element whose `subject`, `purpose`, `tenant_boundary`, `retention`,
  `owner` or `control` is **empty**. Those are the six string fields of
  ADR-0020 §1's eight; `necessary` is a boolean and `recipients` a list, so
  "present" is all either can be;
- a `control` outside `{redact, none, forbid}` or a `subject` outside
  `{payer, staff, merchant, none, system}` — a typo like `redcat` would
  otherwise classify a column as protected when it is not;
- a copy whose `kind` is not `column`, a `column` copy missing its table or
  column, or a file whose `version` is not `1`;
- a **registered** non-database surface with a duplicate id, an id colliding
  with an element name, or an empty `surface`/`description`. (It cannot refuse
  an _unregistered_ one — nothing derives that list. The count of surfaces
  "not yet statically enumerable" is printed, not enforced.)
- **a migration whose DDL the parser cannot read** — by name, see below.

**Implements:** `cargo xtask verify-privacy-inventory`, `verify_privacy_inventory`
in `.xtask/src/main.rs`.

**It parses the migrations themselves,** and that is the decision to remember.
`schemas/vpay.cstack` is a _projection_ — it deliberately models less than the
whole database — so the schema file is not the authoritative surface and a
manifest would be a second artifact that can itself drift. The parser is
string-aware and models `CREATE TABLE`, `ALTER TABLE … ADD/DROP/RENAME COLUMN`
and `DROP TABLE`, so the derived set is the **final** schema, not the union of
every column ever written.

> **DDL the parser cannot read is refused BY NAME, not skipped** (hardened
> `c4c542f3`, 2026-09-18). `AS SELECT`, `PARTITION OF`, `INHERITS` and a
> `CREATE TABLE` with no column list each fail naming the file and the table;
> a `(LIKE other)` body fails with its own message (it parses, but yields no
> columns, and "a table with no columns is one the inventory can never be
> asked about"); and `ALTER TABLE … RENAME TO` fails too, because every column
> would stay filed under the old name and the inventory would have to keep
> naming a table that no longer exists in order to pass.
>
> **Why a refusal and not a skip:** an unreadable table derives as nothing, and
> then _both_ directions agree about a table neither can see — direction A
> cannot report columns it never derived, and direction B has no stale row
> because an honest author classified none. **A green gate over an
> unclassified table is the one thing this gate exists to make impossible**,
> and that is also the limit of the both-directions property in general: the
> two directions share one oracle, so when the oracle is silent they agree.

> **The keyword match was one literal space, and every way it was wrong failed
> OPEN.** Until `c4c542f3` the parser did `stmt.to_uppercase().find("CREATE
TABLE")`. Measured, each against that parser: `CREATE  TABLE` (two spaces),
> `CREATE\tTABLE`, or a `CREATE` left at the end of a wrapped line **matched
> nothing at all** — the whole table, every column, invisible to both
> directions; `ALTER  TABLE t ADD COLUMN email` silently lost `email`, and
> `DROP  TABLE t` silently left `t` standing, reopening the exact defect
> `drop_table_targets` had been added to close. There was **no word boundary**
> (`RECREATE TABLE` contains `CREATE TABLE`) and **no string-literal
> awareness** — and these migrations' `COMMENT ON` bodies are essays _about
> migrations_, so one containing the words `DROP TABLE customers` would have
> removed the real table from the derived set. The only thing keeping the old
> spelling correct was that all 48 migrations happen to be typed with exactly
> one space, and nothing in this repository reformats SQL. `kw_end` now matches
> whitespace-tolerantly, at a word boundary, outside string literals — and on
> the original bytes rather than an uppercased copy, because `to_uppercase` is
> not length-preserving (`'ﬁ'` is three bytes, `"FI"` is two) and offsets from
> the copy were indexing the original.

**Schema qualification is dropped**: `authkestra.oauth_*` is filed under its
bare name, which is the spelling the inventory uses. Two tables of the same
bare name in different schemas would merge into one entry, and a column of
either would satisfy a classification written for the other. No such pair
exists (31 created tables, 31 distinct bare names, as of 2026-09-18).

**How you trip it:** adding a migration with a new column and not adding the
`kind: column` copy to an element in `schemas/privacy-inventory.yaml`. Dropping
a column and leaving its copy behind trips the other direction — and dropping a
**table** without deleting its rows is the same failure with more rows; that is
a real hole this gate's own review found, on migration 0009's
`merchant_api_keys`.

**The file's shape**, so you can add a row without opening it: `version: 1`,
then `elements:` keyed by a stable element name, each carrying the eight
classification fields, then `copies:` — a list of
`{kind: column, table: …, column: …}`. Then `non_db_surfaces:`, a list of
registered disclosure surfaces (**10 as of 2026-09-18, 6 of them
`enumerable: false`**, which the gate counts and prints but does not enforce).
The published prose companion is `docs/reference/personal-data-inventory.md`.
The gate's own behaviour is pinned by **26 mutation cases as of 2026-09-18**
(16 before the parser hardening the same day) in `mod privacy_inventory_tests`.

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
`verify: ok` line the recipe prints afterwards means the fourteen gates passed
and says nothing about the numbers `verify-docs` printed.

## `verify-citations` — a gate, but not one of the fourteen

`just docs-check-citations` → `cargo xtask verify-citations`. ~~The thirteenth
gate.~~ **Renumbered 2026-09-18:** `verify-versions` and
`verify-privacy-inventory` took the thirteenth and fourteenth places, and this
one was never in the list they are in anyway. `justfile`'s own header calls it
"a sixteenth check", counting `verify-docs` as the fifteenth thing `just
verify` prints. It is **not** in `just verify` and **not** in `just ci`,
because it needs the network and an authenticated `gh`. It is deliberately excluded from `cargo xtask verify-all`
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
