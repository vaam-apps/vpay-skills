# Staff single sign-on (ADR-0030)

_Written against the vpay change that carries ADR-0030 (branch
`feat/staff-oidc-sso`; **this page is true only of a vpay that has merged it** —
on `a33aac61` or `69d822ab` none of it exists). Read from the code and the flow
page, and the Keycloak behaviour below was measured by the change's own tests on
`quay.io/keycloak/keycloak:26.6.1`, 2026-10-19/20. Nothing here has run in a
cluster._

The decision is `docs/adr/0030-staff-sign-in-federates-to-an-oidc-provider.md`
and the process is `docs/flows/dashboard-auth/sso.md`. `SKILL.md` has the
password and TOTP steps; this page is what an agent touching
`vpay_api::staff::sso` must not rediscover.

## The shape, in six sentences

vpay stays the OpenID Provider for `/dash/v1`; **only the human step
federates**. `staff_auth.sso` (`vpay_config::StaffSso`) present = SSO on, absent
= off: there is no `enabled` key and you must not add one. Three unauthenticated
routes — `GET /dash/v1/staff/sso` (`{enabled}`), `POST /staff/sso/start`,
`POST /staff/sso/callback` — and two dashboard routes, `/api/auth/sso/start`
(POST) and `/api/auth/sso/callback` (GET). A callback ends in the same
`authenticated` `staff_sessions` row the TOTP step ends in, and the dashboard
then runs the unchanged code leg. The provider's id_token is verified, read and
discarded; its access token is never read, and `userinfo` is never called.
Password + TOTP is untouched and is the break-glass path.

## Things that look like bugs and are not

- **The pending flow is not a `staff_sessions` row.** It is a row in
  `staff_sso_attempts` (migration `0050`), because `staff_sessions.staff_id` is
  `NOT NULL` and a first sign-in has no staff member yet. It is **consumed by
  `DELETE`** — the row count is the single-use guard — and a wrong `state` burns
  the flow.
- **The callback returns a different token from the one it was called with.**
  `{"outcome":"signed_in","session":"<new>"}`. That is session-fixation defence;
  the dashboard replaces its cookie. Do not "simplify" it into promoting the
  pending token.
- **A `200` can mean "go round again".** `{"outcome":"reauthenticate", …}` is the
  one `prompt=login` retry, started when a token's **only** defect is a missing
  second factor. Everything else is a uniform `401`. The retry flow carries
  `is_retry` in the row; a retry that fails the same rule is refused.
- **An IdP outage is a `401`**, not a `503`, with `sso identity provider
unavailable` in the log. Deliberate.
- **`redirect_uri` in `staff_auth.sso` is the dashboard's callback**, not a vpay
  URL. Do not confuse it with `dashboard_client.redirect_uris`, which is the
  OAuth leg between the dashboard and vpay-as-OP and still has no route.
- **The tests' identity provider is the in-process `wiremock` crate**, not the
  `wiremock/wiremock` container the rail suites use: it must serve tokens signed
  at run time with the flow's own nonce.

## The rules you can break by editing

| Rule                                                                                                                                                  | Where                                                              | What proves it                                                                                                                     |
| ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| Linked by `(iss, sub)` only, **never email**; the address is written once at provisioning                                                             | `sso/mod.rs` `provision`, `Credentials::find_federated`            | `a_second_sign_in_reuses_the_link_and_never_matches_on_email`, `an_email_that_belongs_to_an_unlinked_member_is_refused_not_linked` |
| No claim touches `is_admin` or `merchant_id`; JIT members are non-admin members of `dashboard_client.merchant_id`                                     | `sso/mod.rs` `provision`                                           | `roles_stay_local_whatever_the_token_claims`                                                                                       |
| A row for another issuer verifies nothing; discovery's `issuer` must equal the configured one byte for byte                                           | `find_federated(configured issuer, sub)`, `provider.rs` `discover` | `a_credentials_row_for_another_issuer_verifies_nothing`, `a_discovery_document_for_another_issuer_is_refused`                      |
| Order: state → code → verify → nonce → **group → disabled → second factor (+retry)** → link/provision. The retry is last so no other defect earns one | `sso/mod.rs` `callback`                                            | `no_other_failure_earns_a_retry` (tokens that also lack the factor)                                                                |
| Algorithm allow-list is asymmetric only; `kid` required; key family must match                                                                        | `sso/verify.rs` `ALLOWED_ALGORITHMS`                               | `alg_none_and_hs256_are_refused`, `an_es256_token_verifies_against_an_ec_jwk`                                                      |
| Provisioning is one transaction with the link                                                                                                         | `vpay_db::staff::provision_federated_in_tx` (`run_in_tx`)          | `provisioning_a_federated_member_is_atomic`                                                                                        |
| `staff_auth.sso` with both `accepted_acr` and `accepted_amr` empty is a boot refusal; so is a block without `dashboard_client` or the staff secrets   | `vpay_config::config::validate_staff_sso`                          | the `an_sso_block_with_no_second_factor_rule…` config cases                                                                        |

## What Keycloak 26.6.1 really issues (measured; other providers differ)

`acr` is a level, not a method: `"1"` after a password, `"0"` on an SSO-cookie
re-entry. **`amr` is empty unless the realm's executions carry
`default.reference.value`** (`pwd`, `otp`), a person whose OTP is enrolled
_during_ the sign-in (`CONFIGURE_TOTP`) gets `["pwd"]`, and **an SSO-cookie token
loses `amr` after about a second unless the executions carry
`default.reference.maxAge`**. So production is `accepted_amr: ["otp"]`,
`accepted_acr: []`, and the realm needs both settings; the fixture is
`backends/tests/integration/tests/fixtures/keycloak-realm.json`. The callback
logs `acr` and `amr` on every second-factor miss — read that line first.
`staff_sso_keycloak.rs` boots the real thing (one test, ~15 s).

## Not built — do not describe as built

Linking an existing password member to an SSO identity (no `staff link`);
back-channel or RP-initiated logout; a continuous group check (a removed person
keeps a session ≤12 h and a token ≤15 min; `staff_members.status` is the kill
switch); any provider but Keycloak; a cluster run; a Cypress case for SSO. R1
(single-factor credentials) is still reserved for `hotp`, `magic_link`,
`email_otp`, `phone_otp` and `webauthn`.
