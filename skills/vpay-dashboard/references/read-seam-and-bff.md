# The `/dash/v1` read seam, the token lifecycle, and the BFF

_Verified against vpay `93c6dfd0` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

## Two transports, one seam

`/dash/v1` serves `payment_intents` as a **cursor-paged REST route** shared
with the merchant API. Everything else this app lists is a **CrateStack
procedure**, reached at `POST /dash/v1/$procs/<name>`.
`src/dash/resource-name.ts` holds both spellings in one file on purpose — "a
resource whose procedure name is guessed at the call site is a 404 nobody
notices until a screen is empty":

| Resource constant    | Served by                                     |
| -------------------- | --------------------------------------------- |
| `PAYMENT_INTENTS`    | `GET /dash/v1/payment_intents` (REST, cursor) |
| `REFUNDS`            | `procedure searchRefunds`                     |
| `WEBHOOK_DELIVERIES` | `procedure searchWebhookDeliveries`           |
| `CUSTOMERS`          | `procedure searchCustomers`                   |
| `CHECKOUT_SESSIONS`  | `procedure searchCheckoutSessions`            |

**`payment_intents` is deliberately absent from `PROCEDURE_OF`.** It is a
_partial_ map, not a total one: a lookup that misses means "this resource is
not a procedure", which is a different thing from "unknown resource". Do not
"fix" it by adding an entry.

`resource-name.ts` exists separately from `provider.ts` because `provider.ts`
is server-side — it reaches `readDash`, the config and `node:crypto` — so a
client component importing the constant from it dragged the whole server graph
into the browser bundle and the build failed with `Reading from "node:crypto"
is not handled by plugins`. `provider.ts` re-exports it, so there is still one
spelling of the string.

### The `$procs` transport mounts FIVE procedures, and the code says otherwise

**Docs↔code disagreement. The code wins: five are mounted.** Verified
2026-09-16 by reading the schema, the registry and the router.

`dashboard_procedure_router` in `backends/crates/vpay-db/src/schema.rs` calls
`cratestack_schema::axum::procedure_router(..., Payments, ...)`, which mounts
**every procedure in the registry**. `impl ProcedureRegistry` on
`search_payment_intents::Payments` has five methods, and `schemas/vpay.cstack`
declares five `procedure`s. So the mounted set is:

```
POST /dash/v1/$procs/searchPaymentIntents
POST /dash/v1/$procs/searchRefunds
POST /dash/v1/$procs/searchWebhookDeliveries
POST /dash/v1/$procs/searchCustomers
POST /dash/v1/$procs/searchCheckoutSessions
```

Three places still describe a single procedure, and all three are stale — they
were true for Lane C and went stale when the four later procedure bodies
landed:

- `backends/crates/vpay-db/src/schema.rs`'s test is _named_
  `no_generated_model_route_is_mounted_only_the_one_procedure_is`, and its
  assertion message says "the one procedure this transport mounts".
- `backends/crates/vpay-api/src/lib.rs` describes the boundary as "mounted in
  front of `POST /$procs/searchPaymentIntents`".
- `backends/crates/vpay-db/src/schema/search_payment_intents.rs` is the module
  name for a file whose `impl ProcedureRegistry` now covers all five (the four
  later bodies live in sibling modules — `search_refunds.rs`,
  `search_webhook_deliveries.rs`, `search_customers.rs`,
  `search_checkout_sessions.rs` — and the registry delegates to them, because
  the trait lives inside the private macro expansion and every body has to be a
  child of that module).

**Two consequences for anyone adding a sixth procedure:**

1. **The router will mount it automatically.** Declare it in
   `schemas/vpay.cstack`, add the registry method, and the route exists. There
   is no mount list to edit — which also means there is no mount list that
   would have stopped you.
2. **Nothing will test it for you at the transport layer.** That test probes
   exactly one path — a `GET` on `/$procs/searchPaymentIntents`, expecting
   `405` rather than `404`, because CrateStack procedures are `POST`-only and a
   matched route answering the wrong method is the discriminator. **The other
   four are mounted and unprobed**, and a sixth would be too. Its nineteen-table
   half — asserting no generated model CRUD route is mounted — does still cover
   the whole model set, and has its own note about a model declared after the
   list was written being exactly the one nobody has proved unmounted.

If you add a procedure, add a probe for it in that test and fix the test's
name while you are there.

## Instants: two formatters, because there are two wire shapes

