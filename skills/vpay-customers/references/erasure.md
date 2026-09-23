# Erasure — the six tables, the marker CHECK, and the closed race

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`vpay_db::customers::erase_in_tx` is the whole of it. Migration
`0041_customers-address-and-anonymisation.sql` is the schema half. The flow
page is `docs/flows/customers/privacy-and-erasure.md`.

## The branch, and the race the database catches

Inside one transaction, under the caller's `SELECT … FOR UPDATE` on the
customer row:

```sql
SELECT NOT (UNREFERENCED) FROM customers WHERE id = $1
```

`UNREFERENCED` is three `NOT EXISTS` clauses — `payment_intents`,
`checkout_sessions`, `invoices` — and
`the_sweep_guard_names_every_table_that_can_reference_a_customer` pins the set
at three. The third was added on 2026-09-07 when invoices landed; without it
the erasure takes the hard-delete branch, the `NO ACTION` foreign key raises
`23503`, the whole transaction rolls back, and the payer is left un-erased with
the merchant told nothing.

The row lock does **not** stop a concurrent `POST /v1/payment_intents` taking a
share lock and inserting a reference, so the branch can be wrong by one race in
exactly one direction — and the `23503` rolls it back, leaving a live customer
and a retry that takes the other branch. The opposite race cannot happen:
history is never removed.

Foreign keys are `NO ACTION` and **stay that way**. `ON DELETE SET NULL` was
the alternative and is worse: vpay never detaches a payment from the payer it
was taken from, because that is the record a dispute is settled with.

## "Every copy" is six tables

An erasure that only rewrites `customers` is the characteristic defect here.
`erase_in_tx` writes all of these **in the transaction that erases the row**:

| Table                                         | What was in it                                                                                                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `customers`                                   | the eleven identifier columns                                                                                                                                  |
| `events.data`                                 | **every** `customer.*` body ever written stores the whole rendered object, and nothing prunes `events`                                                         |
| `charges.payer_ref` / `payer_ref_masked`      | the payer's MSISDN as the rail was given it — reachable from a customer only _through_ an intent                                                               |
| `charges.failure_raw` / `refunds.failure_raw` | the rail's own words, verbatim: a decline may quote the subscriber's number back                                                                               |
| `idempotency_keys.response_body`              | the exact JSON a `POST /v1/customers` answered, kept 24 hours to replay                                                                                        |
| `webhook_deliveries.payload_sha256`           | cleared — see below; this one protects a delivery, not the payer                                                                                               |
| `manual_payments.reference` (step A)          | the merchant's out-of-band payment reference; also its copies in `invoice.*` bodies, their live deliveries' digests and excerpts, and stored invoice responses |

The last row exists since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged
in vaam-apps/vpay#251; migration `0049`), written by `redact_out_of_band_references` in the
erasure's transaction. It is the first copy in this table that the
**merchant** typed rather than vpay derived, and it is still `subject: payer`:
a reference exists to say how the payer paid. The whole-database scan now
seeds a paid-out-of-band invoice whose reference names the payer and expects
`manual_payments.reference` among the places it finds the literal before the
erasure. _(This table is as verified on 2026-09-16 plus that row; the
`payment_intents.last_payment_error_*` and `webhook_deliveries.response_excerpt`
statements added on 2026-09-18 are not in it — see vpay's
`docs/reference/personal-data-inventory.md`. The count "six" is not restated
here for that reason.)_

`provider_requests` needs no statement, and that is a property of its schema
(`0016`: a status code and an attempt number, no bodies) rather than an
oversight. `refunds.reason` is left alone — it is the merchant's free text
about their own refund, the same kind of thing `metadata` is.

Two of these were found only because a test caught them, not because anyone
enumerated them. `charges.failure_raw` and `refunds.failure_raw` were added on
2026-09-11 **after they survived an erasure in a test**. The marker replaces
the whole string rather than the number inside it: a redaction that had to
recognise every spelling a rail might use fails silently on the first one it
has not seen. `failure_code` beside it survives, so _why_ a payment failed is
still answerable.

