---
name: vpay-dashboard
description: The staff operator console. /dash/v1 accepts only COSE_Sign1 messages under a paired, non-exportable P-256 console key (since 2026-10-27, ADR-0036), with /svc/v1 where a service holding console_pairing (vaam) pairs and revokes keys, the `vpay-server staff pair-key`/`revoke-key` break-glass, the two REST reads and the five `$procs` searches behind an allowlist, and `dashboard:`/`service_clients:` config. Also frontends/apps/dashboard, which is one notice page that cannot sign in until VP-6b, and the surviving components VP-6b will render. Load before touching /dash/v1, /svc/v1, console_keys, a dashboard procedure or the dashboard app, before restoring any page, and before trusting anything that says a staff member signs in with a password, TOTP, SSO, a session or a bearer token.
---

# vpay dashboard

> **Verified against vpay `f885170c` (2026-10-27).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

> **Read this first. Since 2026-10-27 the dashboard application cannot sign
> anybody in. That is the intended state of the tree, not a regression.**
> vaam-apps/vpay#383 (merged as `f885170c`; ADR-0036 is dated 2026-10-26, and
> vpay calls the work VP-6a) deleted staff passwords, TOTP, sessions, the
> authorization code, the bearer token and OIDC SSO. With them went every
> page, the BFF, the middleware and the shell of `frontends/apps/dashboard`.
> **The console's own sign-in (key generation in the browser, the pairing
> request, signing every request, the sealed reader) is VP-6b, and it is not
> built.** Neither is vaam's side of the pairing. On a vpay before
> `f885170c`, this skill's previous version (stamped `73c3a7cf`, 2026-10-21)
> describes your tree. What it said is the last section below.

