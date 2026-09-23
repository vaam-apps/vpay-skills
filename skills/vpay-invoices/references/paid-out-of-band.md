# Paid out of band — a statement, not a payment

_Written 2026-09-23 against the vaam-apps/vpay step A integration branch
(`feat/rfc-0004-step-a`, RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251). None of this
exists on a vpay `master` older than that merge. See
[VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Flow page: `docs/flows/invoices.md` § "Paid out of band". Decisions: D5–D19 in
`docs/adr/0024-customer-filters-and-manual-payments.md`, all accepted
2026-09-23.

```text
POST /v1/invoices/{id}/pay
paid_out_of_band=true
out_of_band[method]=cash|cheque|bank_transfer|other   optional, default other
out_of_band[reference]=…                               optional, 1–500 chars
out_of_band[received_at]=<unix seconds>                optional, default now
```

- **The bare flag is enough.** A Stripe-shaped client sends
  `paid_out_of_band=true` and nothing else; vpay records `other` (D5, D11 —
  D11 said "required" in the ADR's first draft and was reversed the same day).
- **Every malformed input is a `400` naming its parameter**, including
  `success_url`/`cancel_url` sent with the flag, `out_of_band[…]` sent without
  it, an unknown `out_of_band[…]` key (named as `out_of_band`, never echoed),
  and `received_at` more than 30 s ahead or earlier than the second
  `finalized_at` falls in. Then the ordinary `409`s: not `open`, or an intent
  attached that is not `canceled` — cancel it first, exactly as for `void`.
- **No configured URLs and no publishable key are needed**: the fork in
  `pay_once` is taken before `merchant_clients[].invoices` is resolved.
  Moving that resolution above the fork turns seven cases red.
- **Nothing is posted to the ledger, and nothing verifies it.** No money
  crossed `payer_clearing`; under pass-through (RFC-0001) vpay has no account
  it could have arrived in. Never add a posting "for completeness", and never
  describe an out-of-band `paid` as money vpay saw. **Only the merchant `/v1`
  surface writes it** — an operator cannot (ADR-0008's dashboard writes are
  unbuilt).
- **`bank_transfer` here is a label, not a rail.** Matching a bank transfer to
  an invoice is RFC-0007, Draft and unbuilt.
- **A canceled intent stays attached** (D15), so `hosted_invoice_url` on such
  an invoice still points at the canceled attempt's checkout session. Recorded,
  not fixed; a voided invoice behaves the same.
- **The record cannot disagree with its bill.** The insert copies amount,
  currency, tenant and mode off the invoice inside the statement
  (`INSERT … SELECT … FROM invoices WHERE id = $2 AND paid_out_of_band`), and
  the composite foreign key `manual_payments_agree_with_their_invoice` onto
  `invoices_payment_record_key` makes a disagreeing row unstorable. See the
  `vpay-data-layer` skill for why that is a foreign key and not a CHECK.
- **`reference` is personal data** (`payment_reference`, `subject: payer`):
  erasure redacts it everywhere it was copied, and a new reference on an
  erased customer's invoice is a `400` naming `out_of_band[reference]` — the
  payment without one still records. The `vpay-customers` skill has the lock
  order that makes that race-free.
- **It does not stamp the customer's `last_used_at`**, and neither does
  hosted `pay`; only creating the invoice does. An existing gap, not a new one.

Code: `vpay_api::v1::invoices::pay_out_of_band_once`,
`vpay_db::invoices::pay_out_of_band_in_tx` (three statements, in a designed
order — `docs/reference/vpay-db/customers-and-invoices.md`). The refund
counter `add_refund_for_intent_in_tx` gained `AND NOT paid_out_of_band`.

## The evidence, as of 2026-09-23 on the branch

`backends/tests/integration/tests/invoices.rs` is 29 cases, 29 passed, 0
ignored — thirteen of them for this. `vpay-db`'s `tests/repositories.rs` adds
two (`paying_out_of_band_records_the_invoices_own_amount_and_posts_nothing`,
`paying_out_of_band_keeps_the_canceled_intent_and_nothing_can_move_it_again`).
`the_out_of_band_invariants_are_enforced_by_the_database_itself` writes every
new constraint's refused row straight past the API, pinned by constraint name.
Two mutations were run: resolving `merchant_clients[].invoices` before the fork
turns seven cases red, and removing the intent the rewritten
`the_invoice_invariants_are_enforced_by_the_database_itself` fixture attaches
turns it red on `paid_names_how`. Gate output:
`docs/status/verification/2026-09-23-manual-payments.md`.
