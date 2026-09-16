# The dashboard: what looks wrong and is not

_Verified against vpay `f063ee96` (2026-09-15). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Source: `docs/runbooks/demo/dashboard-sign-in.md`. These are the things a new
agent "fixes" and should not.

| What you see                                             | Why it is correct                                                                                                                                                                                                                                            |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **The payer column is an em dash**                       | `charges.payer_ref_masked` is never written by anything. The detail page renders the column as null rather than deriving a mask from the payer's unmasked number. **The day the column is written, the value appears.** Do not derive a mask at render time. |
| **The list has a "Methods" column and no "Rail" column** | `GET /dash/v1/payment_intents` returns no charge, so the only rail-shaped value there is the set of rails the intent _may_ be confirmed against. The rail that actually took it is on the detail page.                                                       |
| **There is no page count**                               | The API serves cursors and a `has_more`, and no count anywhere.                                                                                                                                                                                              |

## Two things it cannot do, by design

**Anything at all to a payment.** `/dash/v1` refuses every non-`GET` method **at
the boundary, before the router matches**. There is no re-poll, no replay, no
refund and no annotation — and therefore no `audit_log`, because there is
nothing yet to audit. ADR-0008 wants one row per dashboard write; that write
surface does not exist.

**Any other slice.** Webhooks, checkout sessions, balances, settings and rail
health are not built, and **the navigation does not link to them** —
`frontends/apps/dashboard/src/layout.test.tsx` fails if it ever does. So adding
a nav entry for an unbuilt page is a test failure, deliberately.

## The demo's dashboard is down on purpose

In `just demo`, the dashboard was for a long time the one service of the file
set that stays down, and `README.md` calls that "a statement rather than an
optimisation". exp28 (2026-09-07) added `dashboard` to `demo_services` so
`just demo` starts it, and issue #78 (2026-09-10) made its host port
`demo_dashboard_port` so a second stack can publish it elsewhere.

Before #78 it was the one service that could not be moved: `compose.e2e.yml`
published it on a literal `3000` and the OP had `http://localhost:3000/…`
registered as the dashboard client's `redirect_uri`, **matched byte for byte**.
Two stacks could not both serve a dashboard, and the second `demo-up` died on
the port bind. A stack B started today wants `demo_dashboard_port=13000`
alongside its other overrides.

## Signing in

`/dash/v1`'s read routes and staff sign-in are real and proven over HTTP against
a real Postgres (`backends/tests/integration/tests/dashboard_read_surface.rs`,
`…/staff_sign_in.rs`). Create an account with the third binary mode:

```bash
vpay-server staff add --config … --database-url … --merchant … --email … --name …
```

Staff authentication is ADR-0017: argon2id with a deployment pepper, RFC 6238
TOTP with a sealed secret, a strictly-increasing replay guard, mandatory
enrolment and a one-time password. ADR-0019 moved _where the credential material
lives_ without changing any of that. ADR-0018 adds a cross-tenant **read-only**
admin role.
