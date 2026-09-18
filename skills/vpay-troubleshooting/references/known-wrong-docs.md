# Known-wrong documentation, and which source wins

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

vpay documents itself exhaustively. The prose drifts faster than anything else
in it, and **the repository knows this and records it in place.** Learning to
read those records is the difference between an agent that trusts a stale
sentence and one that checks.

## The operational rule

> **When prose and a machine-checked source disagree, the machine-checked source
> wins — and you fix the prose in the same commit.**

`AGENTS.md` says this about itself, in the paragraph describing `just verify`:

> The recipe is the list; this paragraph is a description of it, and **it has
> gone stale at nearly every count it has carried**… If that list and the recipe
> disagree, **the recipe is right: read it, and fix this paragraph in the same
> commit.**

Which source is "machine-checked" depends on the claim:

| A claim about                    | The authority                                                                   |
| -------------------------------- | ------------------------------------------------------------------------------- |
| what a recipe does               | `just --show <recipe>` / `just --evaluate`, never the comment above it          |
| which gates run                  | the `verify` recipe's dependency list                                           |
| what is built                    | `docs/status.md` — and only because `verify-status` reads it                    |
| a flag, its env var, its default | `cargo run -p vpay-server -- --help` on the running binary                      |
| a toolchain version              | `rust-toolchain.toml`, which CI reads                                           |
| an exemption                     | ADR-0016's table, which `verify-serde` reads in both directions                 |
| an SDK capability                | `docs/sdks/parity.md`, which `verify-sdk-parity` reads                          |
| everything else in `docs/`       | **prose.** `docs/README.md`: "A document here is prose unless a gate reads it." |

`README.md` states the binary case explicitly: run `--help`, "that is more
trustworthy than any doc if the two disagree." `docs/flows/configuration.md`
repeats it about its own flag table.

## The three correction shapes

Learn to spot these. Each one is a sentence somebody checked on a date.

1. **`~~struck through~~`** — a claim that was true and stopped being true. Kept
   rather than deleted "because this chart was written against the earlier state
   and a reader comparing the two should see the gap close rather than wonder
   whether it was ever real."
2. **`**Corrected <date>:**`** — the replacement, dated.
3. **`this said "X" until <date> and had been wrong since <date>`** — two dates,
   because _when it broke_ and _when it was noticed_ are different facts.

`docs/README.md` defends the habit outright: those paragraphs "are the reason
the documents are worth trusting", and when the eleven longest documents were
split on 2026-09-11 the split was judged by a whitespace-tolerant diff proving
**zero lines missing** — no struck-through claim or dated measurement could be
dropped or paraphrased in the move.

`CONTRIBUTING.md` in this skills repo carries the same rule: "When you correct
something, say what it said before… A silent correction teaches nobody."

## The census

A non-exhaustive list of sentences this repository has caught being wrong. The
point is not the individual facts — it is the _rate_.

