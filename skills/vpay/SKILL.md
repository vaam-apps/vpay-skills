---
name: vpay
description: "Orientation for working in the vpay repository — a Rust + TypeScript payment orchestrator for Cameroon mobile money rails (MTN MoMo, Orange Money). Load this before any task in vpay: it carries the two machine-enforced rules, what is actually built versus scaffold, the repository map, and which of the other vpay-* skills to load for the work at hand. Use when reading, planning, reviewing or changing anything in vpay."
---

# vpay

> **Verified against vpay `7a79684e` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

A payment orchestrator for Cameroon mobile money. Rust backend, TypeScript
frontends, merchant SDKs in Rust, Node and Flutter.

**Read `docs/status.md` before you believe anything.** It is short on purpose
(~290 lines) and it is the contract behind this repository's second rule. The
whole history lives under `docs/status/`, one page per area.

## The one thing to get right

> **vpay is a scaffold. It compiles, lints clean, its tests pass — and it
> cannot take a payment. Do not deploy it.**

As of 2026-09-15 exactly one real rail call has ever been made: a EUR
`mtn_momo` PaymentIntent created, confirmed and settled against **MTN's
sandbox**. No real payer, no production rail, no rail other than MTN's sandbox,
no webhook to any merchant endpoint outside the repository, no pod has ever run
vpay.

Everything else in this repository's history settled against a WireMock host
that answered the way the documents say a rail answers.

**The failure mode this repository is organised against is making itself look
more finished than it is.** In descending order of damage:

- filling an unimplemented function with something plausible that returns a
  hard-coded success;
- writing a test that asserts nothing so a suite goes green;
- adding a mock adapter to make local development easier;
- rendering fake rows in the dashboard so a screenshot looks good;
- marking something ✅ in the status pages because it compiles.

Each is worse than leaving the gap visible. If you cannot implement something
properly, leave `ProviderError::NotImplemented("<crate>::<fn>")`, declare it in
`docs/status.md`, and say so plainly in your summary.

The test: **would a test fail if it broke?** If no, it is not done.

## The two machine-enforced rules

Both are refused by `just verify`, and CI runs it.

**1. No test doubles in shipping processes.** No mock, fake, stub or dummy may
be reachable from `vpay-server` in any of its modes (`serve`, `worker`,
`staff`). `vpay-testkit`, `wiremock`, `testcontainers`, `mockall` and `fake`
may appear only under `[dev-dependencies]`. A stub rail is a **WireMock host in
configuration**, reached over HTTP exactly as a real rail is — never a linked
implementation or a `cfg` variant. (`cargo xtask verify-no-mocks`)

**2. Never claim a feature is done when it is not.** Unwritten code returns
`ProviderError::NotImplemented`, never a plausible success, an empty list or a
zero. Every such token must be declared in `docs/status.md`, and the gate fails
in **both** directions — an undeclared token fails, and so does a declaration
naming code that no longer carries one. (`cargo xtask verify-status`)

**Do not reach for `#[ignore]`.** AGENTS.md still prescribes
`#[ignore = "not implemented: …"]`, but `just verify-ignored` pins
`expected_ignored := "0"` and is step 7 of `just ci`, so adding one fails the
build. The sanctioned way to mark a test that needs a running stack is a Cargo
feature (`required-features = ["live-stack"]`). This is the authority rule in
miniature: the recipe wins over the prose.

## Before you start, and when you finish

```bash
just verify        # twelve gates + one advisory report — before AND after
cat docs/status.md
```

When you finish: `just ci`, then update the status page your change belongs to
**in the same commit**, then the relevant `docs/flows/*.md` **Status** section.
Then state explicitly, in your summary, what you did _not_ do.

`docs/status.md` § "Where a new row goes" is the map. Details:
`vpay-docs-status`.

## Repository map

