# What is not built — the list this resource exists to keep honest

`docs/flows/invoices.md`'s opening paragraph says why this list exists: _"a
document that lists only what exists is how somebody comes to believe an
invoice gets sent."_ Every entry below is a **gap**, dated, not a decision
against the feature, unless it says otherwise. Verified against code
2026-09-16.

The one navigational trap: the page's own link to this list,
`[What is not built](#what-is-not-built)`, points at a heading that does not
exist. The list is bold prose under `## Status`. Scroll to the bottom of the
page.

## Nothing sends an invoice anywhere

**No PDF, no e-mail, no hosted invoice page.** There is nothing to send and
nothing to send it with. `hosted_invoice_url` is a **checkout session URL**,
not a document — a payer following it sees the existing checkout page (the
merchant's name and the amount) and pays through the rails that already exist.

**The payer's checkout page does not show the invoice number** (2026-09-07,
recorded as a gap rather than a decision). `pay` writes `Invoice {number}` into
the intent's `description`, so the number _does_ reach the payer's browser —
`GET /v1/browser/checkout/sessions/{id}` expands the intent and `description`
is one of its keys — and `frontends/apps/checkout` renders the amount and the
merchant name only. So a payer following a `hosted_invoice_url` from an e-mail
sees a bill they cannot tie to the document that asked for it. Closing it is a
change to the checkout screens, not to this resource.

## No tax, no discount, no credit note

A line is a description, a quantity and a unit amount. **A merchant who needs
VAT puts it on a line.** There is no tax field, no rate, no discount and no
credit note object.

A refund against the intent that paid an invoice **leaves the invoice `paid`**
and adds to `amount_refunded`. There is no credit note, no sixth status and no
second `invoice.paid` (decided 2026-09-10, issue #91 D5; it was an open
maintainer question from 2026-09-07). Migration `0042`'s header carries the
argument; it is reversible in two places — that migration and one statement.

## No rail can refund, so `amount_refunded` is `0` everywhere

`mtn_momo::refund` is `ProviderError::NotImplemented` (refunds are MTN's
Disbursements product, for which this deployment has never held a credential);
Orange Money answers `Unsupported` (its Web Payment product documents no refund
API). `POST /v1/refunds` is **unrouted** and `vpay_db::Refunds` exposes no
`create`.

So `amount_refunded` is `0` on every invoice in every deployment,
`apply_refund_succeeded` is called by **no shipping binary**, and the cases
that prove it seed a `pending` refunds row with a raw `INSERT`. What is built
is the database's answer and the transaction that writes it; what is not built
is everything that would produce a refund in the first place.

## No dunning, no timer, no automatic `uncollectible`

`due_date` is stored and **read by nothing**. There is no reminder job, no
retry schedule and no timer that moves an invoice anywhere. A merchant learns
about a write-off from `GET /v1/invoices?status=uncollectible`.

## No subscriptions and no recurring invoices

The maintainer's decision of 2026-09-05 is that these will be built **in-house
on CrateStack** when they are built. Nothing here forecloses it, and
`invoice_items` deliberately has **no `subscription` column** rather than a
null one. That is also why there is no pending-charge inbox: `POST
/v1/invoice_items` writes straight onto a named draft.

## No partial payments

One intent at a time, and `paid` means paid in full.
`paid_means_nothing_remaining` is where that stops being a sentence in a
document — a `paid` invoice with anything remaining is a row Postgres refuses.
A merchant taking a deposit issues two invoices.

## Two Stripe event types are never emitted

`invoice.marked_uncollectible` and `invoice.payment_failed`. Neither is in
`type_is_a_documented_event`, because nothing writes them (migration `0023`'s
lockstep rule: the vocabulary moves with the writer). Do not add either label
without its writer in the same commit.

## `invoice.*` webhook bodies carry `lines.data` empty

Restated here because it is the divergence most likely to be discovered in
production rather than read. The `/v1` object always carries the lines; the
event body never does. See the SKILL.md quote block for why, and for the three
render sites.

## No dashboard screen, no browser test

`/dash/v1` exposes nothing about this resource. **No Cypress case renders an
invoice's `hosted_invoice_url`** — the URL is asserted to be a real
checkout-session URL by the integration suite and the checkout page it points
at is covered by `checkout.cy.ts`, but nothing has driven a browser from an
invoice to a paid charge.

## `sdks/stripe-compat` has no invoice cases

That suite drives the real `stripe@22.6.1` package rather than either merchant
SDK, so the closed SDK parity rows say nothing about it. It gets no parity rows
of its own (ADR-0015 decision 4) and is recorded here instead.

## What was closed, so you do not re-open it as a gap

Both of these read as gaps in older prose and are **done**:

- **Both merchant SDKs ship the whole resource** (2026-09-08, exp33):
  `client.invoices()` / `client.invoice_items()` in Rust,
  `client.invoices` / `client.invoiceItems` in Node, thirteen methods each —
  five CRUD, four transitions, four on a line — and both event unions know the
  four `invoice.*` types. Fifteen ✅/✅ rows in `docs/sdks/parity.md`.
- **Each SDK has a live suite against a real `vpay-server`**:
  `sdks/rust/tests/live_invoices.rs` (behind the `live-stack` feature, so
  `cargo nextest run --workspace` neither builds nor counts it) and
  `sdks/nodejs/src/invoices.live.test.ts` (its own vitest project, excluded
  from `pnpm test`). `just sdk-live` brings the stack up; CI's `e2e` job runs
  both. **Neither skips** — with no `VPAY_BASE_URL` they fail naming the
  variable, which is the whole difference between the row being closed and
  being laundered.

`backends/tests/integration/tests/invoices.rs` still drives **raw HTTP** and
should: it was written before the clients existed, and a suite rewritten to
drive one would assert the SDK's encoding rather than the server's contract.

## The three mutations that escaped the delivered suite

Worth knowing because they are the shapes this resource's tests fail at
(2026-09-07 review, each measured by applying the mutation and re-running):

1. deleting `NO_LIVE_INTENT` from `attach_intent` — every wire case green while
   two concurrent `pay` requests minted two intents for one bill;
2. unscoping either list cursor — every case green while another merchant's
   `in_…` as `starting_after` paged the caller's own rows;
3. keying the settlement's invoice flip on anything but its own intent — all
   seven delivered cases green while a settlement paid an invoice it was never
   bound to.

## One more stale document to distrust

`docs/plans/2026-09-06-data-layer.md` is cited by the first line of migration
`0034`'s header **and** `0036`'s. **No such file is tracked or on disk.**
Neither `.sql` can be corrected in place — `sqlx::migrate!` checksums a
migration's whole bytes and `just verify-migrations` pins them — and
`verify-links` does not read `.sql`, so nothing but a reader was ever going to
catch it. The same rule is why `0006`, `0013` and `0017` carry their
corrections in the flow docs rather than in the files.
