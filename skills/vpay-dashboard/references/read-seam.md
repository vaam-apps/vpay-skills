# The `/dash/v1` read seam: two transports, five procedures, and what the app lost

_Verified against vpay `f885170c` (2026-10-27), the merge of
vaam-apps/vpay#383, for every section. This page was
`read-seam-and-bff.md` until that day and covered the BFF, the token
lifecycle and the generated-client caller, all deleted by #383; what it said
about them is kept, struck, at the end. Version-sensitive claims carry the
date they became true — see
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

The credential on every read below is a COSE_Sign1 message under a paired
console key: [console-keys.md](console-keys.md). This page is the shape of
the reads, which #383 did not change ("the routes and the tenancy rule are
unchanged", `docs/status/backend.md`).

## Two transports, one seam

`/dash/v1` serves `payment_intents` as a **cursor-paged REST route** shared
with the merchant API (`DASH_ROUTES` in `vpay_api::dash`, two `GET` rows).
Everything else is a **CrateStack procedure** at
`POST /dash/v1/$procs/<name>`:

| Resource constant (`src/dash/resource-name.ts`) | Served by                                     |
| ----------------------------------------------- | --------------------------------------------- |
| `PAYMENT_INTENTS`                               | `GET /dash/v1/payment_intents` (REST, cursor) |
| `REFUNDS`                                       | `procedure searchRefunds`                     |
| `WEBHOOK_DELIVERIES`                            | `procedure searchWebhookDeliveries`           |
| `CUSTOMERS`                                     | `procedure searchCustomers`                   |
| `CHECKOUT_SESSIONS`                             | `procedure searchCheckoutSessions`            |

`searchPaymentIntents` is the fifth procedure. No screen calls it today,
because no screen calls anything (see `SKILL.md`). Until 2026-10-27 the
Overview called it six times per render.

`resource-name.ts` survived #383 with `PROCEDURE_RESOURCES` and
`isProcedureResource`. `payment_intents` is in neither, because it is not a
procedure. Do not add it. **Its comment is stale at `f885170c`**: it says the
resource-to-procedure mapping is `LIST_OF` in `src/dash/cratestack-client.ts`,
and #383 deleted that file. As of 2026-10-27 nothing maps a resource to a
procedure, because nothing calls one. That is VP-6b's to rebuild.

## The `$procs` transport: five served, more mounted, an allowlist between

