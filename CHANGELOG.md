# Changelog

Each release records the vpay range it was verified against, which skills
changed, and — the section worth reading — **any claim that stopped being true**,
so someone upgrading can find the thing that will break them.

Releases are named for the date of verification and the vpay commit verified
against, because that commit is usually between two vpay releases. See
[VERSIONING.md](VERSIONING.md). _(Corrected 2026-09-23: this sentence said "vpay
publishes no release tags and its workspace version has never moved off
`0.1.0`". vpay has cut release tags since 2026-09-17 — `v0.1.1` through
`v0.5.0` as of 2026-09-23 — and its workspace version is `0.5.0`. The naming
rule stands, for the reasons VERSIONING.md now gives.)_

## v2026-10-09-69d822ab

Verified against vpay `69d822ab` (2026-10-09), the squash of vaam-apps/vpay#273,
which carried vaam-apps/vpay#268 (RFC-0002 PR 3, issue #147) with the
maintainer's review fixes, and on top of vaam-apps/vpay#272 (the npm advisory
sweep). `vpay-tooling` was briefed by @Valsuh45 in
vaam-apps/vpay-skills#28, restamped and corrected here.

### Added

- `vpay-tooling`: the `debug_protections` boundary of `verify-privacy-inventory`.
  A registered type that derives `Debug`, a registration naming no declared
  type, or one naming no inventory element fails the gate. A derive is matched on
  its last path segment. The lockstep row covers adding a type that holds a payer
  identifier, rail failure text, a rendered API body or a credential.

### Claims that stopped being true

- `vpay-tooling`: none. The boundary is new. Figures vaam-apps/vpay#268
  stated that did not survive its review: 17 registrations is 24 on
  `master`, 31 gate tests is 33, and xtask's 313 tests is 315.
- `vpay-checkout`, `vpay-dashboard`, `vpay-frontend` and `workspace.md`, since
  vaam-apps/vpay#272 the same day: the two vpay apps are on Next `^15.5.27`, not 15.5.25. `examples/shop` pins 16.3.8, not
  16.3.4. Both changed on 2026-10-09 to clear GHSA-vcvr-r3jv-pc5j and
  CVE-2026-94483. `read-seam-and-bff.md`'s note keeps the versions the review
  read, now in the past tense.

## v2026-10-09-87166eaf

Re-verified against vpay [`87166eaf`](https://github.com/vaam-apps/vpay/commit/87166eafb71eb105ff90a9958377866840447748)
(2026-10-09), `v0.5.0` plus fourteen commits (`git describe`: `v0.5.0-14`; no
release has been cut since `v0.5.0`). The previous entry here is `a33aac61`
(2026-09-29), and `87166eaf` is **two merged pull requests** later:
[vaam-apps/vpay#270](https://github.com/vaam-apps/vpay/pull/270) (the Helm chart
accepts `global`, `8cad7d1`) and
[vaam-apps/vpay#271](https://github.com/vaam-apps/vpay/pull/271) (ADR-0028, the
client-assertion `jti` horizon, `87166ea`). Together they touch 22 files, and this
release covers all of them: every path in that diff that a skill claims in
`coverage.json` was read, and the skills that describe them were corrected.

**What "re-verified" means here is narrower than in the last entry, and a stamp
should be read that way.** The check was the `git diff a33aac61..87166eaf` of the
claimed paths, plus a grep of `skills/` for every symbol and claim those two PRs
touched (`jti`, `client_assertion`, `sweep_expired`, `stopgap`, `value sets`,
`Accepted in part`, `not yet written`). It was **not** a re-read of any reference
page end to end. Eight skills changed content and carry the new stamp on their
`SKILL.md` (`vpay-conventions`, `vpay-customers`, `vpay-data-layer`,
`vpay-merchant-api`, `vpay-ops`, `vpay-reconciler`, `vpay-tooling`,
`vpay-troubleshooting`); the reference pages that were edited carry a stamp that
**names the section** and the older stamp for the rest: `adr-index` (the 0026 and
0028 rows and notes, "Not accepted", the numbering section), `erasure` (§ "Which
payments an erasure reaches"), `configuration` (boot step 7), `config-boot` (boot
step 7), `recipes` (§ Helm) and `deployment` (§ "Verifying the chart"). The other
twelve skills keep the `a33aac61` stamp, which is the honest record of when they
were last read. **`coverage.json`'s `baseline` did not move** (it is still
`a33aac61`; the gate requires a stamp to contain it, and every new stamp does).
Five claims were added to it: ADR-0028 under `vpay-conventions`, `vpay-reconciler`
and `vpay-merchant-api`, `merchant_token_flow.rs` under `vpay-merchant-api`, and
`deploy/helm/fixtures/wrapper` under `vpay-tooling`.

