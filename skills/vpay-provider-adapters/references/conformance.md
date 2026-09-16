# The conformance suite

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`backends/tests/conformance/` — package `vpay-tests-conformance`, one test
file (`tests/adapter_conformance.rs`, ~3 250 lines as of 2026-09-16 — it was
~2 400 before the refund family landed) and one mappings directory per rail
under `wiremock/`.

## The rule

> ONE suite, parameterised over every adapter. **Adding a rail means making
> this pass — not writing a new suite.** That is the real test of whether the
> provider port is a port or just a folder.

The wire-level cases were written **before** the adapters, deliberately: this
file was the specification the MTN and Orange implementers coded against.

Nothing in it is `#[ignore]`d. `just verify-ignored` holds the count at
**zero** for this suite, so an ignored test here would be hiding a regression
rather than declaring an absence. **67 tests, 67 passed, 0 skipped as of
2026-09-16** (`docs/status/verification/2026-09-16-w3-merge.md`); it was 54 on
2026-09-15, before the refund-destination and refund-wire families.

## No `if rail == …` in a test body

Each case runs once per rail, against that rail's own stub container.
Differences are either capability values (`flow`, `supports_refunds`) or
table data. From the file's own header:

> If a case ever needs `if rail == MtnMomo` to pass, **the port has leaked
> and that is the finding, not the fix.**

The rail's name appears in **eight** table functions — seven until
`refund_body_pattern` joined them on 2026-09-15 — `mappings_dir`, `start`,
`documented_declines`, `declared_failure_codes`, `documented_callback_body`,
`callback_url_pattern`, `refund_body_pattern`, `return_url_pattern`; plus the
two places that construct **every** adapter rather than selecting one,
`adapters()` and `a_required_rail_parses_its_own_destination`.
`declared_failure_codes` reads the adapter's own `PRODUCED_FAILURE_CODES` —
the suite keeping its own copy of that list would be the failure mode the
list exists against.

## The stub is a container, never an in-process double

ADR-0006: a rail stub is a WireMock **host in configuration**, reached over
HTTP exactly as a real rail is. `vpay_testkit::containers::start_wiremock`
starts `wiremock/wiremock` from the **same** `wiremock/{rail}/mappings`
directory `compose.yml` bind-mounts, so a mapping fixed for one is fixed for
both.

**The Rust `wiremock` crate must never appear in this package's manifest.**
It is an in-process double that would replace the very transport these cases
exist to exercise. `cargo xtask verify-no-mocks` cannot police a
dev-dependency — the whole package is test-only — so the guard is a comment
in `Cargo.toml` plus the fact that nothing there would compile against it.

## The mapping contract a new rail must satisfy

Every case steers by **reference**: MTN matches the `X-Reference-Id` header,
Orange matches the body's `order_id` (which is `ChargeRef::reference_id`
rendered). So both rails stub the **same UUIDs**, and adding a rail means
adding one mappings directory rather than editing the test file.

| constant          | UUID (low bits) | what your mappings must do                                                                                                                                                                                                                                                                                             |
| ----------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `REF_ACCEPTED`    | `…0202`         | accept the submit (MTN 202; Orange 201 with a `pay_token` + `payment_url`)                                                                                                                                                                                                                                             |
| `REF_DUPLICATE`   | `…00dd`         | report the reference as already existing, **twice**, naming the **same payment** both times. A push rail with no key material at all; a redirect rail with the token it already minted. A redirect rail's pair must be a WireMock **scenario**, so the second answer is served by a mapping that _could_ have differed |
| `REF_UNKNOWN`     | `…0404`         | answer the status query `404`                                                                                                                                                                                                                                                                                          |
| `REF_UNAVAILABLE` | `…05aa`         | answer the status query `503`                                                                                                                                                                                                                                                                                          |
| `REF_SLOW`        | `…0560`         | answer the status query after a `fixedDelayMilliseconds` well above `SHORT_REQUEST_TIMEOUT` (100 ms)                                                                                                                                                                                                                   |
| `REF_SCENARIO`    | `…0ce0`         | a WireMock scenario: first status query `PENDING`, second `SUCCESSFUL`                                                                                                                                                                                                                                                 |
| `REF_REDIRECT`    | `…0302`         | answer the **submit** `307` with `Location: REDIRECT_TARGET`, **and** stub that path with the rail's _accepted_ answer                                                                                                                                                                                                 |
| `REF_HUGE`        | `…0b16`         | answer the status query `200` with a valid body padded past `MAX_RAIL_BODY_BYTES` (256 KiB)                                                                                                                                                                                                                            |
| decline block     | `…0f01…`        | one reference per documented failure reason in your `mapping.rs`                                                                                                                                                                                                                                                       |

