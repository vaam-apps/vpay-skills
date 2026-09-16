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
   reads when deciding whether to load the skill: say what it covers *and* when
   to reach for it. Under 80 characters fails the gate.
2. Keep `SKILL.md` to roughly one sitting — about 100–150 lines. Depth goes in
   `references/*.md`, and every reference page must be linked from `SKILL.md`
   (the gate refuses orphans, because a page nothing links is a page no agent
   opens).
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

**When you correct something, say what it said before.** vpay's documents are
full of "this said X until date Y and was wrong", and that is not
self-flagellation — it is the only signal a reader has about which sentences on
a page have been checked recently. A silent correction teaches nobody.

## When vpay changes

The parity rule in [README.md](README.md) is the contract: code, docs, skills,
or it has not landed. In practice:

| The vpay change                    | What has to happen here                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------ |
| A new `docs/flows/` page           | A skill claims it in `coverage.json`, with prose that covers it                      |
| A route mounted or unmounted       | `vpay-merchant-api` or `vpay-dashboard`, whichever owns the surface                  |
| A `NotImplemented` token retired   | Every skill that described it as unbuilt — grep for it                               |
| A new gate, or a gate that changed | `vpay-tooling`, and `vpay-troubleshooting` if it has a non-obvious failure mode      |
| A toolchain pin bumped             | `vpay-tooling`                                                                        |
| An SDK capability added            | `vpay-sdks`, and the parity row in vpay itself                                       |
| A path renamed or moved            | Whatever `verify-coverage` says — that is exactly the half of the gate that catches it |

## Reviewing

The question to ask of a diff here is the one vpay asks of its own: **does
anything I wrote imply something works that I have not seen work?**

For a skill there is a second: **does anything I left out lead an agent
somewhere expensive?** Omission is the characteristic defect of a briefing, and
it does not show up in review the way a wrong sentence does.
