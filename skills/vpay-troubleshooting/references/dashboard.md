# The dashboard: what looks wrong and is not

_Verified against vpay `f885170c` (2026-10-27), the merge of
vaam-apps/vpay#383, for every section; it was last verified at `d3a8810b`
(2026-09-16), and the sections that changed say what they said before.
Version-sensitive claims carry the date they became true — see
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Sources: `docs/runbooks/demo/dashboard-sign-in.md` (rewritten by #383) and
`docs/flows/dashboard-auth.md`. These are the things a new agent "fixes" and
should not. `vpay-dashboard` is the skill for the surface itself; its
`references/console-keys.md` has the full refusal table.

## Since 2026-10-27: nobody can sign in, on purpose

| What you see                                                                                          | Why it is correct                                                                                                                                                                                                                                                                                                |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Every dashboard URL shows "This console signs in with a paired key … not available until VP-6b"**   | The app is one notice page since #383. Its sign-in, session, BFF and pages are deleted. The console's own key pairing and request signing are VP-6b and **not built**. There is no form to fix and no route to restore. `src/layout.test.tsx` fails if a second page appears                                     |
| **`vpay-server` exits `78` with `ConfigError::RetiredKey` naming `dashboard_client` or `staff_auth`** | Both keys were deleted. Naming either is refused **even as a null** (`dashboard_client: ~`). Delete the key from the base file and every overlay. To serve the console add `dashboard: {merchant_id: …}` and a `service_clients:` entry; to run without one, add nothing                                         |
| **Every `/dash/v1` request answers `401` with an unsigned body, whatever you send**                   | A request must be a COSE_Sign1 message under a paired console key, a `GET` included. No signature, an unknown, revoked or expired key, a merchant's or a service's key, a bad signature, the wrong audience and a replay all get **the same bytes**, by design. A bearer token is not a credential here any more |
| **`/dash/v1`, `/dash/v1/$procs/…` and `/svc/v1` are all `404`**                                       | The configuration has no `dashboard:` block, so none of the three is mounted. That is how a deployment runs without a console                                                                                                                                                                                    |
| **`/svc/v1/console_keys` is `404` on the management tier**                                            | `/svc/v1` is on the **business** surface (ADR-0036 decision 11). A `surfaces: [management]` process does not serve it                                                                                                                                                                                            |
| **A `POST` to `/dash/v1/payment_intents` is `405` (or `415` if signed), not `403`**                   | Since #383 the signed transport layer refuses writes before the router. ~~`403` from the boundary~~ was the answer until 2026-10-27                                                                                                                                                                              |
| **`/dash/v1/$procs/retrievePaymentIntent` (or any merchant MCP procedure) is `404`**                  | `DASHBOARD_PROCEDURES` admits the five searches only (since vaam-apps/vpay#377, 2026-10-27). The generated router mounts every procedure, and the allowlist keeps the merchant ones off the staff surface                                                                                                        |
| **Pairing answers `403` for a person who is already an administrator**                                | Pairing never yields an administrator's key. Use `vpay-server staff pair-key`, the break-glass                                                                                                                                                                                                                   |
| **Pairing a key that was revoked or expired answers `409`**                                           | A thumbprint is unique for ever and the row is never deleted. Pair a **new** key                                                                                                                                                                                                                                 |
| **A key that worked yesterday is `401` today**                                                        | Keys live exactly 30 days from `paired_at`, and revocation takes effect on the next request. Nothing is cached                                                                                                                                                                                                   |

## Reading `/dash/v1` by hand in the demo

`just demo-staff` creates `ada@example.test` with **no credential** (since
2026-10-27) and writes the `stf_…` id to `.e2e/vpay-demo/staff-id.txt`. Attach
a key you hold with the break-glass, through the demo's compose file set:

```bash
vpay-server staff pair-key --staff "$(cat .e2e/vpay-demo/staff-id.txt)" \
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
ADR-0018's cross-tenant **read-only** admin role stands.

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
port is only a port. The container renders the notice.