Plus one `ProviderConfig` whose credentials are wrong (`Credentials::Rejected`
in `start`), and a mapping answering it `401`, to prove a credential failure
is not reported as a payer's problem.

`REF_REDIRECT`'s target is stubbed with the _accepted_ answer on purpose: a
followed redirect then fails `redirects_are_refused_and_never_followed`
twice over — once because `submit` would return `Ok`, and once because the
request would appear in the stub's journal.

## The refund family — new on 2026-09-15, and what it does not prove

Five wire cases plus two container-free ones, all steered by **the refund's
own reference**, never the charge's:

| constant                    | value           | what your mappings must do                                                    |
| --------------------------- | --------------- | ----------------------------------------------------------------------------- |
| `REF_REFUND_ACCEPTED`       | `…0d02`         | accept the transfer (MTN: `202` with an empty body)                           |
| `REFUND_PAYEE`              | `+237600000200` | the payee, as a merchant sends it — the registered-holder number reused       |
| `REFUND_PAYEE_CANONICAL`    | `237600000200`  | the same payee in the shape `RefundTarget::mobile_money` hands the rail       |
| `REFUND_PAYEE_UNREGISTERED` | `+237600000404` | a payee the rail refuses — asserted a **decline**, never an accepted transfer |

`REF_REFUND_ACCEPTED` is deliberately **not** `REF_ACCEPTED`: the refund
path's reference is the refund's own, and a suite reusing the charge's would
model the very confusion that makes a second partial refund of one charge come
back `409 RESOURCE_ALREADY_EXIST` and be reported as _accepted_.

The payee is built by calling the adapter's **own** `parse_destination` (the
suite's `refund_destination()` helper), not by this file constructing a
`RefundTarget` — so every case exercises the wire key the adapter owns as well
as the transfer it produces.

**`claims_refunds` gates the wire cases on three states, not two.** It tested
only `supports_refunds` until 2026-09-15; both halves moved that day and left
a pair it could not express:

| Rail           | `supports_refunds` | `refund`                                 | the wire cases                    |
| -------------- | ------------------ | ---------------------------------------- | --------------------------------- |
| `mtn_momo`     | `true`             | written — MTN's Disbursements `transfer` | run                               |
| `orange_money` | `true`             | `NotImplemented("orange_money::refund")` | assert the token, then return     |
| _(none today)_ | `false`            | the port's default                       | assert `Unsupported`, then return |

The middle state is **not a silent skip**: a rail that declares the capability
and has not built the call owes a `NotImplemented` token naming _itself_, and
that is asserted through the same container the wire cases use, so the
`orange_money` parameterisation of every refund case still proves something.
Gating on the capability alone made
`a_duplicate_refund_reference_is_accepted_and_never_paid_twice` fail on Orange
the moment the two branches met.

**What `a_refund_on_a_rail_that_refunds_reaches_the_rail_and_is_accepted`
proves, and what it does not.** MTN's stub answers `202` **only** for a
request carrying the bearer minted from `/disbursement/token/` _and_ the
per-product `Ocp-Apim-Subscription-Key`; anything else — the Collections
bearer already in the adapter's cache, the Collections key, a plural path
segment — matches no mapping and becomes `ProviderError::Config`. The `submit`
before it is not decoration: it puts a Collections bearer in that cache first,
so the case fails if the refund reuses it. What none of it proves is that
**MTN behaves this way** — nothing in this repository has ever called MTN's
Disbursements product, in sandbox or anywhere else, and a stub faithful to the
documentation but not to the rail would pass every assertion.

Two cases run with **no container at all**, because parsing a merchant's
parameters is pure: `a_destination_is_offered_exactly_when_the_capability_demands_one`
pins the suite's own helper (measured 2026-09-15: with its `Required` arm
returning `None`, all 54 cases then in the suite still passed), and
`a_required_rail_parses_its_own_destination` asserts the property no adapter
can assert about itself — that a rail declaring `Required` has actually
**overridden** `parse_destination` rather than inheriting the port's
`Unsupported`, canonicalises the payee, and refuses `600000200`,
`not a phone number`, `+0600000200`, `+1234567` and an **empty sub-map** as
`Malformed` without any refusal naming the value it refused.