`src/format.ts`'s `formatInstant` takes **unix seconds** (the REST list
serialises an integer). `src/dash/procedure-list.ts`'s `formatIsoInstant`
takes an **RFC 3339 string** (a CrateStack procedure serialises a `DateTime`).
Passing one to the other yields `Invalid Date` or, worse, a plausible date in 1970. An unparseable value is returned **verbatim** rather than replaced: if
vpay answers something this cannot read, an operator should see what it said.

## The token lifecycle

`src/server/dash-read.ts` and `src/server/gate.ts`.

The `/dash/v1` access token lives `staff_auth.access_token_ttl_seconds`,
**900 by default**. A staff session runs thirty minutes idle / twelve hours
absolute (ADR-0017 decision 2). So the credential a page reads with dies a
quarter of an hour into a session that has most of a working day left — and
`requireStaff` used to mint a token only when the row carried **none**.

Measured on the real stack: fifteen minutes after signing in, every render of
`/payments` said "The bearer token is invalid, expired, or was not issued for
this endpoint", with the row still holding the same dead token afterwards.
`dashboard.cy.ts` runs in under thirty seconds and never reached it.

Three rules now:

1. **Re-mint at 80 % of TTL.** `REMINT_AFTER_FRACTION = 0.8` in `gate.ts`, so a
   token is replaced with a fifth of its life in hand (180 s on the default
   TTL). `>=` and not `>`: at exactly the margin the fifth is spent.
2. **Retry a `401` exactly once**, after a fresh mint. A second `401` is not an
   expiry — the scope was revoked, the registration changed, the clock is
   wrong — and retrying again would be a loop with a credential operation in
   it. The original refusal is rendered, with its request id.
3. **Never retry a `403`.** `/dash/v1` answers `403` for a token that is valid
   and not allowed (wrong scope, wrong merchant claim), which a new token from
   the same registration would answer identically.

Re-minting is _more_ checking than carrying one token, not less:
`vpay_api::staff::oauth::authorize` re-reads the staff row and re-checks that
the account is active, that `merchant_id` is still the dashboard client's
binding, and that `password_change_required` is not set, **on every mint**.

`readDash` takes both tokens as arguments rather than reading them, so the
whole of it runs against a stubbed `fetch` — `dash-read.test.ts` is what would
have caught the fifteen minutes.

## `notFound()` is a security property, not a convenience

`app/(dash)/payments/[id]/page.tsx` maps vpay's `404` to `notFound()`
**because the detail read is tenant-scoped**: another merchant's id answers the
identical `404` a nonexistent one does. Rendering "you may not see this" for
one and "no such payment" for the other would turn the page into an oracle for
which ids exist in other tenants. It is answered on the server, before anything
reaches the client screen, so the two bodies stay indistinguishable whatever
the browser does next.

This is also why a client-side data layer is not a plumbing detail: a `useOne`
in a client component **cannot call `notFound()`** on a resolved fetch, and
choosing what replaces it is a decision about a cross-tenant id oracle on a
payment system.

## There is no total, and no page count

