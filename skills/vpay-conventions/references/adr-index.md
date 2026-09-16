# The ADR index, and what is still undecided

_Verified against vpay `93c6dfd0` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`docs/adr/`. An ADR is **immutable once accepted**: to change a decision you
write a new ADR that supersedes it. You never edit one.

The one documented exception is ADR-0016's serde exemption _table_, which the
ADR itself calls "the one part of this document a routine change touches —
adding or removing a row is a change to an accepted decision's data, not to
the decision."

## Accepted

| ADR                                            | Decision, in one line                                                                                                                                                                                                                                                              |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0001 record-architecture-decisions             | Numbered, immutable ADRs in `docs/adr/`; supersede, never edit.                                                                                                                                                                                                                    |
| 0002 provider-port                             | One `ProviderAdapter` trait; **providers are rows in a table, never enum variants**; the core branches on capability _values_, never a provider code. Adding a rail is an INSERT plus an adapter crate, never a schema migration.                                                  |
| 0003 yaml-configuration                        | All administration is YAML in git with `${ENV}` placeholders, validated at boot and reconciled in one transaction; validation failure exits non-zero without serving traffic; the dashboard cannot change any of it.                                                               |
| 0004 musl-mimalloc                             | Static musl binaries into `FROM scratch` with mimalloc. **Superseded in part by 0014** — only the architecture named in its Decision.                                                                                                                                              |
| 0005 rustls-only                               | rustls everywhere; `openssl`, `openssl-sys`, `native-tls` banned in `deny.toml`; TLS verification never disabled, including against stub hosts.                                                                                                                                    |
| 0006 no-mocks-in-main-processes                | No test double reachable from the shipping binary; a stub rail is a WireMock host in configuration. Gate: `verify-no-mocks`.                                                                                                                                                       |
| 0007 lint-policy                               | A panic in a payment path is a defect. Deny `unwrap`/`expect`/`panic`/`todo`/`unimplemented`/float arithmetic; forbid `unsafe`; tests exempt via `clippy.toml`.                                                                                                                    |
| 0008 dashboard-scope                           | The dashboard **observes and acts on records**; it never administers configuration and never holds a merchant secret key. Every write produces an `audit_log` row.                                                                                                                 |
| 0009 dashboard-oidc-provider                   | vpay runs Authkestra's `authkestra-op` **in-process as its own OP**; authorization-code + PKCE. _Its "Scope boundary" paragraph is superseded by 0010; its audience half by 0017._                                                                                                 |
| 0010 merchant-auth-private-key-jwt             | `/v1` authenticates merchants with OAuth2 `client_credentials` + `private_key_jwt`, not API keys — because `SqlxOpStore::find_client` hardcodes `token_endpoint_auth_method: None` and `jwks: None`, so a DB-backed registry cannot serve `private_key_jwt` at the pinned version. |
| 0011 error-modelling                           | Errors typed at the leaves, composed per layer, classified once, `anyhow` only at the edge. See [errors.md](errors.md).                                                                                                                                                            |
| 0012 rail-configuration-requirements-in-config | **Interim.** `vpay_config::config::REQUIRED_RAIL_KEYS` is the _one_ sanctioned place outside an adapter crate where a provider code is matched on; it moves behind the port the day the port grows a `required_settings()` hook.                                                   |
| 0014 builder-host-musl-triple                  | `backends/Dockerfile` passes `--target` set to the **builder's own host triple** read from `rustc -vV`, never a hardcoded `x86_64-unknown-linux-musl` — hardcoding it on an arm64 host "fails outright". Supersedes 0004's architecture only.                                      |
| 0015 sdk-parity                                | `sdks/rust` and `sdks/nodejs` are held to parity per capability, machine-checked by `verify-sdk-parity` against `docs/sdks/parity.md`.                                                                                                                                             |
| 0016 engineering-standards                     | Six engineering standards, three machine-checked. See the table in `SKILL.md`.                                                                                                                                                                                                     |
| 0017 staff-authentication                      | How a staff member signs in to `/dash/v1`: argon2id with a deployment pepper, RFC 6238 TOTP with a sealed secret, a strictly-increasing replay guard, mandatory enrolment, a one-time password. **Supersedes the audience half of 0009** — the literal `vpay:dash/v1` is retired.  |
| 0018 cross-tenant-admin-reads                  | A cross-tenant **read-only** admin role for `/dash/v1` (`is_admin` on `staff_members`). `/dash/v1` still answers `403` to every non-`GET`. Extends 0017.                                                                                                                           |
| 0018 privacy-controls-and-evidence             | Six GDPR workstreams share **one** personal-data inventory and one evidence architecture (actor / tenant / target / outcome / time / correlation). ⚠ duplicate number — see below.                                                                                                 |
| 0019 credential-model                          | Credentials become their own object, generic over kind and subject. Amends 0017's decision 1 about _where the material lives_, not about how a staff member proves identity. Does not touch 0018.                                                                                  |
| 0021 flutter-checkout-plugin                   | A Flutter checkout plugin as a payer surface — a native Activity / UIViewController calling the hosted page, **polling, never a URL**.                                                                                                                                             |

