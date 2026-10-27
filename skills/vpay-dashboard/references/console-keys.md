# Console keys: pairing, one signed call, revocation, and what they replaced

_Verified against vpay `f885170c` (2026-10-27), the merge of
vaam-apps/vpay#383, by reading `docs/flows/dashboard-auth.md`, ADR-0036,
`docs/status/backend.md` § "`/dash/v1` is a signed message under a paired
console key", `docs/reference/vpay-config.md` and the identifiers below in the
code. **Nothing here was re-run for this page**: the test names are checked to
exist, not to pass. On a vpay before `f885170c` none of this exists, and
`/dash/v1` takes a bearer token minted by a staff sign-in. See
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

The mechanism is `docs/flows/dashboard-auth.md` (rewritten by #383, 244 lines
as of 2026-10-27, one page with no directory). The decision is
`docs/adr/0036-the-console-is-a-signed-message-under-a-paired-key.md`
(Accepted, dated 2026-10-26). It supersedes ADR-0017 and ADR-0030 entirely,
and ADR-0008's "OIDC session" credential. It does not touch ADR-0018's tenancy
rule. The signed transport itself (what a signature binds, the replay cache,
the coarse refusal) is `/v1`'s, from ADR-0035 and
`docs/flows/signed-transport.md`. This page is only what differs on `/dash/v1`
and `/svc/v1`.

## The invariant, verbatim

> **No `/dash/v1` request is handled unless it is a COSE_Sign1 message signed
> by a console key that is registered to an active member of the dashboard's
> merchant, was paired less than 30 days ago, has not been revoked, is
> addressed to this deployment, and has not been seen before. Every answer to
> such a request is a COSE_Sign1 message signed by this deployment.**

A console key is a **P-256 key a browser generated with WebCrypto
`extractable: false`**, `ESP256` on the wire, stored in `console_keys`
(migration `0056`). vpay answers under one of the two Ed25519 response keys
`/v1` is sealed with. The layer is `EnvelopeMode::Required`: **there is no
optional mode, no compatibility mode and no token fallback.** Do not add one
to make a client easier to write.

## What is built and what is not (as of 2026-10-27)

| Built and proven against a real Postgres                                               | Not built, and not to be described as built      |
| -------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `console_keys` (`0056`), its 30-day `CHECK`, the prefix index (`0057`)                 | the console's own sign-in in the browser (VP-6b) |
| the `/dash/v1` and `$procs` `EnvelopeLayer`s, under console keys                       | vaam's side of `/svc/v1`: its key and its call   |
| `POST /svc/v1/console_keys`, `DELETE /svc/v1/console_keys/{thumbprint}`                | the pairing screen on vaam's admin phone         |
| `vpay-server staff pair-key` and `staff revoke-key`                                    | listing a person's keys (only pair and revoke)   |
| `service_clients:` and `dashboard:` in the YAML, `RetiredKey` for the two deleted keys | any rate limit on `/svc/v1`                      |
|                                                                                        | a run in a cluster; the Helm render was not run  |

The suites: `backends/tests/integration/tests/console_pairing.rs` (10 cases
per `docs/status/backend.md`), `dashboard_read_surface.rs`,
`dashboard_procedure_transport.rs`, `vpay-db`'s `repositories.rs`
(`the_console_key_table_refuses_malformed_rows` and the `migration_0056_…`
case) and `vpay-server`'s `tests/cli.rs`. The guard-by-guard proof is
`docs/status/verification/2026-10-26-vp6a-console-keys.md`. The unsigned-`401`
cases assert the layer's own body, not only the status, so loosening
`Required` to `Optional` fails them.

**The only signing client in the repository is test code**
(`backends/tests/integration/tests/support/sealed.rs`). vpay ships no
console, CLI or SDK that signs a `/dash/v1` request, so as of 2026-10-27 a
person reads `/dash/v1` only through a client they bring. The dashboard app
renders a notice saying so.

## Pairing: two doors, one function

Both end in `vpay_api::console_keys::store_key`.

