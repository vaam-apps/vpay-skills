# Staff authentication: the limiter, the one answer, the sessions

_Verified against vpay `a33aac61` (2026-09-29), by reading the three pages under
`docs/flows/dashboard-auth/` and checking the identifiers below against the
code at that commit. Version-sensitive claims carry the date they became true —
see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

The overview is `docs/flows/dashboard-auth.md` and the decision is ADR-0017.
`SKILL.md` has the sign-in steps, the forced password change and the BFF; this
page holds what the three detail pages add, so an agent that touches
`vpay_api::staff` does not have to rediscover it. **Nothing here was re-run for
this page**: the numbers are the flow pages' own and the identifiers are checked
to exist, not to behave as described.

## Which page is which

| Page under `docs/flows/dashboard-auth/` | What it holds                                                                                                                | Where it is covered                                                 |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `rate-limiting.md`                      | The rate-limit budget, and which address it counts                                                                           | § The limiter, below                                                |
| `sessions-and-refusals.md`              | Ending other sessions on a password change; a mistyped code; every refusal one `401`                                         | §§ Every refusal is one answer, Sessions, below                     |
| `scope-and-tokens.md`                   | One OAuth2 scope; token lifetimes; replacing a token; JWKS; a "where each piece lives" table (not reproduced here — read it) | § Scope and tokens, below, and `read-seam-and-bff.md` (the re-mint) |

## The limiter

