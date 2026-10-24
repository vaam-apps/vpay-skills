---
name: vpay-provider-adapters
description: The vpay provider port (`vpay-provider`) and how to add a payment rail — the `ProviderAdapter` trait and its fourteen methods as of 2026-10-23 (payer_fields since 2026-09-16, accepted_exposure since 2026-10-20, prepare, default_callback_registration and verify_callback since 2026-10-23), the `Unsupported` vs `NotImplemented` rule that decides which error a missing operation returns, the rail-agnostic refund destination (`RefundDestination`, `RefundTarget`, `parse_destination`), the capability flags the core branches on instead of provider codes, the shared HTTP/token modules, the shared conformance suite a new adapter must pass, where per-provider state is written (`docs/status/providers.md`), the Draft designs for Stripe and PayPal adapters and for callback registration (RFC-0010, -0011, -0013), and the Notch Pay rail (RFC-0012), admitted under ADR-0031 and merged, called against Notch Pay's sandbox only (2026-10-21, step 5; load `vpay-notchpay` for it). Load this before writing or changing any `vpay-adapter-*` crate, before touching `backends/crates/vpay-provider`, and before adding a rail.
---

# The provider port, and adding a rail

> **Verified against vpay `a33aac61` (2026-09-29).** The two Notch Pay passages
> that say what step 5 did ("RFC-0012" in the RFC list, and "`backends/crates/vpay-notchpay`
> is the Notch Pay client") were also read against vpay `e28292d`
> (2026-10-21), the merge of vaam-apps/vpay#310. The RFC-0013 entry's
> sentence on what Notch Pay's sandbox stored for an API-created endpoint was
> read against vpay `82a68f4e` plus the pull request that adds
> `docs/status/verification/2026-10-23-notchpay-sandbox-two-step-and-channels.md`,
> restamped with its merge. The method count, the
> `default_callback_registration` bullet and the RFC-0013 entry's
> `callbacks.register` default were read against vpay's `master` at `9499f09d` plus the pull request that adds `docs/status/verification/2026-10-23-notchpay-defaults-and-wrong-network.md`, restamped with its merge. Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).
> What they say about Notch Pay's 2026-10-24 webhooks (the
> `default_callback_registration` bullet, the RFC-0012 and RFC-0013 entries
> and the RFC-0014 heading) was read against vpay `4df5e0d2` (2026-10-24), the
> merge of vaam-apps/vpay#364.

One trait, `vpay_provider::ProviderAdapter`, in
`backends/crates/vpay-provider/src/lib.rs`. The core owns the payment
lifecycle, ledger, reconciliation and failure taxonomy; an adapter owns
exactly one rail's wire protocol (ADR-0002).

**`if provider == "mtn_momo"` outside a `vpay-adapter-*` crate is a defect,
and nothing greps for it.** The structural guard is that the port is only
ever held as `Box<dyn ProviderAdapter>` — which is also why the trait is
`#[async_trait]` rather than using native `async fn`, since that is not
dyn-safe. The core branches on capability _values_, never on a code.

## The headline: `Unsupported` vs `NotImplemented`

Two ways for an operation to be absent, and they are not interchangeable.
Getting this wrong is the single most likely mistake in an adapter.

|                     | `ProviderError::Unsupported`                              | `ProviderError::NotImplemented("crate::fn")`                              |
| ------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------- |
| Means               | **The rail has no such API, permanently**                 | **Unbuilt work on our side**                                              |
| Whose fact          | the rail's                                                | ours                                                                      |
| Capability flag     | `false`                                                   | stays **`true`** — the rail can do it                                     |
| How to produce it   | do **not** override the trait method; inherit the default | **override** the method and return your own token                         |
| Category / severity | `Conflict` (409), severity overridden to `Error`          | `NotImplemented`                                                          |
| Seen by             | nothing — it is a settled answer                          | `cargo xtask verify-status`, which fails if it is not in `docs/status.md` |

`refund`, `parse_destination` and `account_holder_name` all default to
`Err(ProviderError::Unsupported)` in the trait.

**The trap:** a rail whose API _does_ exist but which you have not written
must **override the default with its own `NotImplemented` token**. Inheriting
`Unsupported` would hide our gap as a fact about the rail, and
`verify-status` would never see it.

~~`vpay_adapter_mtn_momo::Adapter::refund` is the live example, and
`vpay-adapter-orange-money` does the opposite: it does not override `refund`
at all, because Orange documents no refund API for Web Payment.~~
**Corrected 2026-09-16 — both halves of that moved on 2026-09-15 (RFC-0003
§ 5), and the example is now the other rail:**

| Rail           | `supports_refunds` | `refund` answers                                       |
| -------------- | ------------------ | ------------------------------------------------------ |
| `mtn_momo`     | `true`             | a real `POST /disbursement/v1_0/transfer`              |
| `orange_money` | `true`             | `NotImplemented("orange_money::refund")` — the example |

**Neither shipping rail answers `Unsupported` for `refund` any more**, and a
skill, doc or test that says one does is stale. Orange's flag flipped because
an Orange refund _is_ an outbound transfer: the rail refunds, so `Unsupported`
would be a lie about Orange, and what is missing is vpay's work — this
repository holds no Orange transfer specification. The reason changed, not
just the value.