1. **`POST /svc/v1/console_keys`**, called by a **service** (vaam), never by a
   person or a merchant. Body
   `{vaam_user_id, public_key, possession_proof, issued_at}` plus optional
   `display_name` and `label`, with `deny_unknown_fields` (`PairBody`, which
   calls that "a security property": a `role` or `merchant_id` is a `400`, not
   a field silently ignored). The order of checks: the signing key is a
   `service_clients` key whose entry holds `console_pairing`; the shape of
   every field; `|now - issued_at| ≤ 300 s` (`PROOF_MAX_SKEW_SECS`); then the
   possession proof, ECDSA-SHA256 over
   `"vpay.console-pair.v1\n" + publicKey + "\n" + issuedAt` (`PROOF_TAG`,
   `possession_message`). Then the person is looked up by
   `(issuer = "vaam", subject = vaam_user_id)` in `credentials`
   (`kind = oidc`, the link row ADR-0019 designed; `VAAM_ISSUER`).
   - **Not linked yet:** one transaction provisions a **plain** member
     (`is_admin = false`) of `dashboard.merchant_id`, the link and the key
     (`pair_console_key_in_tx`). A refusal leaves no member and no link.
   - **Linked:** an administrator is refused (`403`
     `console_key_administrator`, logged as such); a disabled member or one of
     another merchant is `403` `console_key_not_permitted`, one answer for both.
   - Answers `201` with `{thumbprint, staff_member_id, expires_at, …}`, `200`
     for a repeat of the same pairing.
2. **`vpay-server staff pair-key --staff <email|stf_…> --public-key <key>
[--label …]`**, the break-glass (owner ruling Q2), for an **existing,
   active** member. No possession proof, because an operator with database
   access already holds more authority than a pairing grants. It creates no
   member, changes no flag, and prints one line:
   `<staff id> <key id> <thumbprint> <expires_at>`.

`public_key` is the 65-byte SEC1 **uncompressed** point as base64url. A
compressed point is refused, not normalised
(`a_bad_proof_a_stale_one_and_a_compressed_point_write_nothing`).

### What a pairing never does

These are the rules an agent is most likely to "simplify" away:

- **It never creates or upgrades an administrator.** The body has no role
  field and rejects one. A provisioned member is `is_admin = false`. An
  existing member's flag is never written. A linked administrator is refused
  outright, because a service holding `console_pairing` can name **any**
  linked person, and this route must not be a way to a key that speaks as an
  administrator. Proof: `pairing_never_creates_or_upgrades_an_administrator`.
  **The only way to key an administrator is `staff pair-key`, and the only way
  to make one is `vpay-server staff add --admin`.** Both are run by an
  operator. ADR-0036 says this outright; it is not an accident.
- **It never names a tenant.** There is no field for one. The tenant comes
  from `dashboard.merchant_id` in configuration.
- **It never moves or reinstates a key.** A thumbprint is unique for ever and
  the row is never deleted (`vpay_db::ConsoleKeys` has no `delete`). Another
  member's key, a revoked key and an expired key are all `409`
  `console_key_taken` (`KeyTaken`). Two vaam ids that differ only in letter
  case collide on `staff_members.email` and get their own `409`,
  `console_key_identity_taken`, "cannot be provisioned", and nothing is
  written (`an_identity_collision_is_not_a_taken_key_and_writes_nothing`).
- **The proof binds the key and the instant, not an audience.** ADR-0036
  decision 2 records why that is enough: the proof only reaches vpay relayed
  by the trusted service, and a lifted proof can only register the key its
  signer chose, for the person the service already names. Do not "harden" it
  by adding an audience the browser cannot know.

## Revocation, expiry, and the per-request read

- **Revocation has two doors**, both through `console_keys::revoke`:
  `DELETE /svc/v1/console_keys/{thumbprint}` and
  `vpay-server staff revoke-key --thumbprint <64 hex>`. Either may revoke
  **an administrator's** key. The asymmetry with pairing is deliberate:
  revoking grants nothing, and a service that could not cut off the most
  privileged keys would be useless in the incident it exists for. Idempotent
  (`a_revocation_takes_effect_on_the_next_request_and_is_idempotent`).
- **A key lives exactly 30 days (720 h) from `paired_at`**, enforced by the
  `CHECK` `console_keys_lives_exactly_thirty_days` and by the read. There is no
  job to run and no extension. A console pairs again with a **new** key.
- **Revocation and expiry take effect on the next request.** The resolver
  (`vpay_api::envelope::console`, `ConsoleKeyResolver`) reads `console_keys`
  on every request and **caches nothing**. Do not add a cache to save the
  read: that turns "revoked" into "revoked within the TTL". The read is
  `thumbprint LIKE '<kid>%'` before the signature is verified, so every
  well-formed `kid` costs one query; `0057` adds a `text_pattern_ops` index,
  partial on `revoked_at IS NULL`, because the unique btree answers a prefix
  `LIKE` only under the C collation (ADR-0036 decision 12).