The stored `customer.*` bodies carry the payer's position nested inside
`data.object.address`; the redaction replaces the whole `address` key with the
redacted object rather than walking into it, so no nested component can survive.

The event for this erasure is inserted **before** the redaction statements, so
the redaction covers it too. The invariant is "no `events` row holds this
payer's identifiers", not "none except the newest".

## Why `webhook_deliveries.payload_sha256` is cleared

`webhook_deliveries` stores a digest and not the bytes (`0022`), so it holds no
copy of the payer. It gets a statement for the opposite reason to a leak: the
digest is recorded by the first signed attempt and compared on every later one,
so rewriting `events.data` makes a pending delivery fail that comparison and
**dead-letter**, with an operator-facing message blaming "a renderer changed
under a live delivery". The merchant then never learns the payer was erased,
and un-parking a dead letter is manual. Only deliveries that can still be
attempted are cleared; `succeeded` and `exhausted` ones are forensics.
`an_erasure_mid_ladder_redelivers_the_redacted_body_instead_of_dead_lettering`
in `tests/webhooks.rs` is the proof.

## The marker CHECK, and the `IS NOT DISTINCT FROM` trap

`anonymized_customers_carry_the_marker` (`0041`) refuses any row whose
`anonymized_at` is set and whose eleven identifier columns are not all in their
erased state: the literal `[redacted]` in the nine text ones, `NULL` in the two
coordinate ones.

**It is spelled `IS NOT DISTINCT FROM`, not `=`, and that is not style.** A
CHECK is violated only when its expression evaluates to FALSE, and
`name = '[redacted]'` over a NULL `name` is NULL, which **passes**. The `=`
spelling therefore accepted an `anonymized_at` row with a NULL identifier —
the first state a missed assignment produces, since the columns an erasure most
easily misses are the ones nobody filled in. The review caught this on
2026-09-11. If you add a twelfth identifier column, add it to this CHECK in the
same spelling; the uniformity is what makes an odd one visible.

The two coordinate columns are `NULL` rather than a marker because they are
`BIGINT` and **there is no integer that is not a possible place** — a marker
value would be a coordinate somewhere real on a row claiming the payer is gone.
`anonymized_at` is what still says a payer was there.

All eleven are written, including components the payer never filled in, because
_which fields a record carried is itself information about the person_.

`at_least_one_identifier` was **not** relaxed and does not need to be: the
marker is not NULL. It now backstops the erasure in the one direction that
matters — an erasure that NULLed the three identifiers instead of marking them
is refused outright.

`address_coordinates_are_both_or_neither` and the two shape CHECKs
(`phone_is_a_canonical_msisdn`, `address_country_is_iso_3166_1_alpha_2`) each
carry an `anonymized_at IS NOT NULL` disjunct, so the marker CHECK is the
**only** constraint that can fire on a row claiming to be erased. That is what
lets the marker CHECK be tested one column at a time.

**All of these are multi-column and therefore invisible to `cratestack migrate
baseline` in both directions.** The drift report cannot be the guard for them.
`postgres_smoke.rs` carries the multi-column CHECK inventory, and
`an_anonymised_customer_carries_the_marker_in_every_identifier_column` writes
the rows they exist to refuse.

## The evidence, and the one limit it has

`an_erasure_leaves_no_payer_identifier_in_any_column_of_any_table` scans
**every** `text`, `character varying` and `jsonb` column `information_schema`
reports in `public`, before and after, for five fixture literals — found in
seven named places before the `DELETE` and nowhere after. A test that named
tables would have named the wrong ones, which is exactly what happened to
issue #68.

Three stated limits: `public` only; it can only find a copy of a literal the
fixture wrote; and it reads text and jsonb, so the **two `BIGINT` coordinate
columns are outside it in principle**. Widening it to numeric columns would not
help — every integer is a possible coordinate, so a hit and a miss both mean
nothing. Those two are asserted **directly, by name, as NULL**. A `jsonb`
column cast to `TEXT` does render a number as digits, so the fixture's latitude
_is_ findable in `events.data` and `idempotency_keys.response_body` before the
erasure and must not be after.