`ListPage` in `src/dash/provider.ts` carries `data`, `hasMore` and `cursor` and
nothing else, because `/dash/v1`'s `ListObject` is `{ object, data, has_more,
url }`. A count invented here — `data.length`, or a `0` standing in for
"unknown" — would be a number a pager could divide by, and the control it drew
would be decorative.

`has_more` means "a row exists past the limit **in the direction just
walked**", which is a different sentence backwards than forwards:
`ending_before` reverses the rows before answering. `pageCursors` in
`src/payments-query.ts` is where that inversion is resolved, in one place.
Making it read `has_more` the same way in both directions fails
`src/dash/provider.test.ts` and `src/payments-query.test.ts` twice.

## The BFF

Route handlers under `app/api/dash/`; the substance is `src/server/bff.ts`.
They authenticate on the same httpOnly cookie and proxy to `/dash/v1` with the
token read out of the `staff_sessions` row on that request — the bearer never
leaves this process. **They are an authenticated surface a script on this
origin can call, which the app did not have before.**

Every property is written as a mutation in `src/server/bff.test.ts`:

- A request with **no session cookie** is refused and reaches vpay **not at
  all** — the test asserts the stub `fetch` was never called, because a `401`
  answered _after_ an upstream round trip is a surface anyone can use to make
  this server open connections. (Reading the cookie _after_ the upstream read
  fails this twice.)
- A request this dashboard did not issue is refused by `csrf.ts`'s
  `originIsAllowed`, **plus** `Sec-Fetch-Site: same-origin` — which is what a
  browser sends instead of an `Origin` on a same-origin `GET`.
- The bearer appears in no response header and no body, asserted against the
  serialised response while the stubbed vpay echoes the token into a header,
  into the envelope's `url`, and into an extra top-level field.
- No caller-supplied merchant id, audience or scope is forwarded: the upstream
  query string is built by `apiQueryString` from the five parameters
  `queryFrom` reads and nothing else is looked at.
- A repeated parameter takes its **first** value, the same as the page does.
  `Object.fromEntries(searchParams.entries())` keeps the last, which would make
  one URL mean two different pages depending on which surface read it.

### The method policy

Only `GET` is exported, and **no write can reach the file**. Next
auto-implements two methods:

- **`HEAD` is bound to the `GET` handler itself**, body discarded — so a `HEAD`
  costs the same session read, the same possible token mint and the same
  upstream call. Consistent with `/dash/v1`, which admits `GET` and `HEAD`.
  Left as it is.
- **`OPTIONS` answered `204` with `Allow: GET, HEAD, OPTIONS` before the
  handler ran at all** — before the origin check and before the cookie was
  read. The one answer this surface could give without passing its own gate.

`middleware.ts` now matches `/api/dash/:path*` and answers **`405` with no
`Allow` header** to every method that is not `GET` or `HEAD`, before the route
module is reached. Measured against a real `next start`:

| Probe                               | Before          | After              |
| ----------------------------------- | --------------- | ------------------ |
| `OPTIONS /api/dash/payment_intents` | `204` + `Allow` | `405`              |
| `OPTIONS /api/dash/nope`            | `404`           | `405`              |
| `POST /api/dash/payment_intents`    | `405` (Next's)  | `405` (this one's) |
| `GET /api/dash/nope`                | `404`           | `404`              |

**For `OPTIONS`, route existence really is unanswerable now. For `GET` it is
not**, and this must not be read as saying otherwise: an unauthenticated `GET`
to a route that exists reaches the gate where a path nothing serves gets Next's
`404`. `OPTIONS` was the loudest discriminator and the only one reachable
without passing a check, never the only one — making the `GET` pair uniform
would mean answering `404` to an honest signed-out client.

`middleware.test.ts` holds the pair: one runs the real route module through
**Next's own `autoImplementMethods`** and measures the `204` and `Allow` it
would still produce; the next asserts what the caller actually gets. Making
`middleware` return `NextResponse.next()` unconditionally goes red three times.

### Two corrections the review made, both worth carrying

The review recorded its findings against **Next 16.3.4**, which is
`examples/shop`'s pin; this app resolves **15.5.25**. The finding held; the
version named in it was not this app's. The test measures the installed one
rather than repeating either number.

`Sec-Fetch-Site` is not merely "a pre-2020 browser" question — **Safari has
sent it only since 16.4 (March 2023)**, so this surface answers `403` to every
older WebKit. That costs nothing while nothing calls it, and is a decision for
whatever eventually does.

### Browser evidence

Five cases in `frontends/tests/e2e/cypress/e2e/dashboard.cy.ts` drive
`/api/dash/**` from a real Chrome against the real stack — **the only evidence
in this repository about what a browser actually puts on the wire here.** The
decisive one is `refuses a cross-origin FRAME of the same URL, which carries no
Origin at all`: an `<iframe>` is a navigation, so it carries no `Origin`, and
`SameSite=Lax` still attaches the session to a **same-site** request — so it
reaches this surface with a signed-in cookie and nothing but `Sec-Fetch-Site`
identifying it. Delete that comparison from `apiIsSameOrigin`, rebuild the
image and re-run: `200` where `403` was expected, and every other case stays
green.

## Known-stale prose

Two, both verified 2026-09-16.

**The "one procedure" sentences in the Rust transport are stale — five are
mounted.** Full detail above. Code wins.

**`frontends/apps/dashboard/README.md`'s counts are a 2026-09-11 snapshot.** It
says the BFF is "**Two** `GET` route handlers" and quotes a suite of "22 files,
243 tests". There are now **six** route files under `app/api/dash/`
(`payment_intents`, `payment_intents/[id]`, `checkouts`, `customers`,
`deliveries`, `refunds`) and the suite is **311 cases in 30 files, 0 skipped**
per `docs/flows/dashboard/status-built-and-not-built.md`. The README's
_properties_ are all still right; only its counts lag.
