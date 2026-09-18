# vpay-skills

Agent skills for [vpay](https://github.com/vaam-apps/vpay) — a payment
orchestrator for Cameroon mobile money rails.

vpay is large and unusually opinionated. It has fourteen machine-enforced gates
(as of 2026-09-18; twelve until 2026-09-17), a
documentation tree with three tiers and its own discipline, an adapter port that
rails must be reached through, and one cardinal rule: **never make the system
look more finished than it is.** An agent dropped into that repository without
context does not fail slowly — it fails by writing something plausible, going
green, and moving a status marker it has not earned.

These skills are the context. Install the one that matches the work:

```bash
npx skills add https://github.com/vaam-apps/vpay-skills --skill vpay
```

Start with `vpay`. It is the orientation skill and it routes to the rest.

## The skills

| Skill                                                      | Load it when                                                              |
| ---------------------------------------------------------- | ------------------------------------------------------------------------- |
| [`vpay`](skills/vpay/)                                     | Anything in the repo. Orientation, the two rules, the map to these others |
| [`vpay-conventions`](skills/vpay-conventions/)             | Writing Rust or TypeScript here — errors, serde, lints, architecture      |
| [`vpay-tooling`](skills/vpay-tooling/)                     | Running anything — `just`, the gates, xtask, CI, the toolchain pins       |
| [`vpay-troubleshooting`](skills/vpay-troubleshooting/)     | Something broke and the message is not the cause                          |
| [`vpay-docs-status`](skills/vpay-docs-status/)             | Finishing a change — status pages, flow docs, ADRs, the parity rule       |
| [`vpay-payments`](skills/vpay-payments/)                   | PaymentIntent states, money, the ledger, crash safety                     |
| [`vpay-reconciler`](skills/vpay-reconciler/)               | The worker — job loop, retry ladders, leases, crash recovery, SIGTERM     |
| [`vpay-merchant-api`](skills/vpay-merchant-api/)           | The `/v1` wire contract, merchant auth, idempotency                       |
| [`vpay-webhooks`](skills/vpay-webhooks/)                   | Events, the outbox, the signature scheme, delivery                        |
| [`vpay-provider-adapters`](skills/vpay-provider-adapters/) | The port, the failure taxonomy, conformance — and adding a rail           |
| [`vpay-mtn-momo`](skills/vpay-mtn-momo/)                   | MTN MoMo specifics — the push flow, tokens, the sandbox                   |
| [`vpay-orange-money`](skills/vpay-orange-money/)           | Orange Money specifics — the redirect flow and the payer window           |
| [`vpay-frontend`](skills/vpay-frontend/)                   | The web workspace, Tailwind v4 + daisyUI 5, and the `verify-ui` gate      |
| [`vpay-checkout`](skills/vpay-checkout/)                   | The payer-facing surfaces: hosted, browser, mobile                        |
| [`vpay-dashboard`](skills/vpay-dashboard/)                 | The merchant dashboard, `/dash/v1`, staff auth, the BFF                   |
| [`vpay-customers`](skills/vpay-customers/)                 | The Customer object, addresses, erasure, retention, name lookup           |
| [`vpay-invoices`](skills/vpay-invoices/)                   | Invoices and invoice items                                                |
| [`vpay-data-layer`](skills/vpay-data-layer/)               | CrateStack, migrations, repositories, sqlx, Postgres tests                |
| [`vpay-sdks`](skills/vpay-sdks/)                           | The merchant SDKs and the parity rule that binds them                     |
| [`vpay-ops`](skills/vpay-ops/)                             | Configuration, deployment, the reconciler, observability                  |

## Why a separate repository

Two reasons, and the second is the load-bearing one.

A skill is installed, not cloned. `npx skills add` fetches one directory into
`.agents/skills/` and pins its hash in `skills-lock.json`. That works for any
project that talks to vpay — a merchant integrating the Node SDK gets
`vpay-merchant-api` without vendoring vpay's whole tree.

And skills must be able to move at a different speed from the code. A skill is
not documentation-of-record; it is a briefing, and a briefing that has to clear
vpay's fourteen gates to be corrected is a briefing nobody corrects.

The cost of that separation is drift, and drift is what the gate below exists to
refuse.

## The parity rule

> **Every feature lands in three places or it has not landed: the code, the
> docs, and the skills.**

vpay's own rule is that a status page which lags is worse than none, because
people trust it. A skill that lags is worse still, because an agent does not
merely trust it — it acts on it, at machine speed, across every session that
loads it.

So this repository ships a gate, and it fails in **both** directions:

```bash
node tools/verify-coverage.mjs /path/to/vpay
```

- **docs → skills.** A page in `docs/flows/` that no skill claims fails the
  gate. That is the half that catches a feature shipping with no briefing.
- **skills → docs.** A path claimed in `coverage.json` that no longer exists in
  vpay fails the gate. That is the half that catches a skill still describing
  something that moved or was deleted.

A one-directional gate rots in the direction nobody looks. vpay learned this
with `verify-status`, which gained its second direction on 2026-09-03 and has
been the model for every gate since.

`coverage.json` is the map. An entry earns its place by prose that actually
covers the path — the gate checks the claim exists, and a reviewer checks it is
true.

CI runs the gate against vpay's `master` daily and on every push, so a vpay
merge that outruns this repository shows up here as a red build rather than as a
confidently wrong agent three weeks later.

## Versioning — read this before you trust a skill

> **A skill is true of _a_ vpay, not of vpay.** A feature that exists on
> `master` may be absent from the tree you are editing.

Every `SKILL.md` is stamped, under its title, with the vpay commit it was
verified against:

> **Verified against vpay `f063ee96` (2026-09-15).**

That stamp is machine-enforced — the gate refuses a skill that carries none, or
one whose stamp disagrees with `coverage.json`'s `baseline`. And
`verify-coverage` reports how far the checkout you point it at has drifted:

```
baseline: these skills were verified against vpay f063ee96 (2026-09-16);
this checkout is 9a3d5732 (2026-09-14) — 0 commit(s) newer,
2 commit(s) it does not have.
```

Run against an older vpay, the gate legitimately fails on paths that do not
exist there yet. That is not a bug; it is the tool telling you these skills are
newer than that tree.

Inside the prose, the mechanism is **dated claims** — "one shipping binary since
2026-09-07 (issue #77); it was two before that". vpay has no release tags and
its workspace version has never moved off `0.1.0`, so a commit and a date are
the only honest version identity available.

Full policy, release naming, and how to write a version-sensitive claim:
[VERSIONING.md](VERSIONING.md). What changed between releases, including any
claim that **stopped** being true: [CHANGELOG.md](CHANGELOG.md).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The short version: a skill is judged on
whether an agent that read it does the right thing, not on whether it is
complete. Prefer the caveat over the tour, and date anything that could go
stale.

## Licence

Apache-2.0, matching vpay.