### Claims that stopped being true

> **ADR-0026 is `Accepted (2026-10-08)`, not "Accepted in part".**
> [vaam-apps/vpay#271](https://github.com/vaam-apps/vpay/pull/271) edited the
> ADR's Status block (and only that): the maintainer confirmed D1–D8 on
> 2026-10-08, after vaam-apps/vpay#256 merged, and the text of D1–D8 is unchanged.
> `vpay-conventions`' ADR index, `vpay-data-layer` and `vpay-reconciler` all said
> "Accepted in part" and told an agent not to describe D1–D8 as
> maintainer-decided; those sentences are struck. Two findings from the review that
> preceded the confirmation are recorded in ADR-0028, not in the immutable
> ADR-0026: D8's `jti` item is concluded, and **D6's list of comparisons against a
> fact omits the checkout confirm gate** (`SessionGate::admit_confirm`, in
> `vpay-api/src/v1/return_trip.rs`), which is on the right side of D6's rule, so no
> behaviour changes.

> **A spent client-assertion `jti` is no longer deleted at its `exp`.**
> [ADR-0028](https://github.com/vaam-apps/vpay/blob/87166eafb71eb105ff90a9958377866840447748/docs/adr/0028-a-spent-jti-outlives-the-validators-leeway.md)
> (vaam-apps/vpay#271, 2026-10-09). `authkestra-op` 0.7.1 leaves `jsonwebtoken`'s
> 60 s leeway at its default, on a whole-second `now`, so the API accepts an
> assertion until `exp + 61 s`. The hourly sweep deleted at `expires_at < now()`,
> and a sweep inside those 61 s (about 1.7 % per assertion at zero skew) let a
> captured, already-spent assertion be replayed **once** for a second merchant
> access token. `ClientAssertions::delete_expired_client_assertion_jtis` now takes
> `retain_after_exp: Duration`, the statement is
> `expires_at < now() - ($1::BIGINT * INTERVAL '1 microsecond')` on the database's
> clock, and the worker passes `vpay_worker::CLIENT_ASSERTION_JTI_RETENTION`
> (5 minutes). A test refuses an assertion older than that horizon, so a leeway bump
> fails CI. `expires_at` still stores the raw `exp`. Not covered: a database clock
> more than about four minutes ahead of an API replica's. `vpay-merchant-api`
> now says so beside the `jti` namespace limitation, and `vpay-reconciler` has a
> section on the sweep. ADR-0026 D8's other named column,
> `oauth_signing_keys.expires_at`, is untouched and **still open**.

> **There is no boot-time sweep, and there is a worker job loop.** `vpay-ops`'
> boot list (and its one-line summary "…announce key → sweep → bind…") and
> `vpay-troubleshooting`'s `config-boot` listed "step 7: sweep expired
> client-assertion `jti`s and `idempotency_keys` once — non-fatal boot-time
> stopgaps, because there is no worker job loop", and "the worker sweeps nothing".
> Both sweeps were removed from `vpay-server` in Step 4 (2026-09-03); the worker's
> hourly `sweep_expired` job runs them. The steps are struck, with the numbering
> kept. **vpay's own `docs/flows/configuration.md` still lists that step 7 at
> `87166eaf`**, so the skills now disagree with a vpay page on purpose; the code
> (`backends/apps/vpay-server`, which contains no sweep) is what the skills follow.
> The pre-ADR-0028 doc comment on `delete_expired_client_assertion_jtis` made the
> same two errors and is quoted in `vpay-merchant-api`.

> **The 2026-10-08 erasure decision is written down, and it has a cost.**
> `vpay-customers` said the maintainer's decision to keep "either payer's erasure
> redacts the ambiguous old intent" was "relayed to this repository, not yet
> written in any vpay document at `a33aac61`". It is written, in
> `docs/flows/customers/privacy-and-erasure.md` (§ "The erasure covers every copy
> vpay kept, not just the row", the paragraph "Decided 2026-10-08, and one
> consequence ADR-0027 does not name"), by vaam-apps/vpay#271. ADR-0027 is not
> edited. The paragraph also records the consequence the skills now carry: **if `Y`
> paid and `X` is erased, `Y`'s later refund hands the rail the redaction marker as
> `payer_ref` and cannot be paid out through vpay**; `charges.provider_txn_id` and
> `provider_reference_id` are `subject: none` in `schemas/privacy-inventory.yaml`
> and survive, so the payout can be made at the rail. Only intents whose sessions
> all predate vaam-apps/vpay#253 can have this shape with a charge.

> **`just helm-check` has four value sets, a wrapper-chart step, and the chart
> accepts `global`.** `vpay-tooling`'s `recipes` and `vpay-ops`' `deployment` said
> "three value sets" and "all three renders". The recipe's own comment dates the
> correction to four to 2026-09-16, so the pages were wrong when they were last
> stamped. [vaam-apps/vpay#270](https://github.com/vaam-apps/vpay/pull/270)
> (2026-10-09) made `values.schema.json` list a top-level `global` (an object, any
> keys; nothing in the chart reads it, so it is accepted and ignored) and added the
> "wrapper chart" step: `deploy/helm/fixtures/wrapper`, a parent that depends on the
> chart by `file://` path and sets `global`, must lint, render the same objects as
> the root default render, and still refuse an unknown key under `vpay:`. Before it,
> any chart that depended on vpay's failed `helm lint` and `helm template`, and no
> step saw it because every render used the chart as the root. `deployment` also
> claimed "zero-skipped is asserted" for `kubeconform`; the recipe passes
> `-strict -summary` and no `-ignore-missing-schemas` and checks no skipped count,
> so that parenthesis is struck.

> **`docs/adr/` runs `0001`–`0028`**, not `0027` (0028 merged 2026-10-09), and
> `adr-index`'s "0028 is the next free number and an open vpay PR is expected to take
> it" is struck: it did. `0029` is next free on `master`; open PRs were not checked.

### What did NOT change, and is the point

- **The banner.** Exactly one real rail call has ever been made (MTN's sandbox,
  2026-09-15), no rail has returned money, no cluster has run vpay. Nothing in the
  two pull requests moved it, and nothing here claims a cluster ran the wrapper
  chart: it proves the `file://` dependency only, and the published OCI chart has
  never been consumed as a dependency from a registry.
- **`jti` is still a global namespace.** The primary key is `jti` alone; `jti` MUST
  still be a UUID v4. ADR-0028 changes when a row is deleted, not what it keys on.
- **Nothing a merchant sends or receives changed** in either pull request: no wire,
  schema or migration change. Migration `0011`'s header comment still says no cleanup
  job exists; migrations are checksummed and it stays stale.
- **Fifteen gates and one report.** Neither pull request added a gate.
- **One number is stale and was left alone:** `vpay`'s `SKILL.md` says the `justfile`
  is 5 299 lines (counted 2026-09-29, dated in place); `wc -l` at `87166eaf` is
  5 342. It is a dated count, so it was not edited.
- **The 24 guards and 25 fixtures** in `helm-check` are as before; the recipe's
  success line gained "wrapper chart (global)".

### Decided this release

- **Stamps name their section.** The reference pages edited here carry a stamp that
  says which section was read at `87166eaf`, as the `a33aac61` release did.
- **A struck step keeps its number.** Boot "step 7" is struck, not deleted, and the
  steps after it keep theirs, because pages cite boot steps by number.

## v2026-09-29-a33aac61

Re-verified against vpay [`a33aac61`](https://github.com/vaam-apps/vpay/commit/a33aac61029c5e2d81e77217cd9ea23d43f946ac)
(2026-09-29), `v0.5.0` plus twelve commits (`git describe`: `v0.5.0-12`; no
release has been cut since `v0.5.0`). It is the cratestack 0.12.0 → 0.15.0 bump
([vpay#259](https://github.com/vaam-apps/vpay/pull/259)). The previous entry
here is `7a79684e` (2026-09-16); the pull requests in between (#20–#27) moved
individual skills to `b747e5d5`, `67c90ea5` and `f68fda09` and were recorded in
git history and in each page's own dated corrections, **not** in this file. This entry
covers the whole interval, and the vpay commits it reads are
`0799a8d2..a33aac61` (53) for the six skills that were still stamped there.

**The baseline moved for all twenty skills at once, and "re-verified" has a
precise meaning here — read it before you trust a stamp.** For every skill
the check was: the `git diff` of every vpay path the skill claims in
`coverage.json`, from that skill's previous stamp to `a33aac61`; a targeted grep
of the skill for every symbol, count and claim those diffs touched; and, where
vpay corrected a claim the skill had been quoting as "still wrong in vpay", the
strike-through below. It was **not** a line-by-line re-read of every reference
page. The pages that were read in full, or whose changed sections were, carry
the new stamp in their own header; a page that was edited in one section carries
a stamp that **names that section** and the older stamp for the rest; the others
keep the older stamp, which is the honest record of when they were last read in
full. Reference pages **not re-read** (older stamp untouched): `vpay-conventions`
(`serde`), `vpay-customers` (`account-holder-lookup`), `vpay-checkout`
(`state-machine`), `vpay-dashboard` (`read-seam-and-bff`, `what-a-screen-may-show`:
`f68fda09`, and vpay changed nothing they cover afterwards but the cratestack
bump), `vpay-frontend` (`verify-ui`, `workspace`), `vpay-invoices`
(`constraints-and-transitions`, one sentence edited; `not-built`;
`paid-out-of-band`), `vpay-merchant-api` (`errors`, `objects`), `vpay-mtn-momo`
and `vpay-orange-money` (all four references), `vpay-payments` (`ledger`),
`vpay-provider-adapters` (`conformance`), `vpay-sdks` (`generated-code`,
`tauri-plugin`), `vpay-tooling` (`gates`, a dated snapshot left alone on purpose;
`recipes`, two sentences edited), `vpay-troubleshooting` (`config-boot`,
`dashboard`, `demo-compose`, `deployment`, `node-web`) and `vpay-webhooks`
(`events`). Two numbers in them are therefore **unverified at `a33aac61`**:
`verify-serde`'s 104 types (`serde`) and `verify-sdk-parity`'s 767 / 45 / 40
(`parity`, `tauri-plugin`), both measured on `b747e5d5`; vpay's own pages record
the second at `9184e42` and nobody re-ran either on `a33aac61`.

Twenty skills changed their stamp; nineteen changed content (`vpay-orange-money`
changed its stamp only: nothing it covers changed, and its banner claims were
re-checked against `docs/status.md` and a grep for `NotImplemented`).

### Claims that stopped being true

> **A checkout session's customer is no longer stored "on the session only".**
> Since [ADR-0025](https://github.com/vaam-apps/vpay/blob/a33aac61029c5e2d81e77217cd9ea23d43f946ac/docs/adr/0025-session-customer-onto-intent.md)
> (vpay#253, 2026-09-23) `POST /v1/checkout/sessions` with a `customer` on a
> customer-less intent writes it onto the intent, in the session insert's own
> transaction. `vpay-merchant-api` said "do not fix it by changing
> checkout-session creation" and called it ADR-0024's open question 3;
> `vpay-customers` repeated it; `vpay-conventions`' ADR index listed it as an
> open maintainer decision. All three are struck. Still true: **intents from
> before 2026-09-23 were not backfilled**, so for them the three `customer=`
> filters still disagree. Two races that used to succeed now refuse: a session
> that loses the intent to a concurrent session naming another customer answers
> the existing `400` naming `customer` (it used to be created, so one intent could
> carry sessions naming two payers), and one whose customer was erased between the
> pre-check and the insert answers the existing `409` (it used to attach the
> erased customer to the intent).

> **Customer erasure reaches payments through `checkout_sessions.customer_id`.**
> [ADR-0027](https://github.com/vaam-apps/vpay/blob/a33aac61029c5e2d81e77217cd9ea23d43f946ac/docs/adr/0027-erasure-reaches-through-checkout-sessions.md)
> (vpay#257). One constant, `vpay_db::customers::PAYERS_INTENTS`, finds the
> intents for the three per-payment statements: intents naming the customer, plus
> customer-less intents a session naming the customer points at; the guard is the
> intent's own customer being `NULL` or the erased one. An old intent whose
> sessions named two payers is redacted by either payer's erasure (the maintainer
> decided to keep this on 2026-10-08, a decision this repository was told about
> and that no vpay document at `a33aac61` records). A session create now takes
> `FOR SHARE` on its customer before the intent; the old order deadlocked with
> erasure (`40P01`, a `503`). `vpay-customers` described the reach as "through an
> intent". `erase_in_tx`'s doc comment, which this repository quoted as saying
> "six more statements", says **thirteen more, fifteen in all** since 2026-09-23.

> **The database's clock schedules jobs.**
> [ADR-0026](https://github.com/vaam-apps/vpay/blob/a33aac61029c5e2d81e77217cd9ea23d43f946ac/docs/adr/0026-the-database-clock-schedules-jobs.md)
> (vpay#256), status "Accepted in part" on `master`. `enqueue_in_tx` takes a
> `Duration`; `record_attempt` takes the retry rung; `live_charges_stale_since`
> takes a window back from `now()`; `Jobs::oldest_runnable_run_at` is now
> `oldest_runnable_age`. `vpay-troubleshooting` said `seed_singletons` stamps
> `run_at` from the test process's clock ("two clocks"): that stopped being true
> on 2026-09-23 and the page now says what is and is not still two clocks.
> Facts (session `expires_at`, `last_used_at`, a webhook signature's `t=`) keep
> the application's clock on purpose.

> **cratestack is `0.15.0`, not `0.12.0`** (vpay#259, 2026-09-29): `justfile`,
> `Cargo.toml`, `vpay-db`'s `cratestack-codec-json`, both `install-cratestack-cli`
> pins in `ci.yml` and the twelve `Cargo.lock` entries. Policy reads now run on
> the caller's transaction (cratestack #1117), so a repeat `create_in_tx` inside
> one open transaction answers `Ok(None)`, **not** `PersistenceError::Denied`;
> `.do_nothing()` holds one connection, not two, so the `MAX_CONNECTIONS / 2`
> worker ceiling that `vpay-reconciler` explained by the second connection is now
> a conservative ceiling that vpay left alone on purpose. The drift totals did not
> move (201 changes over 26 relations, 19 unmappable). **Three 0.12.0
> measurements were not re-done by vpay and are now marked "unverified at
> 0.15.0" rather than restated**: that foreign keys are not introspected, that
> `jsonb`/`bytea` do not round-trip, and that the grammar has no `@@check(expr)`.

> **`just migrations-manifest` runs on macOS** (vpay#252, merged 2026-09-23).
> `vpay-data-layer` said the PR was open and the recipe failed there.

> **`@vaam-apps/ui` is `^0.4.0` in both apps**, not `^0.1.2` (`^0.2.4` from
> vpay#249 on 2026-09-23, `^0.4.0` from #258 on 2026-09-25, whose title says "the
> dashboard" and which also moved the checkout). `pnpm install` can fail on a
> brand-new `@vaam-apps/ui` with `ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION` until
> `pnpm-workspace.yaml`'s `minimumReleaseAgeExclude` names it.
> `vpay-checkout`'s story and axe numbers were measured at `^0.1.2` and have not
> been re-measured at `^0.4.0` in any vpay page; the page says so.

> **The Helm chart publish job has run.** `vpay-ops` said `publish-chart` "has
> never run". It ran on 2026-09-20: `v0.2.2` pushed an unsigned, mislabelled
> `0.2.1` and failed at `cosign sign` (helm and cosign do not share a credential
> store); `v0.3.0` pushed and signed cleanly. `Chart.yaml`'s `version:` is owned
> by release-please since 2026-09-20 and must equal the tag (`0.5.0`); the
> republish guard has four outcomes, not three (a published, unsigned chart is
> resumed, not refused); the chart package was measured **public**, which
> corrects the assumption that a first push is private. `cosign verify` has still
> never been run, and no cluster has installed the chart.

> **`config/application.yml` references ten environment variables, not seven.**
> The three `MTN_DISBURSEMENT_*` names arrived with vpay#178 on 2026-09-16.
> `vpay-ops` said seven; `vpay-mtn-momo` dated the move 2026-09-15.

> **Four vpay documentation claims this repository was quoting as "still wrong"
> were fixed on 2026-09-24 (vpay#255)**, a PR prompted by an earlier pass over
> these skills: `schemas/vpay.cstack`'s "FIVE models" header (now fourteen of
> twenty, 36 statements); the route probe's `MODEL_TABLES: [&str; 19]`, which
> had missed `manual_payments` (now `[(&str, &str); 20]`, checked against the
> macro's own `MODELS` in both directions); `resource-contract.md`'s "the only
> `DELETE`" (and its silence about the invoice routes); `InvoiceStatus`'s
> `draft ──void──> void` edge and `void_in_tx`'s "a `draft` or `open`";
> `provider-port.md`'s six-method table (now nine); `docs/flows/errors.md`'s
> two-leaf `exit_code_for`; `stripe-sdk-compat.md`'s "`customer` dropped";
> `parity.md`'s "iOS and macOS compiled by nobody". Where a skill told an agent
> to distrust one of these, it now says it is fixed. **Not fixed in vpay, and
> still named**: `checkout_sessions.rs`'s `run_until` comment ("seed_singletons
> stamps from this process's clock"), the `0.12.0` mentions in `schemas/vpay.cstack`
> and `postgres_smoke.rs`, and "both binaries" in `config_reconcile.rs`,
> `lock_keys.rs` and `repository.rs`.

> **Smaller, each dated where it is corrected:** `docs/status.md` is 653 lines,
> not "~290" (275 on 2026-09-11; `vpay`, `vpay-docs-status` and
> `reading-the-docs.md` all said ~290); `justfile` is 5 299 lines, not 4 704;
> `verify-ignored` is the sixth dependency of `just ci`, not "step 7"; `vpay`'s
> gate-count history had fourteen and twelve backwards;
> `vpay_api::provider_callback` also _enqueues_ the `poll_charge` job when none is
> queued (`vpay-reconciler` said it only pulls one forward); the confirm's poll
> job is committed `POLL_AFTER_CONFIRM_GRACE` out, not at `run_at = now()`;
> `@vaam-apps/ui` registers **two** themes (`dark`, `light`), not one;
> `EXPECTED_ASSERT_SITES` is 74, not 71; `customers.rs` is 28 cases, not 24, and
> `invoices.rs` 30, not 29.

### What did NOT change, and is the point

- **The banner.** Exactly one real rail call has ever been made (MTN's sandbox,
  2026-09-15). Orange has never been called, no webhook has reached a merchant
  outside the repository, no cluster has run vpay, and no rail has returned money.
  `orange_money::refund` is still the workspace's only declared `NotImplemented`
  token (re-grepped, not just re-read).
- **Fifteen gates and one report**, 49 migrations, 23 paths / 37 methods in
  `V1_ROUTES`, 20 `model`s of which 14 are queried (36 statements, recounted).
- **The dashboard skill's content.** vpay changed nothing it covers after
  `f68fda09` except the cratestack bump.

### Decided this release

- **Twenty-one flow pages are now claimed individually.** `customers/*`,
  `dashboard/*`, `dashboard-auth/*`, `hosted-checkout/*`, `webhooks/*` and
  `merchant-auth/verification-and-limits.md` were covered only by their
  directory's claim, which the gate honours only for pages that existed at the
  baseline, and only when the baseline commit is in the vpay clone. Each is now a
  claim in `coverage.json`, and each owning skill has a table saying where the
  page is covered. Four of them were **not covered at all** and now are:
  `dashboard-auth/rate-limiting.md`, `dashboard-auth/scope-and-tokens.md` and
  `dashboard-auth/sessions-and-refusals.md` became `vpay-dashboard`'s new
  `references/staff-auth.md`, and `merchant-auth/verification-and-limits.md`
  (the `jti` namespace limitation, which is an onboarding requirement) became a
  section of `vpay-merchant-api`. The rest were already described; the tables
  only say where.
- **`verify-coverage` reads vpay's working tree**, not its git history, for the
  path and page checks (it uses git only for drift, "existed at baseline" and
  stamp ancestry). Run it against a clean checkout of `master`.
  `vpay-docs-status`' parity page now says so.
- **ADR-0026 is described as "Accepted in part"**, which is what `master` says.
  A vpay PR confirming D1–D8 was in flight; when it merges, change the
  `vpay-conventions` row and the sentence in `vpay-data-layer` and
  `vpay-reconciler`. ADR-0028 is not listed; it lands with that PR.
- **The page budget was not touched.** `vpay-data-layer`, `vpay-customers`,
  `vpay-dashboard`, `vpay-merchant-api`, `vpay-ops` and `vpay-troubleshooting`
  grew; the reasons are the same as CONTRIBUTING's: negative claims and
  corrections belong where the agent will read them.

### Known discrepancy, not resolved here

The brief this release was written from counted **21 flow pages added after the
old baseline** and **14 stale stamps**. Neither is what the gate shows: one page
(`tauri-checkout.md`, already claimed) was added after `0799a8d2`, and the 21
are the detail pages that were covered by directory inheritance; moving the
baseline to `a33aac61` makes all twenty stamps stale at once, not fourteen.
vpay's own `CLAUDE.md` says `docs/status.md` "is 259 lines as of 2026-09-11";
`git` says 275 that day.

## v2026-09-16-7a79684e

Re-verified against vpay [`7a79684e`](https://github.com/vaam-apps/vpay/commit/7a79684e98afe8a7feeba16cc51da1334ac03b4a)
(2026-09-16), which is `93c6dfd0` plus exactly one commit:
[vpay#178](https://github.com/vaam-apps/vpay/pull/178), the refund path end to
end — destinations, the first ledger postings, five `/v1` routes, and both SDKs.

**The baseline moved for all twenty skills at once**, and unlike the previous
release that was not a formality: #178 is a behaviour change, not documentation.
Fourteen skills changed. Three were re-briefed by agents who owned disjoint file
sets (`vpay-provider-adapters`, `vpay-payments`, `vpay-merchant-api`;
`vpay-mtn-momo`, `vpay-orange-money`, `vpay-customers`; `vpay-sdks`,
`vpay-webhooks`, `vpay-invoices`), and a seam pass corrected the rest.

### Claims that stopped being true

> **`POST /v1/refunds` is no longer a `404`.** Five refund methods are mounted
> across three paths — create, retrieve, update, list, cancel. `V1_ROUTES` is
> **23 paths, 37 methods**, up from 21 and 33. Anything saying the create is
> unrouted, or that only the retrieve is mounted, is now false.

> **`vpay_db::Refunds::create` writes `refunds` rows.** The module is no longer
> "two reads and no write"; the `no_over_refund` CHECK is reachable from a
> merchant request for the first time.

> **The ledger has its first writer.** `vpay-db/src/ledger.rs` exists;
> `Settlement::apply_succeeded` posts a CAPTURE and `apply_refund_succeeded` a
> REFUND. `Transaction::validate()` now balances **per currency** — it summed
> across currencies before, and mixed-currency postings committed.

> **`mtn_momo::refund` is written** — a real `POST /disbursement/v1_0/transfer`.
> It is no longer a `NotImplemented` token.

> **Neither rail answers `Unsupported` for `refund` any more.** Orange's
> `supports_refunds` is `true` and its `refund` is a declared
> `NotImplemented("orange_money::refund")`. Every "Unsupported on Orange" is
> false, and the _reason_ changed as well as the value: an Orange refund is a
> transfer back, so its absence is work vpay owes rather than a fact about the
> rail.

> **`charge.refunded` and `charge.refund.updated` are emitted**, for the first
> time. Both were documented-but-never-emitted before.

> **`refunds.create` is no longer an SDK method with no route.**
> `balance.retrieve` is now the only one.

> **`verify-status` gained a third direction** — a `<rail>::…` token must be
> carried by that rail's crate — and reports **one** token, not two and not
> zero. Counts that moved: 48 migrations (was 44), `verify-sdk-parity`
> 603 proving tests / 36 dated gaps (was 550/35).

### What did NOT change, and is the point

**No rail in this repository has ever returned money to anyone.** Mounting a
route is not a rail call, and `docs/status.md`'s load-bearing banner is
unchanged. Specifically:

- **MTN's Disbursements product has never been called** — not in production,
  not against the sandbox, not once. `mtn_momo::refund` is WireMock-proven and
  rail-unproven, and no real Disbursements credential exists in the project.
- **Nothing settles a pending refund.** There is no refund poll ladder
  (RFC-0003 open question 8), so every refund these routes create stays
  `pending` forever, `invoices.amount_refunded` never moves, and `refunds.fee`
  is written by nothing.
- **A refund whose transfer was instructed cannot be cancelled** — a
  double-payout hole was closed. Since nothing settles refunds, cancel's
  remaining subject is a create that died before recording its attempt.
- **The destination is persisted nowhere.** No column; retention is RFC-0003
  open question 3, undecided. vpay cannot tell an operator which payee a refund
  went to. The rail's records can, by `provider_reference_id`.

### Decided this release

- **Page length.** Four `SKILL.md` files now exceed CONTRIBUTING's 100–150 line
  budget: `vpay-mtn-momo` (370), `vpay-orange-money` (312),
  `vpay-provider-adapters` (278), `vpay-payments` (267). They were **not**
  split. The reasoning is recorded in CONTRIBUTING.md item 2; in short, each is
  long because of a negative claim, and a reference page is one an agent _may_
  open. The budget was not raised, and whether it should move is left to the
  maintainer.
- **A test name that misleads.**
  `a_rail_without_the_refund_capability_answers_unsupported` describes an arm
  that runs on no rail — no rail declares `supports_refunds: false` since
  2026-09-15. vpay keeps the name because dated records cite it. Every skill
  that names it now says so; `vpay-mtn-momo` no longer counts it among the
  seven conformance cases without that caveat.

### Known discrepancy, not resolved here

The brief this release was written from gives `verify-links` as **1736**.
vpay's own `docs/status.md` gate table, re-run on 2026-09-16, says **1 731
links in 375 files** — measured on `888b00c3`, the last commit before the table
was filled in, not on the merge commit. Both may be right. No skill asserts
either number as of `7a79684e`; `vpay-tooling` quotes vpay's figure and names
the commit it was measured on.

## v2026-09-16-93c6dfd0

Re-verified against vpay [`93c6dfd0`](https://github.com/vaam-apps/vpay/commit/93c6dfd0cba6237bf581c6d68d40a8b70f003d65)
(2026-09-16), which is `f063ee96` plus
[vpay#179](https://github.com/vaam-apps/vpay/pull/179) (the docs↔skills parity
rule) and [vpay#180](https://github.com/vaam-apps/vpay/pull/180) (audit-web's
prose).

**The baseline moved for all twenty skills, and here is the basis for that.**
Both merged PRs are documentation-only — verified, not assumed: the `justfile`
diff between `f063ee96` and `93c6dfd0` contains zero non-comment lines, and
`package.json` is byte-identical once its `//pnpm` prose array is excluded. No
behaviour any skill describes changed, so the nineteen skills untouched by this
release remain accurate at the new baseline. The one that did change is
`vpay-tooling`, below.

### Claims that stopped being true

> `vpay-tooling` and `vpay-troubleshooting` said `just audit-web` was **not** in
> `just ci` and that `just ci` ran offline, and that the audit ceiling was
> `--audit-level=high`. Those were accurate readings of vpay's prose and wrong
> about vpay's behaviour — the recipe had said otherwise since 2026-09-11
> (issue #103). vpay#180 fixed the prose; these skills now say `audit-web` is in
> `just ci`, that moderates fail, and that **`just ci` needs the network**.

The two discrepancies were `vpay-tooling`'s headline evidence for its authority
rule ("the recipe body wins over its own comment"). They are recorded as spent
rather than deleted, because finding them is what the rule was worth.

Newly carried: issue #103's allowlist for an accepted advisory is **not built**,
so a moderate advisory in a transitive dev dependency blocks every merge with no
sanctioned way to accept it.

## v2026-09-16-f063ee96 — first release

Verified against vpay [`f063ee96`](https://github.com/vaam-apps/vpay/commit/f063ee9647867699f3b12622a62ac0004609373b)
(2026-09-15), the commit that recorded vpay's first settlement against a real
MTN sandbox.

~~Twenty skills, covering all 23 pages in vpay's `docs/flows/`.~~
**Corrected 2026-09-19:** `docs/flows/` has 45 pages (46 counting
`README.md`), not 23. 23 is `verify-coverage`'s own shallow,
one-level-deep count — the exact defect this same entry's "Reviewed"
section below already names as found and fixed before this release
("`verify-coverage` enumerated `docs/flows` one level deep, seeing 23
pages of 45, while reporting success"). The gate was fixed; this
headline sentence, written before that fix, was not.

### Reviewed

Two adversarial review passes before first publication, with distinct lenses:
factual accuracy against the tree, and fitness for purpose. Both were run
against vpay `f063ee96`.

The factual sweep mechanically extracted and grepped every backticked identifier
across all twenty skills — 339 file paths, 429 snake_case names, 146 constants,
351 CamelCase names, 26 env vars, 11 metric names — and every one resolves,
except two whose absence _is_ the claim (`SQLX_OFFLINE`, `ChargeObject`). All 35
route claims are mounted.

Nine factual defects and six fitness defects were found and fixed before this
release. The worst was in the gate itself: `verify-coverage` enumerated
`docs/flows` one level deep, seeing 23 pages of 45, while reporting success.

### Claims that stopped being true

Nothing yet — this is the first release. Future entries go here, in the shape:

> `vpay-merchant-api` said `POST /v1/refunds` was unrouted. Routed since
> `<date>` (`#<PR>`). If you generated code against the old claim, it 404'd.

### Known to be true only of this commit

Called out because they are the claims most likely to age, and an agent on a
different tree needs to know which ones to re-check first:

- **One real rail call has ever been made** — a EUR `mtn_momo` intent settled
  against MTN's sandbox on 2026-09-15. Orange has never been called. Before
  `f063ee96`, no real rail call had been made at all, and
  `docs/runbooks/live-sandbox-test.md` and `config/application-live.yml` did not
  exist.
- **`mtn_momo::refund` is the workspace's only `NotImplemented` token.** It was
  eight on 2026-09-03.
- **`just verify` is twelve gates.** This count has been wrong in vpay's own
  prose at nearly every value it has held — three, five, seven, nine, ten,
  eleven. Read the `verify` recipe, not any document.
- **`rust-toolchain.toml` pins `1.98.0`** — it was `1.95.0` until 2026-09-05.
- **`@vpay/ui` was deleted on 2026-09-12**; both apps compose the published
  `@vaam-apps/ui`. On an older tree the package exists and is imported.
- **`vpay-server` is one binary with three modes** since 2026-09-07 (issue #77).
  It was two binaries and two images before that, and the retired
  `ghcr.io/vaam-apps/vpay-worker` package still answers a `docker pull`.
- **`schemas/vpay.cstack` compiles into `vpay-db`** since 2026-09-06. Before
  that a syntax error in it was not a build failure.
- **The schema drift constants are 190 changes over 25 relations**, read from
  source on 2026-09-16 rather than re-measured.
