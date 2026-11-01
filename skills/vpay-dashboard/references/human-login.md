# Staff sign-in (`human_login`): password, TOTP, session, SSO, and where each is served

_Verified against vpay `10f42747` (2026-10-30), the merge of vaam-apps/vpay#394,
by reading `docs/flows/dashboard-auth/human-login.md`,
`docs/flows/dashboard-auth/sso.md`, `docs/flows/dashboard-auth.md`, ADR-0037
decision 8, `docs/status/backend.md` § "Staff sign-in is served again…" and
§ "Staff SSO is served again…", `docs/status/frontend.md` § VPA-5a, 5b and 5c,
`frontends/apps/dashboard/README.md`, and the identifiers below in the code.
**Nothing here was re-run for this page**: test names are checked to exist,
not to pass. On a vpay between `f885170c` (2026-10-27) and `623d659f`
(2026-10-30) none of this is served, and `staff_auth`/`dashboard_client` are a
boot refusal (`RetiredKey`). See
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

This page exists because the sign-in was deleted on 2026-10-27 (vpay#383,
ADR-0036) and restored, **as one deployment's choice**, three days later. Both
facts are true of `master`; which one is true of a deployment is its YAML.

## Served, or not: one condition

Owner ruling R5 (2026-10-27) made the ways in a YAML list: `auth.methods`,
any of `keypair`, `machine_tokens`, `hmac_webhooks`, `human_login`. **Absent
means all four** (ADR-0037). `human_login` is _served_ when it is permitted
**and** both of its blocks are written, `staff_auth` (password pepper, TOTP
key) and `dashboard_client` (the one authorization-code client):
`AuthMethods::human_login_served` in `backends/crates/vpay-config/src/auth.rs`.
Five places ask that one function and cannot disagree (and discovery,
`GET /.well-known/vpay-auth`, lists `human_login` only when it is served):

| What                                      | Served                                   | Not served                                    |
| ----------------------------------------- | ---------------------------------------- | --------------------------------------------- |
| `/dash/v1/staff/*`, `/dash/v1/oauth/*`    | mounted (`STAFF_ROUTES`, 11 as of 10-30) | the honest `404`                              |
| `/dash/v1` and `$procs` envelope          | `Optional` (`Off` with `keypair` off)    | `Required`: unsigned is the coarse `401`      |
| the scope step, for a request not signed  | validates a staff access token           | `500`: only reachable if the layer is missing |
| `/v1/oauth/jwks.json` on a management pod | mounted (the validator reads its own)    | only where `machine_tokens` or MCP reads it   |
| `vpay-server staff add`                   | prints a one-time password (must change) | prints the `stf_…` id                         |

Half a sign-in is refused at boot, never quietly unserved: a block without
the method in an explicit list is `AuthMethodDisabled`, the method named
without both blocks (or the default with one block of two) is
`AuthMethodNotConfigured`. `dashboard.merchant_id` and
`dashboard_client.merchant_id` must agree when both are written
(`DashboardTenantMismatch`). **`/dash/v1` is mounted by `human_login` alone**
(tenant from `dashboard_client.merchant_id`), while `/svc/v1` follows the
`dashboard:` block alone, because a session-only deployment has nothing to
pair into (`vpay_api` `lib.rs`, the `serves_dash`/`serves_svc` comment).

**vaam's deployment sets `auth.methods: [keypair, hmac_webhooks]`** (ADR-0037,
Consequences), so none of this page is served there, and the console key is
the only way in on vaam. Do not write vaam-facing docs or code as if a
password exists.

## The flow (VPA-3, vpay#390, since 2026-10-30)

vpay is the OpenID Provider and authenticates the person itself:
`POST /dash/v1/staff/login` (email + password, argon2id with the pepper) →
a `pending_totp` session whose answer says `next: totp`, or `totp_enrolment`
with an `otpauth` URI and a sealed secret on a first sign-in →
`POST /staff/totp` (RFC 6238 ±1 step, compare-and-swap counter) →
`authenticated` →
`GET /oauth/authorize` (PKCE S256, exact `redirect_uri`, a 60 s single-use
code) → `POST /oauth/token` → an RS256 access token, audience
`dashboard_client.client_id`, the `vpay_merchant_id` claim.
`/staff/password` is the forced change; `/staff/session`,
`/staff/session/stage` and `/staff/logout` complete the eight non-SSO routes.

- **The session**: 256 bits from the OS CSPRNG, stored as its SHA-256 in
  `staff_sessions`; 30 minutes idle, 12 hours absolute, checked on read.
  Nothing deletes an expired row (refused and left, so a log can tell
  "expired" from "never existed"). Logout **deletes** the row. The minted JWT lives out its TTL
  (`staff_auth.access_token_ttl_seconds`, 10 to 3600, default 900).