**Changed 2026-10-27 (vaam-apps/vpay#377, ADR-0033 step 3).**
`dashboard_procedure_router` in `backends/crates/vpay-db/src/schema.rs` wraps
CrateStack's generated `procedure_router`, which mounts **every** procedure
in `schemas/vpay.cstack`. Since #377 that is twelve as of 2026-10-27: the five
dashboard searches plus the merchant MCP procedures (`retrievePaymentIntent`,
`retrieveRefund`, and five `mutation procedure`s such as
`openPaymentIntent` and `issueRefund`). CrateStack 0.16 has no attribute that
keeps a procedure off REST.

So `/dash/v1/$procs` is filtered by **`DASHBOARD_PROCEDURES`**, a
`[&str; 5]` in `backends/crates/vpay-api/src/lib.rs`, applied by
`only_dashboard_procedures`. Any other `$procs/<name>` is the honest `404`:

```http
POST /dash/v1/$procs/searchPaymentIntents
POST /dash/v1/$procs/searchRefunds
POST /dash/v1/$procs/searchWebhookDeliveries
POST /dash/v1/$procs/searchCustomers
POST /dash/v1/$procs/searchCheckoutSessions
```

`a_merchant_mcp_procedure_is_not_served_to_the_dashboard`
(`dashboard_procedure_transport.rs`) pins it. The code's comment names two
more guards: each MCP procedure's own `@allow` refuses a dashboard context,
which carries no `audience` claim, and the dashboard's registry has no port to
run them with.

~~The `$procs` transport mounts five procedures, and the router mounts every
procedure in the registry, so a sixth is routed automatically.~~ **Corrected
2026-10-27:** true until #377. **A sixth dashboard procedure now needs an
entry in `DASHBOARD_PROCEDURES` as well**, or it is a `404` on `/dash/v1`.
And an entry there for an MCP procedure is how a merchant write would reach
the staff surface. Review that list as a security boundary.

**Three places in the Rust still say "one procedure"** (re-read 2026-10-27):
the `vpay-db` test `no_generated_model_route_is_mounted_only_the_one_procedure_is`
(name and assertion message), the comment on it in `vpay-db/Cargo.toml`, and
the module name `schema/search_payment_intents.rs`, whose
`impl ProcedureRegistry` covers all the searches through sibling modules
(the trait lives inside the private macro expansion, so every body must be a
child of that module). Code wins. The test's model half covers all **twenty**
models as of 2026-10-27 (`MODEL_TABLES: [(&str, &str); 20]`, compared both ways
against `cratestack_schema::MODELS`; `ConsoleKey` is in it since #383, which
removed `StaffSession` and `OauthAuthorizationCode`). So a model you declare
fails that test until you add its pair. Its procedure half probes one path,
so add a probe if you add a procedure.

## `@vpay/dash-client`: generated, and only its types may be imported

`frontends/packages/dash-client` is still `cratestack generate-typescript`
output, written by `just gen-dash-client` and drift-checked by
`just check-schema`. **Never edit it.** Any edit to `schemas/vpay.cstack`, a
comment included, makes it stale, because its `src/runtime.ts` embeds the
schema's SHA-256. #383 regenerated it.

**Since 2026-10-27 no module runs it.** The dashboard's ESLint
`no-restricted-imports` rule (`allowTypeImports: true`) now refuses **every**
value import of `@vpay/dash-client`. The message reads "Only the generated
client's types may be imported: no module runs it until VP-6b builds the
signed reader." It used to exempt `src/dash/cratestack-client.ts`, which #383
deleted. When VP-6b builds the signed reader, that one module is the
exemption, and the reason is unchanged: the package has a class per
**model**, vpay mounts none of those routes, and `PaymentIntentApi.list()`
would reach the hand-written `GET /dash/v1/payment_intents` and cast its
envelope to an array with no error.

`@cratestack/refine` (`0.16.0`) and Refine are still dependencies of the app.
`src/dash/procedure-resources.ts` (the hand-written manifest that refuses any
undeclared filter key, any sort, `getOne` and writes) and
`src/components/procedure-list-view.tsx` survive with their tests, but no page
renders them. `just gen-dash-client` still passes no `--refine`, because that
manifest binds models.

## Instants: two formatters, because there are two wire shapes

Still true, and `src/format.ts` survived: `formatInstant` takes **unix
seconds** (the REST list serialises an integer), `formatIsoInstant` an
**RFC 3339 string** (a procedure serialises a `DateTime`). Passing one to the
other yields `Invalid Date` or, worse, a plausible date in 1970. An
unparseable value is returned **verbatim**, so an operator sees what vpay
said. (`formatIsoInstant` was re-exported from `src/dash/procedure-list.ts`
until #383 deleted that file.)

## There is no total, and no page count

`GET /dash/v1/payment_intents`'s `ListObject` is `{ object, data, has_more,
url }`. A count invented on a screen (`data.length`, or `0` for "unknown")
would be a number a pager could divide by. The procedures do answer
`totalCount` (`count(*) OVER ()`), and **`@cratestack/refine` reports `0` for
an unknown one**: past the end of the set a procedure answers
`totalCount: null`. Read the count from the procedure's page
(`ProcedureListPage` in `src/dash/procedure-paging.ts`), never from Refine's
`total`.

`has_more` means "a row exists past the limit **in the direction just
walked**". `ending_before` reverses the rows before answering, so it reads
differently backwards. `pageCursors` in `src/payments-query.ts`, which
survived, resolves that in one place. `src/payments-query.test.ts` fails twice
if it reads `has_more` the same way in both directions. (So did
`src/dash/provider.test.ts`, deleted 2026-10-27.)

## A tenant-scoped `404` must stay indistinguishable

The detail read is tenant-scoped: another merchant's id answers, **byte for
byte**, the `404` a nonexistent id does (`dashboard_read_surface.rs`, claim 2,
still pinned under console keys). Until 2026-10-27 the payment page mapped it
to `notFound()` on the server, before anything reached the browser. That page
is deleted. **Whoever restores it in VP-6b inherits the rule**: rendering
"you may not see this" for one and "no such payment" for the other turns the
page into an oracle for which ids exist in other tenants. A client-side
`useOne` cannot call `notFound()` on a resolved fetch, so where the read runs
is a security decision, not plumbing.

## What this page said before 2026-10-27, and why it stopped being true

vaam-apps/vpay#383 deleted every module these sections described. They are
kept, struck, because an agent on an older tree will meet the code, and an
agent on this one may meet a document that still describes it.

- ~~**The BFF**: six `GET` route handlers under `app/api/dash/`
  (`src/server/bff.ts`), authenticated on the httpOnly session cookie, proxying
  to `/dash/v1` with the bearer read out of the `staff_sessions` row; refused a
  cookie-less request before any upstream call, checked `Origin` plus
  `Sec-Fetch-Site: same-origin`, forwarded no merchant id, audience or scope.
  `middleware.ts` answered `405` with no `Allow` to every method but
  `GET`/`HEAD` on `/api/dash/*`.~~ **Deleted 2026-10-27**, with
  `middleware.ts`, `csrf.ts` and the five `dashboard.cy.ts` cases that drove
  `/api/dash/**` from a real Chrome. `docs/status/frontend.md` marks those
  cases "Retire with the BFF; restate if VP-6b adds a browser-reachable
  surface". Whether the app should have one at all was decision RD5 in the
  Refine plan, and it is still a maintainer's call. **Do not recreate
  `app/api/dash/` as part of restoring a page.**
- ~~**The token lifecycle**: the `/dash/v1` access token lived 900 s; the app
  re-minted at 80 % of the TTL (`REMINT_AFTER_FRACTION = 0.8`), retried a
  `401` exactly once and never a `403` (`withOneRemint`).~~ **Deleted
  2026-10-27:** there is no token. A key expires 30 days after pairing, and a
  refused key means re-pairing, not re-minting (`docs/status/frontend.md`,
  case 15's successor).
- ~~**The procedure transport is the generated client**: only
  `src/dash/cratestack-client.ts` imports `@vpay/dash-client` at runtime, every
  call goes through `callProcedure`, and four screens read through
  `readProcedureList`.~~ **Deleted 2026-10-27** (the module, the screens and
  `src/server/dash-read.ts`). The rule survives in a stricter form; see above.
- ~~The README's test count and four other passages lag the code.~~
  **Superseded 2026-10-27:** #383 rewrote `frontends/apps/dashboard/README.md`
  (85 lines as of that day). It describes the notice app, and this page found
  nothing in it that contradicts the code.
