# MTN's three failure vocabularies

_Verified against vpay `93c6dfd0` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

All of this lives in `backends/crates/vpay-adapter-mtn-momo/src/mapping.rs`,
kept in one module away from the transport, because **this is the part that
drifts**: MTN grows a `reason` string and the only visible symptom is a
rising `provider_error` rate. A table in one file is a table a reviewer can
diff against the rail's documentation; the same rows spread across `match`
arms in the request code are not.

MTN uses three vocabularies and collapsing them would lose information.

## 1. `FAILURE_REASONS` — outcomes

The `reason` on a `FAILED` status, and the `code` on a 400. Twelve rows, in
the same order as the table in `docs/flows/adapter-mtn-momo.md` so the two
can be diffed by eye.

| MTN reason                      | `FailureCode`                                                    |
| ------------------------------- | ---------------------------------------------------------------- |
| `NOT_ENOUGH_FUNDS`              | `insufficient_funds`                                             |
| `COULD_NOT_PERFORM_TRANSACTION` | `payer_timeout`                                                  |
| `EXPIRED`                       | `payer_timeout`                                                  |
| `PAYMENT_NOT_APPROVED`          | `payer_declined`                                                 |
| `APPROVAL_REJECTED`             | `payer_declined`                                                 |
| `PAYER_NOT_FOUND`               | `invalid_payer`                                                  |
| `PAYER_LIMIT_REACHED`           | `payer_limit_reached`                                            |
| `SENDER_ACCOUNT_NOT_ACTIVE`     | `payer_account_blocked`                                          |
| `PAYEE_NOT_FOUND`               | `invalid_payee`                                                  |
| `PAYEE_NOT_ALLOWED_TO_RECEIVE`  | `payee_account_blocked`                                          |
| `NOT_ALLOWED`                   | `provider_account_blocked` — **our** partner account; this pages |
| `SERVICE_UNAVAILABLE`           | `provider_unavailable`                                           |

`failure_code()` is **case-insensitive**: MTN documents and sends these
uppercase, but a table that silently stops matching because a rail changed a
string's case is a table that fails open into `provider_error`.

The fallback is `FailureCode::ProviderError`, and it is **deliberately not an
error** — an unmapped reason is still a decline, and the raw string travels
with it in `ChargeStatus::Failed { raw }` so an operator can add the row.
`raw` is never empty: a rail that failed a charge without saying why is
itself worth seeing, so a reasonless `FAILED` still carries
`"FAILED (no reason given)"`.

## 2. `CONFIGURATION_CODES` — HTTP 500 that is really our mistake

`INVALID_CURRENCY`, `NOT_ALLOWED_TARGET_ENVIRONMENT`,
`INVALID_CALLBACK_URL_HOST`.

All three are permanent until a human edits configuration.
`internal_error(code, raw)`:

| the 500's body carries          | result                                          |
| ------------------------------- | ----------------------------------------------- |
| one of the three codes          | `ProviderError::Config` — stops the poll ladder |
| any other code                  | `ProviderError::Transport`                      |
| no JSON, or JSON with no `code` | `ProviderError::Transport`                      |

`INTERNAL_PROCESSING_ERROR` can mean the wallet platform is down **or** that
the payer had no funds. The ambiguity resolves to the rail's side, because
reporting it as a decline would close a charge that may still be alive,
whereas reporting it as transport leaves the status query to settle it.

Never a decline on a 500: a payer must not be told they were refused because
someone's gateway fell over.

## 3. Everything else on a 500 — the rail, not us

A transport failure the poll ladder retries. No table needed.

## The two lists that record how far the table is evidenced

Both are `pub`, and both are data rather than prose, because the useful
question — "is this mapping backed by the rail's documentation or by ours?" —
is one an operator asks about a _specific_ string while reading a
`failure_raw`, and a list they can grep answers it.

### `UNPUBLISHED_REASONS`

`COULD_NOT_PERFORM_TRANSACTION` and `SENDER_ACCOUNT_NOT_ACTIVE`. Mapped rows
that MTN's published `ErrorReason` enum does **not** contain. They rest on
`docs/flows/adapter-mtn-momo.md` alone, which is itself a reconstruction.
They are kept because removing them would turn two mapped declines back into
`provider_error`.

Removing a row from **this** list is a claim that MTN has published it.
Removing it from `FAILURE_REASONS` turns a mapped decline into
`provider_error`. Different edits, different meanings.

### `UNMAPPED_REASONS`

Seven published codes that stay `FailureCode::ProviderError` **on purpose**,
so a reviewer diffing MTN's enum against the table can see the difference is
accounted for rather than missed:

| code                                                                              | why it is not mapped                                                                                                                                                                                                                                                                                                                       |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `INTERNAL_PROCESSING_ERROR`                                                       | the rail could not say what happened; as a terminal `FAILED` reason there is nothing left to retry and nothing to tell a payer                                                                                                                                                                                                             |
| `RESOURCE_NOT_FOUND`                                                              | about the _reference_, not the payment — the status query answers this as HTTP 404, which is `ChargeStatus::NotFound`                                                                                                                                                                                                                      |
| `RESOURCE_ALREADY_EXIST`                                                          | the duplicate-reference answer, handled as HTTP 409 → `Submitted`; as a `FAILED` reason it would be MTN contradicting itself                                                                                                                                                                                                               |
| `TRANSACTION_CANCELED`                                                            | **the one genuinely open row.** `payer_declined` would read well, but MTN publishes no description saying _who_ cancels, and the Collection API's only cancel operations are `CancelInvoice` and `CancelPreApproval`, neither of which vpay calls. Guessing "the payer" would put a sentence in front of a buyer on the strength of a verb |
| `INVALID_CURRENCY`, `NOT_ALLOWED_TARGET_ENVIRONMENT`, `INVALID_CALLBACK_URL_HOST` | `CONFIGURATION_CODES`, ours to fix; inventing a payer-facing code for our own misconfiguration would blame the wrong party                                                                                                                                                                                                                 |

## `PRODUCED_FAILURE_CODES` — the declared vocabulary

`pub const PRODUCED_FAILURE_CODES: [FailureCode; 11]` — every code this
adapter can put on a charge, ordered as `vpay_core::FailureCode::ALL` is so
the two can be diffed by eye. **MTN reaches all eleven.**

It cannot simply be derived from `FAILURE_REASONS`, and that is why it is its
own constant: two of the eleven are not table rows.

- `ProviderAccountBlocked` also arrives from HTTP `401`/`403`
  (`rail_credentials_refused`, `token::mint`), with no `reason` string
  anywhere.
- `ProviderError` is `failure_code()`'s fallback, so it is reachable by
  construction and can never be absent.

The conformance suite reads this constant from the adapter
(`declared_failure_codes`) rather than keeping a copy, and
`the_declines_prove_every_code_each_rail_can_produce` holds the declared list
and the stubbed decline rows to each other. It exists because the shop's
buyer copy, the demo's test-number panel and the SDK docs each promise
outcomes per rail — and until issue #59 they promised `payer_declined`, which
**no adapter produced**.

## Where the reason strings come from — and the correction

Until 2026-09-10 the table's nine rows were transcribed from
`docs/flows/adapter-mtn-momo.md`, itself a reconstruction. They are now
checked against MTN's **published** vocabulary: the `ErrorReason.code` enum
in the Collection API's OpenAPI components document, retrieved 2026-09-10
from the portal's unauthenticated schema endpoint. It enumerates **seventeen**
codes, snapshotted verbatim as `PUBLISHED_ERROR_REASONS` in that module's
tests.

Spelled out rather than fetched: a test that reached the network would fail
on a train, and the point of the list is to be a **snapshot someone
re-retrieves deliberately**.

**A correction worth knowing about, because it runs in both directions.**
The module header said, until 2026-09-11, that `ErrorReason` was "the whole
Collection API's error schema, not `requesttopay`'s", that MTN "publishes no
per-operation subset", and that every new row was therefore a deliberate
assumption. **The document says otherwise**: `RequestToPayResult.reason` is
`$ref: "#/components/schemas/ErrorReason"` in the same document, and
`RequesttoPayTransactionStatus` answers `RequestToPayResult` on its `200`. So
these rows are a _citation_ about `requesttopay`, not an inference from a
neighbouring operation. That is the difference between `PAYMENT_NOT_APPROVED`
being evidence and being a guess a maintainer would be right to revert — and
it sharpens `UNPUBLISHED_REASONS` from "absent from a big shared schema" to
"absent from the enum MTN types this exact field with".

**What a schema cannot say is that a value ever _arrives_.** As of
2026-09-15, no `FAILED` body from MTN has ever been observed by this
repository; the 2026-09-15 sandbox run settled.

## If you add a rail, copy this shape

The full comparison of the table against MTN's enum is in
`docs/plans/exp48-failure-codes-notes/opus.md`. The shape to copy for a new
adapter: a mapping table, a `pub PRODUCED_FAILURE_CODES`, a snapshot of the
rail's published vocabulary as a test const, and an assertion that every
published code is **either mapped or deliberately unmapped**. A code that is
neither now fails the build.