- **One refusal**: every sign-in failure is `ApiError::StaffSignInRefused`,
  one `401`, one sentence; the stage is in the log only. A missing account
  still costs one argon2id verification.
- **Rate limits are back on `rate_limit_windows`**: ten attempts per five
  minutes per email **and** per source address, five current-password
  attempts per session, refused before any credential work. The source is
  the transport peer unless `staff_auth.trusted_proxies` names proxies.
- **No refresh token, then or now.** The client is registered for
  `authorization_code` alone; a `refresh_token` request is `400` on
  `grant_type` while served, `404` when not
  (`the_refresh_token_grant_is_not_honoured_while_the_method_is_served`). Do not
  "restore" one: it never existed on this surface.

## One credential per request

| Request on `/dash/v1` where `human_login` is served | Answer                                                   |
| --------------------------------------------------- | -------------------------------------------------------- |
| signed (console key)                                | exactly the key path in `console-keys.md`, sealed        |
| signed **and** `Authorization: Bearer`              | `400` naming `authorization`, sealed                     |
| no signature, no token                              | `401` `missing_bearer_token`                             |
| a token that does not validate                      | `401` `invalid_token`; `503` if the JWKS is down         |
| valid, without this tenant's claim or scope         | `403` (a `client_credentials` token can never read here) |
| member disabled, gone or another tenant's           | `403`, one answer for the three                          |
| an unsigned write, with a good token or none        | still the layer's `405`, before any token is read        |

A token passes the member re-read and then the **same** `resolve_tenancy` a
key does, so ADR-0018's admin override cannot differ by door. A session's
answers are **plain JSON**, unsealed and without contract-lock protection:
the trade a deployment makes by serving the method. `/svc/v1` stays
`Required` whatever `auth.methods` says.

## Single sign-on (VPA-4, vpay#391, since 2026-10-30)

Served exactly when `human_login` is **and** `staff_auth.sso` is written; there
is no switch of its own. Written without the method it is `AuthMethodDisabled`
(naming `staff_auth`). A provider that cannot be built is a boot refusal
(`StaffLoginError::Unusable`, exit `78`), where before VPA-4 it logged and
fell back. `GET /dash/v1/staff/sso` says `enabled`; `start` and `callback` are the
`404` when it is `false`. Code: `backends/crates/vpay-api/src/staff/sso/`.

- **Only the human step federates.** The callback verifies the provider's
  id_token (algorithm allow-list, `kid`, signature, `iss`, `aud`, `azp`,
  `exp`, `iat`, nonce, group, second factor), discards it, and ends in the
  same `authenticated` `staff_sessions` row; vpay then runs its own code leg.
- **Members kept through 0056 resolve again** by `(issuer, subject)`
  (`find_federated`); an email is never matched. A changed issuer string
  orphans every link.
- **Every refusal is the same `401`**; the reason is a log line prefixed
  `sso`. At most one `prompt=login` retry, enforced by `is_retry` on the
  `staff_sso_attempts` row.
