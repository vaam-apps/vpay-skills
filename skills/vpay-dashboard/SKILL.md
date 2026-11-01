---
name: vpay-dashboard
description: The staff operator console. /dash/v1 takes a COSE_Sign1 message under a paired, non-exportable P-256 console key (since 2026-10-27, ADR-0036) and, only where auth.methods serves human_login with staff_auth and dashboard_client written, a staff access token from a password + TOTP or OIDC SSO sign-in (restored 2026-10-30, ADR-0037, vpay#390/#391); vaam serves keys only. Covers /svc/v1 pairing for a service holding console_pairing, the `staff pair-key`/`revoke-key` break-glass, /dash/v1/staff/* and /oauth/*, the two REST reads and five `$procs` searches behind an allowlist, and frontends/apps/dashboard, which signs a person in and shows an account page where human_login is served, a paired-key notice elsewhere, and no page that reads /dash/v1 anywhere. Load before touching /dash/v1, /svc/v1, console_keys, staff sign-in, auth.methods human_login, a dashboard procedure or the dashboard app, and before trusting anything that says staff sign-in is deleted, or that it exists on every deployment.
---

# vpay dashboard

> **Verified against vpay `10f42747` (2026-10-30)** for the description, the
> banner, "Two halves", the invariant's note, the surface, "Things that look
> like bugs", "The app today", the demo, the flow-page table and every passage
> marked **Corrected 2026-10-30**; the rest was last read at `f885170c`
> (2026-10-27). Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

> **Read this first. Whether a staff member can sign in is deployment
> configuration, since 2026-10-30.** vaam-apps/vpay#383 (`f885170c`, 2026-10-27,
> ADR-0036, "VP-6a") deleted staff passwords, TOTP, sessions, the
> authorization code, the bearer token and OIDC SSO. Under owner ruling R5 and
> ADR-0037 (`auth.methods`), vpay#388, #390 and #391 (VPA-2, 3 and 4) restored
> them **behind the `human_login` method**: served only when it is permitted
> (absent `auth.methods` permits all four methods) **and** both `staff_auth` and
> `dashboard_client` are written. vpay#392–#394 (VPA-5a/b/c) gave the app its
> sign-in, an account page and an SSO button back. **vaam sets
> `auth.methods: [keypair, hmac_webhooks]`, so on vaam the console key is still
> the only way in.** Still not built anywhere: the console's own key sign-in
> (key generation, pairing, signing, the sealed reader: VP-6b), vaam's side of
> the pairing, and **any page that reads `/dash/v1`**. The sign-in is
> [references/human-login.md](references/human-login.md).
>
> ~~Since 2026-10-27 the dashboard application cannot sign anybody in. … The
> console's own sign-in … is VP-6b, and it is not built.~~ **Corrected
> 2026-10-30:** true of every vpay from `f885170c` up to, not including,
> `9099eccb` (vpay#392); from it on, the app signs a person in wherever
> `human_login` is served. On a vpay before `f885170c`, this skill's version
> stamped `73c3a7cf` (2026-10-21) describes your tree; the last section below
> is what it said.

Load `vpay-frontend` first for the workspace, the styling stack and
`verify-ui`. The flow docs are `docs/flows/dashboard-auth.md`, its
`dashboard-auth/` directory (~~deleted by #383~~ **back since 2026-10-30** with
`human-login.md` and `sso.md`), `docs/flows/dashboard.md` and the four pages
under `docs/flows/dashboard/`. Decisions: ADR-0008 (the dashboard observes, it
does not administer), ADR-0018 (which tenant a request reads), **ADR-0036**
(the paired key) and **ADR-0037** (`auth.methods`; decision 8 is
`human_login`). ~~ADR-0017 and ADR-0030 are superseded by ADR-0036 and describe
deleted code.~~ **Corrected 2026-10-30:** ADR-0030's status line says it was
restored (VPA-4) as part of `human_login`; ADR-0017's still says only
"superseded", but the sign-in it decided is served again under ADR-0037
decision 8. Read ADR-0037 before believing either status line.

## Two halves, and what each can do

| Half                                                     | State at `10f42747` (2026-10-30)                                                                                                                                                                                                                                                                                                      |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **vpay's `/dash/v1`**, its `$procs` transport, `/svc/v1` | **Built and proven** against a real Postgres (`dashboard_read_surface.rs`, `dashboard_procedure_transport.rs`, `console_pairing.rs`, `staff_sign_in.rs`, `staff_sso.rs`). Reachable by a client that signs with a paired key (vpay ships none; the only one is test code) and, where `human_login` is served, by a staff access token |
| **`frontends/apps/dashboard`** (`@vpay/dashboard`)       | **A sign-in and an account page** where vpay serves `human_login` (`/login`, `/login/totp`, `/login/password`, `/`); "This console signs in with a paired key" (`PairedKeyOnly`) on `/login` where it does not. **No page reads `/dash/v1`.** It cannot sign with a key (VP-6b)                                                       |

~~**`frontends/apps/dashboard`**: one notice page. `/` and the `not-found`
boundary both render `ConsoleUnavailable` … It calls vpay not at all and reads
no environment.~~ **Corrected 2026-10-30 (vpay#392):** `ConsoleUnavailable` is
deleted; `not-found` renders `NotFoundNotice`; the app reads six environment
variables and calls `/.well-known/vpay-auth`, `/dash/v1/staff/*` and
`/dash/v1/oauth/*`. `docs/status.md`'s frontend row says the same. The app's
`README.md` (rewritten again by #392–#394) is accurate: read it.

## The invariant

> **No `/dash/v1` request is handled unless it is a COSE_Sign1 message signed
> by a console key that is registered to an active member of the dashboard's
> merchant, was paired less than 30 days ago, has not been revoked, is
> addressed to this deployment, and has not been seen before.** _(verbatim,
> `docs/flows/dashboard-auth.md`)_

**Corrected 2026-10-30:** that sentence is exact only where `human_login` is
not served (vaam, and any deployment without both blocks). Where it is served,
`/dash/v1` runs the envelope `Optional`: the sentence holds for every signed
request, and an unsigned one must carry a staff access token instead
(`docs/flows/dashboard-auth.md`, the paragraph under the invariant; ADR-0037
decision 8). A signature **and** a bearer is a `400`.

The tenancy rule is unchanged by #383 or by VPA-3: a token goes through the
same `resolve_tenancy` a key does. A request reads exactly the tenant
`dashboard.merchant_id` names, fixed in YAML and checked at boot, unless the
key belongs to an **administrator**, who may name another registered merchant
(ADR-0018). For a non-administrator `?merchant_id=` is not even parsed.

Everything about the key is in
[references/console-keys.md](references/console-keys.md): the two pairing
doors, what a pairing never does, revocation on the next request, the 30-day
`CHECK`, the refusal table, the config and the upgrade cutover. **Read it
before touching `vpay_api::console_keys`, `vpay_api::envelope::console`,
`vpay_api::svc` or the `dashboard:`/`service_clients:` config.**

## The surface (as of 2026-10-30)

~~It is mounted only when the configuration has a `dashboard:` block.~~
**Corrected 2026-10-30 (vpay#390):** `/dash/v1` is mounted by a `dashboard:`
block **or** by `human_login` being served (tenant from
`dashboard_client.merchant_id`); `/svc/v1` by `dashboard:` alone. With neither,
all of it is the honest `404` and every `vpay-server` mode still boots.

| Route                                      | What it is                                                                                                                          |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `GET /dash/v1/payment_intents`             | Cursor-paged; `?status=`, `?created_gte=`, `?created_lte=`. `?customer=` is a `400` naming `customer` (since 2026-09-23)            |
| `GET /dash/v1/payment_intents/{id}`        | The intent, its charge, its refunds, its event timeline. Another tenant's id is byte for byte the `404` of an id that never existed |
| `POST /dash/v1/$procs/<search>`            | The five searches in `DASHBOARD_PROCEDURES`; any other procedure name is a `404` (since 2026-10-27, vaam-apps/vpay#377)             |
| `POST /svc/v1/console_keys`                | On the **business** surface, not management. A service key holding `console_pairing` pairs a key                                    |
| `DELETE /svc/v1/console_keys/{thumbprint}` | The same caller revokes one, an administrator's included                                                                            |
| `/dash/v1/staff/*`, `/dash/v1/oauth/*`     | **Only where `human_login` is served** (since 2026-10-30, vpay#390/#391): eleven routes, SSO's three among them; `404` elsewhere    |

On a keys-only deployment the body of every request is a COSE_Sign1 message,
**a `GET` included**, and every answer past the layer is sealed. Where
`human_login` is served, an unsigned request with a staff bearer is answered
in plain JSON ([references/human-login.md](references/human-login.md)). The shapes, the procedures and the
allowlist are in [references/read-seam.md](references/read-seam.md).

**Writes are refused by the layer, and the answer changed on 2026-10-27.**
~~Any method but `GET`/`HEAD` is a `403` from the boundary before the router
matches.~~ **Corrected 2026-10-27 (#383):** a plain `POST`/`PUT`/`PATCH`/`DELETE`
on a read route is the `EnvelopeLayer`'s own `405` with `Allow: GET`, and a
**signed** one is its `415`. Both come before the router, whatever key signed
it (`a_write_method_is_refused_by_the_layer_not_by_the_route_table`). The
property is the one ADR-0008 needs: no write can arrive here unlogged, because
no `audit_log` exists. ADR-0018's text still says `403`; ADRs are not edited.
Serving `human_login` does not change it: an unsigned write with a good token
is the same `405` (`a_session_may_not_use_a_method_the_surface_does_not_serve`,
re-read 2026-10-30). Writes are key-only and refused.

## Things that look like bugs and are not

~~**Every path of the app renders the same notice.** … `src/layout.test.tsx`
("serves `/` and nothing else") fails if a second `page.*` appears.~~
**Corrected 2026-10-30 (vpay#392):** the app has four pages, `/`, `/login`,
`/login/totp`, `/login/password`, and `src/layout.test.tsx` ("the app has the
pages the sign-in needs, and no others") pins exactly that list. **On a
keys-only deployment `/login` says "This console signs in with a paired key"
and draws no field.** That is the page working, not a broken sign-in: do not
"fix" it by writing `staff_auth` into vaam's configuration. **Still: do not
restore a read page on its own.** The account page links to nothing because
nothing reads `/dash/v1`; the page that does changes that test in the same
commit.

~~**A configuration that still says `dashboard_client:` or `staff_auth:` does
not boot, even as `dashboard_client: ~`.** It exits `78` with
`ConfigError::RetiredKey`.~~ **Corrected 2026-10-30 (vpay#390):** true only of
releases between VP-6a and VPA-3. The two keys are `human_login`'s blocks
again, and `RetiredKey` with its table is deleted. What refuses now is half a
sign-in: one block of two, or the method named without both, is
`AuthMethodNotConfigured`; a block with the method left out of an explicit
`auth.methods` (vaam's case) is `AuthMethodDisabled`. Do not soften either.

**Pairing refuses an administrator.** `POST /svc/v1/console_keys` for a person
who is already an admin is a `403`, by design: a service holding
`console_pairing` can name any linked person. An admin is keyed only by
`vpay-server staff pair-key`, and made only by `staff add --admin`.

~~**`rate_limit_windows` and `vpay_db::RateLimits` exist and nothing calls
them.**~~ **Corrected 2026-10-30 (vpay#390):** the sign-in limiter is back
(`vpay_api::staff::rate_limit`) and spends from `rate_limit_windows` where
`human_login` is served. `docs/flows/dashboard-auth.md` § Status still calls
them "unused scaffolding" at `10f42747`; the code wins. Rate limiting `/svc/v1`
is still listed as not built. Do not wire the sign-in limiter to it as if that
were the plan.

**`@vpay/dash-client`, `@cratestack/refine` and Refine are still dependencies
of an app that runs none of them.** ESLint refuses any value import of the
generated client ("no module runs it until VP-6b builds the signed reader").
Only its types may be imported.

**There is no `dashboard.cy.ts`.** It was **deleted, not skipped**, on
2026-10-27 (19 cases), because it drove the deleted sign-in.
`docs/status/frontend.md` § "What VP-6b must restore" lists every case: ten to
restore over the new sign-in, nine retired with the mechanism they tested.
~~As of that day no browser test of the dashboard exists.~~ **Corrected
2026-10-30 (vpay#394):** `dashboard-human-login.cy.ts` (seven tests) drives the
password sign-in against the compose stack. It reads no payment and drives no
SSO (the stack has no identity provider), so it restores none of the ten.

## The app today

The sign-in half is [references/human-login.md](references/human-login.md).
The presentational components and the pure logic under them survive with
their tests and Storybook stories, because a read page will render them again
(from the app's `README.md`):

| Area                  | Where                                                                                                                                        |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Payments table/detail | `src/components/payments-table.tsx`, `payment-detail.tsx`, `payments-filters.tsx`, `payments-pager.tsx`                                      |
| Procedure lists       | `src/components/procedure-list-view.tsx`, `procedure-table.tsx`, `procedure-pager.tsx`, over `src/dash/procedure-resources.ts`               |
| Query and paging      | `src/payments-query.ts` (cursors, filters, the URLs), `src/dash/procedure-paging.ts`                                                         |
| Words and colour      | `src/format.ts`, `src/payment-status.ts`, `src/status-systems.ts`, from `@vpay/tokens`                                                       |
| Failure shape         | `src/server/api.ts` (`ApiFailure`, the request id, the object types, and the sign-in calls again), `read-failure.tsx`, `src/dash/failure.ts` |
| ~~The notice~~        | ~~`console-unavailable.tsx`~~ deleted 2026-10-30; `PairedKeyOnly` in `src/components/login-notices.tsx`, on `/login` only                    |

`app/healthz/route.ts` survives. It takes no dependency and probes nothing.
Probing `/dash/v1` from it would take the dashboard pod's readiness down with
every `vpay-server` rollout. Do not "improve" it into a deep health check.

**Deleted on 2026-10-27, not hidden**: `app/login/**`, `app/api/**` (the SSO
routes and the BFF), `app/signed-out`, `app/(dash)/**` (every page, screen and
`loading.tsx`), `middleware.ts`, everything under `src/server` but `api.ts`,
the sign-in, TOTP, password, SSO, enrolment, signed-in-bar and app-shell
components, `src/dash`'s providers, readers, `resources.ts` and the
generated-client caller, `src/config`, and `qrcode`. The list is
`docs/status/frontend.md`'s. **Corrected 2026-10-30:** restored by #392/#393,
adapted: `app/login/**`, `app/signed-out`, `app/api/auth/sso/{start,callback}`,
`src/server`'s `cookies`, `csrf`, `pkce`, `oauth`, `gate`, `session`, `actions`,
`enrolment`, `signed-out`, `sso`, `src/config`, the sign-in, TOTP, password,
enrolment, SSO and signed-in-bar components, and `qrcode@1.5.4`. **Still
deleted**: `app/(dash)/**`, `app/api/dash/` (the BFF), `middleware.ts`, the
app shell, `src/dash`'s providers, readers and `resources.ts`, and the
generated-client caller. The honest-absence rules the surviving
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
  ten end-to-end cases. _(At `10f42747` vpay names the read pages both as
  VP-6b's and as "a later ticket" separate from it, e.g. `docs/status.md`'s
  frontend row against the app's README. Either way, none exists.)_ VP-6b also
  decides which credential wins when a browser holds both a session and a
  key. Whether the app regains a browser-reachable surface like the BFF is RD5
  in the Refine plan, still a maintainer's call; the e2e table marks those
  cases "restate if VP-6b adds one".

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

~~Since 2026-10-27, `just demo-staff` creates `ada@example.test` with **no
credential** and writes the `stf_…` id to `demo_staff_id_file`.~~ **Corrected
2026-10-30 (vpay#394, VPA-5c):** the generated demo overlay serves
`human_login`, so `just demo-staff` keeps the one-time password in
`.e2e/vpay-demo/staff-password.txt` and fails if `staff add` printed an id
instead (a stale overlay). Sign in on **`http://localhost:3000`**, not another
host name: the cookies are `__Host-` and `Secure`. There is no SSO button: no
provider. To read `/dash/v1` by hand you still need a key: read the member's
`stf_…` id from `staff_members`, run `vpay-server staff pair-key`, and use a
client that signs (`docs/runbooks/demo/dashboard-sign-in.md` has the lines).
`demo_dashboard_merchant` (default `shop-merchant-tenant`) picks the tenant
and must move on both `demo-up` and `demo-staff`; `demo_dashboard_port`
(default `3000`) moves the port, and the overlay's `redirect_uri` with it
(~~"no longer tied to a registered `redirect_uri`"~~, true 2026-10-27 to
2026-10-30).

## Flow pages, and where each is covered

| Page                                                 | Covered in                                                                                                                                                                                                               |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `docs/flows/dashboard-auth.md`                       | `references/console-keys.md`; the invariant, the surface and the VP-6b list above                                                                                                                                        |
| `docs/flows/dashboard-auth/human-login.md`           | `references/human-login.md` (since 2026-10-30)                                                                                                                                                                           |
| `docs/flows/dashboard-auth/sso.md`                   | `references/human-login.md` § "Single sign-on" (restored 2026-10-30)                                                                                                                                                     |
| `docs/flows/dashboard.md`                            | this page. Its § Status, unchanged at `10f42747`, still says a staff member "can sign in and read this merchant's payments": the first half is true again where `human_login` is served, the second is not true anywhere |
| `docs/flows/dashboard/slice-1-gaps.md`               | the refund and `customer` sections above                                                                                                                                                                                 |
| `docs/flows/dashboard/status-built-and-not-built.md` | `references/what-a-screen-may-show.md`                                                                                                                                                                                   |
| `docs/flows/dashboard/status-read-seam-and-bff.md`   | `references/read-seam.md`; its BFF evidence is history since 2026-10-27                                                                                                                                                  |
| `docs/flows/dashboard/status-styling-and-demo.md`    | the demo section above and `references/what-a-screen-may-show.md`; the styling stack is `vpay-frontend`'s                                                                                                                |

~~`docs/flows/dashboard-auth/rate-limiting.md`, `sessions-and-refusals.md`,
`scope-and-tokens.md`, `sso.md`~~: deleted by #383 (2026-10-27), with the
sign-in they described. **Corrected 2026-10-30:** `sso.md` is back (VPA-4);
the other three's matter is in `human-login.md`, and they stay deleted.

## More

- [references/human-login.md](references/human-login.md): `human_login`,
  where it is served and where not (vaam), the password, TOTP and session
  flow, one credential per request, SSO, the app's sign-in pages, cookies and
  variables, the demo overlay, the 0056/0058 upgrade and member recovery.
- [references/console-keys.md](references/console-keys.md): pairing (two
  doors, one function), what a pairing never does, revocation and expiry on the
  next request, one call and its refusals, `dashboard:`/`service_clients:` and
  the `RetiredKey` cutover (releases between VP-6a and VPA-3 only), the
  0056/0058 upgrade, where each piece lives, and what the deleted
  `staff-auth.md` and `sso.md` pages said, each marked with what is back.
- [references/read-seam.md](references/read-seam.md): the two transports, the
  five procedures and the `DASHBOARD_PROCEDURES` allowlist, the generated
  client's type-only rule, the two instant formatters, why there is no total,
  the tenant-scoped `404`, and the BFF and token lifecycle it used to describe.
- [references/what-a-screen-may-show.md](references/what-a-screen-may-show.md):
  the honest-absence rules, status colour, the theme, which `@vaam-apps/ui`
  components the surviving code imports, the filters and skeletons, the a11y
  gate, and the deleted shell.

## What this page said before 2026-10-27

Each of these was true at `73c3a7cf` and is false at `f885170c`. **Corrected
2026-10-30:** the ones marked _back_ are true again on a vpay from `9099eccb`
on, **where `human_login` is served**, and still false on vaam:

- ~~The app **is the OAuth client**: it holds the staff session in an
  httpOnly cookie, presents it as `X-Vpay-Staff-Session`, and runs the
  authorization-code leg server-side.~~ No session, cookie, code or token
  exists. _Back_ (#392), with the cookie renamed `__Host-vpay_dash_session`.
- ~~`/login`, `/login/totp`, `/login/password`, `/overview`, `/payments`,
  `/payments/{id}`, `/refunds`, `/deliveries`, `/customers`, `/checkouts`,
  `/signed-out`.~~ Every one is deleted. Only `/healthz` and the notice remain.
  _The three `/login` pages and `/signed-out` are back_; `/` is an account
  page; every read page is still deleted.
- ~~There is no route at `redirect_uri`, and nothing is missing.~~ There is no
  `redirect_uri`: `dashboard_client` is a retired key. _Back_: the
  `redirect_uri` is registered and never visited; SSO's callback is
  `/api/auth/sso/callback`, a different URI.
- ~~Two screens are wired to the BFF; do not delete `src/server/bff.ts`.~~
  The BFF is deleted.
- ~~Four environment variables (`VPAY_DASH_API`, `VPAY_DASHBOARD_CLIENT_ID`,
  `VPAY_DASHBOARD_REDIRECT_URI`, `VPAY_DASHBOARD_SCOPE`) and the optional
  `VPAY_DASHBOARD_PUBLIC_ORIGIN` configure the app and fail closed.~~ The app
  reads no environment. _Back_, plus `VPAY_DASHBOARD_SSO_ISSUER`.
- ~~Navigation is a gate (`DASH_RESOURCES`), the shell is mounted once in
  `app/(dash)/layout.tsx`, and navigation does not reload the document.~~ The
  shell, the rail and the resource registry are deleted.
- ~~The Overview shows twelve exact counts read from the search
  procedures.~~ Deleted; its rules are kept in `what-a-screen-may-show.md`.
- ~~`/login/password` is a forced step; `staff add` writes a one-time
  password.~~ There is no password. `staff add` prints an id. _Back_ where
  served; elsewhere it still prints the id.
- ~~The `$procs` transport mounts five procedures and a sixth is routed
  automatically.~~ Since #377 (2026-10-27) the router mounts every procedure,
  merchant MCP ones included, and `DASHBOARD_PROCEDURES` admits five.
