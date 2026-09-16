# What is still unverified about Orange, and what each item blocks

_Verified against vpay `f063ee96` (2026-09-15). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

**As of 2026-09-16, no request from this repository has ever reached Orange
Money.** This page is the list of assumptions that sit underneath
`vpay-adapter-orange-money`, taken from
`docs/flows/adapter-orange-money.md` § "To confirm" and § "Still unverified
against the real rail". Read it before you change anything in that crate, and
**date anything you resolve**.

Nothing here is a bug report. Each item is a question with a stated default,
and the default is written into the code so the uncertainty is visible.

## The assumptions in the wire path

| #   | Assumption                                                                                                                                                                                                                                         | What it blocks / costs if wrong                                                                                                                                                                                                                                                                  |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | **The error-body vocabulary for `webpayment`.** Orange documents none, so a 4xx that is not a 401/404 becomes `Rejected{provider_error}` carrying the raw body                                                                                     | Every such failure lands in the "unmapped, alert on it" bucket of `docs/flows/failures.md`. It is **not a settled mapping**                                                                                                                                                                      |
| 2   | **The sub-reasons of `FAILED`.** Same bucket, same reason — the status string is the whole of the rail's word                                                                                                                                      | Decides whether the eight codes under "what this rail cannot say" stay unreachable. If sub-reasons exist they become `STATUS_TABLE` rows, `PRODUCED_FAILURE_CODES` grows, and the shop's `cannotExpress` list shrinks                                                                            |
| 3   | **Whether a repeated `order_id` really is idempotent.** The stub returns the same `pay_token`, and the port _requires_ a duplicate to be `Submitted` rather than an error                                                                          | If Orange charges twice for a repeat, crash recovery is a double charge. This is an assumption about Orange, not an observation                                                                                                                                                                  |
| 4   | **Whether the notification carries `pay_token`.** The adapter carries it through _when present_ and never requires it                                                                                                                              | `ref_extra` repair from a callback — already unavailable for a second reason (below)                                                                                                                                                                                                             |
| 5   | **The 401 → re-mint → retry path is unproven.** No mapping returns 401 from `webpayment` or `transactionstatus` _after_ a good token; only the 401 on the token endpoint itself is covered (`bad_credentials_are_not_reported_as_a_payer_problem`) | A token expiring in flight is the commonest real-world 401, and nothing exercises it                                                                                                                                                                                                             |
| 6   | **`lang` defaults to `fr`** when a deployment configures none                                                                                                                                                                                      | The one defaulted field in the request body; wrong wording on Orange's page, nothing worse                                                                                                                                                                                                       |
| 7   | Transaction and daily limits for XAF                                                                                                                                                                                                               | Unknown failure modes at volume                                                                                                                                                                                                                                                                  |
| 8   | **Whether Orange exposes an account-holder lookup at all**, and under which product and credential (issue #47). Orange has a KYC/customer product; **its route is not confirmed from this repository and is not being claimed**                    | While unknown, `supports_account_holder_lookup: false` and the port's `Unsupported` is correct. If the answer is "yes, here", the flag becomes `true`, the adapter overrides `account_holder_name` with its **own `NotImplemented` token** until it is written, and `docs/status.md` grows a row |
| 9   | **How long the real hosted page gives a payer**, and what `transactionstatus` answers while they are on it                                                                                                                                         | Nothing is blocked: `INITIATED` and `PENDING` both map to `Pending` and the poll ladder is indifferent to how many rungs it spends there. It is written down because the _stub_ now has an answer and a reader must not mistake it for a measurement of Orange                                   |
| 10  | **Whether `FAILED` carries a sub-reason under any field name**                                                                                                                                                                                     | Item 2 restated from the other side. MTN's equivalent question has a published answer — a seventeen-value `ErrorReason.code` enum — and comparing the adapter against it is what closed `payer_declined` on that rail                                                                            |

## `notif_token` verification is unbuilt — in two places

The adapter returns the received `notif_token` in `ref_extra` and **fails
closed** when there is none. That is the only thing it can do: it holds no
state, so it cannot compare.

Since Step 8 (2026-09-04) **the callback route does not compare it either**.
`vpay_api::provider_callback` **discards the returned `ref_extra`** rather
than merging unverified rail material onto the charge.

`docs/flows/adapter-orange-money.md` carries this as a struck-through
sentence — it said the comparison "is the callback route's job, and that
route is not built yet", which was true until Step 8 and is wrong in a second
way now. **The current text on that page wins**: the route exists and
discards the material.

Consequence, and it is the reason this is on this page rather than in a
backlog: the adapter's fail-closed check is **load-bearing in production, not
only in tests**. It is the only thing between an unauthenticated POST and a
queued poll.

## The stub's hosted page is not Orange's

`backends/tests/conformance/wiremock/orange/mappings/stub-hosted-page.json`
serves `/stub-hosted-page/{pay_token}` with a Pay link and a Cancel link so a
browser can finish the redirect leg — and since Step 9 a browser does
(`shop-hosted.cy.ts`).

**The real rail stores `return_url` and `cancel_url` against the `pay_token`
at submit and renders them from its own state.** The stub does not.

### The payer window (2026-09-10, issue #58)

The stub gives **one unconditional `PENDING`, four more while a payer is on
the page, then `EXPIRED`.** Four conformance cases cover that chain:

- `a_charge_no_payer_has_looked_at_is_pending_once_and_then_settles`
- `the_hosted_pages_pending_chain_is_bounded_and_ends_in_an_expiry`
- `the_payers_exit_from_the_hosted_page_decides_the_charge` — two cases:
  `#pay` → `SUCCESS`, `#cancel` → `EXPIRED`, each **following the link the
  page rendered** rather than one the test built, so a link pointed back at
  the merchant fails them.

**These are cases about the stub, not about this adapter.** The
`Pending`/`Succeeded`/`Failed(payer_timeout)` mapping did not change a line
when they landed. They exist because the stub's _old_ timing made the demo's
Orange test numbers unreachable from a browser, which was a false green on
the shop's own checkout panel.

How many seconds the chain is worth against the worker's own ladder is
`the_pending_chain_gives_a_payer_at_least_thirty_seconds` in
`backends/tests/integration`, which reads the mapping file and
`vpay_worker::poll_delay` and multiplies.

If the real page's window turns out to be minutes rather than seconds, **the
thing that changes is the escalation in `docs/flows/reconciler.md`, not this
adapter.**

## What _is_ proven, and by what

Worth knowing so you do not re-prove it, and so you do not over-claim it.

**Unit tests** (57 in the crate, 57 passed, 0 skipped, measured 2026-09-03):
token-URL derivation, the status table, the request body's shape (`amount` as
a JSON number), callback parsing, `ref_extra`'s shape, payment-URL
validation. All pure — no socket.

