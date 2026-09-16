# Reading vpay's documentation

vpay's documentation has conventions that will mislead you if you do not know
them. Three in particular.

## 1. Prose about counts goes stale; the machine-checked source does not

The repository records its own drift, out loud. AGENTS.md's gate paragraph said
"three gates" until 2026-09-05 and had been wrong since 2026-09-03. CLAUDE.md
said "three" until 2026-09-06, then "ten" until `verify-migrations` landed.
Both now carry the history of having been wrong.

**The rule the repository states for itself: if a prose count and the
`justfile` recipe disagree, the recipe is right.** Read the recipe, and fix the
paragraph in the same commit.

Generalise it. When a document and a machine-checked artefact disagree, the
artefact wins:

| Question                              | The authority                                          |
| ------------------------------------- | ------------------------------------------------------ |
| What gates are there?                 | the `verify` recipe in `justfile`                      |
| What is unimplemented?                | `cargo xtask verify-status`, whose output is the list  |
| Do the SDKs agree?                    | `docs/sdks/parity.md`, which is a gate                 |
| What does the schema declare?         | `schemas/vpay.cstack`, which now compiles              |
| Which toolchain?                      | `rust-toolchain.toml`, `.nvmrc`, `flutter-toolchain.toml` |
| What routes exist?                    | the router construction code, not the flow docs        |

Flow documents describe what is **designed**. `docs/status.md` describes what
is **built**. A flow page can be entirely accurate and describe nothing that
exists — that is not a defect in the page, it is the tier working as intended.
Every flow page ends with a **Status** section precisely so the two are never
confused; read it first.

## 2. "This said X until date Y and was wrong" is a feature

Several pages carry struck-through claims and dated corrections rather than
silent edits. This looks like clutter. It is the most valuable thing on the
page, for two reasons:

- It tells you which sentences have been checked recently, and which have been
  sitting unexamined since they were written.
- It tells you what the plausible-but-wrong belief was — which is usually the
  belief you were about to form.

**When you correct a document here, follow the convention: say what it said
before, and date it.** Do not quietly fix it. The repository has been burned by
counts that were wrong for days while reading correct.

When a document outgrows one sitting it becomes an overview **plus a
directory** — never a shorter document. Nothing is dropped or paraphrased in
the move. Eleven pages split this way on 2026-09-11, `docs/status.md` among
them (6 151 lines → ~290, with everything preserved under `docs/status/`).

## 3. Links are half-checked, and citations are not checked by default

`cargo xtask verify-links` (in `just verify`) checks that every relative link
in a tracked `*.md` resolves to a **tracked** file — so a link satisfied by an
untracked scratch file on your machine fails the build rather than passing
locally.

**It checks the destination path and never the `#anchor`.** A link to a heading
that has been renamed still passes. So when you rename a heading, grep for
anchors pointing into that page. A sweep on 2026-09-11 found four stale ones.

`cargo xtask verify-citations` (`just docs-check-citations`) checks that every
CI run id, pull request and issue a document cites actually exists. It is
**not** in `just verify` or `just ci` because it needs the network and a GitHub
token. Run it when you add or edit a document that cites one. A citation that
does not resolve is a false claim: strike it through with a dated correction —
do not replace it with an id you have not checked.

## The documentation tiers

Putting a document in the wrong tier is a real review comment here.

| Tier                    | Holds                                                     | Mutability                      |
| ----------------------- | --------------------------------------------------------- | ------------------------------- |
| `docs/adr/`             | A decision that has been taken                             | **Immutable** — supersede, never edit |
| `docs/flows/`           | A process: what happens, in what order, what can go wrong | Edited as the process changes   |
| `docs/reference/<crate>.md` | Why the *code* is shaped the way it is                | Edited as the code changes      |
| `docs/rfc/`             | A proposal under discussion                                | Until decided                   |
| `docs/runbooks/`        | Something an on-call person must do                        | Edited freely                   |
| `docs/status/`          | What is actually built, with dated evidence                | Append; nothing is deleted      |
| `docs/plans/`           | Per-change working notes, including review findings        | Historical                      |

A change to a *process* belongs in `flows/`. A change to how the *code
expresses* it belongs in `reference/`. Long reasoning does **not** belong in an
80-line module header — it belongs in `docs/reference/<crate>.md`, with the
module carrying one paragraph and a link.

## Where the hard-won lore actually is

`docs/plans/*/review.md` files are the adversarial-review findings from each
change. They are the densest source of "this passed every test and was still
wrong" in the repository, and they are easy to miss because `docs/plans/` looks
like an archive of process notes.

One example worth internalising: `amount: row.amount - row.fee.unwrap_or(0)`
passed all 244 of `vpay-api`'s tests. The case that catches it —
`a_reported_fee_never_moves_the_payers_amount` — was added on review, and holds
the invariant the field exists to protect: `amount` is the payer's money and is
never net of the fee.
