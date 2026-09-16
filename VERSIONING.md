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

**There is none, in the usual sense.** As of 2026-09-16:

- `Cargo.toml`'s workspace `version` is `0.1.0` and has never moved.
- The repository carries **no release tags** — the only tags are two working
  refs from a rebase.
- Images are published by digest and by `github.sha`.

So a vpay version is **a commit**, and the only honest way to date a claim is
the calendar. That is not a workaround; it is why vpay's own documents are
written the way they are, in dated sentences like "the one shipping binary
since 2026-09-07 (issue #77)" and "this said X until date Y and was wrong".

**Follow that convention here.** It is the whole mechanism.

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
it was verified against**, not for a vpay version that does not exist:

```
v2026-09-16-f063ee96
```

`CHANGELOG.md` records, per release: the vpay range covered, which skills
changed, and — most importantly — **any claim that stopped being true**, so
someone upgrading can find the thing that will break them.

## Installing a specific version

`npx skills add` fetches the default branch:

```bash
npx skills add https://github.com/vaam-apps/vpay-skills --skill vpay
```

What pins it is the **lockfile**, not the command. `skills-lock.json` records a
`computedHash` of the exact content installed, so a project that has installed a
skill keeps that content until someone re-runs the command. Commit
`skills-lock.json`.

**If you are pinned to an old vpay**, that is exactly the case where you should
_not_ silently take the latest skills. Check the tag whose vpay commit is
nearest your own, and read `CHANGELOG.md` between there and now.

## What the gate can and cannot tell you

`tools/verify-coverage.mjs` checks that every `docs/flows/` page is claimed and
that every claimed path exists in the checkout you point it at. Run against an
**older** vpay it will legitimately fail on paths that do not exist yet — that
is not a bug, it is the tool telling you these skills are newer than that tree.

It cannot read prose. It cannot tell you that a sentence about a route is true.
Only a dated claim and a reader can do that.