| Where                                   | What it said, and for how long                                                                                                                                                                                                                                                                                                                                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `AGENTS.md`, the gate count             | "three gates" until 2026-09-05, wrong since 2026-09-03. Then "five" for a day. Then seven, nine, ten, **twelve** — each correction dated, with the branch that caused the drift named. **Thirteen** on 2026-09-17 (#201) and **fourteen** on 2026-09-18 (#187), and this time the prose did not follow: `master` ran thirteen while `AGENTS.md`, `docs/status.md` and the `justfile` header all still said twelve. |
| `CLAUDE.md`, the same count             | "three" until 2026-09-06, wrong since 2026-09-03; "ten" until 2026-09-07. It now defers: "**AGENTS.md carries the count… that is the copy to trust, and this one now agrees with it.**"                                                                                                                                                                                                                            |
| `CLAUDE.md`, `schemas/vpay.cstack`      | Said "**not** wired into the build, its syntax is unverified, do not try to make it compile". **Wrong in two stages** — syntax checked from 2026-09-05, _compiled_ from 2026-09-06.                                                                                                                                                                                                                                |
| `AGENTS.md`, TypeScript conventions     | Named Headless UI, framer-motion and vaul — "none of which is a dependency of any `package.json` in this repository". Corrected 2026-09-07, **and again 2026-09-12** when `@vpay/ui` was deleted.                                                                                                                                                                                                                  |
| `docs/flows/configuration.md`           | "said 'neither binary calls it' until 2026-09-02; that had been false since 2026-08-11."                                                                                                                                                                                                                                                                                                                           |
| `docs/flows/configuration.md`           | A "correction of the correction": an earlier pass said the DB-CHECK claim was false. True of the `.cstack` grammar, not of the raw SQL schema.                                                                                                                                                                                                                                                                     |
| `docs/flows/errors.md`                  | `~~verify-errors counts 14 error types~~` — corrected 2026-09-04 to 15.                                                                                                                                                                                                                                                                                                                                            |
| `README.md`                             | `~~a missing --database-url exits 1~~` — corrected 2026-09-10.                                                                                                                                                                                                                                                                                                                                                     |
| `deploy/helm/vpay/README.md`            | The guard count "**said 15 until 2026-09-10**" and had been wrong since the sixteenth landed. It is 22.                                                                                                                                                                                                                                                                                                            |
| `deploy/helm/vpay/README.md`            | Two `~~struck-through~~` Status bullets ("the liveness probes point at a listener that does not exist", "every PrometheusRule query names a metric no build emits") — both corrected the same day.                                                                                                                                                                                                                 |
| `deploy/helm/vpay/README.md`            | "an earlier draft of this section claimed [the render] was byte-identical — that claim was true of the Gateway API work alone and stopped being true when the rail callback fix landed."                                                                                                                                                                                                                           |
| `docs/runbooks/demo/known-flake.md`     | `~~Not fixed in this step.~~` → fixed; then `~~After the fix, just demo ran green~~` — "**Corrected 2026-09-04: no demo run after the fix is recorded.**"                                                                                                                                                                                                                                                          |
| `config/application.yml`                | The MTN host "used to say `wiremock`, a host no compose file defines — **never caught because the stack had never been started**."                                                                                                                                                                                                                                                                                 |
| `docs/status.md`                        | The load-bearing "no HTTP call to a real rail has ever been made" sentence, narrowed by ten dated addenda and finally retired 2026-09-15 — and **replaced with a narrower one**, not deleted.                                                                                                                                                                                                                      |
| `docs/plans/exp11-notes/opus-review.md` | A whole review finding whose content is: `CLAUDE.md`'s "Things that will waste your time" still said the toolchain pin is `1.95.0`.                                                                                                                                                                                                                                                                                |

Repository-wide, `grep -c "and was wrong\|Corrected 2026\|~~"` over `docs/`
gives double digits on `docs/status/backend.md`, `docs/status/merchant-sdks.md`,
`docs/status/cratestack.md`, `docs/status/frontend.md`,
`docs/status/infrastructure.md`, `docs/flows/deployment.md` and
`docs/adr/0017-staff-authentication.md`.

**The best worked example, and it is now closed.** `AGENTS.md` and
`audit-web`'s own comment both said `audit-web` was **not** in `just ci` and
that `just ci` worked offline, while the `ci:` recipe listed it — wrong since
2026-09-11, found 2026-09-16 by reading the recipe, fixed the same day. The
prose was wrong in five places at once (two `justfile` comments, the `ci`
recipe's coverage list, `package.json`'s `//pnpm` block, and `AGENTS.md`),
which is the shape to expect: one recipe change, several descriptions of it,
and no gate that reads any of them. See **vpay-tooling** for what is true now.

## Two kinds of number you must not read as current

**`docs/plans/*` are dated records, never current truth.** `docs/README.md`
lists them as "dated records, not current truth", and the 2026-09-11 split
deliberately did not touch that directory: "those are dated records of what was
run on a day, and **rewriting one falsifies it**." A test count, a container
count or a load average in a plan note describes one tree on one machine on one
afternoon.

`docs/status.md` refuses to restate one even when it would be convenient: "That
244 is a dated measurement of the pre-rebase tree and is **deliberately not
restated**."

**A gate's "last printed" column is one tree on one day.** `docs/status.md`'s
gate table carries what each gate printed on a named commit, and the page says
so: "Those numbers are a measurement of one tree on one day, not a promise."
`docs/status/gates.md` goes further — read its numbers "as date-stamps rather
than totals: it calls `verify-sdk-parity` the reader of 'the fourth
machine-checked document', which was true on 2026-09-03, when four of the
fourteen gates there are now (2026-09-18) existed."

## Known-stale links

`verify-links` checks a destination **path** and never a `#anchor`. So a link to
a heading that has been renamed, or that moved to another page, **still passes
the build**. A 2026-09-11 sweep found four stale anchors; three were fixed and
**the fourth is named in `docs/README.md` rather than quietly left** —
`docs/flows/invoices.md` links to a "What is not built" heading that file does
not have.

**Practical consequence:** if you rename a heading, re-read the anchors pointing
into that document. Nothing will tell you. Four doc comments in
`backends/crates/vpay-db/src/` link to `vpay-db.md#cratestack`, which is why
that heading is kept on purpose.

## What to do when you find one

1. Believe the machine-checked source. Measure it if you can.
2. Fix the prose **in the same commit as whatever you were doing**.
3. **Say what it said before, and since when.** Use one of the three shapes.
   Do not silently overwrite.
4. If it is a citation (a CI run id, a PR, an issue) that does not resolve:
   `AGENTS.md` — "A citation that does not resolve is a false claim: strike it
   through with a dated correction. **Do not replace it with an id you have not
   checked.**" `cargo xtask verify-citations` (`just docs-check-citations`)
   checks these; it needs the network and a GitHub token and is **not** in
   `just verify`. See **vpay-docs-status**.