- **The counters are rows in Postgres** (`rate_limit_windows`, migration
  `0038`), shared by every replica, since 2026-09-10 (issue #79 item 2). They
  were per-process memory before, so three replicas admitted three times the
  attempts. `two_replicas_share_one_sign_in_budget` is the proof: two servers
  over one database, six wrong passwords against a budget of five, `429` on the
  sixth.
- **The numbers are configuration**: `staff_auth.rate_limits.sign_in` and
  `.change_password`, each `{ attempts, window_seconds }`, defaulting to
  **10 / 300** and **5 / 300** (read from `vpay-config` at `a33aac61`).
  Whether the shared default should be lower is a maintainer decision and has
  not been taken; the exp36 review kept ten.
- **Why a durable counter is not a denial-of-service amplifier**, because the
  next agent will ask: the row key is a SHA-256 of `"<action>:<dimension>:<value>"`
  (the caller chooses the value, and a column holding it verbatim is a copy of
  an email address written down by the act of guessing it); each statement
  deletes up to 32 elapsed rows `FOR UPDATE SKIP LOCKED` (two concurrent
  sign-ins cannot deadlock on the sweep); and it is one
  `INSERT … ON CONFLICT … DO UPDATE … RETURNING attempts`, so two replicas
  serialise on the row lock instead of both reading nine and both admitting.
- **It fails closed**: a database failure is a refusal, never an allowance.
- **The window is fixed, not sliding.** An attacker who straddles a boundary
  gets twice the limit in one instant; a sliding window needs a row per attempt,
  the unbounded table the digest key and the sweep exist to avoid.
- **Sign-in is limited per email and per address, and the check runs before any
  credential work**, because an attempt over budget must not cost an argon2id
  verification. Both counters move on every attempt, including a refused one.
- **The second factor spends from the same sign-in budget**, and counts on the
  _failure_ path (there is no argon2id to protect, and a correct code that
  spent a unit would halve how many people can sign in). It did not count at all
  until 2026-09-07: thirty consecutive wrong codes were thirty `401`s.
- **The password change has its own budget**, keyed by the session
  (`change_password:session`, 5 per 5 minutes by default), deliberately not the
  sign-in budget: a thief with a stolen cookie must not be able to lock the owner
  out of their own login by guessing there.

### Which address it counts

**The transport peer, unless the peer is a machine the operator named.**
`staff_auth.trusted_proxies` (addresses and CIDRs) is **empty by default**, and
empty means the peer. When the peer is listed, the client is the **first
untrusted hop of `X-Forwarded-For` walking from the right**. Never the leftmost
entry: that is whatever the caller wrote, a new one per request, and it hands
every caller a fresh bucket while the limiter keeps reporting that it limits.
The walk ends at the peer if the peer is not listed, a hop does not parse, every
hop is ours, there is no header, or a field line is unreadable. Repeated
`X-Forwarded-For` lines are **one chain** (RFC 9110 § 5.2, the nearest hop is the
last hop of the last line). `Forwarded` (RFC 7239) is deliberately not read. An
unparseable `trusted_proxies` entry takes the **login** down (the read surface
mounts with no staff login) rather than narrowing the list silently.

With the list empty and a reverse proxy in front, **every staff member shares one
per-address budget**, because the peer is the proxy; the per-email budget still
binds per account.

Four `staff_sign_in.rs` cases need distinct client addresses and use
`127.0.0.2`–`127.0.0.35`; on macOS that is `just loopback-aliases`, see
`vpay-troubleshooting`.

## Every refusal is one answer

No such address, wrong password, disabled account, wrong code, replayed code,
expired session, idle session, forged session, session at the wrong stage: one
`401`, one sentence, `error.code = "authentication_error"`. The step that
refused reaches the log and never the body. The **timing** is equalised too:
`StaffCredentials::verify_absent_account` costs an address with no account one
argon2id verification, so "no such account" and "wrong password" cannot be told
apart by either shape or time.

Two consequences an agent trips on, both from the exp36 review (2026-09-10):

- **A `401` from a credential endpoint ends nothing.** `POST /staff/password`
  and `POST /staff/totp` answer the same `401` for "that is not your password /
  code" as for "your session is over", so the caller cannot tell them apart and
  **must not clear the session cookie on either**. Whether the session is over is
  the next render's question. The dashboard once cleared it on any `401` and
  bounced a mistyped code back to the password form.
- **That needed an eighth staff route**, `GET /dash/v1/staff/session/stage`:
  `{"stage": "pending_totp" | "authenticated"}` for a live session whose account
  is active, `401` for everything else, **and nothing about the person** (a
  caller there has presented a password and no second factor). It does not touch
  `last_seen_at`. What it admits: a holder of a stolen `pending_totp` session can
  keep guessing codes at one screen, bounded by the shared sign-in budget.

## Sessions

- **Changing a password takes the password in force and ends every other
  session** of that staff member (since 2026-09-10, issue #79 item 3). The
  caller's own session survives on purpose; any authorization code the deleted
  sessions had in flight goes with them (the cascade on
  `oauth_authorization_codes.session_id`). Without the current password, the
  session cookie alone could take the account for up to twelve hours.
  `SKILL.md` has the forced first-sign-in change.
- A session lives **30 minutes idle, 12 hours absolute**.

## Scope and tokens

- **One OAuth2 scope**, because the dashboard is read-only. No mutating
  dashboard use case is being built, so ADR-0008's write actions and its
  `audit_log` row per write are described and **not built**; a real mutating use
  case needs its own scope and write path.
- **Authorization code 60 s, single use** (compare-and-swap on
  `oauth_authorization_codes`). **Access token 900 s by default**
  (`staff_auth.access_token_ttl_seconds`, bounded 10–3600). **No refresh token
  and no ID token are issued**, and there is no revocation endpoint in
  `authkestra-op`. The "refresh" is the authorization-code leg run again on the
  live session, which re-reads the staff row and re-checks active status, the
  merchant binding and `password_change_required` on every mint. The mitigation
  for a stolen access token is the short TTL plus the session row: signing out
  deletes the row, so the _obtainability_ of the token is revoked even though the
  JWT stays valid until it expires.
- **The dashboard re-mints at 80 % of the TTL** (`REMINT_AFTER_FRACTION = 0.8`
  in the dashboard's `server/`), from `staff_sessions.access_token_expires_at`
  (migration `0040`), which `GET /dash/v1/staff/session` carries. See
  `read-seam-and-bff.md` for the `stale-token` arm and the reactive retry.
- **Discovery, `/userinfo` and the JWKS are not served on `/dash/v1`.** There is
  one issuer and one JWKS, on `/v1/oauth`; the dashboard's validator fetches
  `/v1/oauth/jwks.json`. ~~vpay publishes `/dash/v1/.well-known/openid-configuration`
  and `/dash/v1/jwks.json`~~ was wrong and was corrected by vpay on 2026-09-20.
- **Signing keys** are RS256 only. `oauth_signing_keys` holds the public JWK, at
  most one active key (partial unique index), and **no private key material** —
  the PEM is injected from a Secret at boot. The flow page's remark that no code
  rotates a row "yet" is **stale at `a33aac61`**: `SigningKeys` and
  `ensure_active_in_database` exist (ADR-0026 D8 names the 24-hour
  `ROTATION_OVERLAP` written from the API host's clock). Read the code, not the
  sentence.
