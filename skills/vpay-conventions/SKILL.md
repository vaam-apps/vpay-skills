---
name: vpay-conventions
description: How to write Rust and TypeScript in vpay — the two machine-enforced rules, ADR-0016's six engineering standards and precisely which three a gate checks, the thiserror/Classify error hierarchy, the serde wire convention and its exemption table, the clippy deny list, and the six architecture rules with the ADRs behind them. Load this before writing or changing any code in vpay, before adding an error type or a serialisable type, and before reaching for an exemption to make a gate pass.
---

# vpay conventions

> **Verified against vpay `b747e5d5` (2026-09-23).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

Source of truth is `AGENTS.md`. This page is the operative subset plus the
things that bite. For the recipes and gates themselves see **vpay-tooling**;
for status pages, the `NotImplemented` bullet shape and the finishing sequence
see **vpay-docs-status**.

## The two rules

Both are machine-enforced by `just verify`, and CI runs it (`AGENTS.md`, "The
two rules").

**1. No test doubles in shipping processes** (ADR-0006, gate
`cargo xtask verify-no-mocks`). Nothing mock, fake, stub or dummy may be
reachable from `vpay-server` in any of its three modes (`serve`, `worker`,
`staff`). In practice:

- `vpay-testkit`, `wiremock`, `testcontainers`, `mockall`, `fake` may appear
  **only** under `[dev-dependencies]`.
- A stub rail is a **WireMock host in configuration** (`compose.yml`), reached
  over HTTP exactly as a real rail is. Never a linked implementation, never a
  conditionally-compiled variant, never a feature flag.
- Cost, accepted deliberately: local development needs Docker rather than an
  in-process fake.

**2. Never claim a feature is done when it is not** (gate
`cargo xtask verify-status`). Unwritten code returns
`ProviderError::NotImplemented("<crate>::<fn>")` — never a plausible success,
an empty list or a zero. The full declaration mechanics belong to
**vpay-docs-status**. The test that settles the question: _would a test fail if
it broke? If no, it is not done._

## ADR-0016's six standards

The ADR's own thesis, and the reason to read it once:

> Three of the six standards below are mechanical enough to check; three are
> not, and **saying which is which is most of the value of writing them down.**

| #   | Standard                                                                                                                      | Gate                                                               | What only a reviewer can judge                                                                                                                       |
| --- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Errors** — `thiserror` at the leaves, composed per layer, classified once, `anyhow` only at a binary edge                   | `cargo xtask verify-errors`                                        | whether a leaf's `category()` is the _right_ category, and whether an override carries the comment ADR-0011 asks for                                 |
| 2   | **Adapters** — a rail is reached only through `ProviderAdapter`; its failures are _mapped_ into `FailureCode`/`ProviderError` | `verify-errors` + the shared conformance suite + `verify-no-mocks` | that a new rail's failures were mapped rather than flattened, **and that no `if provider == "…"` appeared outside `backends/crates/vpay-adapter-*`** |
| 3   | **serde** — everything vpay serialises spells the wire convention                                                             | `cargo xtask verify-serde`                                         | **whether an exemption's reason is honest**                                                                                                          |
| 4   | **SOLID and DRY**                                                                                                             | **none, deliberately**                                             | all of it                                                                                                                                            |
| 5   | **Repositories** — `vpay-db` exposes traits; implementations are `pub(crate)`                                                 | `cargo xtask verify-repositories`                                  | whether a new method belongs on an existing trait or a new one; whether a query behind `op_store_pool` is a repository method nobody wrote           |
| 6   | **Docs** — a doc-comment example is compiled and run                                                                          | `just test-doc` (+ `verify-docs`, a _report_)                      | whether a module doc is a paragraph and a link or an 80-line essay                                                                                   |

**The two deliberately ungated ones are the ones to watch.**

- **Standard 4 has no gate on purpose.** ADR-0016: "A cyclomatic-complexity or
  duplicate-token gate measures the shape of code rather than whether a
  responsibility is in the right place, and the cheapest way to pass one is a
  worse design that scores better." `verify-docs` _reports_ every production
  function of 80 lines or more as the closest honest proxy, and is not a gate
  for the same reason. Ask instead: _what would have to change for this to be
  wrong, and does that live in one place?_
- **ADR-0002's provider-code rule has no gate either**, and ADR-0016 says so:
  "no gate reads for it today, and that is a known gap". So
  `if provider == "mtn_momo"` outside `backends/crates/vpay-adapter-*` will
  pass every gate in the repository. It is still a defect. The one sanctioned
  exception is `vpay_config::config::REQUIRED_RAIL_KEYS`, which carries a
  comment saying so and pointing at ADR-0012 (interim, until the port grows a
  `required_settings()` hook).

Standard 6's report has a trap ADR-0016 names outright: **`verify-docs` must
not become a gate**, because the cheapest way to pass a comment-volume gate is
to delete the `# Errors` sections ADR-0011 depends on.

### The migration rule

> **Existing code is migrated as it is touched; a new crate complies from its
> first commit. Do not open a sweep. Do not, in particular, add an exemption
> row to make a gate pass on code you did not have to touch.**

Standard 5 has **no exemption mechanism at all** — "there is no exception
today and an escape hatch nobody needs is the one that gets used."

## Depth

- [references/errors.md](references/errors.md) — the three tiers, `Category` as
  the whole policy table, why delegation is wholesale, and how to add an error.
- [references/serde.md](references/serde.md) — what "spelling the wire
  convention" means, and the two-directional exemption table.
- [references/adr-index.md](references/adr-index.md) — every ADR in one line,
  and the decisions still open.

## Architecture rules

| Rule                                                                                                                                                                         | Where it comes from                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **Rails live behind the port.** Branch on capability _values_ (`flow`, `supports_refunds`), never on a provider code                                                         | ADR-0002 — and nothing greps for it                              |
| **No environment branching.** No `if (sandbox)`, no `NODE_ENV` check, no profile-selected bean. A profile selects a config _file_, never a code path                         | ADR-0003, `docs/flows/configuration.md`                          |
| **Money is integer minor units.** XAF is zero-decimal: `5000` means 5,000 FCFA. One conversion function, `Money::to_provider_string`                                         | `docs/flows/money.md`; float arithmetic is denied workspace-wide |
| **Never let a payer act on a transaction you cannot name.** Push rails: persist the reference before submitting. Redirect rails: persist the rail's token before redirecting | `docs/flows/crash-safety.md`                                     |
| **One charge per intent, forever.** A plain unique index. Retry means a new `PaymentIntent`                                                                                  | `docs/flows/payment-lifecycle.md`                                |
| **Callbacks are hints.** `parse_callback` returns identifiers only, never a status. The authenticated status query is the only thing that moves money                        | `docs/flows/provider-port.md`                                    |

## Lints

`clippy.toml` is seven lines and the whole policy (ADR-0007 — "a panic in a
payment path is a defect"):

```toml
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-panic-in-tests  = true
allow-dbg-in-tests    = false
```

Denied in production code: `unwrap`, `expect`, `panic`, `todo`,
`unimplemented`, float arithmetic. `unsafe` is **forbidden**, not denied.
Note the fourth line: **`dbg!` is denied even in tests.**

If clippy complains about `expect` in something you think is a test, you are
outside a `#[cfg(test)]` module — that is the exemption's scope, and it is the
single most common trip in this repo.

**TLS: rustls only.** `openssl`, `openssl-sys` and `native-tls` are banned
outright in `deny.toml` (ADR-0005), so a transitive dependency that pulls one
in fails CI rather than silently linking. Adding one needs a new ADR. TLS
verification is never disabled, including against stub hosts.

`RUSTSEC-2023-0071` is in `deny.toml`'s `ignore` list with its full reasoning —
accepted by the maintainer 2026-08-09, no patched `rsa` release exists, and
the entry genuinely fires. Do not "fix" it.

## TypeScript

- TS strict, plus `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`.
- Both apps compose the published **`@vaam-apps/ui`**. `frontends/packages/ui`
  (`@vpay/ui`) was **deleted 2026-09-12** and `just verify-ui` refuses any
  import of it. Its one theme registers under daisyUI's built-in name `dark`,
  not `bumblebee`. `@base-ui/react` left the repository with `@vpay/ui`.
- Status colour and copy come from `@vpay/tokens`, and since 2026-09-12 that
  is machine-enforced: `verify-ui` check 7a-ii refuses `text-state-<hue>-fg` /
  `bg-state-<hue>-bg` / `border-state-<hue>-border` / `text-destructive`
  written directly in an app. Status presentation goes through
  `defineStatusSystem` (`StatusPill`/`StateChip`) or an `InlineBanner` variant.
  See the repo-local `vaam-ui` skill in `.agents/skills/vaam-ui/`.
- The dashboard never holds a merchant API key; it calls `/dash/v1`
  server-side under an OIDC session (ADR-0008).

`AGENTS.md`'s TypeScript section has been corrected twice (2026-09-07 and
2026-09-12) and names libraries that are not dependencies of anything here.
Trust `package.json` and `just verify-ui` over that paragraph.

## Before you open a PR

`just fmt`, then `just ci`. Then the question this repo cares about most:
**does anything I wrote imply something works that I have not actually seen
work?** If so, fix the claim, not just the code.