`cargo xtask verify-status` therefore reported **exactly one** token from
2026-09-16 — Orange's — down from eight on 2026-09-03 and from **two** partway
through 2026-09-15. **It reports two again with the Notch Pay adapter
(2026-10-20, vaam-apps/vpay#295, `357c825`):
`notchpay::refund`**, a third rail that refunds and whose refund vpay has not
written (`vpay-notchpay` skill). `orange_money::refund` is the same string that left the
list on 2026-09-03 meaning something else entirely (then: Orange has no refund
API; now: vpay has not written the transfer), so do not read a token's
reappearance as a regression. The gate also gained a **third direction** on
2026-09-15 — a token whose prefix names a shipping rail must live in **that
rail's crate** — because the two older directions compare token _strings_ and
a copy-paste between adapters was invisible to both. See
[references/adding-a-rail.md](references/adding-a-rail.md) § 8.

`Unsupported` is a 409 for the merchant but `Severity::Error` for us:
reaching that arm means the core skipped the `Capabilities` check it is
supposed to branch on first, and logging it at `Conflict`'s default `Info`
would bury our own bug among merchants' typos.

**Where a provider's gaps are written down: `docs/status/providers.md`
(since 2026-10-20).** One matrix, a row per provider (`mtn_momo`,
`orange_money`, `notchpay`, `stripe`, `paypal` and `flutterwave` as of
2026-10-20 — the last has no RFC and no code, and its admission is `unknown`;
`fapshi` since 2026-10-23, also with no RFC and no code, from a desk study)
and a column per thing the port asks of an adapter. It writes a gap
by the rule above and by no other: a cell reads **unsupported** where the rail
cannot (`ProviderError::Unsupported`, the capability flag `false` beside it)
and **token** where the rail can and vpay has not
(`NotImplemented("rail::op")`, declared in `docs/status.md`). The rest of its
legend — `proven`, `stub-only`, `sandbox`, `client`, `designed`, `gated`,
`unknown` — says how far a real call is. It says no new error type is needed
and none should be added: a third "not supported" would be a second way of
saying one of these two. Read the row before you touch a rail, and move the
row in the same change that moves the rail. **No gate reads the matrix**:
`verify-status` reads the tokens in `docs/status.md` and nothing here, and the
page itself says deriving its capability columns from each adapter's
`Capabilities` is not built — so a stale cell fails nothing.

## The methods

**Fourteen as of 2026-10-23.** ~~"The ten methods"~~ was this heading until
then, "Twelve" until later the same day, and "Thirteen" until RFC-0014. Ten since the Notch Pay adapter
(2026-10-20, vaam-apps/vpay#295, on `master` since `357c825`, after this
page's stamp): `accepted_exposure`, below; eleven with
`ensure_callback_endpoint` (2026-10-22); twelve with `prepare` (2026-10-23,
ADR-0032, below); thirteen with `default_callback_registration` (2026-10-23,
RFC-0013's amendment, below); fourteen with `verify_callback` (2026-10-23,
RFC-0014, below). Everything else in this section was nine.

~~The seven methods~~ **Corrected 2026-09-23:** the trait has **nine** `fn`s
at `b747e5d5`. The code block below always listed eight, and
`payer_fields` became the ninth on 2026-09-16 (vpay#186). This page went on
saying "seven" through its `7997536b` stamp.
**`parse_destination` was added on 2026-09-15 (RFC-0003 open question 4) and
`refund` grew a `destination` parameter the same day.** An adapter written
against the shape before that does not compile against this port.

```rust
fn code(&self) -> &'static str;                   // == a rail code in payment_method_types
fn capabilities(&self) -> Capabilities;
async fn submit(&self, charge, config)      -> Result<Submitted, ProviderError>;
async fn query_status(&self, charge, config)-> Result<ChargeStatus, ProviderError>;
fn parse_callback(&self, body: &[u8])       -> Result<CallbackRef, ProviderError>;
fn parse_destination(&self, raw: &serde_json::Map<String, Value>)
                                            -> Result<RefundTarget, ProviderError>; // default: Unsupported
async fn refund(&self, charge, amount, destination: Option<&RefundTarget>, config)
                                            -> Result<Refunded, _>;   // default: Unsupported
async fn account_holder_name(&self, msisdn, config)-> Result<Option<AccountHolder>, _>; // default: Unsupported
fn payer_fields(&self) -> &'static [PayerField];  // default: &[] — since 2026-09-16
fn accepted_exposure(&self) -> Option<AcceptedExposure>; // default: None — since 2026-10-20
```

- **`accepted_exposure` is ADR-0031's boot warning, through the port**
  (2026-10-20). ADR-0031 amended ADR-0029 D1 rule 3. A status read
  authenticated by a key the provider calls **public** is admissible only if
  the adapter declares what that key lets anyone else do. `None` (the
  default) is right for every rail whose read takes a secret. `notchpay` is
  the only rail that declares one. `vpay_api::boot::warn_about_accepted_exposures`
  logs it at `WARN` for every _configured_ rail, in `serve` and `worker`.
  `Measured` must forward it, like `payer_fields`: a wrapper that took the
  default would silence the warning on every rail
  (`the_accepted_exposure_is_forwarded_not_defaulted`). An adapter admitted
  on ADR-0031's second limb that does not override it is the "way B done
  silently" both RFC-0012 and ADR-0031 refuse.

- **`payer_fields` is what the payer must type, and vpay enforces it
  generically** (since 2026-09-16, vpay#186). A push rail that collects an
  instrument declares its `PayerField`s as a `const`. MTN declares one:
  `msisdn`, `Phone { region: "CM", phone_type: Mobile }`, required. A redirect
  rail keeps the default `&[]`. The checkout's per-rail `RailSpec` renders the
  declaration. `vpay_api::v1::payer_fields` validates `confirm` against it,
  refuses any key the declaration does not name, and **checks the phone number
  for real** before the rail is asked. So a stub or test steering on a
  documentation MSISDN (`2376000000xx`) now gets vpay's own `400` on the
  confirm path and never reaches the mapping. MTN's `requesttopay.json`
  decline row moved to `237670000400` for that reason. The conformance suite calls adapters
  directly and is unaffected. **Its default is a trap for wrappers:**
  `Measured` must forward it. It silently answered `&[]` until that was caught
  against the live demo stack on 2026-09-16, and
  `payer_fields_is_forwarded_to_the_inner_adapter_and_not_defaulted` in
  `vpay_provider::measured` now
  pins it. Any new decorator over `dyn ProviderAdapter` must forward every
  defaulted method. ~~`docs/flows/provider-port.md` does not mention
  `payer_fields` at all.~~ **Corrected 2026-09-29:** since vaam-apps/vpay#255
  (2026-09-24) its interface table lists all nine methods, `payer_fields`
  included (it listed six, and said "three network methods" where there are
  four, until then). Its "Adding a rail" checklist still has **no step** for
  `payer_fields`, at `a33aac61`.

- **`ensure_callback_endpoint` is the eleventh method (2026-10-22, RFC-0013).**
  `async`, defaulting to `ProviderError::Unsupported`; called only on a rail
  whose `callback_url_mode` is `Registered`, by the worker's
  `ensure_callbacks` job and `vpay-server callbacks ensure`, under the
  registration lock. It takes `EndpointCheck { marker, write }` and the
  rail's `ProviderConfig`, and answers `EndpointReport { state, retirable }`,
  `state` one of `Present { endpoint_id, repaired }`, `Created`, `Drifted`,
  `Foreign`, `Missing`. **Four rules** (its doc comment): list the account's
  endpoints whole, every page; create at most once per call, and only when
  `write`; never modify or delete an endpoint without this deployment's
  marker; a create whose answer was lost is not repeated in the same call.
  `Measured` forwards and counts it (operation `ensure_callback_endpoint`).
  Only `notchpay` overrides it. _(This bullet said "RFC-0013 proposes a tenth
  method …; it does not exist" until 2026-10-22.)_

- **`prepare` is the twelfth method (2026-10-23, ADR-0032).** It is
  `async`, takes `(&ChargeRef, &ProviderConfig, PriorSubmits)`, answers
  `Result<RefExtra, ProviderError>`, and defaults to `ProviderError::Unsupported`; called only on a rail whose
  `submit_steps` is `PrepareThenSubmit`, before `submit`. It creates the
  payment on the rail **without prompting anyone** and returns the key
  material the rail is addressed by. The core writes a `prepare` row,
  calls it, commits the answer on the charge (`Charges::record_prepared`),
  and only then writes the `submit` row and calls `submit`, which on such a
  rail is the prompt alone. `PriorSubmits` is the core's evidence: `None`
  when no `submit` row exists for the charge (nothing ever asked the rail to
  prompt), `Some` when one does. `Measured` forwards and counts it
  (operation `prepare`). Only `notchpay` overrides it, as initialize. The
  error table gains a `prepare` column: `Rejected` there also covers a rail
  holding a payment no submit prompted for (`PriorSubmits::None`), and
  `NeedsReconciliation` a duplicate a submit may have prompted (`Some`).
  **Do not move the commit into the adapter**: an adapter that writes to the
  database is a core concern in a rail crate (ADR-0002).

- **`default_callback_registration` is the thirteenth method (2026-10-23,
  RFC-0013's amendment).** Synchronous, `-> CallbackRegistrationDefault`
  (`Ensure` | `Off`), defaulting to `Ensure`. It is what the registration
  pass does on a `Registered` rail when the deployment leaves
  `providers[].callbacks.register` unset; an explicit setting always wins.
  Only the binary's plan reads it (`vpay-server`'s `callback_plan`, through
  `From<CallbackRegistrationDefault> for vpay_worker::callbacks::Registration`),
  and it logs one `info` line for a rail it leaves `off`. **Only `notchpay`
  overrides it, with `Off`**: the maintainer's "Default to `off` for Notch
  Pay", after the sandbox delivered nothing to an API-created endpoint
  (2026-10-23). _(This said the sandbox "stored an API-created endpoint with
  `url: null`" until 2026-10-24: the listing shows `null` for a
  dashboard-registered endpoint that delivers too.)_ It is a method and not a `Capabilities` field
  for `accepted_exposure`'s reasons: nothing persists it and the core never
  branches on it. It exists so the binary does not match on `"notchpay"`,
  which ADR-0002 forbids outside an adapter crate. There is no `Check`
  default. `Measured` must forward it, like every defaulted method
  (`the_default_callback_registration_is_forwarded_not_defaulted`): a wrapper
  that took the default would put Notch Pay back on `ensure`.

- **The error-surface table in the trait's own doc comment is the contract.**
  It is a markdown grid, one row per `ProviderError` variant and one column
  per method. An adapter may raise fewer variants than a row allows; it may
  **not** raise a variant the table does not give it. There is deliberately
  no `ProviderError::retryable()` — retry policy is `Classify::retry` and a
  second oracle beside it is what ADR-0011 exists to prevent.
- **`parse_callback` is synchronous so that it cannot make a network call.**
  A callback is a hint; an adapter that could fetch something while
  "parsing" one could smuggle a status out of an unauthenticated request. It
  returns identifiers only — returning a status is a design error, and both
  shipping adapters' callback wire types have **no field** for the rail's
  `status` at all, so there is nowhere to put one. **`parse_destination` is
  synchronous for the same reason**, from 2026-09-15: it reads a merchant's
  parameters, and an adapter that could call a rail while "parsing" them
  would turn one refund request into an unbounded number of outbound calls.
  The two are the only pure functions in the error-surface table, and the
  blank `Config`/`Rejected`/`Transport` cells in their columns are as
  load-bearing as the ticks — there, those variants would not merely be
  unused, they would be untrue.
- **`submit` must report a duplicate as `Submitted`, never as an error.**
  That is what makes same-reference retry safe after a crash. On a two-step
  rail that is `submit`'s own duplicate (Notch Pay's "already processed",
  read and reported `Submitted`); `prepare`'s duplicate is the `409` above,
  decided by `PriorSubmits` (2026-10-23).
- **`query_status` must remain callable indefinitely**, long after any payer
  prompt expired. A rail with no record is `ChargeStatus::NotFound`, which is
  **not** an error and **not** a decline.
- Only `ProviderError::Rejected` says the money did not move. `Transport`,
  `Malformed` and `NeedsReconciliation` leave the charge's fate **unknown**,
  which is the distinction the whole port is arranged around.
- **`Malformed` versus `NeedsReconciliation`** (the seventh variant, since
  2026-10-23, vaam-apps/vpay#313). `Malformed` is an answer the adapter could
  not read, which a later read may fix; the poll ladder retries it at `Warn`.
  `NeedsReconciliation { context }` is an answer the adapter understood and
  refused for good, or lost key material it cannot ask without. Only a person
  can settle it, and the worker escalates the charge to `unresolved` on the
  first one. It gets code `charge_needs_reconciliation`, `Error`, and the
  API's `502`. Pick by "can asking again change it?", never by how bad it
  looks.

## Capabilities

`flow` (`Push` | `Redirect`), `supports_refunds`,
`supports_partial_refunds`, `delivers_callbacks`, `requires_ip_allowlist`,
`supports_account_holder_lookup`, `refund_destination: RefundDestination`
(`Origin` | `Required`, **new on 2026-09-15**), `submit_steps: SubmitSteps`
(`One` | `PrepareThenSubmit`, **new on 2026-10-23**, ADR-0032: whether a
charge is one call that prompts, or `prepare` then `submit` with the
rail-issued key material committed between them; `One` on `mtn_momo` and
`orange_money`, `PrepareThenSubmit` on `notchpay`, not persisted, no
coherence rule, and `prepare_is_unsupported_exactly_on_a_one_step_rail` holds
the method to it), and — **new on 2026-10-22**
(RFC-0013) — `callback_url_mode: CallbackUrlMode` (`PerRequest` |
`Registered`): whether the rail is told its callback URL on every submit
(`mtn_momo`'s `X-Callback-Url`, `orange_money`'s `notif_url`) or holds it
registered with the account (`notchpay`). A new adapter must set it; the
conformance case `the_submit_tells_the_rail_where_to_call_back` now asserts
its per-rail census against this value, and
`the_endpoint_check_is_unsupported_exactly_on_a_per_request_rail` holds the
method to it.

`Capabilities::is_coherent()` is `!supports_partial_refunds ||
supports_refunds`. It is **mirrored by a real DB CHECK** —
`partial_refunds_imply_refunds` in
`backends/migrations/0002_create-providers.sql`, proven to fire by
`partial_refunds_without_refunds_is_rejected_by_the_database` in
`backends/tests/integration/tests/postgres_smoke.rs`. Belt and braces, not a
substitute either way. `vpay_api::v1::boot::boot_seeds` also refuses an
incoherent pair at boot (`ConfigError::IncoherentCapabilities`, exit 78).

`Capabilities::callbacks_are_coherent()` is the second rule, kept **apart**
from `is_coherent` on purpose: `Registered` requires `delivers_callbacks`.
It has no CHECK, because `callback_url_mode` is not persisted; boot refuses
it as `ConfigError::IncoherentCallbackMode` (exit 78), and `vpay-server`'s
`every_registered_callback_url_belongs_to_a_rail_that_calls_back` asserts it
over every linked adapter. Do not fold it into `is_coherent`, whose rules are
exactly the ones a database CHECK mirrors.

`supports_account_holder_lookup` and `refund_destination` are **deliberately
not persisted**: unlike the other four they have no column in `providers` and
no field on `vpay_db::ProviderSeed`. Nothing reads a capability out of that
table — `vpay_api` resolves an adapter in-process and asks it — so a column
would be a second copy of an answer the linked code already owns.

**`refund_destination` is deliberately NOT in `is_coherent`**, and the
obvious rule ("a rail that cannot refund must declare `Origin`") was
considered and refused: neither value means anything when refunds are off, so
the rule would be an arbitrary sentinel called coherence, and there is no
`providers` column to mirror it with — a Rust-only "coherence" rule with no
database half would be a different kind of thing wearing that method's name.
`a_refund_destination_is_inert_to_coherence` pins the decision and what would
have to change to reverse it. Read the field **only when `supports_refunds`
is true**: on a rail that cannot refund the value is inert, no code path
reaches it, and it must still be a truthful statement about that rail's refund
product — because it is the value that governs the moment `supports_refunds`
flips, and Orange's flipping on 2026-09-15 is not hypothetical.

## The refund destination — `RefundDestination`, `RefundTarget`, `parse_destination`

New on 2026-09-15 (RFC-0003 § 1 and open question 4). A refund on a
mobile-money rail is an **outbound transfer**, so it needs a payee; a refund
on a card or wallet goes back to the instrument that paid. The port carries
that difference as a capability, never as a rail code.

| `refund_destination` | The core, before it calls `refund`                                                                                           | `destination` argument |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| `Required`           | refuses a request **without** a payee                                                                                        | always `Some`          |
| `Origin`             | refuses a request **carrying** one (accepting and dropping it would tell a merchant their nominated payee had been honoured) | always `None`          |

Both rails vpay carries declare `Required`, so the `Origin` arm ships with
**no shipping example**. Since 2026-10-17 (RFC-0010 § 7, delivery step 1) it
has an integration guard instead of a census: two cases in
`backends/tests/integration/tests/refunds.rs` drive `POST /v1/refunds` on an
`Origin` rail that exists only in that test binary —
`an_origin_rail_refuses_a_destination_through_the_route` (the `400`) and
`a_refund_without_a_destination_succeeds_on_an_origin_rail_and_names_the_payment`
(the accepting arm). The conformance case
`every_shipping_rails_refund_destination_agrees_with_its_parser` states the
rule — a `Required` rail overrides `parse_destination`, an `Origin` rail does
not — and no longer asserts that no `Origin` rail ships. _(Until 2026-10-17
that case was `no_shipping_rail_returns_a_refund_to_the_paying_instrument`,
whose message listed those two cases and an RFC-0003 § 1 amendment as the
price of relaxing it; all three were paid in the change that renamed it.)_

**An `Origin` rail is told which payment it unwinds.** The refund
`ChargeRef`'s `ref_extra` carries the charge's `provider_txn_id` under
`vpay_api::v1::refunds::REFUND_PROVIDER_TXN_ID_KEY` (the string
`provider_txn_id`), or nothing when the charge recorded none — the column is
nullable, and a `Required` rail never needs it. A `Required` rail ignores the
key; both shipping adapters do, and a `Required` adapter that reads it is a
bug. An `Origin` adapter reads it and answers `ProviderError::Config` when it
is missing — the port's settled answer for a broken invariant: never an
invented id, never a decline. Until 2026-10-17 `ref_extra` was empty on every
refund.

**The adapter parses the wire shape, not the core.** The merchant sends
`destination[<rail_code>][…]`; the core strips its own envelope — the outer
key is a `code()`, which is core knowledge by definition — and hands the
**rail-scoped inner map** to `parse_destination`. Symmetric with
`parse_callback`, and that symmetry is the whole point: a bank-account
upstream tomorrow is a new adapter method body and **zero core change**. Both
shipping adapters read the key `msisdn`.

**The adapter owns the key; the port owns the number.**
`RefundTarget::mobile_money` is **fallible and canonicalising** (maintainer's
decision, 2026-09-15 — it was infallible before), the field is private and
that constructor is the only way in, so an invalid destination cannot be
built and no adapter re-spells the rule. It **requires a leading `+`**,
unlike `GET /v1/account_holders`, which still takes the bare national form.
The full rules — the `+` asymmetry, why a refusal may never name the number,
and why `parse_destination`'s `Malformed` must be **translated** by its
caller rather than forwarded — are in
[references/adding-a-rail.md](references/adding-a-rail.md) § "A `Required`
rail".

**If the core's invariant is ever broken — a `Required` rail handed `None` —
answer `ProviderError::Config`.** Settled 2026-09-15 by
`vpay_adapter_mtn_momo::Adapter::refund`, the first adapter here to make a
real transfer call (RFC-0003 open question 6). Not `Rejected`, which blames a
rail nobody asked; not `Malformed`, which is about an answer that does not
exist; not `Unsupported` or `NotImplemented`, which on a rail that refunds
are both lies. `Config` is the closest true sentence the enum offers, and
what matters is its classification — it stops the poll ladder, it pages, and
it never reaches a payer as a decline.

## An `Ok` from `refund` is an acceptance. It is **not** a settlement

Read this before writing anything that turns a `Refunded` into a stored
status. `Refunded` has **no status field** and the trait has **no refund
status read** — there is no `query_refund_status` to pair with `query_status`.
So the strongest thing an adapter can mean by `Ok` is _the rail took the
instruction_, and on the one rail that implements it that is literally so:
MTN's Disbursements `transfer` answers `202 ACCEPTED` with an empty body and
its outcome is read back from
`GET /disbursement/v1_0/transfer/{referenceId}`, **a call vpay does not
make**.

A caller that writes `refunds.status = 'succeeded'` on an `Ok` is telling a
merchant money moved on the strength of a response that did not say so.
`pending` is what an `Ok` supports, and `pending` is what `POST /v1/refunds`
writes. RFC-0003 open question 8 stays open because closing it needs a refund
poll ladder that does not exist. **No rail has ever returned money to
anyone**, and `mtn_momo::refund` is WireMock-proven and rail-unproven: no real
MTN Disbursements credential exists in this project, so a deployment reaching
that call today gets `ProviderError::Config` naming the credential it lacks.

## What the port owns, and an adapter must not re-implement

- **`vpay_provider::http`** — `client()` / `client_with_timeouts()`. Vendored
  Mozilla roots via `webpki_roots`; **`reqwest::Client::new()` _panics_ in
  the `FROM scratch` runtime image** (ADR-0004), so this is the only correct
  client. It also _removes_ two reqwest defaults: redirects are **not
  followed** (a 3xx arrives intact and must be refused as `Malformed`, never
  taken — taking it would replay your credentials at whatever `Location`
  named) and proxy environment variables are ignored. `read_rail_body`
  bounds every response at `MAX_RAIL_BODY_BYTES` (256 KiB);
  `Response::text()`/`bytes()` read to end of stream and let the peer decide
  how much memory a worker allocates.
- **`vpay_provider::http::path_segment(x)?`** — the one way a value you did
  not write (a payer's number, a caller's reference; not a typed `Uuid` or a
  constant of yours) gets into a URL path, because the request carries
  **your deployment's credentials**. It percent-encodes everything outside
  RFC 3986's unreserved set. It was `fn(&str) -> String` until 2026-10-20 and
  now returns `Result<String, ProviderError>`: an **empty input, or one made
  only of dots** (`""`, `.`, `..`, `...`), is refused as
  `ProviderError::Malformed` before anything is sent, with a fixed message
  that never quotes the input. Escaping cannot cover those inputs: `.` and
  `..` are unreserved, and URL parsers read `%2E%2E` as `..` too, so
  `…/payments/..` is the API root and `…/payments/` the collection. `...` is
  not a dot segment to either URL standard and is refused **as a decision** —
  nothing an adapter interpolates is legitimately all dots. The argument is
  in `docs/reference/rails.md`, § "A path segment the caller chose". What that
  asks of an adapter:
  - **Call it and propagate with `?`.** Do not map the refusal to `Ok(None)`
    or a decline. It is `Malformed` for a request that never left the
    process — the same compromise as `parse_destination`'s (§ 4b of
    [adding-a-rail.md](references/adding-a-rail.md)).
  - **Build the URL before the request, the token fetch included**, so a
    refused value costs the rail nothing. MTN's `account_holder_name` calls
    `account_holder_url(..)?` before `send_authorized`.
  - **Write no copy of the guard.** There is no unchecked variant to reach
    for, and a private "reject `..`" check beside it is the second copy that
    disagrees with the port's.
  - **Test the adapter's _use_ of it, not the helper.** Put the URL in a pure
    function and assert the dot inputs are refused there; then assert at the
    method that they come back `Malformed` against a config pointing at a
    port nothing listens on, where a request that did leave the process would
    come back `Transport`. MTN's pair is
    `the_lookup_url_escapes_the_payer_reference_it_interpolates` and
    `a_dot_segment_reference_is_refused_before_any_request`; a test of the
    helper alone stayed green when the adapter's call was deleted (the
    `vpay-mtn-momo` skill's account-holder-lookup page).
- **`vpay_provider::token`** — `fingerprint(&[&str])` (length-prefixed
  SHA-256; hash the **secret** halves too, so rotating only a secret evicts
  the cache on the next call), `usable_until()`, `CachedToken` with a
  redacting `Debug`. The refresh margin is **per-rail** and passed in.
- **`vpay_provider::measured::Measured`** — the counter and histogram. It is
  applied **once**, in `vpay_api::v1::boot::adapters_by_code`, the single
  funnel every rail call passes through. Do not instrument inside an adapter.
  One increment per **port call**, not per HTTP request; `parse_callback` is
  not counted.

Adapters take the process's one `reqwest::Client` by clone. Neither shipping
adapter derives `Default` — `Default` would have to invent a client.

Every deployment gets `DEFAULT_CONNECT_TIMEOUT` (5 s) and
`DEFAULT_REQUEST_TIMEOUT` (20 s), carried on `ProviderConfig` rather than on
the client because one client is shared by every rail. **Apply
`config.request_timeout` per request.** See the timeout lesson in
[references/conformance.md](references/conformance.md) — Orange did not, and
the suite said it was fine.

`ProviderConfig`, `AccountHolder` and `RefundTarget` all have hand-written
redacting `Debug` impls. So do both adapters' credential and wire types, and
so does `vpay_api::v1::refunds`' `CreateParams` — which holds a payee's number
in a raw `serde_json::Map` that has none of its own. Keep it that way.

## `ProviderConfig` is per deployment today, whatever its doc says (2026-10-16)

`ProviderConfig`'s doc comment calls it "Opaque per-merchant, per-rail
configuration", and the port hands it to every method per call. **The core
builds one per rail, not per merchant.** `vpay_api::v1::ResourceConfig::from_config`
projects each `providers[]` entry into one `RailConfig`, `MerchantClient` names
no provider account, and `charges` records `provider_code` and no account.
Measured on vaam-apps/vpay `master` at `44c5bf5`.

So on a deployment with two `merchant_id`s, both merchants are paid into the
one account `providers[].credentials` belongs to on each rail. **Pass-through
holds only on a deployment that serves one merchant.** Do not write an adapter
that assumes it is handed a particular merchant's account, and do not describe
a multi-merchant deployment as pass-through. Draft RFC-0009 proposes the fix;
the port's trait would not change.

**What decides aggregation is whose money it is, not how many `merchant_id`s
there are (2026-10-22).** The maintainer decided the merchants served for now
are projects of one company. RFC-0009's revision (vaam-apps/vpay#336) treats
that as the company being a merchant: one account per rail holding the
company's own money is not aggregation. The operator's rule is therefore one
deployment per **legal entity**: several projects of one company may share
one, and a merchant that is another company, a subsidiary included, needs its
own. **Nothing in code enforces this yet.** RFC-0009's phase A
(`deployment.legal_entity`, a `legal_entity` on each merchant, a boot refusal
for a second company, `charges.provider_account_id`) is unbuilt, and counsel
has not been asked. Do not configure a second company onto a deployment, and
do not describe phase A as built.

## Proposed port changes that do not exist (2026-09-23)

~~Four Draft RFCs~~ **Ten RFCs as of 2026-10-20** — the four merged by
vaam-apps/vpay#244 as `7997536b` (`docs/rfc/0004`, `0006`, `0007`, `0008`),
and 0009 to 0014 added from 2026-10-16 to 2026-10-20 (0014 is the last
paragraph of this section) — would change this port or add to it. (This
sentence said "Four" until 2026-10-20, although the list below had grown to
seven by 2026-10-17, and "Nine" for part of 2026-10-20.) **None of it is built beyond what each
bullet says, and the capability list above is complete as of 2026-10-20.** Do
not write an adapter against any of the following, and do not treat a Draft
section as a decision:

> **Rewritten 2026-10-20 for the Notch Pay adapter.** RFC-0012 and RFC-0013
> were first read on vpay's `claude/happy-hopper-r3h6rz` branch, merged the
> same day as `b8be107` (vaam-apps/vpay#285), and re-read at `b8be107`, whose
> tree is identical to that branch's last commit; both RFCs had changed only in
> formatting. The RFC-0012 bullet and the client-crate section below were then
> rewritten from vaam-apps/vpay#295 (read at `bf96e50`), which builds the
> adapter and merged to `master` as `357c825` the same day. On a tree older
> than that, the client is linked by nothing and RFC-0012 is gated. This
> page's stamp is not moved: moving it means re-verifying the whole page
> (VERSIONING.md).

> **Qualified 2026-09-23.** Since vaam-apps/vpay step A (RFC-0004 §§ 5–6,
> merged in vaam-apps/vpay#251), RFC-0004 is Draft **except** § 5's `customer` list filters
> and § 6's manual payments, accepted through ADR-0024 and built. **Neither
> touches this port**: no capability, no adapter method, no `ProviderFlow`.
> Everything below — § 7 included — is still Draft. One name collides: step
> A's `out_of_band[method]=bank_transfer` is a **label a merchant sends** when
> recording an invoice paid outside vpay. It is not a rail, reaches no
> adapter, and is not RFC-0007's `bank_transfer`, whose shape below is still
> an open question.

> **Qualified 2026-10-17, and again on 2026-10-23.** RFC-0004 § 7 and
> RFC-0008 are **accepted** through ADR-0029 (vaam-apps/vpay#275). On
> 2026-10-23 **one** of their identifiers was built:
> `Capabilities::method_types` (§ "Method types" below), with D12's two
> wallet spellings and D13's `collapse`, both confirmed by the maintainer
> that day. **The rest below still does not exist**, and an adapter written
> against it is written against a design, as RFC-0010 and RFC-0011 are.
> ADR-0029 D2 says each capability lands with its first consumer; do not add
> one to the port ahead of the adapter that declares it. D8 and D9 (fee
> ascending) are **reopened**: the maintainer asked for "a score of each
> payment solution for each checkout", and RFC-0008's 2026-10-23 revision
> proposes one. D14–D17 (`card`'s presentation, `routing.requires`, fee
> schedules on the provider account) are still proposed.

- **RFC-0004 § 7 — one adapter per provider.** Every card or bank PSP that
  meets four hard rules gets its own adapter. The rules: vpay never holds the
  money; no card number reaches `vpay-server`; a secret-authenticated status
  query; a stub-able contract. Refunds, off-session charging and bank
  transfer become capabilities. It would add `supports_off_session`,
  `charge_stored`, `method_types`, `amount_limits` and
  `ProviderFlow::Instructions`. _Accepted 2026-10-17 (ADR-0029)._ **Only
  `method_types` exists, since 2026-10-23**; the other four do not.
- **RFC-0006 — card processing without a PSP.** A separately deployed card
  vault (`vpay-vault`), never linked into `vpay-server`, which appears to the
  core as one more rail (`vpay_vault`). It would also add
  `ProviderFlow::Embedded`, later.
- **RFC-0007 — bank transfers reconciled from statements.** It leaves open
  whether `bank_transfer` is an adapter whose `query_status` answers
  `Unsupported`, or a new kind of settlement source.
- **RFC-0008 — routing.** It separates the method type (`mtn_wallet`) from
  the rail (`notchpay`). _Accepted 2026-10-17 (ADR-0029, less its § 7 on
  health); **revised 2026-10-23** to a score per checkout, and **partly
  built** the same day_ (§ "Method types" below): `payment_method_types`
  accepts `mtn_wallet` and `orange_wallet` beside rail codes, and the router
  is `vpay_core::routing::route`. ~~"Today `payment_method_types` holds rail
  codes … no router exists."~~ That was true until 2026-10-23.

- **RFC-0009 — provider accounts per merchant** (added 2026-10-16, not
  part of vaam-apps/vpay#244; **revised 2026-10-22** in vaam-apps/vpay#336).
  It has two phases:
  - **Phase A** is for projects of one company: `deployment.legal_entity`, a
    `legal_entity` on each merchant, a boot refusal for a second company, and
    `charges.provider_account_id`, recorded as `default`. No trait change.
  - **Phase B** is for other companies, on Stripe Connect's shape, the
    maintainer's preference:
    - `acct_…` accounts in the database;
    - the operator's platform client honouring `Stripe-Account`;
    - a separate onboarding application (`frontends/apps/onboarding`, with
      `/onboard/v1` and account links);
    - per-account callbacks at
      `/provider/{code}/{provider_account_id}/callback`;
    - sealed rail secrets;
    - the original § 2 two-level merge, where a key at both levels refuses
      to boot;
    - **application fees**, the one place the trait would change: the
      submit request would carry the fee, and only rails that can split a
      payment may take it. They conflict with ADR-0029 D1 rule 1 as
      written, and need an amending ADR and counsel first.

  **None of it exists**: see the section above for what the tree does today.
  `Stripe-Account` is still accepted and ignored for every client, and
  `application_fee_amount`, `transfer_data` and `on_behalf_of` are still
  refused with a `400`. _(This bullet described only the 2026-10-16 proposal
  — `provider_accounts` in YAML, "No trait method changes" — until
  2026-10-22.)_

- **RFC-0010 — a Stripe adapter** (added 2026-10-17). `vpay-adapter-stripe`,
  rail code `stripe`, Stripe's hosted Checkout as one more `Redirect` rail:
  `POST /v1/checkout/sessions` with the charge reference as the
  `Idempotency-Key`, `GET /v1/checkout/sessions/{id}?expand[]=payment_intent`
  as the authoritative read, `POST /v1/refunds` by `payment_intent`. It
  would be the **first `RefundDestination::Origin` rail**. The two core
  changes the RFC names **landed on 2026-10-17 as its delivery step 1**:
  `refund_at_rail` puts the charge's `provider_txn_id` into the refund
  `ChargeRef`'s `ref_extra` under `REFUND_PROVIDER_TXN_ID_KEY`, and the
  conformance premise guard was relaxed with its three debts paid — the
  refund-destination section above has both. Steps 2–7 are unbuilt. Its
  callback URL is registered with Stripe, never sent per request,
  so `the_submit_tells_the_rail_where_to_call_back` cannot hold on it as
  written. **No crate, no mapping, no stub exists**, and nothing has called
  Stripe from this repository.
- **RFC-0011 — a PayPal adapter** (added 2026-10-17). `vpay-adapter-paypal`,
  rail code `paypal`, Orders v2 as one more `Redirect` rail:
  `POST /v2/checkout/orders` with `payment_source.paypal` and the charge
  reference as `PayPal-Request-Id`, the `payer-action` link as the redirect,
  `GET /v2/checkout/orders/{id}` as the read, refunds on the capture id. Its
  one design decision: **PayPal's wallet is not captured on approval**
  (`ORDER_COMPLETE_ON_PAYMENT_APPROVAL` is documented for alternative
  payment methods only), so the RFC puts the
  `POST /v2/checkout/orders/{id}/capture` call **inside `query_status`**
  when the order is `APPROVED`, idempotent on a derived request id, rather
  than adding a `complete` method to the port. It times out an unapproved
  order locally (PayPal's status enum has no expired value) from PayPal's
  documented three-hour validity. It proposes two small core changes of its
  own — the hosted checkout's return page pulling the poll forward, and
  `sandbox` as a livemode stub marker — and shares RFC-0010's two. **No
  crate, no mapping, no stub exists**, and nothing has called PayPal from
  this repository.
- **RFC-0012 — a Notch Pay adapter** (added 2026-10-20; Draft, **admitted
  under ADR-0031 (rule 3 amended) and built the same day**, in
  vaam-apps/vpay#295, `357c825`). Notch Pay fails ADR-0029 D1's third rule
  **as ADR-0029 wrote it**: `GET /payments/{reference}` accepts the key
  Notch Pay calls public. The maintainer first chose to ask Notch Pay in
  writing (way A) and emailed them. With no answer the same day, they wrote
  "Let's do a full implementation and warn the developer about the risks in
  prod. They never answered to me too". **ADR-0031** then amended rule 3 for
  every provider, on six conditions (way B, taken openly). The adapter,
  `vpay-adapter-notchpay`, is a `Push` rail on `cm.mobile` (and, since
  2026-10-23, `cm.mtn` / `cm.orange` per charge), the first
  shipping `Origin` rail, with refunds as the declared token
  `notchpay::refund`. **It has called Notch Pay's sandbox, and nothing
  else**: RFC-0012's step 5 ran on 2026-10-21 (three test-mode payments
  through the adapter, from the maintainer's MacBook: one `succeeded`, one
  `failed`, one short `amounts.total` refused as `Malformed` and never a
  success; since 2026-10-23 that refusal is `NeedsReconciliation`, which
  escalates the charge to `unresolved` at once), after step 1's probe the same day. No production, no live key,
  no webhook received by vpay, no refund. _(This said "no webhook received"
  until 2026-10-24, when the sandbox delivered four to a capture endpoint
  registered in its dashboard, and the adapter's `verify_callback` verified
  them offline; none has reached a running vpay.)_ Load the `vpay-notchpay` skill before
  touching it. Write "admitted under ADR-0031",
  **never** "rule 3 passes", and do not weaken any of ADR-0031's conditions:
  `X-Grant` on every call, boot refusing a missing private key, the
  documented exposure, the boot warning, the amount check, the open question.
- **RFC-0013 — vpay registers its own callback endpoints. Steps 2–4 built
  2026-10-22, for Notch Pay** (vaam-apps/vpay#340, merged as `8504536f`).
  Read `docs/rfc/0013-vpay-registers-its-own-callbacks.md` § "As built"
  before touching it. What exists: the port's `CallbackUrlMode`,
  `callback_url_mode`, `callbacks_are_coherent` and
  `ensure_callback_endpoint` (above); `JobKind::EnsureCallbacks`, the sixth
  singleton, dedupe key `ensure:callbacks`, migration `0051`, hourly and
  five minutes after a `Transport` or database failure, **never `Err`**; the
  lock `vpay_db::lock_keys::CALLBACK_REGISTRATION` (`vpaycbk`,
  `pg_try_advisory_xact_lock`, one transaction per rail);
  `providers[].callbacks.register` (`ensure` | `check` | `off`, refused on a
  `PerRequest` rail as `ConfigError::CallbacksOnPerRequestRail`; unset, the
  adapter's `default_callback_registration`, which is `ensure` everywhere but
  `notchpay`, `off` since 2026-10-23 — it said "`ensure` default" until then);
  `vpay-server callbacks ensure [--check]`; the counter
  `vpay_callback_endpoint_checks_total{provider,state}`, which the chart's
  `VpayCallbackEndpointUnconfirmed` alerts on (2026-10-23,
  `docs/runbooks/callback-endpoints.md`). A new `Registered` rail is covered
  by it with no rule change, because the rule groups `by (provider)`. The marker is
  `vpay:{deployment.name}:{rail}:default`. **Never run against a real Notch
  Pay account** (delivery step 5); Stripe and PayPal have no adapter. On
  2026-10-23 the Notch Pay probe made the same `POST /webhooks` call with a
  public URL: the sandbox listed it with `"url": null`, delivered nothing in
  21 minutes, and answered `404` to `GET`, `PUT` and `DELETE` by id. So on
  that evidence an endpoint `ensure_callbacks` creates on Notch Pay receives
  nothing, for a reason still unknown, and `present` or `created` means
  "listed", not "delivering" (the `vpay-notchpay` skill). One registered by
  hand in the dashboard does deliver: four signed webhooks to a capture
  endpoint on 2026-10-24, none yet to a running vpay. _(This said the
  sandbox "stored `"url": null`" until 2026-10-24: the listing shows `null`
  for the dashboard endpoint too.)_ **So since 2026-10-23 an unset `register`
  asks a Notch Pay account nothing** (above); `ensure` must be written. Where
  it differs from the RFC: `Drifted` and `retirable` are new; Notch Pay's
  endpoint is recognised by a URL tag in its description, because Notch Pay
  answers `"url": null`; nothing is ever retired or repaired on Notch Pay
  (no working `DELETE`, no `PUT`), which since 2026-10-23 is the maintainer's
  decision rather than a gap ("an operator's task only, not code"); a
  counter, not a gauge.
  _(The original entry, "added 2026-10-20; Draft, nothing in it is built", is
  kept below as written.)_
- ~~**RFC-0013 — vpay registers its own callback endpoints** (added 2026-10-20;
  Draft, **nothing in it is built**).~~ On a rail whose callback URL is
  registered with the provider account instead of sent on each request —
  Stripe, PayPal and Notch Pay; only Notch Pay has an adapter, and only on a
  vpay branch (2026-10-20) — vpay would make
  sure one endpoint it created exists at the effective callback URL, in a
  worker singleton job (proposed kind `ensure_callbacks`, hourly) and a
  proposed command `vpay-server callbacks ensure`, **never on the boot path**,
  and would never modify or delete an endpoint it did not create.
  `mtn_momo` and `orange_money` take the URL per request (`X-Callback-Url`,
  `notif_url`) and would be asked nothing. It proposes two port additions,
  **`Capabilities::callback_url_mode`** (`PerRequest` | `Registered`) and
  **`ProviderAdapter::ensure_callback_endpoint`** (defaulting to
  `Unsupported`), and a `providers[].callbacks.register` setting. **None of
  those names exists** — no `callback_url_mode`, `CallbackUrlMode`,
  `ensure_callback_endpoint` or `EnsureCallbacks` appears under `backends/` as
  of 2026-10-20. The RFC ties the port shape to the first adapter that
  registers an endpoint (its delivery step 2, which also needs a pull request
  against `vpay-skills`); until then do not add either to the port. The
  maintainer's decision of 2026-10-20 is the premise — "never miss a
  webhook" — and the design, the schedule and the open questions (seven as of
  2026-10-20) are still Draft. One of them is a rail that can register its URL
  only in a dashboard, as Flutterwave's documentation reads: it would declare
  `Registered` and leave `ensure_callback_endpoint` at `Unsupported`, and
  whether that needs a third mode is open (question 7).
  **Until it lands, registering a webhook with Stripe, PayPal or Notch Pay is a
  manual step for the merchant or operator** (RFC-0010 § 9, RFC-0011 § 9,
  RFC-0012 § 9) — _no longer true of Notch Pay from 2026-10-22_; on MTN a production callback host is a domain registered on
  MTN's portal with the API user, and the RFC finds nothing there for vpay to
  register at startup.
  Registration would change how soon a hint arrives, not what a hint can do:
  callbacks stay hints, and the poll ladder settles a charge whose callback
  never came (RFC-0013 § "What a missed callback costs").

**`backends/crates/vpay-notchpay` is the Notch Pay client** (2026-10-20;
`docs/reference/vpay-notchpay.md`), an internal crate (`publish = false`)
transcribed from Notch Pay's OpenAPI specification 2.1.0. On `master` at
`b8be107` it was **linked by nothing**. Since `357c825` (vaam-apps/vpay#295),
`vpay-adapter-notchpay` sits on it and `vpay-server` links the
adapter. Since 2026-10-21 (RFC-0012's step 5) it has called Notch Pay's
sandbox, and only that. What carries over to any
change you make there:

- **Do not write a `wire.rs` for Notch Pay, and do not re-transcribe the
  specification.** This is the one rail whose wire is a crate boundary rather
  than a module (see `references/adding-a-rail.md` § 3). The adapter keeps
  `lib.rs` and `mapping.rs`. There is no `token.rs`, because Notch Pay
  authenticates with static keys in headers.
- **The client says what Notch Pay answered; the adapter decides what it
  means.** A call returns `Result<Answer<T>, ProviderError>`: `Answer::Success`
  or `Answer::Refused` (any other `4xx`, with Notch Pay's own words), and
  `ProviderError` only for `Transport` (`408`, `429`, `5xx`, no answer),
  `Malformed`, or `Config` at construction. What a `409` or a `422` means for a
  charge is `mapping.rs`'s call, not the client's. The crate defines no error
  type of its own, on purpose.
- **It sends both keys on every request.** That is ADR-0031's condition 1,
  and every conformance stub mapping requires `X-Grant`. Its
  `reqwest::Client` must not follow redirects (a followed one replays
  `X-Grant`); the clients in `vpay_provider::http` refuse them. **Since
  2026-10-20 it applies a per-request deadline**
  (`Client::with_request_timeout`), which the adapter sets from
  `ProviderConfig::request_timeout`.
- **What it deliberately omits**: refunds, cancel, webhook update and delete,
  transfers, and webhook signature verification, which is the adapter's
  `verify_callback` over `vpay_provider::callback`'s helper since 2026-10-23
  (RFC-0014): the client never sees a webhook.
- **Where Notch Pay's own pages disagree with its specification**, the client
  follows the specification, and RFC-0012 § 10 lists each disagreement for the
  first sandbox session. Treat every one of them as unverified.

A card PSP adapter is not blocked on engineering. The 2026-09-23 desk
evaluation (`docs/plans/2026-09-23-card-provider-evaluation.md`,
public documentation only) found **no provider shown to accept Visa and
Mastercard for a Cameroon merchant on a hosted page**. The providers' written
answers come first.

**Fapshi — a desk study, no adapter (2026-10-23).**
`docs/plans/2026-10-23-fapshi-evaluation.md` read Fapshi's documentation and
`openapi.json` only: no account, no key, no call. On paper it is a mobile-money
aggregator (MTN and Orange in Cameroon, XAF) that passes all four hard rules,
**rule 3 as ADR-0029 wrote it**: the status read takes `apiuser` and a secret
`apikey`, so it would need no ADR-0031 exception and no boot warning. The page
names four blockers before any code:

- a resent Direct Pay may prompt the payer twice (no idempotency key, and
  `externalId`'s uniqueness is undocumented), and only a lookup by an id of
  vpay's, `GET /transaction/{userId}`, could guard it, if that lookup is
  consistent;
- Direct Pay is off in live mode and approved case by case;
- there is no refund API;
- the terms name "Fapshi Inc" and no licence, regulator or partner bank.

There is no `vpay-fapshi` crate, no `fapshi` adapter code and no RFC. **Do not
start one** before the page's written questions are answered and the
maintainer asks for it; the page lists the order such work would follow.

**RFC-0014 — verifying rail callbacks: the port and Notch Pay's check are
built** (2026-10-23, against vpay `master` `82a68f4e` plus the branch
`claude/happy-hopper-r3h6rz-webhook-signatures`; stub-proven, and since
2026-10-24 verified offline against four real Notch Pay deliveries, signed
over the raw body; none has reached a running vpay, vaam-apps/vpay#364).
_(This said "stub-proven; never verified against a real Notch Pay delivery"
until 2026-10-24.)_ The maintainer answered "Yes, build RFC-0014"
on 2026-10-23, lifting RFC-0012's "hash key is evidence only" rule for it.

- **The port** (`vpay_provider::callback`, new): `CallbackRequest { headers,
body, received_at }` (a hand-written `Debug` that prints header names and
  the body's length, registered in `debug_protections`), `CallbackVerdict`
  (`Verified`, `VerifiedPrevious`, `Invalid(&'static str)`, `NotChecked`,
  `NoSecret`, `Unavailable(&'static str)`; `#[non_exhaustive]`; `Verified`,
  `VerifiedPrevious`, `NotChecked` and `NoSecret` admit the pull-forward,
  `Invalid` and `Unavailable` do not), and
  **`ProviderAdapter::verify_callback(request, charge, config)`**, the
  fourteenth method, defaulting to `NotChecked`. `parse_callback` is unchanged. `Measured`
  forwards it and counts nothing. **An adapter that declares a check and
  keeps that default fails closed**: the route answers `Unavailable` and
  logs `error`. _(2026-10-23 security review: this bullet said "only the
  first three admit the pull-forward", which named `Invalid` and missed
  `NoSecret`; `VerifiedPrevious` and the fail-closed rule are the review's.)_
- **`Capabilities::callback_check`** (`None` | `BodyToken` | `Signature`) says
  what **vpay's adapter** checks, not what the rail sends;
  `callback_check_is_coherent` (`!delivers_callbacks ⇒ None`) is a boot
  refusal (`IncoherentCallbackCheck`). A new rail literal must set it.
- **The secret is a credential under two port-fixed names**,
  `CALLBACK_SIGNING_SECRET` and `CALLBACK_SIGNING_SECRET_PREVIOUS`, read
  through `callback_signing_keys(config)` (each tagged
  `vpay_provider::SigningKey`, so a match under the previous one answers
  `VerifiedPrevious`): not RFC-0014 § 5's illustrative
  `callbacks.signing_secrets` list. Boot is generic over them (ADR-0002),
  and refuses a previous secret without a current one and a secret with
  surrounding whitespace.
- **One MAC comparison** for the workspace: `hmac_sha256_hex_matches`
  (`Mac::verify_slice`, constant time). Do not write an `==` on a MAC in an
  adapter; `the_comparison_is_the_constant_time_one` reads the source.
- **Who checks:** `notchpay` (`Signature`, `x-notch-signature`). `mtn_momo`
  and `orange_money` declare `None` and answer `NotChecked`. **Orange's
  `BodyToken` check is not built** (RFC-0014 open question 2: nobody has
  observed that the notification's `notif_token` equals the stored one), nor
  are Stripe's and PayPal's (no adapter).
- **What it keeps:** callbacks stay hints (a verdict only decides whether
  the poll is pulled forward), and every answer stays a byte-identical
  `202`. The conformance census `a_forged_callback_is_refused_by_a_rail_that_checks`
  branches on the capability, and `documented_callback_signature` is the
  per-rail data row a new signing rail must add.

## Method types and the router (built 2026-10-23)

A **method type** is what the payer uses; a **rail** is the adapter that
processes it. Each adapter declares what it serves in
`Capabilities::method_types` (`vpay_core::routing::MethodTypes`, a set):

| Rail           | `method_types`                |
| -------------- | ----------------------------- |
| `mtn_momo`     | `mtn_wallet`                  |
| `orange_money` | `orange_wallet`               |
| `notchpay`     | `mtn_wallet`, `orange_wallet` |

- **A new rail declares at least one type**, or boot refuses it when
  enabled (`ConfigError::RailServesNoMethodType`), and the conformance
  suite's `every_adapter_declares_the_method_types_it_serves` pins the exact
  table above, so a rail added or changed fails there until the table names
  it. `card` and `bank_transfer` are reserved spellings: no adapter may
  declare them until RFC-0010 / D4 build them.
- **`ChargeRef::method_type`** carries the payer's choice to `submit`,
  `query_status` and `refund`. `None` means a charge confirmed by the code of
  a rail serving several types, or one from before migration `0053`. An
  adapter serving one type may ignore it; `notchpay` sends `cm.mtn` /
  `cm.orange` from it and its `settings.channel` (`cm.mobile`) when it is
  `None`.
- **The router names no rail.** `vpay_core::routing::route` is one pure
  function: a hard eligibility filter (serves the type, enabled, settles the
  currency, the pin), then a score per candidate with a breakdown. Never add
  a rail-code branch to it; `vpay-server`'s `the_router_names_no_rail` reads
  its source and fails. **The formula is provisional** (`provisional-1`, a
  configured weight per rail, `routing.weights`, default 100, ties to the
  lower rail code). RFC-0008's R8 was answered on 2026-10-23: the next score
  component is the estimated fee; the completion rate is reported for the
  operator to turn into a weight by hand, never scored (§ R8.1, "Go with
  (a)"); neither is built; health never enters the score (R8.3); there is no exploration
  (R8.5); the tie-break is kept. A new component is a `Component` variant
  and its breakdown entry, not a sort key. vpay's evidence:
  `docs/status/verification/2026-10-23-method-type-routing.md`.

## Where to go next

- [references/adding-a-rail.md](references/adding-a-rail.md) — the full
  checklist, including two corrections to `docs/flows/provider-port.md` you
  must not follow literally, and what a `Required` rail owes.
- [references/conformance.md](references/conformance.md) — the shared suite,
  the mapping contract a new rail's stubs must satisfy, and the techniques.
- `vpay-mtn-momo` skill — the push rail in detail.
- `vpay-orange-money` skill — the redirect rail in detail.
- `vpay-notchpay` skill — the aggregator push rail admitted under ADR-0031,
  and its production risk.
- `docs/status/providers.md` (the per-provider matrix, 2026-10-20),
  `docs/plans/2026-10-23-fapshi-evaluation.md` (Fapshi, a desk study),
  `docs/reference/vpay-notchpay.md` (the client crate, 2026-10-20),
  `docs/rfc/0012-notchpay-adapter.md`,
  `docs/rfc/0013-vpay-registers-its-own-callbacks.md`.
- `docs/reference/rails.md`, `docs/flows/provider-port.md`,
  `docs/adr/0002-provider-port.md`, `docs/adr/0011-error-modelling.md`.