`no_shipping_rail_returns_a_refund_to_the_paying_instrument` is the premise
guard: both rails declare `RefundDestination::Required`, so every `Origin` arm
in the suite has no live example. Its failure message lists the three things
owed before it may be relaxed — an integration case sending `destination` to
an `Origin` rail through `POST /v1/refunds` and asserting the `400`, a case
that a refund **without** one succeeds there, and RFC-0003 § 1 updated.
**Do not simply widen that assertion.**

## The account-holder family — and the digits-only rule

These steer on a **payer reference** rather than a charge reference, because
`account_holder_name` takes no charge at all — it is a stateless read of a
number.

| constant              | value          | what your mappings must do                                                                 |
| --------------------- | -------------- | ------------------------------------------------------------------------------------------ |
| `MSISDN_REGISTERED`   | `237600000200` | name a holder (`David Mbarga`)                                                             |
| `MSISDN_UNREGISTERED` | `237600000404` | report no record — the rail's own "not found"                                              |
| `MSISDN_SLOW`         | `237600000560` | answer past `SHORT_REQUEST_TIMEOUT`                                                        |
| `MSISDN_HUGE`         | `237600000616` | answer with a valid body padded past the cap                                               |
| `MSISDN_FULL_OF_PII`  | `237600000700` | a **named** holder buried in as much other personal data as the rail could plausibly carry |

**Digits only, all of them, and that is not cosmetic.** `GET
/v1/account_holders` validates Cameroon E.164 server-side, so a hex-lettered
steering number of the kind `requesttopay.json` uses could never reach this
port method from the API. A suite that steered on one would be exercising a
path no caller has.

The refund family reuses two of these numbers deliberately, so one table
answers both questions — but spelled **with a leading `+`**, because
`RefundTarget::mobile_money` requires one where `GET /v1/account_holders` does
not. That asymmetry is the maintainer's decision of 2026-09-15, and the two
spellings are two constants on purpose: a test using one string for both would
not notice if the canonicalisation stopped happening.

**A rail declaring `supports_account_holder_lookup: false` stubs none of
them.** `claims_account_holder_lookup` asserts
`Err(ProviderError::Unsupported)` for it — not `NotImplemented`, and
certainly not `Ok(None)` — out of the same shared body, so "Orange has no
such API" is _checked_ rather than skipped. All five cases still run for that
rail.

The PII case asserts on the name **and on the two halves the rail sends it
in**. Asserting on the joined `"Amina Nkeng"` alone was a hole, found by
mutation on 2026-09-06: a `tracing::debug!(?parsed)` of the adapter's wire
type printed `given_name: Some("Amina"), family_name: Some("Nkeng")` and the
case passed. A leak is a leak of the halves.

## What a conformance charge stands for

`Rail::charge` builds a charge as it looks **after `submit`**, because that
is the only kind `query_status` ever sees in production. Selected by
capability, never by rail name:

- **Push** → `payer_ref: Some("237600000000")`, `ref_extra: {}` (a push rail
  is addressed by the reference we generated and carries no rail key
  material).
- **Redirect** → `payer_ref: None`, `ref_extra: {"pay_token": …}` (the rail
  will not answer for our reference alone).
- **`return_url: Some(..)` on both** — deliberately _not_ selected by flow.
  The core fills it for any charge whose merchant sent one and leaves it to
  the adapter to decide whether the rail has a use for it. A push rail's case
  would be vacuous if the field were `None`.

Seeding a redirect rail's `pay_token` keeps
`not_found_is_never_on_its_own_a_failure` a test of the rail's 404 rather
than an accidental test of our own missing-token branch. Only `pay_token` is
seeded — `notif_token` is deliberately absent, because no port method reads
one from a `ChargeRef`, and seeding a value nothing consumes is how a suite
starts describing a contract it does not check.

## Two techniques worth copying

### The request journal is the only witness for a request that should NOT happen

