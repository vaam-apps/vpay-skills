# Versioning

> **A skill is true of a vpay, not of vpay.**

This is the failure mode this page exists to prevent:

> An agent loads `vpay-merchant-api`, reads that `GET /v1/refunds/{id}` is
> routed, and writes a merchant integration against it. The claim is true of
> vpay's `master`. The merchant is pinned to a commit from before
> [#45](https://github.com/vaam-apps/vpay/pull/45) landed, where that route
> returns `404 unknown_route`. Nothing in the skill said when it became true, so
> nothing warned anyone.

The inverse is just as bad and harder to spot: a skill that still describes
something the current vpay has removed, which an agent then faithfully
reproduces.

## What a vpay "version" actually is

~~**There is none, in the usual sense.** As of 2026-09-16: `Cargo.toml`'s
workspace `version` is `0.1.0` and has never moved; the repository carries
**no release tags**; images are published by digest and by `github.sha`.~~

**Corrected 2026-09-23 — vpay has real releases now, and has since
2026-09-17.** That paragraph was true when it was written and stopped being
true the next day. Measured against vpay `b747e5d5`:

- **Nine release tags**, `v0.1.1` (2026-09-17) through `v0.5.0`
  (2026-09-23), every one on `master`. They are cut by release-please
  (`release-please.yml`, since vpay#201, 2026-09-17) from Conventional-Commit
  history, and they are **lightweight** tags — `git cat-file -t v0.5.0` says
  `commit`.
- `Cargo.toml`'s workspace `version` is `0.5.0`, and release-please moves all
  twenty-four annotated version lines together (`cargo xtask verify-versions`
  fails if one disagrees) — the SDKs, the Flutter and Tauri plugins and the
  Helm chart carry the same number.
- A `v*` tag makes `release.yml` publish images under the version and its
  `major.minor` (`0.5.0`, `0.5`) beside `sha-<40 hex>`; a push to `master`
  publishes `edge`.

**So a vpay version is still a commit for the purpose of stamping, and the
calendar is still how a claim is dated** — but for reasons that survive the
tags, not because there are none:

- **Skills are verified against `master`, which is between tags most of the
  time.** `b747e5d5`, the commit this correction was verified against, is four
  commits after `v0.5.0` and in no release (`git describe --tags b747e5d5` →
  `v0.5.0-4-gb747e5d`). A tag-only stamp could not name the tree that was read.
- **A commit on `master` cannot move; a lightweight tag can** be deleted or
  re-pointed with no trace in history. The gate's `merge-base --is-ancestor`
  check is only as stable as the ref it is given.
- **A reader's checkout is between tags as well.** A dated claim ("since
  2026-09-17") answers them directly; a tag answers them only after they work
  out which tag contains their commit.

What the tags **do** buy is a human-readable anchor. When a stamp or a
changelog entry is written, it is fine — and useful — to say where the commit
sits: "vpay `b747e5d5` (after `v0.5.0`)". `git describe --tags <commit>` gives
that in one line, and `git tag --contains <commit>` tells a reader which
releases include a claim's commit. The machine-checked stamp stays the commit.

vpay's own documents are written the same way, in dated sentences like "the
one shipping binary since 2026-09-07 (issue #77)" and "this said X until date
Y and was wrong". **Follow that convention here.** It is the whole mechanism.

## The three rules

### 1. Every `SKILL.md` names the vpay it was verified against

Directly under the title:

```markdown
> **Verified against vpay `f063ee96` (2026-09-15).** Version-sensitive claims
> below carry the date they became true. On an older or newer vpay, trust the
> repository over this page.
```

`coverage.json`'s `baseline` block carries the same ref in machine-readable
form, and `tools/verify-coverage.mjs` prints how far the checkout you gave it
has drifted from that baseline.

**A stamp may be newer than the baseline; it may never be older.** Re-verifying
one skill against a later vpay and stamping just that one is correct and
expected — the gate checks only that the stamped commit _contains_ the baseline.
Requiring all twenty stamps to move together would make a one-skill correction
cost a full re-verification pass, which is how you get twenty rubber-stamps.

> **Stamp to a commit that will still exist. Measured 2026-09-18.** The gate
> answers "does this stamp contain the baseline" with `git merge-base
--is-ancestor` **inside the vpay checkout it was given**, and CI gives it a
> clone of `master`. A stamp naming a commit on an unmerged vpay **feature
> branch** therefore passes for exactly as long as that branch exists on the
> remote, and fails the day it is deleted — with a message about the stamp
> being "older or unrelated", which is not what went wrong. Two skills stamped
> `9d83ff0e` (a merge commit on vpay's `feat/sheet-polish-192`) passed on
> 2026-09-17 and failed on 2026-09-18, changed by nothing in this repository.
>
> So: **stamp to a commit on vpay's `master`.** If the work you verified
> against has not merged yet, leave the stamp where it was and restamp when it
> has — a stamp that is honestly a little behind is worth more than one that
> is precise today and unresolvable next week.

The baseline also governs coverage. Six flows are an overview plus a directory,
and a claim on the overview covers the detail pages **that existed when the
claim was made**. A page added under a claimed directory _since_ the baseline
fails the gate by name, because inheriting the parent's claim would hide exactly
the case the gate exists to catch: a feature shipped with no briefing.

### 2. A version-sensitive claim carries the date it became true

Not "there is one shipping binary" but "**one shipping binary since 2026-09-07
(issue #77); it was two before that**".

A claim is version-sensitive if a reader on a six-week-old checkout would be
misled by it. In practice that is most claims about:

| Kind of claim         | Write it as                                                  |
| --------------------- | ------------------------------------------------------------ |
| A route exists        | "routed since `<date>` (`#<PR>`)"                            |
| A gate exists         | "the twelfth gate, from `<date>`" — gate counts change often |
| A token was retired   | "`NotImplemented` until `<date>`, real since"                |
| A package was deleted | "deleted `<date>`; nothing may import it"                    |
| A pin was bumped      | "`1.98.0` — it was `1.95.0` until `<date>`"                  |
| A count of anything   | "N as of `<date>`", never a bare N                           |

The cost of the date is six characters. The cost of omitting it is an agent
confidently generating code against a surface that does not exist on the tree
it is editing.

### 3. Say what it was before

When you correct a skill because vpay changed, **strike the old claim through
and date the correction** rather than overwriting it:

```markdown
~~`orange_money::refund` returns `NotImplemented`.~~ **Corrected 2026-09-06:**
it answers `Unsupported` — Orange's Web Payment product documents no refund API,
so this was never unbuilt work.
```

This is vpay's own house style and it is not sentimentality. It tells a reader
two things they cannot get any other way: which sentences on the page have been
looked at recently, and what the plausible-but-wrong belief was — usually the
belief they were about to form.

## Releases

This repository tags a release whenever a batch of skills is re-verified against
a newer vpay. A tag is named for the **date of verification and the vpay commit
it was verified against** — not for a vpay release tag, because the commit
verified against is usually between two of them (see above). ~~…not for a vpay
version that does not exist~~ _(corrected 2026-09-23: vpay has had release
tags since 2026-09-17)_. A `CHANGELOG.md` entry may name the nearest vpay tag
beside the commit:

```text
v2026-09-16-f063ee96
```

`CHANGELOG.md` records, per release: the vpay range covered, which skills
changed, and — most importantly — **any claim that stopped being true**, so
someone upgrading can find the thing that will break them.

## Installing, upgrading, and the pin that actually exists

```bash
npx skills add https://github.com/vaam-apps/vpay-skills --skill vpay   # install
npx skills update                                                      # upgrade
npx skills ls                                                          # what is installed
```

**`npx skills add` always fetches the default branch. There is no way to install
a tag.** Measured 2026-09-16: the `owner/repo@ref` form is accepted and
**silently ignored** — installing `vaam-apps/vpay-skills@v2026-09-16-f063ee96`
produced `main`'s bytes, and so did
`vaam-apps/vpay-skills@this-ref-does-not-exist`, which reported success. The
lockfile records no ref, only `source` and `computedHash`.

~~Check the tag whose vpay commit is nearest your own.~~ **Corrected
2026-09-16:** this page said that until the CLI was actually tested. You cannot
install a tag. The tags remain useful as a **reading** index — pair one with
`CHANGELOG.md` to see what changed — but not as an install target.

So the only pin is the **lockfile**. `skills-lock.json` records a `computedHash`
of the content installed; a project keeps that content until someone runs
`add` or `update` again. **Commit `skills-lock.json`** — on this distribution
model it is the entire version-control story, and
`npx skills experimental_install` restores from it.

**"Updated" does not mean "changed".** `npx skills update` reports
`✓ Updated <skill>` unconditionally, including when the content and the hash are
byte-identical afterwards (measured 2026-09-16). To know whether anything moved,
diff the `computedHash` in `skills-lock.json` before and after — that is the
value that tells the truth.

**If you are pinned to an old vpay**, that is exactly when you should not
silently take the latest skills, because these skills describe a newer tree.
Read `CHANGELOG.md` — in particular its "claims that stopped being true"
sections — before running `update`, and let the gate's drift line tell you how
far apart you are.

## What the gate can and cannot tell you

`tools/verify-coverage.mjs` checks that every `docs/flows/` page is claimed and
that every claimed path exists in the checkout you point it at. Run against an
**older** vpay it will legitimately fail on paths that do not exist yet — that
is not a bug, it is the tool telling you these skills are newer than that tree.

It cannot read prose. It cannot tell you that a sentence about a route is true.
Only a dated claim and a reader can do that.
