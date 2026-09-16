# Adversarial review — fitness for purpose and coherence

Lens: **would an agent that loaded this skill and nothing else do the right
thing on its first attempt?** A separate reviewer is verifying identifiers and
numbers against the vpay tree; this report only touches facts where the fact is
the coherence defect.

Ground truth: `/home/selast/dev/vpay/.claude/worktrees/skills-repo-setup-84f4d7`
(HEAD `f063ee96`, the baseline these skills are stamped against).

`node tools/verify-coverage.mjs <vpay-path>` **passes**, exit 0:
`20 skills, 205 vpay paths claimed, 23 feature pages, 0 exempt` —
`baseline: vpay f063ee96 (2026-09-15) — exactly the tree these skills were
verified against`.

Overall: this is a strong set. Fourteen of the twenty SKILL.md pages lead with
the load-bearing refusal, the "what is not built" material is specific and
dated, and the house voice is consistent enough that I found no marketing
padding and no generic advice worth flagging. The defects below are
concentrated in three places: the **orientation skill**, which is the one every
agent loads and is the least accurate page in the repository; the **seams
between skills that describe the same object**; and the **gate**, whose
headline claim is false for half of `docs/flows/`.

---

## 1. `vpay-payments` describes an invoice transition that does not exist, and `vpay-invoices` says so

**File:** `skills/vpay-payments/SKILL.md` § "Invoices" (line ~181)

**The problem.** It says:

> `draft → open → paid | void | uncollectible`, plus `draft → void` and a
> `DELETE` that removes a draft entirely.

`skills/vpay-invoices/SKILL.md` says the opposite, in a blockquote, having
gone and checked:

> **A draft cannot be voided.** `void_in_tx`'s `WHERE` names `'open'` alone; a
> draft is _deleted_. … **Two doc comments in the repository say otherwise and
> are wrong** …

I read the statement. `backends/crates/vpay-db/src/invoices.rs:1388`:

```
UPDATE invoices SET status = 'void', voided_at = $3, updated_at = $3 \
 WHERE merchant_id = $1 AND id = $2 AND status = 'open' AND {NO_LIVE_INTENT} \
```

`vpay-invoices` is right. `vpay-payments` has copied the exact stale doc
comment that `vpay-invoices` identifies as wrong, and states it as fact.

**Consequence.** `vpay-payments`' description advertises "the charge, refund
and **invoice** state machines" and fires on "state machine", "transition",
"status" — so an agent asked to "add voiding of unpaid invoices" loads
`vpay-payments`, not `vpay-invoices`, reads that `draft → void` is a legal
edge, and writes a handler for it. The database refuses with a `23514` at
runtime, not at review. This is defect class 1 from the master brief inside the
skills repo itself: a designed-but-absent transition in the present tense.

**Fix.** Delete `plus draft → void` from `vpay-payments`, and replace the
paragraph with a pointer: the invoice state rule belongs to `vpay-invoices` and
duplicating it is what produced the divergence. If a summary must stay, quote
`vpay-invoices`' blockquote verbatim.

---

## 2. The orientation skill tells every agent to write `#[ignore]`, which `just ci` refuses

**File:** `skills/vpay/SKILL.md`, § "The two machine-enforced rules", line 66

> Tests for unbuilt behaviour are `#[ignore = "not implemented: … — see
docs/status.md"]`.

**The tree:** `justfile` line 2580 is `expected_ignored := "0"`, and
`verify-ignored`'s body exits 1 on any mismatch:

```
verify-ignored: FAIL — expected exactly {{expected_ignored}} ignored tests;
update docs/status.md and this recipe together
```

