# `docs/flows/sdlc.md` and `docs/flows/emergency-changes.md`

_Read on vpay's `impl/387-sdlc` branch (issue #387, refs #151) on 2026-11-17,
before it merged; restamp against `master` once it has. Version-sensitive
claims carry the date they became true — see
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Two flow pages that write vpay's change-to-production path down as an ISO/IEC
27001:2022 control (A.8.25). **They are policy, not machinery.** Nothing in them
adds a gate; they cite the gates [gates.md](gates.md), [recipes.md](recipes.md)
and [ci.md](ci.md) already describe, and say — after the #387 review made them —
which of their own sentences are enforced and which are practice.

The briefing in one line: **cite a recipe or a gate as evidence of a control,
never one of these pages.**

## `sdlc.md` — what it says, and how much of it is enforced

| Claim on the page                                                                                                                     | Enforced?                                                                                                                                     |
| ------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `just ci` is the path: `fmt-check`, `clippy`, `verify`, `test-rust`, `test-doc`, `verify-ignored`, the web steps, `audit-web`, `deny` | **Yes**, by the recipe — and CI runs the same jobs **subject to path filtering**                                                              |
| `cargo deny` runs on every change                                                                                                     | **Locally only.** CI's `supply-chain` job runs only when a change touches a `Cargo.toml`, `Cargo.lock`, `deny.toml` or `ci.yml`               |
| A security-relevant change gets a threat-assessing review                                                                             | **Practice.** No `CODEOWNERS`; nothing in the tree requires a review or a green gate before merge; branch protection is not evidenced         |
| A threat model precedes a new interface (`docs/security/README.md`)                                                                   | Practice backed by one gate: `verify-security-inventory` fails if the inventory does not move with a new surface, secret class or rail        |
| Advisories are risk-accepted by a dated, reasoned, `REMOVE WHEN` entry                                                                | **The format is convention**; `audit-web` / `deny` only see the id                                                                            |
| Remediation SLA: critical 7 d, high 30 d, moderate 90 d, low next release                                                             | **No.** "Proposed, pending maintainer approval (2026-11-17)"; no ADR or RFC records it, and an allowlist entry makes both gates green forever |
| A named vulnerability owner                                                                                                           | **No.** The maintainer who merges the fix; no standing owner                                                                                  |
| CI evidence is retained                                                                                                               | GitHub's default log retention only; no copy, no per-release evidence bundle                                                                  |
| Releases are signed                                                                                                                   | `cosign sign` runs; **no `cosign verify` of a published artifact has ever been run**                                                          |

The page lists the security-relevant `verify` gates by name and calls the set
"the gates the `verify` recipe lists" rather than a number, so it does not go
stale when a gate is added — the count is in `AGENTS.md` and [gates.md](gates.md)
(17 as of 2026-10-27, unchanged by this batch).

### Its advisory lists disagree with the files — trust the files

`sdlc.md` names the accepted advisories. Written on its branch, the lists
already disagree with `master` and with the other changes landing the same day:

- **Web.** It lists three accepted: `GHSA-vfj7-8cjw-p6xm` (braces, maintainer
  decision 2026-10-09), `GHSA-3g3m-2fjr-6pw4` (glob) and `GHSA-g3hv-75wg-m249`
  (minimatch), and its branch adds the last two to `ignoreGhsas`. But
  vaam-apps/vpay#400, on `master` since 2026-11-17, **cleared glob and minimatch
  by overrides with nothing ignored**, and vpay#402 adds
  `GHSA-fc5q-39xc-vjpf` (picomatch, pending acceptance). The two branches also
  conflict in `pnpm-workspace.yaml`.
- **Rust.** It lists `RUSTSEC-2023-0071`, `RUSTSEC-2024-0449` and
  `RUSTSEC-2026-0387`. vpay#402 adds `RUSTSEC-2026-0543` and
  `RUSTSEC-2026-0440`, both pending the maintainer's acceptance (PR #396's
  carry adds `-0440` too).

Whichever merges last must reconcile the page. Until then
`pnpm-workspace.yaml`'s `auditConfig.ignoreGhsas` and `deny.toml`'s
`[advisories] ignore` are the list; the `vpay-troubleshooting` skill's
`build-toolchain.md` has the 2026-11-17 entries.

## `emergency-changes.md` — an unexercised procedure

What counts as an emergency (a live exploited vulnerability, a production
incident, a regulatory deadline — not "the sprint ends"), the path (still a
branch and PR, never a direct push), the gates that may never be skipped
(compile, the Rust unit and integration suites, `just verify`, no undeclared
`NotImplemented`), maintainer authorisation for skipping any other, and a
retrospective within five business days.

**None of steps 2–4 is enforced**: no `CODEOWNERS`, no branch protection on
record, no record template, no tracker. CI's path filters can skip a suite a
change seems not to touch, so "must pass" means the author ran it. **No
emergency change has ever been made in this repository** (2026-11-17), so no
retrospective has ever run. If you are asked to make one, follow the page and
update its Status in the same change — the page says so itself.

## Where these sit

`docs/flows/README.md` lists both; `docs/security/README.md` links `sdlc.md`.
`schemas/security-inventory.yaml`'s `A.8.25` row is the machine-readable view,
and `verify-security-inventory` checks that file, not these pages.