Load `vpay-frontend` first for the workspace, the styling stack and
`verify-ui`. The flow docs are `docs/flows/dashboard-auth.md` (rewritten by
#383 as one page; its `dashboard-auth/` directory is deleted),
`docs/flows/dashboard.md` and the four pages under `docs/flows/dashboard/`.
Decisions: ADR-0008 (the dashboard observes, it does not administer), ADR-0018
(which tenant a request reads) and **ADR-0036** (the paired key). ADR-0017 and
ADR-0030 are superseded by ADR-0036 and describe deleted code.

## Two halves, and only one of them works

| Half                                                     | State at `f885170c`                                                                                                                                                                                                                       |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **vpay's `/dash/v1`**, its `$procs` transport, `/svc/v1` | **Built and proven** against a real Postgres (`dashboard_read_surface.rs`, `dashboard_procedure_transport.rs`, `console_pairing.rs`). Reachable only by a client that signs with a paired key. vpay ships none; the only one is test code |
| **`frontends/apps/dashboard`** (`@vpay/dashboard`)       | **One notice page.** `/` and the `not-found` boundary both render `ConsoleUnavailable` ("This console signs in with a paired key", "not available until VP-6b"). It calls vpay not at all and reads no environment                        |

`docs/status.md`'s frontend row carries a ⛔ for exactly this. The app's own
`README.md` (rewritten by #383, 85 lines) is accurate and short: read it.

## The invariant

> **No `/dash/v1` request is handled unless it is a COSE_Sign1 message signed
> by a console key that is registered to an active member of the dashboard's
> merchant, was paired less than 30 days ago, has not been revoked, is
> addressed to this deployment, and has not been seen before.** _(verbatim,
> `docs/flows/dashboard-auth.md`)_

The tenancy rule is unchanged by #383. A request reads exactly the tenant
`dashboard.merchant_id` names, fixed in YAML and checked at boot, unless the
key belongs to an **administrator**, who may name another registered merchant
(ADR-0018). For a non-administrator `?merchant_id=` is not even parsed.

Everything about the key is in
[references/console-keys.md](references/console-keys.md): the two pairing
doors, what a pairing never does, revocation on the next request, the 30-day
`CHECK`, the refusal table, the config and the upgrade cutover. **Read it
before touching `vpay_api::console_keys`, `vpay_api::envelope::console`,
`vpay_api::svc` or the `dashboard:`/`service_clients:` config.**

## The surface (as of 2026-10-27)

It is mounted only when the configuration has a `dashboard:` block. Without
one, all of it is the honest `404` and every `vpay-server` mode still boots.

| Route                                      | What it is                                                                                                                          |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `GET /dash/v1/payment_intents`             | Cursor-paged; `?status=`, `?created_gte=`, `?created_lte=`. `?customer=` is a `400` naming `customer` (since 2026-09-23)            |
| `GET /dash/v1/payment_intents/{id}`        | The intent, its charge, its refunds, its event timeline. Another tenant's id is byte for byte the `404` of an id that never existed |
| `POST /dash/v1/$procs/<search>`            | The five searches in `DASHBOARD_PROCEDURES`; any other procedure name is a `404` (since 2026-10-27, vaam-apps/vpay#377)             |
| `POST /svc/v1/console_keys`                | On the **business** surface, not management. A service key holding `console_pairing` pairs a key                                    |
| `DELETE /svc/v1/console_keys/{thumbprint}` | The same caller revokes one, an administrator's included                                                                            |

The body of every request is a COSE_Sign1 message, **a `GET` included**, and
every answer past the layer is sealed. The shapes, the procedures and the
allowlist are in [references/read-seam.md](references/read-seam.md).

**Writes are refused by the layer, and the answer changed on 2026-10-27.**
~~Any method but `GET`/`HEAD` is a `403` from the boundary before the router
matches.~~ **Corrected 2026-10-27 (#383):** a plain `POST`/`PUT`/`PATCH`/`DELETE`
on a read route is the `EnvelopeLayer`'s own `405` with `Allow: GET`, and a
**signed** one is its `415`. Both come before the router, whatever key signed
it (`a_write_method_is_refused_by_the_layer_not_by_the_route_table`). The
property is the one ADR-0008 needs: no write can arrive here unlogged, because
no `audit_log` exists. ADR-0018's text still says `403`; ADRs are not edited.

## Things that look like bugs and are not

**Every path of the app renders the same notice.** That is fail-closed, not a
routing bug: there is nothing it could ask vpay for. `src/layout.test.tsx`
("serves `/` and nothing else") fails if a second `page.*` appears under
`app/`. **Do not restore a page on its own.** A page needs the signed reader
VP-6b builds, and a page nothing can feed is what #383 deleted. VP-6b's first
real page changes that test in the same commit.

**A configuration that still says `dashboard_client:` or `staff_auth:` does
not boot, even as `dashboard_client: ~`.** It exits `78` with
`ConfigError::RetiredKey`, whose message says what to add instead. That is
the cutover. Do not soften it into a warning (console-keys.md, "Upgrading").

**Pairing refuses an administrator.** `POST /svc/v1/console_keys` for a person
who is already an admin is a `403`, by design: a service holding
`console_pairing` can name any linked person. An admin is keyed only by
`vpay-server staff pair-key`, and made only by `staff add --admin`.

**`rate_limit_windows` and `vpay_db::RateLimits` exist and nothing calls
them.** They were the sign-in limiter's. Rate limiting `/svc/v1` is listed as
not built. Do not wire the old limiter to it as if that were the plan.

**`@vpay/dash-client`, `@cratestack/refine` and Refine are still dependencies
of an app that runs none of them.** ESLint refuses any value import of the
generated client ("no module runs it until VP-6b builds the signed reader").
Only its types may be imported.

**There is no `dashboard.cy.ts`.** It was **deleted, not skipped**, on
2026-10-27 (19 cases), because it drove the deleted sign-in. As of that day no
browser test of the dashboard exists. `docs/status/frontend.md` § "What VP-6b
must restore" lists every case: ten to restore over the new sign-in, nine
retired with the mechanism they tested.

## The app today

The presentational components and the pure logic under them survive with
their tests and Storybook stories, because VP-6b will render them again (from
the app's `README.md`):

| Area                  | Where                                                                                                                          |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Payments table/detail | `src/components/payments-table.tsx`, `payment-detail.tsx`, `payments-filters.tsx`, `payments-pager.tsx`                        |
| Procedure lists       | `src/components/procedure-list-view.tsx`, `procedure-table.tsx`, `procedure-pager.tsx`, over `src/dash/procedure-resources.ts` |
| Query and paging      | `src/payments-query.ts` (cursors, filters, the URLs), `src/dash/procedure-paging.ts`                                           |
| Words and colour      | `src/format.ts`, `src/payment-status.ts`, `src/status-systems.ts`, from `@vpay/tokens`                                         |
| Failure shape         | `src/server/api.ts` (`ApiFailure`, the request id, the object types), `read-failure.tsx`, `src/dash/failure.ts`                |
| The notice            | `src/components/console-unavailable.tsx`, rendered by `app/page.tsx` and `app/not-found.tsx`                                   |

`app/healthz/route.ts` survives. It takes no dependency and probes nothing.
Probing `/dash/v1` from it would take the dashboard pod's readiness down with
every `vpay-server` rollout. Do not "improve" it into a deep health check.

**Deleted on 2026-10-27, not hidden**: `app/login/**`, `app/api/**` (the SSO
routes and the BFF), `app/signed-out`, `app/(dash)/**` (every page, screen and
`loading.tsx`), `middleware.ts`, everything under `src/server` but `api.ts`,
the sign-in, TOTP, password, SSO, enrolment, signed-in-bar and app-shell
components, `src/dash`'s providers, readers, `resources.ts` and the
generated-client caller, `src/config`, and `qrcode`. The list is
`docs/status/frontend.md`'s. The honest-absence rules the surviving
components obey (the Methods column, the masked payer, status colour,
skeleton geometry, the a11y gate) are in
[references/what-a-screen-may-show.md](references/what-a-screen-may-show.md).
**Read it before touching a component or restoring a screen.**

## What VP-6b has to build (none of this exists)

This is stated as work, not as a description. Do not write any of it in the
present tense in a doc, a status page or a skill until it has merged:

- **In the browser**: generate a non-extractable P-256 key, sign the pairing
  request (`"vpay.console-pair.v1\n" + publicKey + "\n" + issuedAt`), sign
  every `/dash/v1` request, open sealed answers, and show a re-pair state for
  a revoked or expired key rather than an error page.
- **In vaam**: a registered `service_clients` key, the call to
  `/svc/v1/console_keys`, and the pairing screen on the admin phone. That is
  vaam's own work.
- **In this app**: the pages again, over the surviving components, and the
  ten end-to-end cases. Whether the app regains a browser-reachable surface
  like the BFF is RD5 in the Refine plan, still a maintainer's call; the e2e
  table marks those cases "restate if VP-6b adds one".

As of 2026-10-27 no document read for this page plans listing a person's
keys, rate limiting `/svc/v1`, or any dashboard write.

## Still true of the reads

**A refund's events do not reach the payment detail's timeline** (read from
the code on 2026-09-23 and again on 2026-10-27; no test covers it). Refund
events carry the refund's own `re_…` id as `object_id`
(`vpay_api::v1::refunds::refund_event`), and the detail read asks
`Events::list_for_objects` only for the intent's and the charge's ids
(`vpay_api::dash::payment_intents`). The integration test that seems to prove
otherwise seeds its `charge.refunded` row by hand with the charge's id
(`docs/flows/dashboard/slice-1-gaps.md`).

**Only `customer` is refused on the list** (`refuse_customer`, since
vaam-apps/vpay step A, vpay#251). `ListParams` has no `deny_unknown_fields`, so
any other unknown key is silently ignored. A customer filter here would be a
new capability with its own test, not the removal of the refusal.

**Every refund the procedures return reads `pending`** (since 2026-09-16,
`vpay_db::Refunds::create`): nothing settles a refund, and **no rail has ever
returned money to anyone.** A row is an instruction recorded, never a payout.
No out-of-band invoice payment is visible here either; only `/v1` writes one.

**Offset paging is not stable while rows are being created** (exp54 review,
2026-09-11). The procedures count `OFFSET` afresh per call, so an insert
between two pages repeats a row at the top of the next page and drops one at
the tail. The cursor list (`starting_after` names a row) does not have the
property. Nothing in vpay compensates.

## The demo stack

Since 2026-10-27, `just demo-staff` creates `ada@example.test` with **no
credential** and writes the `stf_…` id to `demo_staff_id_file`. To read
`/dash/v1` by hand, run `vpay-server staff pair-key` with a key you hold and
use a client that signs (`docs/runbooks/demo/dashboard-sign-in.md`). The
dashboard container still starts and renders the notice.
`demo_dashboard_merchant` (default `shop-merchant-tenant`) still picks the
tenant, and `demo_dashboard_port` (default `3000`) still moves the port. The
port is no longer tied to a registered `redirect_uri`, because there is none.

## Flow pages, and where each is covered

| Page                                                 | Covered in                                                                                                                 |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `docs/flows/dashboard-auth.md`                       | `references/console-keys.md`; the invariant, the surface and the VP-6b list above                                          |
| `docs/flows/dashboard.md`                            | this page. Its § Status still says "A staff member can sign in" at `f885170c`; its own header says such text is the record |
| `docs/flows/dashboard/slice-1-gaps.md`               | the refund and `customer` sections above                                                                                   |
| `docs/flows/dashboard/status-built-and-not-built.md` | `references/what-a-screen-may-show.md`                                                                                     |
| `docs/flows/dashboard/status-read-seam-and-bff.md`   | `references/read-seam.md`; its BFF evidence is history since 2026-10-27                                                    |
| `docs/flows/dashboard/status-styling-and-demo.md`    | the demo section above and `references/what-a-screen-may-show.md`; the styling stack is `vpay-frontend`'s                  |

~~`docs/flows/dashboard-auth/rate-limiting.md`, `sessions-and-refusals.md`,
`scope-and-tokens.md`, `sso.md`~~: deleted by #383 (2026-10-27), with the
sign-in they described.

## More

- [references/console-keys.md](references/console-keys.md): pairing (two
  doors, one function), what a pairing never does, revocation and expiry on the
  next request, one call and its refusals, `dashboard:`/`service_clients:` and
  the `RetiredKey` cutover, where each piece lives, and what the deleted
  `staff-auth.md` and `sso.md` pages said.
- [references/read-seam.md](references/read-seam.md): the two transports, the
  five procedures and the `DASHBOARD_PROCEDURES` allowlist, the generated
  client's type-only rule, the two instant formatters, why there is no total,
  the tenant-scoped `404`, and the BFF and token lifecycle it used to describe.
- [references/what-a-screen-may-show.md](references/what-a-screen-may-show.md):
  the honest-absence rules, status colour, the theme, which `@vaam-apps/ui`
  components the surviving code imports, the filters and skeletons, the a11y
  gate, and the deleted shell.

## What this page said before 2026-10-27

Each of these was true at `73c3a7cf` and is false at `f885170c`:

- ~~The app **is the OAuth client**: it holds the staff session in an
  httpOnly cookie, presents it as `X-Vpay-Staff-Session`, and runs the
  authorization-code leg server-side.~~ No session, cookie, code or token
  exists.
- ~~`/login`, `/login/totp`, `/login/password`, `/overview`, `/payments`,
  `/payments/{id}`, `/refunds`, `/deliveries`, `/customers`, `/checkouts`,
  `/signed-out`.~~ Every one is deleted. Only `/healthz` and the notice remain.
- ~~There is no route at `redirect_uri`, and nothing is missing.~~ There is no
  `redirect_uri`: `dashboard_client` is a retired key.
- ~~Two screens are wired to the BFF; do not delete `src/server/bff.ts`.~~
  The BFF is deleted.
- ~~Four environment variables (`VPAY_DASH_API`, `VPAY_DASHBOARD_CLIENT_ID`,
  `VPAY_DASHBOARD_REDIRECT_URI`, `VPAY_DASHBOARD_SCOPE`) and the optional
  `VPAY_DASHBOARD_PUBLIC_ORIGIN` configure the app and fail closed.~~ The app
  reads no environment.
- ~~Navigation is a gate (`DASH_RESOURCES`), the shell is mounted once in
  `app/(dash)/layout.tsx`, and navigation does not reload the document.~~ The
  shell, the rail and the resource registry are deleted.
- ~~The Overview shows twelve exact counts read from the search
  procedures.~~ Deleted; its rules are kept in `what-a-screen-may-show.md`.
- ~~`/login/password` is a forced step; `staff add` writes a one-time
  password.~~ There is no password. `staff add` prints an id.
- ~~The `$procs` transport mounts five procedures and a sixth is routed
  automatically.~~ Since #377 (2026-10-27) the router mounts every procedure,
  merchant MCP ones included, and `DASHBOARD_PROCEDURES` admits five.