`requests_recorded_for` / `requests_matching` POST to the stub's own
`/__admin/requests/count`. Asserting on the adapter's return value alone
cannot distinguish "the redirect was refused" from "the redirect was followed
and the answer was then rejected for some other reason". Every case starts
its own container, so the journal is empty at the top of a test and needs no
reset. The count is dug out of the JSON by hand rather than with
`serde_json`, to keep this package's dev-dependency set minimal.

### The timeout lesson — build the client with production defaults

`start()` builds the `reqwest::Client` with `DEFAULT_CONNECT_TIMEOUT` and
`DEFAULT_REQUEST_TIMEOUT`, and puts the **short** deadline on the
`ProviderConfig` only. That is deliberate, and it was found the hard way:

> With the short deadline on the _client_, an adapter that ignored
> `ProviderConfig::request_timeout` entirely **still passed** — reqwest's
> client-level timeout fired for it, and the case proved nothing about the
> adapter. **Orange did ignore it**, on both its token call and every payment
> call, and this suite said it was fine.

A black-holed Orange host therefore held a worker task for as long as the
shared client allowed. With the deadline on the `ProviderConfig` alone, the
only thing that can make
`an_unavailable_rail_is_a_transport_error_never_a_decline` pass is the
adapter applying the configured deadline **per request**. Attach
`.timeout(config.request_timeout)` to every `RequestBuilder`, token mint
included.

## The case list

Capability-level (no rail, no container — these cover a rail added tomorrow
without anyone touching the file): `every_adapter_declares_coherent_capabilities`,
`adapter_codes_are_unique`, `unimplemented_operations_never_fabricate_success`,
`a_destination_is_offered_exactly_when_the_capability_demands_one`, and
`no_shipping_rail_returns_a_refund_to_the_paying_instrument` — the last two
added 2026-09-15 and 2026-09-16.
~~`refund_is_refused_when_the_capability_is_absent`~~ **no longer exists under
that name**; its subject is `a_rail_without_the_refund_capability_answers_unsupported`
below.

Per rail but container-free: `a_required_rail_parses_its_own_destination`
(2026-09-15).

Wire-level, once per rail: `the_submit_tells_the_rail_where_to_send_the_payer_back`,
`the_submit_tells_the_rail_where_to_call_back`,
`submit_returns_a_reference_and_a_flow_shaped_result`,
`duplicate_submit_reports_submitted_not_an_error`,
`not_found_is_never_on_its_own_a_failure`,
`a_declined_charge_maps_to_the_documented_failure_code`,
`the_declines_prove_every_code_each_rail_can_produce`,
`an_unavailable_rail_is_a_transport_error_never_a_decline`,
`bad_credentials_are_not_reported_as_a_payer_problem`,
`a_callback_body_round_trips_to_identifiers_only`,
`a_rail_without_the_refund_capability_answers_unsupported`,
`pending_then_successful_walks_the_scenario`,
`redirects_are_refused_and_never_followed`,
`an_oversized_rail_body_is_refused_at_the_cap`, the five refund wire cases
(`a_refund_on_a_rail_that_refunds_reaches_the_rail_and_is_accepted`,
`the_refund_is_addressed_to_the_payee_the_merchant_nominated`,
`a_refund_to_a_payee_the_rail_rejects_is_a_decline_and_never_an_accepted_transfer`,
`a_duplicate_refund_reference_is_accepted_and_never_paid_twice`,
`a_refund_never_puts_the_payees_number_in_a_log_line`), and the five
account-holder cases.

**`a_rail_without_the_refund_capability_answers_unsupported`'s name now
describes the arm that does not run**, and it is kept rather than renamed
because live pages and a dated verification record cite it. No rail declares
`supports_refunds: false` since 2026-09-15, so what runs is the other arm: a
rail advertising refunds must not answer `Unsupported`, and when it answers a
token the token must name **that** rail. The property the dead arm exercised —
that the port's `refund` default is `Unsupported` — moved to
`a_rail_with_no_refund_api_takes_the_default_and_answers_unsupported` in
`vpay-provider`, on a stub that overrides nothing. Keeping a case is not the
same as still proving it.

Orange-only, and **about the stub rather than about the adapter**: the four
hosted-page payer-window cases, plus
`a_test_number_typed_on_the_rails_hosted_page_reaches_the_documented_outcome`.
See the `vpay-orange-money` skill before reading anything into them.
