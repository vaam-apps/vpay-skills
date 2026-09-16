# The docs↔skills parity rule

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

> **A feature lands in three places or it has not landed: the code, the docs,
> and the skills.**

## Why a third place

The skills in [vaam-apps/vpay-skills](https://github.com/vaam-apps/vpay-skills)
are a documentation tier with a different audience, and therefore a different
test.

A flow doc describes a process to a reader who will then decide what to do. A
skill briefs an agent that is already doing it. So a skill is judged not on
"is this accurate and complete" but on "**would an agent that read only this do
the right thing on its first attempt**".

That difference is why they live in a separate repository. A briefing that has
to clear twelve gates to be corrected is a briefing nobody corrects. The cost
of the separation is drift, which is what the gate below exists to refuse.

The urgency is rule 2 with a force multiplier attached. A status page that lags
is worse than none because people trust it. **A skill that lags is worse still,
because an agent does not merely trust it — it acts on it, at machine speed, in
every session that loads it.** A skill describing a designed-but-unbuilt feature
in the present tense is this repository's cardinal sin, industrialised.

## What triggers a skills change

| Your change in vpay                              | The skill to change                                                              |
| ------------------------------------------------ | -------------------------------------------------------------------------------- |
| A new or deleted `docs/flows/` page              | Whichever skill claims it in `coverage.json`                                     |
| A route mounted or unmounted                     | `vpay-merchant-api`, or `vpay-dashboard` for `/dash/v1`                          |
| A `NotImplemented` token retired                 | Every skill that described it as unbuilt — grep for it                           |
| A new gate, or a gate that grew a direction      | `vpay-tooling`; `vpay-troubleshooting` if it fails oddly                         |
| A toolchain pin bumped                           | `vpay-tooling`                                                                   |
| An SDK capability added                          | `vpay-sdks`, beside the `docs/sdks/parity.md` row                                |
| An adapter behaviour, or a new rail              | `vpay-provider-adapters` and the per-rail skill                                  |
| A migration, a schema change, a repository trait | `vpay-data-layer`                                                                |
| A path renamed or moved                          | Whatever `verify-coverage` names — that half of the gate exists for exactly this |
| A new reproducible failure mode you hit          | `vpay-troubleshooting`. This one is worth doing even when nothing else changed   |

That last row is the one people skip and shouldn't. If you lost an hour to
something, the next agent will lose an hour to the same thing, every session,
until somebody writes it down.

## The gate

```bash
node tools/verify-coverage.mjs /path/to/vpay
```

It fails in **both** directions, the way `verify-status` has since 2026-09-03:

- **docs → skills.** A page in `docs/flows/` that no skill claims fails. That is
  the half that catches a feature shipping with no briefing.
- **skills → docs.** A path claimed in `coverage.json` that no longer exists in
  vpay fails. That is the half that catches a skill still describing something
  that moved or was deleted.

A one-directional gate rots in the direction nobody looks.

It also refuses a skill whose `name` does not match its directory (that name is
what `--skill` resolves), a description under 80 characters (the description is
the only thing an agent reads when deciding whether to load the skill — too
short and it never triggers), and a `references/` page that `SKILL.md` does not
link (a page nothing links is a page no agent opens).

`vpay-skills`' CI runs it against this repository's `master` daily and on every
push, so a vpay merge that outruns the skills surfaces there as a red build
rather than as a confidently wrong agent three weeks later. **Do not leave it to
the cron.** Open the `vpay-skills` PR alongside yours and link the two.

## Genuinely no skill needed?

Some pages will not warrant one. Say so explicitly rather than leaving the gate
red: add the path to `coverage.json`'s `exempt` list **with a reason** in
`exemptReasons`. The gate refuses an exemption with no reason, for the same
reason ADR-0016's serde table does — an undocumented exception is
indistinguishable from one nobody has looked at since it was written.

## Installing them

```bash
npx skills add https://github.com/vaam-apps/vpay-skills --skill vpay
```

`vpay` is the orientation skill and routes to the rest. This repository dogfoods
them: they install into `.agents/skills/` and pin in `skills-lock.json`, the
same mechanism `vaam-ui` already uses.

## Keeping them current

```bash
npx skills update          # upgrade every installed skill
npx skills ls              # what is installed, and from where
```

Three things measured on 2026-09-16 that will mislead you otherwise:

- **`update` always takes the default branch.** There is no way to install or
  hold a tag — `owner/repo@ref` is accepted and silently ignored, including a
  ref that does not exist.
- **`✓ Updated` does not mean anything changed.** It is printed
  unconditionally. The `computedHash` in `skills-lock.json` is what tells you
  whether content moved; diff it across the update.
- **`skills-lock.json` is the only pin there is.** Commit it. It is what keeps
  a project on known content, and `npx skills experimental_install` restores
  from it.

So upgrading is deliberate, not automatic: nothing re-fetches on its own, and a
project stays on the bytes in its lockfile until someone runs `add` or `update`.
Before you run either against a vpay older than the skills' baseline, read the
skills repo's `CHANGELOG.md` — its "claims that stopped being true" sections are
written for exactly that moment.