## The idempotency race, closed 2026-09-12 (issue #111)

A `POST /v1/customers/{id}` that committed, lost the race to a `DELETE`, and
only then stored its response wrote the pre-erasure payer back into
`idempotency_keys.response_body`, where replaying the key re-read it for up to
24 hours.

The fix is a **shaped exception, not a smarter store**.
`vpay_db::Idempotency::store` takes a `StoredResponse` whose `subject` is
either `ResponseSubject::Verbatim` — every route but `/v1/customers`, one
statement, unchanged — or `ResponseSubject::Customer { id }`, which the three
customer write routes pass. The customer arm wraps the write in a transaction
that reads the payer's row `FOR SHARE` first and redacts the body it just
stored if that payer is gone. Every erasure takes `FOR UPDATE` on that row
first, so the two serialise; a hard-deleted customer leaves no row, so
**absence reads as erased**.

Three things it deliberately is not: it does not inspect the body (the id comes
from the route); it does not define what a redacted customer body is (it calls
`vpay_db::customers`' `redact_stored_responses_in_tx`, the same statement the
erasure runs, so `cargo xtask`'s SQL-interpolation audit still counts one site
and the two sides cannot drift); and it does not touch the response the racing
request is _answered_ with.

The decisive mutation: make `store` treat `ResponseSubject::Customer` as
`Verbatim` and both
`an_update_that_loses_the_race_to_an_erasure_stores_no_payer_identifier`
rounds fail with the payer's name in the stored body.

**If you add a fourth customer write route, it must pass
`ResponseSubject::Customer`.** Nothing greps for that.

**A third variant since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged
in vaam-apps/vpay#251): `ResponseSubject::OutOfBandInvoice { customer_id }`.** The
out-of-band `pay` and a `POST /v1/invoices/{id}` on an invoice paid out of band
pass it, because their stored body carries `out_of_band_payment.reference`.
Same mechanism, keyed on the invoice's customer; it runs the same
redaction statement the erasure runs.
`a_stored_invoice_response_written_after_an_erasure_is_redacted_as_it_lands`
drives the late-write branch. **A new invoice write whose response can carry a
reference must pass it too** — nothing greps for that either.

The paying transaction's lock order is what makes the _refusal_ of a new
reference on an erased payer's invoice race-free: `FOR SHARE` on the customer
first, where every erasure takes `FOR UPDATE` first; the settlement, `void`,
`attach_intent` and the other invoice writes take no customer lock at all.

## What is NOT closed, and is a maintainer's decision

**vpay cannot erase the merchant's own copy, and there is deliberately no
second event type for it.** The merchant received the payer's details in
`customer.created` and in every `customer.updated`, over signed bodies to
endpoints they configured. `customer.deleted` is emitted in the erasure's
transaction on both branches and carries the `cus_…` and the instant, which is
the whole signal a merchant needs. A `customer.redacted` type was considered
and **declined** (2026-09-11, again 2026-09-12): it carries nothing
`customer.deleted` does not, a Stripe-shaped handler has no branch for it, and
it would read as an enforcement vpay cannot perform.

What is missing is a **contract** — whether a merchant is obliged to act, and
within what window. Four options with their costs are written out in
`docs/flows/customers/privacy-and-erasure.md` § "The decision, written out —
NOT TAKEN". **It is not taken, nothing in the repository assumes any of them,
and an agent must not pick one.** Note option D's specific failure mode: a
green "erased" tick in a dashboard that nothing checked is exactly the defect
`AGENTS.md`'s second rule names.

Also open by choice rather than by gap: **an erased customer is still listed by
`GET /v1/customers`** (filtering would make `has_more` describe a different set
from the rows, and would make the list deny a `cus_…` the retrieve answers),
and there is **no `email` filter on the list** — a filter on a payer identifier
turns the list into a lookup. ADR-0024 D4 reaffirmed that on 2026-09-23 while
adding `customer=` to the intent, session and refund lists (step A): a filter
by the merchant's own `cus_…` is not this gap closed.
