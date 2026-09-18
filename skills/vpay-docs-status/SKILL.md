---
name: vpay-docs-status
description: How to finish a change in vpay — which status page takes your row, how to update a flow doc's Status section, when to write an ADR versus a reference page, the NotImplemented declaration shape that verify-status reads, and the docs↔skills parity rule. Load this whenever you are about to commit, open a PR, mark something done, retire a NotImplemented token, or write any document in the vpay repository.
---

# Finishing a change in vpay

> **Verified against vpay `0799a8d2` (2026-09-18).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

Most repositories treat documentation as the thing you do after the work. Here
it is part of the work, two of the gates read it, and the review question that
matters is asked of the prose as much as the code:

> **Does anything I wrote imply something works that I have not actually seen
> work?**

## The finishing sequence

```bash
just fmt
just ci        # includes just test-doc — nextest runs zero doctests
```

Then, **in the same commit as the code** (a status page that lags is worse than
none, because people trust it):

1. **The status page your change belongs to** — see the routing table below.
2. **The relevant `docs/flows/*.md` Status section.**
3. **A dated page under `docs/status/verification/`** carrying your `just ci`
   evidence.
4. **The skill in [vpay-skills](https://github.com/vaam-apps/vpay-skills)** —
   the parity rule, below.
5. **In your summary to the user: what you did _not_ do.** Explicitly.

## Where a new row goes

`docs/status.md` is the current-state page and is short on purpose (~290 lines,
down from 6 151 on 2026-09-11). **Only four things belong on it**: the banner,
the table of areas, the gate table, and the `NotImplemented` declaration — which
lives there because `cargo xtask verify-status` reads that file by name.
Everything else is an archive page under `docs/status/`.

| Your change                                 | The page                                                                                       |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| A capability, a route, an adapter behaviour | `docs/status/backend.md`                                                                       |
| A browser or dashboard change               | `docs/status/frontend.md`                                                                      |
| An image, chart, compose file, migration    | `docs/status/infrastructure.md`                                                                |
| A schema or CrateStack change               | `docs/status/cratestack.md` (+ a dated page in `status/cratestack/` if you measured something) |
| An SDK change                               | `docs/status/merchant-sdks.md` **and** `docs/sdks/parity.md` (a gate)                          |
| A new gate, or a gate that grew a direction | `docs/status/gates.md`, **and** the gate table in `docs/status.md`                             |
| Your `just ci` evidence                     | a dated page under `docs/status/verification/`, newest first                                   |

Six flow pages are an overview **plus a directory** since 2026-09-11. The
**Status** section stayed on the overview in all six. In `webhooks.md` and
`dashboard.md` it is a summary plus an index — put your evidence on the page it
points at, not on the index.

## Declaring a NotImplemented token

Unwritten code returns `ProviderError::NotImplemented("<crate>::<fn>")` — never
a plausible success, an empty list or a zero. Every token in shipping code needs
a bullet under `docs/status.md` § "Unimplemented items tracked by
`verify-status`", and the gate **fails in both directions**: an undeclared token
fails, and so does a bullet naming code that no longer carries one.

**The bullet must not start with a backticked path.** The gate reads exactly
that shape — `- ` then a backtick — as a declared token. A bullet that opens
with a backtick but names no shipping token fails the docs→code half. So open
every bullet with a noun:

```markdown
- `orange_money::refund` — vpay has not written the transfer…   ← a declaration
- the column `refunds.fee` (migration `0031`), nullable…        ← prose, not a declaration
```

_This example named `mtn_momo::refund` until 2026-09-16. That token was
**retired** on 2026-09-15 when the Disbursements `transfer` call was written,
so copying it would have declared a token no shipping code carries — which
fails the gate's docs→code half. `orange_money::refund` is the live one:
**there is exactly one declared token as of 2026-09-16**, down from eight on
2026-09-03 and from two partway through 2026-09-15._

That is the gate working, not a trap. It is how `docs/status.md` can discuss
unpopulated fields in the same section without declaring them unbuilt.

**The scanner lexes.** Since 2026-09-05 a token written in a comment of any
kind, in a `#[doc = "…"]` attribute, or inside any string, raw-string or
character literal is prose and declares nothing. You never have to strip an
honest sentence from a doc comment to keep this gate green.

Tests for unbuilt behaviour: `#[ignore = "not implemented: … — see
docs/status.md"]`. But note that `#[ignore]` is effectively banned —
`just verify-ignored` asserts `expected_ignored = 0`. The sanctioned way to mark
a test that needs a running stack is a **Cargo feature** (`required-features =
["live-stack"]`), not `#[ignore]`.

## Which tier does this document belong to?

Putting a document in the wrong tier is a real review comment here.

| What you have                                  | Where it goes                      | Mutability                            |
| ---------------------------------------------- | ---------------------------------- | ------------------------------------- |
| A decision that has been taken                 | `docs/adr/`                        | **Immutable — supersede, never edit** |
| A process: what happens, in what order         | `docs/flows/`                      | Edited as the process changes         |
| Why the _code_ is shaped the way it is         | `docs/reference/<crate>.md`        | Edited as the code changes            |
| A proposal under discussion                    | `docs/rfc/`                        | Until decided                         |
| Something an on-call person must do            | `docs/runbooks/`                   | Freely                                |
| What is actually built, with evidence          | `docs/status/`                     | Append; nothing is deleted            |
| What an **agent** must know before changing it | a skill in `vaam-apps/vpay-skills` | Freely                                |

Long reasoning does **not** go in an 80-line module header. It goes in
`docs/reference/<crate>.md`, with the module carrying one paragraph and a link.
A module doc that _is_ a document uses `#[doc = include_str!("…md")]`.

## Two habits that are not optional here

**Say what it said before.** When you correct a document, strike the old claim
through and date the correction. This looks like clutter and is the most
valuable thing on the page: it tells a reader which sentences have been checked
recently and what the plausible-but-wrong belief was. Several of this
repository's own headline paragraphs were wrong for days while reading
perfectly — AGENTS.md said "three gates" until 2026-09-05 and had been wrong
since 2026-09-03.

**A document that outgrows one sitting becomes an overview and a directory** —
never a shorter document. Nothing may be dropped or paraphrased in the move,
because the dated measurements and struck-through claims are what make the page
worth trusting.

## Links and citations

`cargo xtask verify-links` (in `just verify`) requires every relative link in a
tracked `*.md` to resolve to a **`git ls-files`-tracked** path — so a link
satisfied by a file you created and never `git add`ed fails the build rather
than passing on your machine alone. It masks fenced code, inline code and HTML
comments, and ignores `http(s)`/`mailto:`.

**It checks the destination path and never the `#anchor`.** Rename a heading and
every anchor into it still passes. Grep for them yourself; a sweep on
2026-09-11 found four stale ones.

`cargo xtask verify-citations` (`just docs-check-citations`) resolves every CI
run id, PR and issue a document cites. It is **not** in `just verify` or
`just ci` — it needs the network and an authenticated `gh` — and it **fails**
rather than skipping when `gh` is missing, because a downgraded check is
indistinguishable in a log from a passing one. Run it when you add or edit a
document that cites one. A citation that does not resolve is a false claim:
strike it through with a dated correction. Do not substitute an id you have not
checked.

## The docs↔skills parity rule

> **A feature lands in three places or it has not landed: the code, the docs,
> and the skills.**

The reasoning is rule 2 with a force multiplier. A status page that lags is
worse than none because people trust it; a skill that lags is worse still,
because an agent does not merely trust it — it acts on it, at machine speed, in
every session that loads it.

Full routing table, the gate, and how to write the companion PR:
`references/skills-parity.md`.
