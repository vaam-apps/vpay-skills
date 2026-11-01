# The dashboard: what looks wrong and is not

_Verified against vpay `10f42747` (2026-10-30), the merge of vaam-apps/vpay#394,
for the first table, the new "where `human_login` is served" table and the two
demo sections, each marked **Corrected 2026-10-30** where it changed; the rest
against `f885170c` (2026-10-27), the merge of vaam-apps/vpay#383. It was
verified at `d3a8810b` (2026-09-16) before that, and the sections that changed
say what they said before.
Version-sensitive claims carry the date they became true — see
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Sources: `docs/runbooks/demo/dashboard-sign-in.md` (rewritten by #383, and
again for VPA-5c), `docs/flows/dashboard-auth.md` and, since 2026-10-30,
`docs/flows/dashboard-auth/human-login.md` and `sso.md`. These are the things a new agent "fixes" and
should not. `vpay-dashboard` is the skill for the surface itself; its
`references/console-keys.md` has the full refusal table.

## Since 2026-10-27: the key is the only way in, unless `human_login` is served

_This heading said "nobody can sign in, on purpose" until 2026-10-30. **Corrected
2026-10-30:** vpay#390–#394 restored the staff sign-in behind `auth.methods`'
`human_login` (ADR-0037). Everything in this table is still exactly true of a
deployment that does not serve it, which includes vaam
(`auth.methods: [keypair, hmac_webhooks]`). The next table is for one that
does._

