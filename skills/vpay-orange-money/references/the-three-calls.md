# The three wire calls, in full

_Verified against vpay `93c6dfd0` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`submit`, `query_status` and `parse_callback` are implemented; `refund` and
`account_holder_name` are the port's `Unsupported` default. Everything below
is proven against a WireMock container **only** — see
[unverified.md](unverified.md).

Every call goes through `vpay_provider::http`: redirects are returned rather
than followed, proxies are ignored, bodies are capped at 256 KiB, and
`.timeout(config.request_timeout)` is attached **per request**. It was not,
until the Step 3 security review: MTN applied the deadline per request and
Orange silently did not, so a black-holed Orange host held a worker task for
as long as the shared client allowed.

## `submit` — non-2xx

`mapping::submitted(&body)` handles the success path. The failure ladder is
`submit_error(status, body)`, and the order of its branches is the design:

| condition     | result                                          | why                                                                                                                                                                                                                                                                   |
| ------------- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `401` / `403` | `Rejected{ProviderAccountBlocked}`              | our partner credentials, refused even after a fresh token. This pages                                                                                                                                                                                                 |
| `5xx`         | `Transport`                                     | the charge's fate is unknown                                                                                                                                                                                                                                          |
| `404`         | `Config`                                        | the payment API is not where the configuration says it is. **A charge must not be declined for that**                                                                                                                                                                 |
| `3xx`         | `Malformed`                                     | checked **before** the catch-all, because the catch-all is a _decline_ and a 3xx is not one — nobody refused the payment, the rail pointed somewhere else. A 307/308 would have replayed this request body, `merchant_key` and all, at whatever host `Location` named |
| anything else | `Rejected{ProviderError}` carrying the raw body | Orange documents no error vocabulary, so this is the "unmapped, alert on it" bucket of `docs/flows/failures.md` — **not a settled mapping**                                                                                                                           |

`rail_reason(body)` bounds what reaches a message at
`MAX_RAIL_REASON_CHARS` (256): enough for an operator to recognise Orange's
own wording, short enough that an HTML error page does not put a screenful
into every log line.

## `query_status` — the full ladder

`POST {base_url}/v1/transactionstatus`, body `{order_id, amount, pay_token}`.

Before any of the below: a `ref_extra` with no non-blank `pay_token` is
`ProviderError::Config` and **never** `ChargeStatus::NotFound`. See the
SKILL.md.

| condition     | result                                                                                                                  |
| ------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `404`         | `ChargeStatus::NotFound` — **checked before `is_success`**; "I have no record" is the canonical answer, never a failure |
| 2xx           | `mapping::charge_status(&body)` — the status table below                                                                |
| `401` / `403` | `Rejected{ProviderAccountBlocked}`                                                                                      |
| `5xx`         | `Transport`                                                                                                             |
| `3xx`         | `Malformed` — taking the hop would have replayed the bearer and the `pay_token` at whatever host `Location` named       |
| any other 4xx | **`Config`**                                                                                                            |

The last row is the one worth understanding: a 4xx on a _read_ is our request
being wrong, not the charge being declined. **A decline arrives as a 200 with
a status string.** Classifying a malformed request as a decline would fail a
live charge.

## `STATUS_TABLE`

| Orange status | `Meaning`                                    |
| ------------- | -------------------------------------------- |
| `INITIATED`   | `Pending`                                    |
| `PENDING`     | `Pending`                                    |
| `SUCCESS`     | `Succeeded` (+ `txnid` as `provider_txn_id`) |
| `EXPIRED`     | `Failed(payer_timeout)`                      |
| `FAILED`      | `Failed(provider_error)`                     |

`INITIATED` is the row most likely to be transcribed wrong, and a careless
transcription turns it into a failure. It is `Pending`.

`FAILED` maps to `provider_error` because **Orange documents no sub-reason**
for it, under any field name. The status string itself is the whole of the
rail's own word, and it is carried into `ChargeStatus::Failed { raw }` (with
the body's optional `message` appended when present, giving e.g.
`"FAILED: debit refused"`). `message` is not in the flow doc's example and is
accepted anyway, because an unmapped `FAILED` must carry the rail's own words
to an operator and a field serde ignores costs nothing.

**An unrecognised status string is `ProviderError::Malformed`, never
guessed.** Losing a settled payment to a renamed status is the failure that
matters here, and `SUCCESS` has not changed what it means.

## `PRODUCED_FAILURE_CODES` — three of eleven

`payer_timeout` (the `EXPIRED` row), `provider_error` (the `FAILED` row) and
`provider_account_blocked` (which is **not** a table row — it arrives from
HTTP 401/403 on the token endpoint, on a submit and on a status query).

`NON_TABLE_FAILURE_CODES` names that third path in code rather than in a
comment, so the sum has to keep adding up.

The other **eight** codes in the core taxonomy are unreachable on this rail,
and that is pinned rather than merely true:
`the_codes_orange_cannot_express_are_unreachable_and_not_merely_unmapped`.
The shop's buyer copy has a `cannotExpress` list that this constant feeds. If
Orange turns out to publish `FAILED` sub-reasons, they become rows in
`STATUS_TABLE`, `PRODUCED_FAILURE_CODES` grows, and the `cannotExpress` list
shrinks — all three checked against each other.

For contrast: MTN reaches all eleven, as of 2026-09-10. The `payer_declined`
this rail cannot produce is produced by MTN, which is what makes the gap
worth stating rather than a property of the whole system.

## `wire.rs` — why nothing in it derives `Debug`

Every type in that module carries either a credential (`merchant_key`) or
rail key material (`pay_token`, `notif_token`, `access_token`). A derived
`Debug` is how those reach a log line: one `tracing::debug!(?body)` added
later, and a token that gates a payer's redirect is in the log stream.

`pub(crate)` types are exempt from `missing_debug_implementations`, so the
lint does not push back — **the omission is deliberate, not an oversight.**
Do not add a derive there.

The same module rule as MTN's applies: **never
`#[serde(rename_all = "snake_case")]` in `wire.rs`.** Orange happens to spell
its fields in snake_case today, which is exactly what makes the attribute
dangerous here rather than harmless — it would read as a promise that these
names are ours to normalise, and the day Orange sends one that is not
snake_case the attribute would rename it away from the rail's own spelling.

## `parse_callback` — what lands in `ref_extra`

| field         | required?                         | goes into `ref_extra`?                      |
| ------------- | --------------------------------- | ------------------------------------------- |
| `order_id`    | **yes**, and must parse as a UUID | no — it becomes `CallbackRef::reference_id` |
| `notif_token` | **yes**, non-blank                | yes                                         |
| `pay_token`   | no                                | yes, when present and non-blank             |
| `status`      | —                                 | **there is no field for it**                |

`pay_token` is carried through so a callback _could_ repair a charge whose
`ref_extra` write was lost. It does not, today:
`vpay_api::provider_callback` discards the whole `ref_extra`. Whether Orange
sends it at all is item 1 of the flow doc's "To confirm" list, which is why
it is carried when present and never required.

## What the unit tests cover, and what they cannot

As of 2026-09-03, 57 unit tests in the crate, 57 passed, 0 skipped
(`cargo nextest run -p vpay-adapter-orange-money`). They cover the **pure**
halves: token-URL derivation, the status table, the request body's shape
(`amount` as a JSON number), callback parsing, `ref_extra`'s shape,
payment-URL validation.

The wire behaviour is covered by `backends/tests/conformance` against a real
`wiremock/wiremock` container. Neither proves anything about Orange.