**Conformance, against a real `wiremock/wiremock` container** (all 13 port
cases pass for this rail; 51 tests in the suite as measured 2026-09-10, 54 as
of 2026-09-15): redirects returned rather than followed
(`redirects_are_refused_and_never_followed`), bodies capped
(`an_oversized_rail_body_is_refused_at_the_cap`,
`a_long_rail_body_is_bounded_before_it_reaches_a_log_line`), per-request
deadlines, payment-URL validation
(`a_payment_url_that_is_not_an_http_url_is_refused`,
`a_payment_url_over_the_column_limit_is_refused`,
`refusing_a_payment_url_does_not_quote_it`), and the bearer cache
(`rotating_only_the_secret_evicts_the_cached_bearer`,
`a_field_boundary_cannot_be_shifted_into_a_collision`,
`the_lifetime_is_measured_from_the_send_not_from_the_answer`).

**One correction worth carrying**, because it shows what the suite's own
seeding can hide: an earlier draft of that flow doc said five of nine
conformance cases passed and four failed on `query_status` for want of a
`pay_token` in the suite's `ChargeRef`. That was fixed **in the suite, where
it belonged** — a `ProviderFlow::Redirect` rail is now seeded with the
`pay_token` its previous `submit` returned, mirroring how a `Push` rail is
seeded with `payer_ref`. The adapter's behaviour did not change: a
`query_status` with no `pay_token` is still `ProviderError::Config` and never
`NotFound`.

## The honest summary to put in a commit message

> Orange's redirect rail has never been called. This change is proven against
> a WireMock host answering the way a reconstructed flow doc says Orange
> answers.