- **A database that cannot answer is a `500`, never a miss.** "No key" would
  make an outage look like a revocation. "Yes" would make it a bypass.
- After the signature verifies, the handler side re-reads the key (revoked
  between the two reads → `403`) and the member (disabled, gone or of another
  merchant → sealed `403`).

A lost browser profile is a revocation, not a password reset. There is
nothing to reset.

## One call

The body of **every** request is a COSE_Sign1 message, a `GET` included.

| Cause                                                                                      | Answer                                                                                                    |
| ------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| no signature, unknown key, revoked key, expired key, bad signature, wrong audience, replay | the layer's coarse `401` (`UNAUTHORIZED`), **unsigned, identical bytes** for every cause                  |
| a merchant's key or a service's key on `/dash/v1`; a console key on `/v1`                  | the same `401`: each surface resolves against its own registry only                                       |
| `POST`/`PUT`/`PATCH`/`DELETE` on a `/dash/v1` read route                                   | the layer's own `405` with `Allow: GET`; a **signed** write is its `415`. Before the router, whatever key |
| an anonymous bodiless `OPTIONS`                                                            | the router's `405`, never a `500`                                                                         |
| a verified key whose member is disabled, gone or of another merchant                       | sealed `403`, one answer for the three                                                                    |
| an administrator naming a merchant this deployment does not serve                          | sealed `400` naming `merchant_id`                                                                         |

**Three layers, not one**: `/dash/v1`'s hand-written reads, CrateStack's
procedure transport under `/dash/v1/$procs/…` (a layer the generated module
builds, because its route table and contract digests are private to
`vpay-db`), and `/svc/v1`. Each has its own contract domain tag in
`vpay-wire` (`vpay/dash-route-contract/v1`, `vpay/svc-route-contract/v1`), so
one surface's digest is never another's. `/dash/v1` resolves keys against
`console_keys`, `/svc/v1` against `service_clients`, `/v1` against
`merchant_clients`, and **boot refuses one public key in two lists**
(`DuplicateCoseKey` spans merchants and services).

## Configuration

```yaml
dashboard:
  merchant_id: acme-cameroon-tenant # must be a merchant_clients entry

service_clients:
  - client_id: vaam
    capabilities: [console_pairing] # a closed set; today only this
    cose_keys:
      - public_key: 11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo # Ed25519, unpadded base64url
        label: vaam-2026-10
```

- **No `dashboard:` means no console**: `/dash/v1`, its procedure transport
  and `/svc/v1` are all the honest `404`, and every `vpay-server` mode still
  boots (owner ruling R4). The base `config/application.yml` carries neither
  block.
- **A capability is a field of this file and nothing else.** No route,
  database row or environment variable can grant one, so granting
  `console_pairing` is a reviewed configuration change, like adding a
  merchant. `SVC_ROUTES` names the capability each route needs, the scope
  step (`vpay_api::envelope::service`) enforces that row, and the handler
  demands a `PairingAuthority`, a type only that step can construct. A new
  `/svc/v1` route without a capability row fails the scope step's own check.
- A service is switched off with the merchants' kill switch,
  `disabled_clients`, effective on its next request
  (`a_disabled_service_is_refused_on_its_next_request`).
- `/svc/v1` is on the **business** surface (`deployment.surfaces` containing
  `business`), not the management tier that serves `/dash/v1`: vaam dials the
  one in-cluster origin it already knows, the `-server` Service (ADR-0036
  decision 11). A `surfaces: [management]` process answers `404` on
  `/svc/v1`. With `networkPolicy.enabled`, list the caller in
  `networkPolicy.serviceIngress.peers`. No HTTPRoute or Ingress publishes
  `/svc/v1`.

### Upgrading a configuration from before `f885170c`

**`dashboard_client` and `staff_auth` are refused at boot, even as a null.**
`dashboard_client: ~` is refused exactly as a populated block is: naming the
key is the refusal. The process exits `78` with `ConfigError::RetiredKey`,
whose message says what to delete and what to add instead
(`RETIRED_KEYS`, `reject_retired_keys` in `vpay-config/src/config.rs`). It is
a hard cutover by design. **Do not soften it into a warning or a silently
ignored key**: a key the binary dropped without a word would leave an operator
believing a sign-in exists that does not.

