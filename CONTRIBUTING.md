# Contributing

## What a skill in here is for

A skill is a **briefing for an agent that is about to change vpay**, not a tour
of vpay. The test of a page is not "is this accurate and complete" but "would an
agent that read only this do the right thing on its first attempt".

Those pull in different directions, and when they do, the briefing wins:

- **Prefer the caveat to the tour.** "The `refunds` table exists and nothing
  writes to it" is worth more than three paragraphs correctly describing the
  columns.
- **Prefer the failure mode to the happy path.** An agent can read the happy
  path out of the code. It cannot read out of the code that reflowing a comment
  in an applied migration stops every existing database from booting.
- **Name the gate, not the principle.** "Branch on capability values, not
  provider codes" is advice. "`if provider == \"mtn_momo\"` outside
  `vpay-adapter-*` is a defect (ADR-0002)" is actionable.
- **Say what is scaffold.** vpay's cardinal rule is that nothing may look more
  finished than it is. A skill that describes a designed-but-unbuilt feature in
  the present tense is the single most damaging thing you can write here,
  because an agent will build on it.

## Adding or changing a skill

1. `skills/<name>/SKILL.md` — frontmatter `name` (must equal the directory
   name) and `description`. The description is the **only** thing an agent
   reads when deciding whether to load the skill: say what it covers _and_ when
   to reach for it. Under 80 characters fails the gate.
2. Keep `SKILL.md` to roughly one sitting — about 100–150 lines. Depth goes in
   `references/*.md`, and every reference page must be linked from `SKILL.md`
   (the gate refuses orphans, because a page nothing links is a page no agent
   opens).

   **Four pages knowingly exceed this, decided 2026-09-16 and recorded rather
   than quietly normalised:** `vpay-mtn-momo` (370), `vpay-orange-money` (312),
   `vpay-provider-adapters` (278) and `vpay-payments` (267). Each is long
   because of a **negative** claim — no rail in this repository has ever
   returned money to anyone; nothing settles a pending refund; MTN's
   Disbursements product has never been called outside WireMock; Orange's
   `refund` is work vpay owes, not a fact about the rail.

   The reason they were not split is the sentence in the parenthesis above,
   taken seriously. A page nothing links is a page no agent opens — so a page
   that _is_ linked is a page an agent _may_ open. That is an acceptable bet
   for depth and a bad one for a warning: splitting moves a certainty into a
   probability, and the thing on the other side of the bet is an agent writing
   code that assumes a refund moved money. This repository's whole purpose is
   that nothing look more finished than it is, and the budget is a heuristic in
   service of the briefing test, not above it — the top of this file says so:
   when the two pull apart, **the briefing wins**.

   Two things stop this being a licence. It is an exception with a stated
   reason, not a raised ceiling: the number above has not moved, and a page
   long for any other reason is still over budget. And it is **temporary by
   construction** — when a refund poll ladder exists and these warnings shrink
   to a sentence each, the four pages should come back under the line. Whether
   the budget itself should move is a **maintainer decision** and is not taken
   here; the fourteen-of-twenty overrun noted below is the evidence for it.

   _Measured 2026-09-16: **fourteen** of the twenty `SKILL.md` files were
   already over 150 lines before this release, median **165.5**; after it,
   **sixteen** are, median **179.5**. So the stated budget and the practised one
   have differed for some time, and this release widened the gap. That is
   recorded as a fact about the corpus, not as permission._

3. Add or update the entry in [`coverage.json`](coverage.json).
4. Run the gate:

   ```bash
   node tools/verify-coverage.mjs /path/to/vpay
   ```

5. Update the table in [README.md](README.md) if you added a skill.

## Citing vpay

Cite paths, identifiers and gate names — they are checkable. `covers` entries
in `coverage.json` are checked to exist; prose is not, so be conservative in
prose.

Two habits carried over from vpay, both worth keeping:

**Date a claim that could go stale.** "As of 2026-09-15, `mtn_momo` is the only
rail ever called against a real sandbox" ages honestly. "`mtn_momo` is the only
rail ever called" does not.

This is not a nicety — it is the only thing standing between a reader on an
older vpay and a confidently wrong answer. A claim is version-sensitive if
someone on a six-week-old checkout would be misled by it, which in practice
means most claims about routes, gates, pins, deleted packages, retired
`NotImplemented` tokens, and **every count of anything**. Write "N as of
`<date>`", never a bare N. [VERSIONING.md](VERSIONING.md) has the full table and
the release policy.

**When you correct something, say what it said before.** vpay's documents are
full of "this said X until date Y and was wrong", and that is not
self-flagellation — it is the only signal a reader has about which sentences on
a page have been checked recently. A silent correction teaches nobody.

## When vpay changes

The parity rule in [README.md](README.md) is the contract: code, docs, skills,
or it has not landed. In practice:

| The vpay change                    | What has to happen here                                                                |
| ---------------------------------- | -------------------------------------------------------------------------------------- |
| A new `docs/flows/` page           | A skill claims it in `coverage.json`, with prose that covers it                        |
| A route mounted or unmounted       | `vpay-merchant-api` or `vpay-dashboard`, whichever owns the surface                    |
| A `NotImplemented` token retired   | Every skill that described it as unbuilt — grep for it                                 |
| A new gate, or a gate that changed | `vpay-tooling`, and `vpay-troubleshooting` if it has a non-obvious failure mode        |
| A toolchain pin bumped             | `vpay-tooling`                                                                         |
| An SDK capability added            | `vpay-sdks`, and the parity row in vpay itself                                         |
| A path renamed or moved            | Whatever `verify-coverage` says — that is exactly the half of the gate that catches it |

## Reviewing

The question to ask of a diff here is the one vpay asks of its own: **does
anything I wrote imply something works that I have not seen work?**

For a skill there is a second: **does anything I left out lead an agent
somewhere expensive?** Omission is the characteristic defect of a briefing, and
it does not show up in review the way a wrong sentence does.