- **Helm**: `networkPolicy.idpEgress` is back (deleted by #383) for a private
  or in-cluster provider, or any provider once `railsEgress` is off.
- **Not built**: no browser has completed it and no e2e case drives it (the
  compose stack has no provider); no linking of an existing member to an SSO
  identity; no back-channel or RP-initiated logout; no continuous group check;
  never run against any provider but a simulated one and Keycloak 26.6.1 in a
  testcontainer.

## The dashboard application's half (VPA-5a/5b/5c, vpay#392–#394)

`frontends/apps/dashboard` signs a person in again **only where vpay serves
`human_login`**. `/login` reads `GET /.well-known/vpay-auth` on every render
and is one of four things: "Sign-in is not reachable" (with the request id)
when vpay cannot be asked, "This console signs in with a paired key"
(`PairedKeyOnly`) when the method is not served, the missing variable names
when it is served and the container is not configured, and the sign-in.
**A discovery that could not be read is never treated as "paired key only".**

| Route                        | What                                                                     |
| ---------------------------- | ------------------------------------------------------------------------ |
| `/login`                     | email, then password: one question per screen; SSO button if configured  |
| `/login/totp`                | the code; a first sign-in shows the QR (the typed key in a bottom sheet) |
| `/login/password`            | the forced change while the one-time password is in force                |
| `/`                          | the account page (who, which merchant, Sign out), or `MintRefused`       |
| `/signed-out`                | route handler: forgets a dead cookie, then `/login`                      |
| `POST /api/auth/sso/start`   | origin-checked; `303` to the provider only on the issuer's origin        |
| `GET /api/auth/sso/callback` | 25 s deadline (others: 5 s); `/`, the provider again, or `?sso=failed`   |

- **Configuration** (`src/config/runtime.ts`): `VPAY_DASH_API`,
  `VPAY_DASHBOARD_CLIENT_ID`, `VPAY_DASHBOARD_REDIRECT_URI`,
  `VPAY_DASHBOARD_SCOPE`, optional `VPAY_DASHBOARD_PUBLIC_ORIGIN` and
  `VPAY_DASHBOARD_SSO_ISSUER`. The chart renders them from
  `dashboard.clientId`, `redirectUri`, `scope`, `ssoIssuer` when set (since
  VPA-5c). `staff_auth.sso.redirect_uri` must end in
  `/api/auth/sso/callback`; `dashboard_client.redirect_uris` is never visited.
- **Cookies** (`src/server/cookies.ts`): `__Host-vpay_dash_session` (12 h),
  `__Host-vpay_dash_enrolment` and `__Host-vpay_dash_sso_pending` (10 min
  each); all `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, no `Domain`.
  **Over plain `http` a browser keeps them only on `localhost`.** On
  `http://dashboard:3000` the sign-in "succeeds" and does not stick. Do not
  branch on the environment to fix it: put it behind TLS.
- **`src/server/dash-fetch.ts` builds every request** with one credential and
  copies nothing from the browser. Its `console-key` arm cannot be constructed
  until VP-6b.
- **Still not here**: any page that reads `/dash/v1` (the account page links
  to nothing), the BFF (`app/api/dash`), `middleware.ts`, `src/dash/*`'s
  providers and Refine auth. `src/layout.test.tsx` pins the four `page.tsx`
  files; a fifth changes that list in the same commit. Which credential wins
  when a browser holds both a session and a key is VP-6b's to decide.

## The demo and e2e stacks serve it (since VPA-5c, 2026-10-30)

`just gen-demo-keys` writes the `demo` overlay with
`auth.methods: [keypair, hmac_webhooks, human_login]`, a `dashboard_client`
and a `staff_auth` whose secrets are `${VAR}`s generated into
`.e2e/staff-auth.env` (mode 0600). `just demo` shares that overlay, so it
serves the sign-in too. `just demo-staff` keeps the one-time password in
`.e2e/<project>/staff-password.txt` and **fails if `staff add` printed a
`stf_…` id**, which means the overlay is stale (`just demo-down`,
`rm .e2e/application-demo.yml`, `just gen-demo-keys`). The stack sets no
`VPAY_DASHBOARD_SSO_ISSUER` and writes no `staff_auth.sso`.
`frontends/tests/e2e/cypress/e2e/dashboard-human-login.cy.ts` (seven tests)
passed 7 of 7 on CI's first `e2e (compose)` run, which was of the spec before
its review round.

## Upgrading, and members who lost something

- **Migration 0056 deleted every `password` and `totp` credential, and 0058
  (VPA-2, vpay#388) does not bring them back.** 0058 recreates
  `staff_sessions`, `oauth_authorization_codes` and `staff_sso_attempts`
  (empty) and drops `credentials_password_and_totp_are_retired`. To keep
  passwords, export before 0056 and import after 0058:
  `docs/runbooks/deploy-and-rollback.md` § "Keeping staff passwords across
  0056 and 0058". `docs/status/backend.md` (2026-10-28) says no tagged
  release contains 0056, so an upgrader from a tag meets the two together; a
  `master` build between them has already lost the rows. `oidc` link rows
  survived 0056.
- **Keep the pepper and the TOTP key, and do not rotate them.** They are the
  only way an old hash or sealed seed verifies.
- **There is no `staff remove`, `staff disable` or `staff reset-password`**
  (gap, 2026-10-30). Lost authenticator: delete the TOTP credential and the
  sessions. Lost password: delete the member and `staff add` again, which
  cascades to their console keys and changes their `stf_…` id. The SQL is in
  the runbook § "Staff who lost a password or an authenticator".

## Tests

`backends/tests/integration/tests/staff_sign_in.rs` (real router, real
socket, Postgres; it binds `127.0.0.2`–`.4`, `.20` and `.30`+ again, see
`vpay-troubleshooting`), `staff_sso.rs`, `staff_sso_keycloak.rs` (Docker, a
Keycloak image, no `#[ignore]`), `backends/crates/vpay-api/src/human_login_tests.rs`,
and in `backends/apps/vpay-server/tests/cli.rs`
`staff_add_mints_a_one_time_password_only_where_human_login_is_served`,
`a_management_pod_serving_human_login_mounts_its_jwks_and_session_validator`
and `a_management_pod_offers_staff_sso_exactly_when_the_block_is_written`.
Verification pages: `docs/status/verification/2026-10-29-vpa3-staff-sign-in.md`,
`2026-10-30-vpa4-staff-sso.md`, `2026-10-30-vpa5a-dashboard-sign-in.md`,
`2026-10-30-vpa5b-dashboard-sso.md`, `2026-10-30-vpa5c-dashboard-e2e.md`.
Nothing here has run in a cluster.