_Docs↔code disagreement, 2026-10-27: `docs/reference/vpay-config.md` says a
file carrying either key "fails to load (`deny_unknown_fields`)". The code's
own comment says `Config` does **not** set `deny_unknown_fields`, which is why
`reject_retired_keys` exists. The code wins._

The rest of the checklist is `docs/flows/dashboard-auth.md` § "Upgrading from
before VP-6a": retire the password pepper, the TOTP key and the SSO client
secret, which nothing reads now; the Helm values `dashboard.clientId`,
`dashboard.redirectUri` and `dashboard.scope` are gone. **The migration signs
everyone out**: `0056` drops `oauth_authorization_codes`, `staff_sessions` and
`staff_sso_attempts` and deletes the `password` and `totp` rows of
`credentials`. The `oidc` link rows stay. `0056` is an applied migration from
the day it merged: do not edit it, write `0058`.

## Where each piece lives

| Piece                                       | Where                                                                                       |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| the table and its 30-day `CHECK`            | `backends/migrations/0056_console-keys.sql`; the prefix index in `0057_…`                   |
| the repository                              | `vpay_db::ConsoleKeys` (`backends/crates/vpay-db/src/console_keys.rs`)                      |
| parsing a key, the proof, pairing, revoking | `vpay_api::console_keys` (`backends/crates/vpay-api/src/console_keys.rs`)                   |
| the key resolver and the tenancy step       | `vpay_api::envelope::console`                                                               |
| `/svc/v1` and the capability check          | `vpay_api::svc` (`SVC_ROUTES`, `PairingAuthority`), `vpay_api::envelope::service`           |
| the operations and their digests            | `backends/crates/vpay-wire`                                                                 |
| the break-glass                             | `vpay-server staff pair-key` / `staff revoke-key` (`backends/apps/vpay-server/src/main.rs`) |

## What this replaced, and what this skill said about it before

Until vaam-apps/vpay#383 merged (2026-10-27), this skill had two reference
pages here, `staff-auth.md` and `sso.md`. **Both were deleted with the code
they described.** What they said, so an agent on an older tree, or reading an
old document, recognises it:

- ~~Staff sign in to `/dash/v1` with a password (argon2id, a deployment pepper)
  and an RFC 6238 TOTP code, get a server-side session (30 minutes idle, 12
  hours absolute) and, through the authorization-code grant with PKCE, a
  900-second RS256 bearer token the dashboard re-mints at 80 % of its TTL
  (ADR-0017).~~ **Deleted 2026-10-27:** no password, TOTP code, session,
  authorization code or bearer token exists on `/dash/v1`. The eight routes
  under `/dash/v1/staff/*` and `/dash/v1/oauth/*` are gone.
- ~~Sign-in is rate-limited per email and per source address in Postgres
  (`rate_limit_windows`, `staff_auth.rate_limits`, `staff_auth.trusted_proxies`),
  and every refusal is one `401`.~~ **Deleted 2026-10-27:** the limiter's
  callers went with the sign-in. **`rate_limit_windows` and
  `vpay_db::RateLimits` remain as unused scaffolding with no caller.** That is
  not an invitation to wire them to `/svc/v1`: a pairing is a signed call from
  a registered service, and rate limiting it is listed as not built.
- ~~Staff can sign in through an OpenID Connect provider (Keycloak) under
  `staff_auth.sso`, with password + TOTP kept as the break-glass (ADR-0030).~~
  **Deleted 2026-10-27 (owner ruling Q1: phone pairing only):** the broker, the
  `staff_sso_attempts` table, `vpay_api::staff::sso`, and
  `staff_sso.rs`/`staff_sso_keycloak.rs` are gone. The break-glass is now
  `staff pair-key`.
- ~~`vpay-server staff add` prints a one-time password and the first sign-in
  forces a change.~~ **Changed 2026-10-27:** it creates a member and **no
  credential** and prints the `stf_…` id alone on stdout.
- `staff_members.last_sign_in_at` still exists and is **never written**
  (2026-10-27). Do not read it as "last seen".

The four `staff_sign_in.rs` cases that needed `127.0.0.2`–`127.0.0.35` (and so
`just loopback-aliases` on macOS) were deleted with that file, and vpay's
`CLAUDE.md` dropped its paragraph about them in the same change.
