---
name: vpay-webhooks
description: vpay's outbound webhooks — the fifteen-type event vocabulary closed by a database CHECK, which two types nothing ever emits, the two refund types that started being emitted on 2026-09-16, the Vpay-Signature and Stripe-Signature HMAC scheme, the two-step outbox, the seven-rung delivery ladder, and the fact that webhook endpoints come only from YAML because there is no endpoint CRUD API. Load this before adding an event type, emitting an event, touching the outbox or the deliverer, or changing anything a merchant verifies.
---

# Outbound webhooks

> **Verified against vpay `93c6dfd0` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

Stripe's scheme, copied exactly, so a merchant's existing verification code
works unchanged.

Two crates: events are **written** by whichever transaction performs the
transition (in `vpay-api` handlers and in `vpay-db`), and **delivered** by
`vpay_worker::webhooks`. `docs/flows/webhooks.md` is the flow page.

## The state of it, as of 2026-09-16

Signing, the outbox, the ladder, the `webhook_deliveries` table and both SDK
verifiers are real and tested. **No webhook has ever reached a merchant
endpoint outside this repository** (`docs/status.md`). Every delivery in this
repository's history went to a test receiver.

## The event vocabulary is closed by the database, not by a Rust enum

Fifteen types, and the closed set is a Postgres CHECK constraint:
`type_is_a_documented_event` on `events.type`, most recently rewritten by
`backends/migrations/0039_events-customer-created-updated.sql`. The Rust side
carries a `String`, not an enum.

**Adding a type is a migration**, and the lockstep rule (migration `0023`) is
that the writer lands with it. ~~The one type that was ever in the vocabulary
with nothing writing it is `payment_intent.canceled`.~~ **Corrected
2026-09-16: there were three**, and `payment_intent.canceled` was the
shortest-lived of them. It sat for a week of releases while
`POST /v1/payment_intents/{id}/cancel` moved the row and told nobody, and a
merchant who settles from signed events could not reach a cancelled state at
all; `charge.refunded` and `charge.refund.updated` sat the same way from
`0018` until 2026-09-16.

**Only real Stripe event types go in this list.** A custom type is silently
dropped by any merchant using `stripe-node`'s typed event union or an
exhaustive `switch`. That constraint is why a _late_ success emits a plain
`payment_intent.succeeded`: an event merchants structurally tend to ignore is
the worst possible carrier for "money actually arrived".

**Thirteen of the fifteen have a writer since 2026-09-16. Two never do:**

| Type                        | Why nothing emits it                                            |
| --------------------------- | --------------------------------------------------------------- |
| `payment_intent.created`    | events are written for _terminal_ transitions; this is progress |
| `payment_intent.processing` | same                                                            |

Do not write code that waits for either of those two. Full writer table:
[references/events.md](references/events.md).

> ~~`charge.refunded` and `charge.refund.updated` have no writer — no rail in
> this repository can refund anything, and nothing writes a `refunds` row at
> all.~~ **Corrected 2026-09-16:** both are **emitted**, by
> `vpay_api::v1::refunds` (RFC-0003 § 2). They were in the original seven of
> `0018` and stayed documented-but-never-emitted until that date — far longer
> than `payment_intent.canceled`'s week.
>
> **A `charge.refunded` does not mean money came back.** It is written in the
> transaction that creates a `pending` refund and carries `status: "pending"`.
> Nothing settles a pending refund (RFC-0003 open question 8), so a handler
> waiting for a `charge.refund.updated` saying `succeeded` waits for ever.
> The three `charge.refund.updated` writers, and the one create path that
> emits both types for one refund inside one HTTP request, are in
> [references/events.md](references/events.md).

## The event row goes in the same transaction as the transition