| Path                   | What is in it                                                                                                                                              |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `backends/crates/`     | The Rust workspace: `vpay-core`, `vpay-api`, `vpay-db`, `vpay-provider`, adapters                                                                          |
| `backends/apps/`       | `vpay-server` — the one shipping binary, three modes                                                                                                       |
| `backends/migrations/` | SQL migrations + `MANIFEST.sha256` (their bytes are pinned)                                                                                                |
| `frontends/apps/`      | `checkout` (the payer page) and the dashboard. The demo shop is `examples/shop`                                                                            |
| `sdks/`                | Merchant SDKs — Rust, Node, Flutter                                                                                                                        |
| `schemas/vpay.cstack`  | The CrateStack schema. **It compiles into `vpay-db`** — a syntax error is a build failure                                                                  |
| `docs/flows/`          | One page per process: what happens, what can go wrong, what invariant holds                                                                                |
| `docs/status/`         | What is actually built, by area, with dated evidence                                                                                                       |
| `docs/adr/`            | Decisions. Immutable — superseded, never edited                                                                                                            |
| `docs/reference/`      | Why the code is shaped the way it is, per crate                                                                                                            |
| `justfile`             | 4 704 lines, two-thirds of them comment (268 KB, 2026-09-16). **The recipe body is the source of truth** — it wins over its own comment and over AGENTS.md |

## Architecture rules you will trip over

- **Rails live behind the port.** `if provider == "mtn_momo"` outside
  `backends/crates/vpay-adapter-*` is a defect. Branch on capability _values_
  (`flow`, `supports_refunds`), never on a provider code. (ADR-0002)
- **No environment branching.** No `if (sandbox)`, no `NODE_ENV` check. A
  profile selects a _config file_, never a _code path_. (ADR-0003)
- **Money is integer minor units.** XAF is zero-decimal: `5000` means 5 000
  FCFA. Float arithmetic is denied workspace-wide.
- **Never let a payer act on a transaction you cannot name.** Persist the
  reference _before_ submitting (push) or redirecting (redirect).
- **One charge per intent, forever.** A unique index enforces it. Retry means a
  new PaymentIntent.
- **Callbacks are hints.** `parse_callback` returns identifiers only, never a
  status. Only the authenticated status query moves money.

## Which skill to load

| The work                                                   | Load                     |
| ---------------------------------------------------------- | ------------------------ |
| Writing Rust or TS here — errors, serde, lints, structure  | `vpay-conventions`       |
| Running anything — `just`, gates, xtask, CI, toolchain     | `vpay-tooling`           |
| Something broke and the message is not the cause           | `vpay-troubleshooting`   |
| Finishing a change — status, flows, ADRs                   | `vpay-docs-status`       |
| PaymentIntent states, money, ledger, crash safety          | `vpay-payments`          |
| The worker — job loop, ladders, leases, crash recovery     | `vpay-reconciler`        |
| The `/v1` wire contract, merchant auth, idempotency        | `vpay-merchant-api`      |
| Events, the outbox, signatures, delivery                   | `vpay-webhooks`          |
| The port, failure taxonomy, conformance, adding a rail     | `vpay-provider-adapters` |
| MTN MoMo specifics                                         | `vpay-mtn-momo`          |
| Orange Money specifics                                     | `vpay-orange-money`      |
| The web workspace, Tailwind/daisyUI, the `verify-ui` gate  | `vpay-frontend`          |
| The payer page — hosted and embedded checkout, the popup   | `vpay-checkout`          |
| Mobile checkout — the Flutter plugin                       | `vpay-sdks`              |
| The dashboard, `/dash/v1`, staff auth, the BFF             | `vpay-dashboard`         |
| Customers, addresses, erasure, retention, name lookup      | `vpay-customers`         |
| Invoices                                                   | `vpay-invoices`          |
| CrateStack, migrations, repositories, sqlx, Postgres tests | `vpay-data-layer`        |
| The merchant SDKs and the parity rule                      | `vpay-sdks`              |
| Config, deployment, images, Helm, observability            | `vpay-ops`               |

See also `references/reading-the-docs.md` — this repository's documentation has
conventions that will mislead you if you do not know them, in particular why a
page tells you what it used to say and was wrong about.
