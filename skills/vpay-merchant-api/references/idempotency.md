# Idempotency

_Verified against vpay `b747e5d5` (2026-09-23). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Two halves: the header and fingerprint in `vpay_api::idempotency`, the storage
and comparison in `vpay_db::idempotency`, and the sequence that joins them in
`PostRequest` (`vpay_api::v1::payment_intents`).

## The header is required, not optional

Stripe treats `Idempotency-Key` as optional. **vpay requires it on every `POST`
under `/v1`** (Step 2 decision D7). A missing or blank one is:

```text
400  idempotency_error is NOT the type here — it is invalid_request_error
     code: invalid_request
     param: idempotency_key
     message: An Idempotency-Key header is required on every POST to /v1.
```

The rejected alternative was generating a fallback key server-side. That is
worthless: a key the server invents differs on every attempt, so a merchant
whose `POST /v1/payment_intents` times out and is retried creates two payments
and finds out from their customer. Refusing is a bug they find in development,
in one request. Both SDKs already send one on every `POST`, so a correct
integration pays nothing for this.

Validation (`IdempotencyKey::parse`):

- absent, or entirely whitespace → treated as absent (a key of spaces names
  nothing, and a client that sent one almost certainly interpolated an empty
  variable);
- longer than **255 bytes** → refused (bytes, not characters — a header value
  is bytes on the wire);
- any byte outside printable US-ASCII → refused. The key is a primary key in
  Postgres, a log field and a support-ticket quotation; restricting it here
  means none of those has to think about encoding.

`IdempotencyKey` is a newtype with a private field, so a handler cannot pass a
merchant id or a path segment where a key belongs.

**`DELETE` requires one too**, on all three paths that answer it:
`/v1/customers/{id}`, `/v1/invoices/{id}` and `/v1/invoice_items/{id}`. It
is the only verb on this API with no body that still carries a key. _(This
named only `DELETE /v1/customers/{id}` until 2026-09-23. The two invoice
`DELETE`s have been mounted since 2026-09-07.)_

## The storage key is `(merchant_id, idempotency_key)`

That is the `PRIMARY KEY` of `idempotency_keys` (migration `0015`). Two
merchants using the same string are already separate rows, which is why the
merchant id is **not** part of the fingerprint below.

## The fingerprint

`vpay_api::idempotency::request_hash(method, path, body) -> [u8; 32]` —
SHA-256, and two properties matter:

**Length-prefixed, not concatenated.** Each field's length goes in ahead of it
as a big-endian `u64`. Plain concatenation would hash `POST` + `/v1/payment_intents`
identically to `POST/v1` + `/payment_intents`, and a merchant could then replay
one endpoint's key on another and be handed the first endpoint's stored
response. Big-endian because the digest is written to a database and compared
on a later request, possibly by a different process on a different machine.

**Over the raw body, before decoding.** Two bodies differing only in
percent-escaping (`%2B` versus a literal `+`) really are different requests to
this API, and hashing the decoded form would let one masquerade as the other.
It also means the hash can be computed before anything is parsed, which is what
lets a replay be answered without the parse running at all.

## The sequence: `PostRequest`

Do not write a second copy of this. `PostRequest` is `pub(crate)` in
`vpay_api::v1::payment_intents` and **every `/v1` write already goes through
it** — `payment_intents::{create, confirm, cancel}`,
`checkout_sessions::{create, expire}`, `customers::{create, update, delete}`,
`invoices::{create, update, delete, finalize, void, mark_uncollectible, pay}`
(the first two transitions share one `transition` helper),
`invoice_items::{create, update, delete}`, and — since 2026-09-16 —
`refunds::{create, update, cancel}`. _(The refund writes were missing from
this list until 2026-09-23. Twenty `PostRequest::read` call sites as of that
date.)_

It exists because the fingerprint is taken over the **raw** body while the
handler needs the **parsed** body, and no two axum extractors can both consume
a body. `PostRequest::read` takes the whole request, extracts and validates the
key, reads the bytes once and hashes them; `PostRequest::form::<T>()` then
hands the _same bytes_ back to the ordinary `VpayForm` extractor, so there is
one decoder for the wire format rather than a second one written for this path.

```rust
let post = PostRequest::read(request).await?;

let claim_id = match post.claim_or_answer(repositories.as_ref(), &scope).await? {
    ClaimOutcome::Owned(claim_id) => claim_id,
    ClaimOutcome::Answered(response) => return Ok(response),
};

// From here the key is CLAIMED, so every path out of this function must end
// it. A validation failure is not a `?` — it releases first:
let validated = match validate(&post, &config).await {
    Ok(validated) => validated,
    Err(error) => {
        post.release(repositories.as_ref(), &scope, claim_id).await;
        return Err(error);
    }
};

// ... do the work, producing Result<Response, ApiError> ...
post.finish(repositories.as_ref(), &scope, claim_id, outcome, subject).await
```

**The `?` operator is the trap.** Between `claim_or_answer` and `finish`, a
bare `?` returns while the key is still claimed, and nothing else ever moves
that row — so every retry under it is answered "still in progress" until the
24-hour window closes. Use the `match` + `release` shape above for anything
that can fail before `finish` runs. `finish` itself releases on every one of
its own failure paths for the same reason.

`claim_or_answer` is one call rather than a `claim` plus a match at every call
site, because the `claim_id` **must not be droppable**: a handler that
destructured the fresh case and threw the id away would compile, and would then
have nothing to end the claim with — leaving the key stuck.