## Not accepted — read the status line before you build on it

**ADR-0013 database-backups-and-retention is `Status: Proposed`.** Its own
first bullet:

> Nothing here is implemented. **No backup of any vpay database has ever been
> taken**, no restore has ever been performed, and no restore drill has ever
> run. This ADR records obligations the schema already creates and proposes a
> policy to meet them; **every number in it is proposed, not measured.**

The recovery objectives, the 30-day PITR window and the 90-day full retention
are all proposed values nobody has measured against anything. The ADR exists
because "the schema already commits vpay to holding things whose loss cannot be
recovered from anywhere else" — money, the only evidence an operator would have
about money, and replay protection, "a security control whose loss is not
visible in any dashboard".

`docs/runbooks/restore-from-backup.md` had every SQL statement in it executed
against a scratch Postgres — but nothing about backups, PITR or the restore
itself has been exercised.

## RFCs — proposals, not decisions

- `docs/rfc/0001-settlement-and-payouts.md` — **Draft**.
- `docs/rfc/0002-gdpr-policy-and-operator-decisions.md` — **Under review**.

## Maintainer decisions recorded in place, not as an ADR status

These are live open questions sitting inside Accepted documents. An agent that
"tidies" one has reversed a decision nobody took.

| Where                                                                 | What is open                                                                                                                                                                                                                                                      |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `docs/flows/errors.md`, the policy-table note                         | Splitting `Category::Idempotency` so an in-flight key can answer `409` as Stripe does. "An ADR-level change and deliberately left as a maintainer decision."                                                                                                      |
| ADR-0017                                                              | Four in-place reservations, including whether the credential lifetime default should move and one the ADR explicitly declines to take ("that is a maintainer decision and this ADR does not take it").                                                            |
| `deploy/helm/vpay/README.md` + `docs/runbooks/provider-error-rate.md` | Whether `VpayProviderErrorRateHigh` should exclude declines. It counts every non-successful port call, which on mobile money includes `charge_declined` — a large, normal share of traffic. To be decided **with measured traffic**, together with the threshold. |
| `docs/flows/hosted-checkout.md`                                       | Why `checkout_not_configured` answers `500` rather than `503`. Recorded as a maintainer's decision, not an oversight.                                                                                                                                             |
| `docs/flows/deployment.md`                                            | Deleting or archiving the retired `ghcr.io/vaam-apps/vpay-worker` GHCR package — "written here as a task with an owner rather than as a fact". Nobody holds the `delete:packages` scope.                                                                          |

## The duplicate ADR number

**Two files are both numbered `0018`**:
`docs/adr/0018-cross-tenant-admin-reads.md` and
`docs/adr/0018-privacy-controls-and-evidence.md`. They are unrelated decisions.

ADR-0021's own header records how it was resolved, and the method is worth
copying: the author ran `git ls-tree origin/master docs/adr/` (20 entries for
19 numbers), checked every **open PR's diff** with `gh pr diff --name-only`,
found that PR #172 already renumbers the second `0018` to
`0020-privacy-controls-and-evidence.md`, and took `0021` — "the next number
free of both the tree and every open PR's diff". The header is titled
**"Number checked at branch time, not assumed."**

So: **before you write a new ADR, check the tree _and_ every open PR.** As of
2026-09-16 `0020` is reserved on a branch and not on `master`.