| What you see                                                                                          | Why it is correct                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`/login` shows "This console signs in with a paired key", and no field**                            | ~~Every dashboard URL shows it: the app is one notice page since #383.~~ **Corrected 2026-10-30:** `/login` asked `GET /.well-known/vpay-auth` and `human_login` is not in it. Since vpay#392 that is the only place it appears (`PairedKeyOnly`); `/` sends you to `/login`, and an unknown path is a plain not-found. The console's own key sign-in is VP-6b and **not built**. If you expected a password sign-in, see the next table's first row |
| **`vpay-server` exits `78` with `ConfigError::RetiredKey` naming `dashboard_client` or `staff_auth`** | You are on a release cut **between VP-6a and VPA-3**: both keys were deleted there, and naming either is refused **even as a null**. **Corrected 2026-10-30:** from VPA-3 (vpay#390) on, `RetiredKey` does not exist and the two keys are `human_login`'s blocks; see `AuthMethodDisabled` below                                                                                                                                                     |
| **`vpay-server` exits `78` with `AuthMethodDisabled` naming `staff_auth` or `dashboard_client`**      | (Since 2026-10-30.) `auth.methods` is written and leaves out `human_login`, as vaam's does, so the block is dormant configuration and refused. Delete the block (`staff_auth.sso` goes with it), or add the method. Do not add the method on vaam                                                                                                                                                                                                    |
| **Every `/dash/v1` request answers `401` with an unsigned body, whatever you send**                   | A request must be a COSE_Sign1 message under a paired console key, a `GET` included. No signature, an unknown, revoked or expired key, a merchant's or a service's key, a bad signature, the wrong audience and a replay all get **the same bytes**, by design. ~~A bearer token is not a credential here any more~~ _(2026-10-30: it is not one here, on a keys-only deployment; it is where `human_login` is served)_                              |
| **`/dash/v1`, `/dash/v1/$procs/…` and `/svc/v1` are all `404`**                                       | The configuration has no `dashboard:` block, so none of the three is mounted. That is how a deployment runs without a console. _(Since 2026-10-30 a served `human_login` mounts `/dash/v1` on its own; `/svc/v1` still needs `dashboard:`)_                                                                                                                                                                                                          |
| **`/svc/v1/console_keys` is `404` on the management tier**                                            | `/svc/v1` is on the **business** surface (ADR-0036 decision 11). A `surfaces: [management]` process does not serve it                                                                                                                                                                                                                                                                                                                                |
| **A `POST` to `/dash/v1/payment_intents` is `405` (or `415` if signed), not `403`**                   | Since #383 the signed transport layer refuses writes before the router. ~~`403` from the boundary~~ was the answer until 2026-10-27                                                                                                                                                                                                                                                                                                                  |
| **`/dash/v1/$procs/retrievePaymentIntent` (or any merchant MCP procedure) is `404`**                  | `DASHBOARD_PROCEDURES` admits the five searches only (since vaam-apps/vpay#377, 2026-10-27). The generated router mounts every procedure, and the allowlist keeps the merchant ones off the staff surface                                                                                                                                                                                                                                            |
| **Pairing answers `403` for a person who is already an administrator**                                | Pairing never yields an administrator's key. Use `vpay-server staff pair-key`, the break-glass                                                                                                                                                                                                                                                                                                                                                       |
| **Pairing a key that was revoked or expired answers `409`**                                           | A thumbprint is unique for ever and the row is never deleted. Pair a **new** key                                                                                                                                                                                                                                                                                                                                                                     |
| **A key that worked yesterday is `401` today**                                                        | Keys live exactly 30 days from `paired_at`, and revocation takes effect on the next request. Nothing is cached                                                                                                                                                                                                                                                                                                                                       |

## Since 2026-10-30, where `human_login` is served

The sign-in is `vpay-dashboard`'s `references/human-login.md`.

| What you see                                                                        | Why it is correct, or what is actually wrong                                                                                                                                                                                                                                           |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **You wrote `staff_auth` and `/login` still says "paired key"**                     | Served needs the method permitted **and** both `staff_auth` and `dashboard_client`. One block of two (or the method named without both) does not run quietly unserved: it is a boot refusal, `AuthMethodNotConfigured`. So check that the process you are asking booted with this file |
| **`/login` says "No sign-in is configured" and lists variables**                    | vpay serves the method; this container lacks `VPAY_DASHBOARD_CLIENT_ID`, `_REDIRECT_URI` or `_SCOPE` (the chart's `dashboard.clientId`, `redirectUri`, `scope`). The operator's problem, not the person's                                                                              |
| **"Sign-in is not reachable", with a request id**                                   | vpay could not be asked (five-second deadline). It is never shown as "paired key only"                                                                                                                                                                                                 |
| **The sign-in succeeds, and the next page is `/login` again**                       | The cookies are `__Host-` and `Secure`; over plain `http` a browser stores them only on `localhost`. Use TLS or `localhost`. Do not make `Secure` conditional (AGENTS.md forbids branching on the environment)                                                                         |
| **Every sign-in failure says the same sentence**                                    | `StaffSignInRefused`, one `401` for a wrong address, password, code, a disabled member or a dead session. The stage is in vpay's log (`stage`), never the body                                                                                                                         |
| **"Signed in, but not authorised" at `/`**                                          | The session is good and vpay refused to mint a `/dash/v1` token (an account of another merchant than the dashboard's). A page, not a redirect, because every redirect would lead back                                                                                                  |
| **A signed request with an `Authorization` header is `400` naming `authorization`** | One request, one credential. Strip the header                                                                                                                                                                                                                                          |
| **An unsigned request is `401` `missing_bearer_token`, not the coarse `401`**       | `/dash/v1` is in `Optional` mode here, so an unsigned request is read as a token request                                                                                                                                                                                               |
| **A `refresh_token` grant is `400` on `grant_type`**                                | There is no refresh token on this surface, and there never was. The app replaces its token at 80 % of its life by running the code leg again                                                                                                                                           |
| **No SSO button**                                                                   | Drawn only when `GET /dash/v1/staff/sso` says `enabled: true` (`staff_auth.sso` written) **and** `VPAY_DASHBOARD_SSO_ISSUER` is set (an `http` issuer counts only on a loopback host). The compose and demo stacks set neither, on purpose: no provider                                |
| **`/login?sso=failed` and no reason**                                               | By design. The reason is in vpay's log, prefixed `sso`. The commonest is a Keycloak realm whose `amr` lacks `otp` (`sso.md` § "What an operator must set up")                                                                                                                          |
| **A member who lost a password cannot be reset**                                    | There is no `staff reset-password`, `disable` or `remove` (gap, 2026-10-30). The SQL is in `docs/runbooks/deploy-and-rollback.md` § "Staff who lost a password or an authenticator"                                                                                                    |
| **Every staff password was lost in an upgrade**                                     | Migration 0056 deleted them and 0058 does not restore them. Only an export taken before 0056 brings them back (the same runbook, "Keeping staff passwords across 0056 and 0058")                                                                                                       |
| **`staff_sign_in.rs` fails on macOS with `EADDRNOTAVAIL` (os error 49)**            | Back since vpay#390, it sends from `127.0.0.2`–`.4`, `.20` and `.30`–`.35`, which macOS does not assign. Its message says run `just loopback-aliases`; **no such recipe exists at `10f42747`**. Run `sudo ifconfig lo0 alias 127.0.0.N up` for each. Linux and CI are unaffected       |

## Reading `/dash/v1` by hand in the demo

~~`just demo-staff` creates `ada@example.test` with **no credential** (since
2026-10-27) and writes the `stf_…` id to `.e2e/vpay-demo/staff-id.txt`.~~
**Corrected 2026-10-30 (vpay#394, VPA-5c):** the generated demo overlay serves
`human_login`, so `just demo-staff` writes the **one-time password** to
`.e2e/vpay-demo/staff-password.txt` and no id file. Sign in at
`http://localhost:3000` with it; you land on an account page and nothing that
reads payments. If `demo-staff` fails saying `staff add` "printed an id, not a
password", the overlay predates VPA-5c: `just demo-down`,
`rm .e2e/application-demo.yml`, `just gen-demo-keys`, start again. To read
`/dash/v1` you still need a key: read the id from the database, then attach a
key you hold with the break-glass, through the demo's compose file set:

```bash
psql … -Atc "select id from staff_members where email = 'ada@example.test'"
vpay-server staff pair-key --staff <that stf_… id> \
  --public-key <SEC1 uncompressed P-256 key, base64url>
```

It prints `<staff id> <key id> <thumbprint> <expires_at>`. A client that holds
the private half then signs each request. **vpay ships no such client**: the
only one in the repository is the integration suites' test support
(`backends/tests/integration/tests/support/sealed.rs`). `staff revoke-key
--thumbprint <64 hex>` ends a key, idempotently. The runbook has the full
`docker compose … run` lines.

~~`vpay-server staff add` creates an account that signs in with a one-time
password, then TOTP. Staff authentication is ADR-0017: argon2id with a
deployment pepper, RFC 6238 TOTP with a sealed secret, a strictly-increasing
replay guard, mandatory enrolment and a one-time password.~~ **Corrected
2026-10-27:** ADR-0017 is superseded by ADR-0036 and none of that exists.
`staff_sign_in.rs` is deleted; `dashboard_read_surface.rs`,
`dashboard_procedure_transport.rs` and `console_pairing.rs` are the proofs now.
ADR-0018's cross-tenant **read-only** admin role stands. **Corrected again
2026-10-30:** all of it is back where `human_login` is served (vpay#390,
ADR-0037 decision 8), and `staff_sign_in.rs` with it. A one-time password is
what `staff add` prints there; the id is what it prints elsewhere.

## The components that come back with VP-6b

As of 2026-10-27 no screen renders these, but the components survive with
their tests and VP-6b will render them again. When they reappear, they will
look like this on purpose:

| What you see                                             | Why it is correct                                                                                                                                                                                                                                                 |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **The payer column is an em dash**                       | `charges.payer_ref_masked` is never written by anything. The detail component renders the column as null rather than deriving a mask from the payer's unmasked number. **The day the column is written, the value appears.** Do not derive a mask at render time. |
| **The list has a "Methods" column and no "Rail" column** | `GET /dash/v1/payment_intents` returns no charge, so the only rail-shaped value there is the set of rails the intent _may_ be confirmed against. The rail that actually took it is on the detail.                                                                 |
| **There is no page count on the payments list**          | The REST list serves cursors and a `has_more`, and no count. The procedure lists' `totalCount` is not a total the cursor list can borrow.                                                                                                                         |

## Two things it cannot do, by design

**Anything at all to a payment.** `/dash/v1` refuses every write method in the
signed transport layer, before the router matches (since 2026-10-27; the
boundary's `403` before that). There is no re-poll, no replay, no refund and
no annotation, and therefore no `audit_log`, because there is nothing yet to
audit. ADR-0008 wants one row per dashboard write; that write surface does not
exist.

**Any other slice.** Webhook configuration and retries, balances, settings and
rail health are not built. ~~The navigation does not link to them, and
`layout.test.tsx` fails if it ever does.~~ _(Until 2026-10-27. The navigation
is deleted with the shell, and `layout.test.tsx` now fails on any page but the
notice.)_ The read-only refunds, deliveries, customers and checkouts lists
existed from 2026-09-14 until #383 deleted their pages; their procedures are
still served.

## The demo's dashboard container

`dashboard` is in `demo_services`, so `just demo` starts it (since exp28,
2026-09-07), and `demo_dashboard_port` (since issue #78, 2026-09-10) moves its
host port so a second stack can publish it elsewhere. ~~The OP had
`http://localhost:3000/…` registered as the dashboard client's `redirect_uri`,
matched byte for byte, so the port and the overlay had to move together.~~
**Corrected 2026-10-27:** there is no dashboard OAuth client any more, so the
port is only a port. The container renders the notice. **Corrected
2026-10-30 (VPA-5c):** the demo overlay has a `dashboard_client` again, with
`http://localhost:<demo_dashboard_port>/dash/v1/callback` as its redirect URI,
and `just gen-demo-keys` regenerates the overlay when the port or
`demo_dashboard_merchant` changes, so they still move together without a hand
edit. The container serves the sign-in.