## The four outcomes

| `IdempotencyClaim` | What the caller gets                                                                                                    |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `Fresh`            | this process owns the key; do the work                                                                                  |
| `Replay(record)`   | the **stored response body, byte for byte**, with its original status **and its original `stripe-should-retry` header** |
| `InFlight`         | `400` `idempotency_error` / **`idempotency_key_in_flight`**, `Retry::AfterBackoff`                                      |
| `Mismatch`         | `400` `idempotency_error` / `idempotency_key_in_use`, `Retry::Never`                                                    |

`idempotency_key_in_flight` is its own code, not a shared `invalid_state`,
precisely so a merchant's client can tell "your own earlier call has not
finished" (fix: wait) from "you changed the body under this key" (fix: their
bug). An SDK's retry logic branches on that string.

**A replay re-emits the stored header, never re-derives it.** Migration `0025`
added `idempotency_keys.response_retry`, written from the rendered response's
own `HeaderMap`. Re-deriving from the stored status would make the replay
disagree with the response it replays — a stored `409` refusal would come back
advertising a retry that cannot succeed, and stripe-node applies a
"retry every 409" rule to exactly that.

## What is stored can be rewritten by an erasure

`Idempotency::store` takes a `vpay_db::ResponseSubject`, and **choosing it is
part of writing a `/v1` route**:

| Variant                            | For                                                                                                                            | Since                   |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ----------------------- |
| `Verbatim`                         | every body that cannot carry payer detail — intents, sessions, invoice items, ordinary invoices, refunds                       | —                       |
| `Customer { id }`                  | `/v1/customers` writes                                                                                                         | 2026-09-13 (issue #111) |
| `OutOfBandInvoice { customer_id }` | `POST /v1/invoices/{id}/pay` with `paid_out_of_band=true`, and `POST /v1/invoices/{id}` on an invoice already paid out of band | 2026-09-23 (vpay#251)   |

The last two store inside a transaction that takes the payer's row under a
share lock. If that payer has since been erased, the store rewrites the body
it just stored. A later erasure also rewrites stored bodies
(`redact_stored_responses_in_tx`, `redact_stored_invoice_responses_in_tx`).
So **"a replay returns the stored body byte for byte" holds only until the
payer is erased.** After that, the replay carries `[redacted]` where the
payer's detail was. A new route whose response can name the payer, but which
stores `Verbatim`, reintroduces the 24-hour erasure window that issue #111
closed. This section was missing from this page until 2026-09-23.

## What is stored, and what is released

**A `4xx` is stored** — Stripe's own behaviour. The merchant caused it,
re-running would produce it again, and answering the retry identically is
cheaper and less surprising than re-executing.

**A `5xx` is not stored, and the key is released.** Two wrong things were
possible here and the second is the one that actually happened:

- freezing the failure for 24 hours, so a merchant retrying after the
  deployment was fixed gets the old outage back; and
- leaving the key `in_flight` — which is what the code did before. Nothing else
  moves such a row, so _every_ retry under that key was answered "a request
  with this Idempotency-Key is still in progress" **for the life of the
  deployment**. Before the rails landed, when every `confirm` ended in the
  adapter's `501`, that permanently burned a key on every confirm a merchant
  made.

Every failure path _after_ the claim releases first, including the three steps
between "the work is done" and "the response is stored" (reading the body back,
parsing it as JSON, and the write itself). A failure to release is logged and
swallowed — the key expires in 24 hours either way, and turning a `501` into a
`500` would replace an accurate answer about the rail with an inaccurate one
about vpay.

## Safety does not rest on the key

Releasing means the retry re-executes, and that is safe **because the
idempotency key is not what stops a payment being taken twice**. The unique
index `one_charge_per_intent` on `charges (payment_intent_id)` is. A
re-executed confirm meets that index and answers `409` rather than charging
again.

Internalise this before you touch the mechanism: the key makes retries _quiet_;
the index makes them _safe_.

## Known gaps, as of 2026-09-23

- ~~**No scheduled sweep.** `idempotency_keys` rows expire logically after 24
  hours (`vpay_db::idempotency::claim` reclaims an expired row), but nothing
  deletes them. A long-lived deployment grows that table monotonically.~~
  **Corrected 2026-09-23:** this was wrong from the day it was written. Since
  Step 4 (2026-09-03), the worker's hourly housekeeping job,
  `vpay_worker::handlers::sweep_expired`, calls `Idempotency::sweep_expired`
  on every pass. So the table holds at most about an hour of expired keys
  past their 24-hour window. vpay's own resource contract made the same
  correction on 2026-09-23. **What is still a gap:** no test asserts that the
  job deletes an idempotency key. The job is exercised only by a checkout
  sessions test that asserts on sessions.
- **No idempotency anywhere but `/v1`.** `/v1/browser`'s confirm calls
  `confirm_once` directly and takes no key — consistent with the browser nest's
  CORS deliberately not allowing the `Idempotency-Key` header, because sending
  one turns a simple request into a preflighted one. `/dash/v1`, `/provider`
  and the `$procs` transport take none either.

## Where the docs are wrong

`docs/flows/merchant-auth/resource-contract.md`'s header table describes
`Idempotency-Key` as "caller-supplied, else a UUIDv4 generated per call". That
describes the **SDKs**, not the server. The server requires the header and has
no fallback. Code wins.