There is no other shape in this repository. `Settlement::apply_succeeded` (in `vpay-db`'s `settlement` module)
moves the charge to `succeeded`, moves the intent, writes the
`payment_intent.succeeded` row and — if the intent pays one — the invoice and
its `invoice.paid`, all in **one** transaction; the customer erasure writes the
anonymisation and the `customer.deleted` in one; the checkout sweep flips
`status` and writes `checkout.session.expired` in one. The two refund types
follow the same shape since 2026-09-16: `POST /v1/refunds` writes the `refunds`
row, its reservation against the intent and `charge.refunded` in one
transaction, and the cancel writes the compare-and-swap, the released
reservation and `charge.refund.updated` in one.

~~(It writes **no ledger entries**. Nothing in this repository does.)~~
**Corrected 2026-09-16:** `Settlement::apply_succeeded` posts a CAPTURE to
`ledger_transactions`/`ledger_entries` in that same transaction. See the
`vpay-payments` skill for the ledger; the refund posting
(`apply_refund_succeeded`) exists and is reached by nothing.

If you emit an event from outside the transaction that made it true, a crash
between the two either tells a merchant about something that did not happen or
leaves something that did happen untold. Both are worse than the extra
statement.

One case **fails closed rather than emitting**: if the intent moved between a
rail's refusal and the `last_payment_error` stamp, the stamp matches no row, so
there is no committed intent to render and **no event is written** — with a
`WARN` naming both omissions. The alternative body would be either stale or
invented.

## Signing

`vpay_worker::signing::signature_header` is the only place in the workspace
that produces the header.

```
Vpay-Signature: t=1753401600,v1=<hex>[,v1=<hex>]
```

- signed payload is `"{timestamp}.{raw_body}"`, HMAC-SHA256, lowercase hex;
- `t` is plain decimal Unix seconds, and **the literal text in the header is
  the text that is signed** — it is written once and reused, because the whole
  scheme rests on those being the same characters;
- **one `v1=` per configured secret, in configuration order.** That is what
  lets a receiver holding _either_ secret verify during a rotation;
- receivers reject a timestamp older than **5 minutes**.

**The same value is sent again as `Stripe-Signature`**, so a merchant can hand
the request straight to `stripe.webhooks.constructEvent` — the official SDKs
verify `t=…,v1=…` over `{t}.{body}` with HMAC-SHA256, byte-identical to this
scheme. Both headers, every delivery.

**An endpoint with no secrets produces a bare `t=…` with no signature, and
that is deliberate.** Every verifier calls it a malformed header, which is
exactly the refusal an endpoint configured with no secret deserves — there is
no such thing as an unsigned delivery a receiver should accept. Do not "fix"
it by skipping the header or by inventing a secret. The deliverer is not
supposed to reach that state anyway: `handle_deliver` records a failed attempt
instead of sending, and boot-time validation is the real guard.

A pre-epoch clock writes `t=-…`, which fails both verifiers' `^\d+$` rule.
Clamping to zero would be worse — the delivery would then be signed with a
timestamp the sender does not believe and would fail the _tolerance_ check
instead, reported as something a merchant could plausibly debug.

## Endpoints come from YAML. There is no endpoint API.

`merchant_clients[].webhooks[]` in `config/application.yml` →
`vpay_config::oauth::WebhookEndpoint` → `vpay_worker::Endpoint { id, url,
secrets }`. `EndpointRegistry` is built at boot.

**There is no `/v1/webhook_endpoints` and no dashboard page for endpoints.** A
merchant cannot register, rotate or delete one through any API; an operator
edits YAML and redeploys. If you are asked to "let a merchant add a webhook
URL", that is a new surface, a new table and a new validation path — not a
missing route.

`Endpoint`'s `Debug` redacts `secrets` to a count. Keep it that way.

## Delivery, in one paragraph

Two-step outbox: `handle_fan_out` turns the `events` backlog into
`webhook_deliveries` rows plus `deliver_webhook` jobs, one transaction per
event; `handle_deliver` renders, signs and sends one; `handle_scan_deliveries`
is the backstop behind both. Delivery **never consults `Classify`** — a
merchant's `500` is not a `ProviderError`. Targets are SSRF-vetted per delivery
and the HTTP client is pinned to the vetted address.

Ladders, timeouts, abandonment and the runbooks:
[references/delivery.md](references/delivery.md).

## Traps

- **The four `invoice.*` event bodies carry `lines.data` EMPTY**, while
  `GET /v1/invoices/{id}` always carries the lines. The event's `data` is
  rendered inside the transaction that wrote the row, and reading the lines
  there would put a second query on a connection holding the invoice number
  sequence's row lock. This is a real difference between what a webhook says
  and what the API says about the same object.
- **`POST /v1/checkout/sessions/{id}/expire` emits nothing** — you asked for
  it. Only the hourly sweep's expiry emits `checkout.session.expired`.
- **`customer.deleted`'s `data.object` carries every identifier already
  `[redacted]`.** vpay stores event bodies for ever; between a merchant's
  convenience in identifying the payer and the payer's erasure being real, the
  erasure wins. This reversed on 2026-09-10 (issue #68) — older prose that says
  the body carries `name`/`email`/`phone` is wrong.
- **`payment_intent.payment_failed` has two writers** and that is deliberate:
  a decline at submit and a decline at a later poll are one thing to a
  merchant. A merchant cannot receive both for one intent, because there is one
  charge per intent forever.
- **A `charge.refunded` reports a `pending` refund and there is no second
  event coming.** An event held back until `succeeded` would be one that never
  arrives. Both refund types render the same ten-key `RefundObject` the API
  returns.
- **The payee's number is on no event body and in no response.** A refund's
  `destination` is persisted in no column at all (RFC-0003 open question 3,
  undecided), so `charge.refunded` cannot carry it even in principle. A
  merchant reconciling who was paid uses the rail's records, by
  `provider_reference_id`. Do not add it: events are stored for ever, which is
  `customer.deleted`'s redaction argument.
- **Delivery is at-least-once, unordered, and can be at-most-zero.** A merchant
  who missed one is pointed at `GET /v1/events`, which renders through the
  **same** `EventObject` the deliverer signs. Two renderers would let the
  fallback answer a different question from the one the webhook asked.

## Where the docs are wrong

Code wins.

- `backends/migrations/0018_create-events.sql`'s `COMMENT ON TABLE events`
  still says the table is "NOT WRITTEN OR READ BY ANY CODE IN THIS REPOSITORY
  — nothing emits events, /v1/events is not routed". Both halves have been
  false since 2026-09-03. **Do not edit that file to fix it** — the bytes of an
  applied migration are pinned by `MANIFEST.sha256` and `verify-migrations`;
  migration `0039`'s comment is current and correct, and a later migration is
  the only way to restate it.