`just verify-ignored` is step 7 of `just ci` (per `vpay-tooling`'s own chain).

**Two skills already carry the correction.** `skills/vpay-docs-status/SKILL.md`
(line 87–91): "`#[ignore]` is effectively banned — `just verify-ignored`
asserts `expected_ignored = 0`. The sanctioned way … is a **Cargo feature**
(`required-features = ["live-stack"]`), not `#[ignore]`."
`skills/vpay-tooling/references/recipes.md:69` says the same.

**Consequence, and why it is worse than it looks.** The orientation skill is
the one every agent loads. `AGENTS.md:95` carries the same wrong sentence, and
`vpay/SKILL.md` reproduced it — which is a direct violation of the rule its own
sibling `vpay-tooling` puts at the top of its page ("**The recipe body wins.**
Over its own comment, over `AGENTS.md`…"). Then consider which skills an agent
writing a test actually loads: `vpay` (wrong advice), `vpay-tooling`
(correction buried in a reference page), and `vpay-conventions` (silent).
`vpay-docs-status`, which holds the correction in its SKILL.md, has a
description scoped to _finishing_ a change — "about to commit, open a PR, mark
something done" — so it will not fire for "write a test for the unbuilt refund
path". The correction is present in the repository and unreachable on the path
that needs it.

**Fix.** In `vpay/SKILL.md`, replace the sentence with the Cargo-feature form
and the `expected_ignored = 0` fact, with a dated note that `AGENTS.md` still
says otherwise. Add one line to `vpay-conventions` § "The two rules". Consider
adding "writing a test" to `vpay-docs-status`' description, or moving the
`#[ignore]`/feature rule into `vpay-tooling`'s SKILL.md where a test author
will meet it.

---

## 3. The parity gate sees 23 of 45 `docs/flows` pages; five are claimed by nobody and cannot fail

**Files:** `tools/verify-coverage.mjs` (the `vpay -> skills` section),
`coverage.json`, and the README's "docs → skills" promise.

README:

> **docs → skills.** A page in `docs/flows/` that no skill claims fails the
> gate. That is the half that catches a feature shipping with no briefing.

The implementation enumerates one directory level:

```js
const flows = readdirSync(flowsDir)
  .filter((f) => f.endsWith(".md") && f !== "README.md")
```

`readdirSync` returns subdirectory _names_ (`customers`, `webhooks`, …), which
do not end in `.md` and are silently dropped. Measured in the checkout:

```
gate sees 23 flow pages
actually on disk: 45
```

vpay's own `CLAUDE.md` says six flows became "an overview plus a directory" on
2026-09-11 — i.e. the blind spot covers precisely the pages where _new_
material has been landing since the restructure.

`coverage.json` partly compensates by claiming four directories wholesale
(`docs/flows/customers`, `dashboard`, `dashboard-auth`, `hosted-checkout`).
Two are not claimed at all, so five pages are unclaimed by any skill **and**
invisible to the gate:

- `docs/flows/webhooks/endpoints-and-egress.md`
- `docs/flows/webhooks/events-api-and-recovery.md`
- `docs/flows/webhooks/outbox-transactions.md`
- `docs/flows/webhooks/status-writers.md`
- `docs/flows/merchant-auth/verification-and-limits.md`

(`docs/flows/merchant-auth/resource-contract.md` _is_ claimed, which makes the
omission of its sibling an oversight rather than a policy.)

**Consequence.** The repository's central safety claim — "a vpay feature that
ships without a skill is a feature every agent will get wrong, and this gate
refuses it" — holds for half the feature corpus. A new page under
`docs/flows/webhooks/` will never turn the build red. The `verify-status`
precedent the README invokes ("a one-directional gate rots in the direction
nobody looks") applies here in miniature: this gate rots in the direction
nobody enumerates.

**Fix.** Walk `docs/flows` recursively, and treat a claimed _directory_ as
covering the pages beneath it (the gate already accepts directory paths on the
skills→vpay side via `existsSync`, so the two halves currently disagree about
what a directory means). Then add the five pages above to `vpay-webhooks` and
`vpay-merchant-api`, or exempt them with reasons.

---

## 4. The routing table sends two kinds of work to the wrong skill

**Files:** `skills/vpay/SKILL.md` § "Which skill to load", and the same table in
`README.md`.

**a. "the reconciler" → `vpay-ops`.** Both tables end with
`Config, deployment, the reconciler, observability | vpay-ops`. `vpay-ops`'
description names "configuration, deployment and observability" and no worker
internals; `vpay-reconciler` is the skill for the job loop, the ladders, lease
reaping and crash recovery — and its own description never uses the word
_reconciler_ at all (it opens "The vpay worker"). So the one table that routes
by topic points "reconciler" at the skill that does not cover it, and the skill
that does cover it does not answer to that name in its description. An agent
asked "the reconciler is re-polling a settled charge" loads `vpay-ops` and
finds nothing.

**b. "mobile" → `vpay-checkout`.** Both tables say `The payer-facing surfaces —
hosted, browser, mobile | vpay-checkout`. `vpay-checkout/SKILL.md` line 19 says
the opposite: "The Flutter payer surface is a different package — load
`vpay-sdks`." `coverage.json` agrees with the skill, not the table:
`docs/flows/mobile-checkout.md` and `sdks/flutter/vpay_checkout_flutter` both
belong to `vpay-sdks`.

**Fix.** Split the ops row (`Config, deployment, images, the chart,
observability`) and give `vpay-reconciler` the reconciler word in both its row
and its `description`. Change the checkout row to "hosted, embedded, popup" and
add a `vpay-sdks` row for the Flutter payer plugin.

---

## 5. `vpay/SKILL.md` tells agents the `justfile` is unreadable, and it is 4 704 lines

**Files:** `skills/vpay/SKILL.md` repository map ("`justfile` | ~270 000
lines"); `README.md` ("vendoring a 300 000-line `justfile`").

`wc -l justfile` → **4704**. `vpay-tooling/SKILL.md` gets it right: "`justfile`
is ~4700 lines and about 95% comment — roughly 90 recipes buried in prose."

**Consequence.** Two different wrong numbers, 57× and 64× over, in the two
documents a reader meets first. The practical harm is not the arithmetic: the
entire skill set rests on "**the recipe body wins** — always confirm with
`just --show <recipe>`", and an agent told the file is a quarter of a million
lines will not open it. It will trust the prose, which is the exact failure the
set exists to prevent.

**Fix.** `~4 700 lines, about 95% comment` in both places, matching
`vpay-tooling`. Keep the "do not read it top to bottom; use `just --show`"
advice, which is the part that is true.

---

## 6. `vpay/SKILL.md`'s repository map puts the demo shop in the wrong tree

**File:** `skills/vpay/SKILL.md`, repository map:

> `frontends/apps/` | `checkout` (the payer page), the dashboard, **the demo shop**

`ls frontends/apps/` → `checkout`, `dashboard`. That is all.
`vpay-frontend/SKILL.md` has it right: `@vpay-examples/shop` lives at
`examples/shop`, and it is described there as "a _third party_ — it depends on
no vpay design-system package", which is the load-bearing point.

**Consequence.** An agent asked to change the demo shop looks in
`frontends/apps/`, does not find it, and — worse — arrives believing the shop
is a first-party app in the workspace rather than a deliberate outsider with a
different Next version and no `@vaam-apps/ui`. That framing is what stops
someone "unifying" the shop's Tailwind config with the two apps'.

**Fix.** Move the shop to an `examples/` row and carry one clause of
`vpay-frontend`'s framing ("a third party, deliberately on its own stack").

---

## 7. No routing row for `/v1/checkout/sessions`, and "checkout" means the frontend

**Files:** the routing tables in `skills/vpay/SKILL.md` and `README.md`.

The `checkout.session` object is a first-class `/v1` resource
(`backends/crates/vpay-api/src/v1/checkout_sessions.rs`, three routes, an
expiry sweep, an event, an `object: "checkout.session"` wire shape). It is
covered — `vpay-merchant-api/references/routes.md:32-34` and `objects.md:97`
carry it — but nothing routes an agent there. The only row containing the word
"checkout" is `The payer-facing surfaces … | vpay-checkout`, which is the Next
app under `frontends/apps/checkout`.

**Consequence.** "Add a field to the checkout session" or "why does
`checkout.session.expired` only fire from the sweep" loads the **frontend**
skill. The answer is two skills away, in a reference page of a skill the agent
was not told to load.

**Fix.** Either add a routing row (`Checkout sessions, the session object,
expiry → vpay-merchant-api`) or name checkout sessions in
`vpay-merchant-api`'s description, which currently says only "route tables".

---

## 8. The version stamp points at a file the reader does not have, and the reference pages carry no stamp at all

**Files:** every `skills/*/SKILL.md` stamp; all 50 `skills/*/references/*.md`;
`README.md` § Versioning; `VERSIONING.md` rule 1.

Two gaps in an otherwise well-designed apparatus.

**a. "See VERSIONING.md" is unresolvable once installed.** The README is
explicit that "a skill is installed, not cloned. `npx skills add` fetches **one
directory** into `.agents/skills/`". `VERSIONING.md` lives at the repository
root and is therefore absent from every installed skill. It is prose, not a
markdown link, so nothing catches it — my link check found 0 broken relative
links precisely because it is not a link.

**b. 50 of 50 reference pages carry no stamp.**
`grep -rL "Verified against vpay" skills/*/references/*.md` returns all fifty.
The gate enforces the stamp on `SKILL.md` and not below it. But the reference
pages are where the copy-pasteable detail lives —
`vpay-merchant-api/references/routes.md` is 249 lines of route tables,
`vpay-tooling/references/gates.md` is 342 lines of gate behaviour — and they
are exactly what an agent reads immediately before writing code. A reader who
follows a link from a stamped page into an unstamped 249-line route table has
lost the one signal the versioning policy exists to deliver.

**Fix.** Replace "See VERSIONING.md" with a self-contained sentence (the policy
is one line: _version-sensitive claims carry the date they became true; trust
the repository over this page_), or copy `VERSIONING.md` into each skill
directory. Add a one-line stamp to each reference page — or, cheaper, extend
the gate to require the stamp on any `references/*.md` over ~100 lines.

---

## Lower-severity findings

**9. `vpay-sdks` lists nine resources under the heading "eight", twice.**
`skills/vpay-sdks/SKILL.md:353` — "The eight merchant resources, in both
languages: `payment_intents`, `checkout.sessions`, `customers`, `invoices`,
`invoice_items`, `refunds`, `events`, `account_holders`, `balance`." That is
nine. The `sdks/nodejs` table row above it also says "eight resources". An
agent checking parity coverage against this list will conclude one resource is
undocumented and go looking for it. Count it, or say which of the nine is not a
resource.

**10. `vpay-dashboard` states a hypothetical sixth procedure as a fact.**
`SKILL.md:48-50`: "**The `$procs` transport mounts five procedures…** Code wins
— five are mounted, and only `searchPaymentIntents` is probed by a test at the
transport layer. A sixth procedure is routed automatically and tested by
nobody." In `references/read-seam-and-bff.md` the sixth is plainly conditional
("Two consequences for anyone **adding** a sixth procedure"). In the SKILL.md
it reads as an existing sixth procedure, contradicting the sentence before it.
`vpay-merchant-api/references/routes.md` lists exactly five. Rewrite as "_if
you add a sixth, the router will mount it automatically and nothing will test
it._"

**11. `vpay-merchant-api` buries its headline warning.** Its description leads
with "the routes that are declared in the wire contract but mounted nowhere" —
correctly, because that is the false-existence trap in this area. The section
"Declared in the contract, mounted nowhere" is at line 498 of 175 lines of
body, below the router tour, auth, scopes, errors, idempotency and body limits.
Every other skill in the set front-loads its refusal. Move it above "Routes are
rows in a table".

**12. Seven skills point at their own reference pages in backticks, not links.**
`vpay`, `vpay-checkout`, `vpay-dashboard`, `vpay-docs-status`,
`vpay-frontend`, `vpay-reconciler`, `vpay-sdks` write `` `references/x.md` ``;
the other thirteen write `[references/x.md](references/x.md)`. The gate's regex
matches both, so this passes. It is the clearest visible seam between the six
authors, and it costs something real: a backticked path does not read as "open
this", and `vpay-sdks` — which has six of them and whose most important content
(`references/parity.md`) is behind one — is the worst case.

**13. The `vaam-ui` companion skill is not cross-referenced from the two skills
that need it.** `vpay-conventions` and `vpay-troubleshooting/references/node-web.md`
both point at `.agents/skills/vaam-ui/`. `vpay-frontend` and `vpay-dashboard`
— the skills an agent building a screen loads — do not, yet
`vpay-frontend/references/verify-ui.md` repeatedly instructs "compose a
`@vaam-apps/ui` component instead" without saying where that component's API is
documented. An agent told what not to write and not where to look writes the
long `className` again.

**14. The stamp's date has undefined semantics and is unenforced.** The stamp
reads `f063ee96 (2026-09-15)`; `coverage.json` carries **both**
`vpayCommitDate: 2026-09-15` and `verifiedAt: 2026-09-16`; the release tag
`v2026-09-16-f063ee96` uses the latter; `VERSIONING.md` rule 1 calls the stamp
"the vpay it was verified against", which reads as the verification date. The
gate's regex captures the date and compares only the sha, so a stamp could
carry any date and pass. Separately, `README.md`'s sample output —
`baseline: these skills were verified against vpay f063ee96 (2026-09-16)` —
does not match what the tool actually prints (`(2026-09-15)`, the commit date,
from `git log --format=%cs`). Decide which date the stamp carries, say so in
`VERSIONING.md`, fix the README sample, and have the gate compare it.

**15. Nine of twenty SKILL.md files exceed the ~160-line target**, topping out
at 201 (`vpay-payments`), 199 (`vpay-orange-money`), 194 (`vpay-invoices`), 191
(`vpay-mtn-momo`), 188 (`vpay-data-layer`), 187 (`vpay-customers`), 177
(`vpay-webhooks`), 175 (`vpay-merchant-api`), 169 (`vpay-ops`). None is padded
— I looked for a tour to cut and did not find one — so this is a note rather
than a finding. If anything goes, it is the duplicated `Category` policy table,
which appears in full in both `vpay-conventions/references/errors.md` and
`vpay-merchant-api/references/errors.md`. They agree today. They are the next
thing to diverge.

---

## Assessment of the versioning apparatus

**The policy is coherent and followable.** The failure mode it names — a skill
true of `master` and false of the tree in front of the reader — is the right
one, the diagnosis (vpay has no tags, `0.1.0` has never moved, so a commit and
a date are the only honest identity) is correct, and the mechanism (dated
claims in prose, in vpay's own house style) is the only one that could work.
The three rules are short enough to be obeyed. `CHANGELOG.md`'s "Known to be
true only of this commit" section is the best idea in the apparatus: it names
the eight claims most likely to age and tells an upgrader which to re-check
first, which is exactly what a changelog for a briefing should do and is not
what changelogs usually contain.

**The stamp is in the right place.** Directly under the title, above all
content, in a blockquote, on all twenty pages — a reader sees it before they act
on the page. It is machine-enforced for presence and for agreement with
`coverage.json`'s `baseline`, and the gate's drift report is genuinely useful
(it correctly reported "exactly the tree these skills were verified against").

**Three gaps**, all covered above: the stamp cites a `VERSIONING.md` that is not
installed alongside it (finding 8a); no reference page carries a stamp, and the
references are what an agent reads last before writing code (finding 8b); and
the stamp's date is semantically undefined and unchecked (finding 14).

**One observation on the release model.** `VERSIONING.md` says a tag is cut
"whenever a batch of skills is re-verified against a newer vpay", and the gate
refuses a skill stamped differently from the baseline. Together these mean a
single-skill correction cannot be published without either re-verifying all
twenty against the new baseline or stamping the corrected skill at the old one.
That is a defensible choice — it is what makes the stamp trustworthy — but it
is not stated, and the first person to fix one sentence will discover it as a
gate failure. Say it in `VERSIONING.md`.

---

## Trying to break `verify-coverage`

It passes, and the implementation is careful: both directions, a named fix in
every message, a stamp check, a description-length floor, a `name` == directory
check, orphan-reference detection in both directions, and drift reported rather
than failed. Defects it does **not** catch:

1. **A feature page in a `docs/flows/` subdirectory** — finding 3. Half the
   corpus. This is the one worth fixing.
2. **A `covers` entry whose skill prose says nothing about it.** Acknowledged
   in the README ("a reviewer checks it is true"), so it is a known limit
   rather than a defect — but with 205 claimed paths and no sampling
   discipline, the reviewer half of that promise is not staffed.
3. **A stamp with a wrong or absurd date** — only the sha is compared
   (finding 14).
4. **An unstamped `references/*.md`** — finding 8b.
5. **A `docs/flows` page claimed by two skills.** The brief asks for "exactly
   one"; the gate only checks "at least one". There are no duplicates today
   (I checked all 205 paths: none appears in two `covers` lists), so this is
   latent, not live.
6. **The `exempt` / `exemptReasons` path has never executed** — both are empty,
   so the two checks over them are untested code in a gate.
7. **A multi-line YAML description** would be truncated at its first line by
   `/^description:\s*([\s\S]+?)(?=\n\w+:|$)/m` (the `/m` flag makes `$` match
   at every line end), and could then trip the 80-character floor spuriously.
   All twenty descriptions are single-line today.

---

## What I did not check

- **Facts, identifiers, counts and paths** inside the skills — delegated to the
  parallel factual reviewer per my brief. The five factual claims I did verify
  (`void_in_tx`'s `WHERE`, `expected_ignored`, the `justfile` line count,
  `frontends/apps/` contents, the `docs/flows` page count) were checked only
  because the coherence finding depended on knowing which side was right.
- **The reference pages in depth.** I read
  `vpay-dashboard/references/read-seam-and-bff.md`,
  `vpay-conventions/references/errors.md` and
  `vpay-merchant-api/references/errors.md` (for the cross-skill checks) and
  sampled `vpay-frontend/references/verify-ui.md`,
  `vpay-merchant-api/references/routes.md` and `objects.md`. The other ~44
  reference pages I checked only for stamp presence, link resolution and
  linkage from their SKILL.md. **If budget exists for one more pass, spend it
  there** — 9 200 of the repository's 12 570 markdown lines are reference
  pages, they carry the most actionable detail, and they are the least reviewed
  part of this set.
- **Whether each skill's "how contributors trip it" material is complete
  against its vpay area.** I sampled: the `#[ignore]` chain (finding 2), the
  `vaam-ui` pointer (finding 13), the migration-checksum rule (well covered in
  three skills, consistently), the erasure six-tables rule (covered, in a
  reference), and the "a stub that answers the way we guessed is not evidence"
  lesson (covered in both adapter skills and `vpay-invoices`). A systematic
  omission audit per area would need the area-by-area reading above.
- **Descriptions under a real trigger harness.** I judged them by reading the
  words against the prompts an agent would carry. Findings 4 and 7 are the two
  I am confident about; the rest read as well-targeted, all are 385–681
  characters, and none is short enough to under-trigger.
- **`CONTRIBUTING.md`** — read only for its relationship to the versioning
  policy, not reviewed as a document.
- **The CI workflow** `.github/workflows/verify.yml` — not read.

## Clean bills

Pages I read in full and found sound, coherent with their siblings, and
correctly ordered — no findings against them:

`vpay-conventions`, `vpay-tooling`, `vpay-troubleshooting`, `vpay-docs-status`,
`vpay-reconciler`, `vpay-webhooks`, `vpay-provider-adapters`, `vpay-mtn-momo`,
`vpay-orange-money`, `vpay-data-layer`, `vpay-frontend`, `vpay-checkout`,
`vpay-customers`, `vpay-ops`.

`vpay-orange-money` and `vpay-ops` are the two best pages in the set: both open
with the refusal ("Orange has never been called"; "No pod has ever run this"),
both distinguish what the stub does from what the rail does, and both say
plainly what a present-tense sentence would cost. They are the model the
orientation skill should be rewritten against.
