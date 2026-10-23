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

## 2026-10-23 — Fapshi, a desk study with no adapter

Read against vaam-apps/vpay's `master` at `be31abcd`, the merge of
vaam-apps/vpay#342, the pull request that adds `docs/plans/2026-10-23-fapshi-evaluation.md` and the
`fapshi` row of `docs/status/providers.md`. The maintainer asked to "study
fapshi as another payment provider too". The study reads Fapshi's public
documentation only: no account, no key, no call.

- `vpay-provider-adapters` (`SKILL.md`): `fapshi` joins the matrix's rows
  with no RFC and no code; a paragraph on what the study found (all four hard
  rules on paper, rule 3 as ADR-0029 wrote it, and four blockers), and an
  instruction not to start an adapter before the written questions are
  answered and the maintainer asks; the page in "Where to go next".

### Stopped being true

- "A row per provider (`mtn_momo`, `orange_money`, `notchpay`, `stripe`,
  `paypal` and `flutterwave`)" in `docs/status/providers.md`: it has a
  seventh row, `fapshi`. Nothing a skill said about any other row changed.

## 2026-10-23 — an answer only a person can act on escalates at once (vaam-apps/vpay#313)

Read against vaam-apps/vpay's `master` at `c672fffc`, the merge of
vaam-apps/vpay#341, which closes vaam-apps/vpay#313 and whose branch was cut from
`8504536f` (the merge of vaam-apps/vpay#340). The maintainer, on #313's three
options: "Go with option 1 on #313, build it". vpay's evidence page is
`docs/status/verification/2026-10-23-needs-reconciliation.md`.

- `vpay-conventions` (`references/errors.md`): the seventh `ProviderError`
  variant, `NeedsReconciliation`, and how to choose between it and
  `Malformed`; the worker's two handlers that re-classify a leaf.
- `vpay-reconciler` (`SKILL.md`): `unresolved` on the first
  `NeedsReconciliation`, from the poll and from the resubmit, and what each
  writes to `provider_requests`.
- `vpay-notchpay` (`SKILL.md`): the three answers that moved (the short
  collection and its siblings, the `409` with no `trx.…`, the read with no
  `trx.…`) and condition 5's wording.
- `vpay-orange-money` (`SKILL.md`): a missing `pay_token` is
  `NeedsReconciliation`, not `Config`.
- `vpay-provider-adapters` (`SKILL.md`): `Malformed` versus
  `NeedsReconciliation`.
- `vpay-ops` (`SKILL.md`, `references/observability.md`): the new
  `error_kind`, and an `unresolved` charge that is not 24 hours old.
- `vpay-notchpay`, `vpay-provider-adapters`: retiring a superseded or
  duplicate Notch Pay webhook endpoint is an operator's task, by the
  maintainer's decision ("#12 is nice but an operator's task only, not
  code"), and leaves the "Not built" list.
- `vpay-frontend`, `vpay-dashboard`, `vpay-checkout`: Next `^15.5.28` in the
  two vpay apps and 16.4.2 in `examples/shop`, the bump in the same pull
  request that clears GHSA-2464-fhr4-w9xw, GHSA-p3m5-59mp-jv5r and
  GHSA-673q-mjm6-r5xr (vpay's
  `docs/status/verification/2026-10-23-npm-advisories.md`).

### Stopped being true

- "`Malformed` is a `Warn`-severity rail error, so the worker retries it on
  the ladder and wakes nobody; only at the 24 h poll horizon does the
  charge become `unresolved`" — about Notch Pay's short collection. It is
  `NeedsReconciliation`, and the charge is `unresolved` after its first such
  read.
- "With none held, `Malformed`" (Notch Pay's `409`). `NeedsReconciliation`,
  and the resubmit escalates the charge instead of riding the ladder.
- "A read of a charge with no `notchpay_reference` is `Config`" and "a
  missing `pay_token` is `Config`". Both are `NeedsReconciliation`, which
  keeps the poll job instead of dead-lettering it.
- "Anything else is `Malformed`" (Notch Pay's condition 5, a success for
  another amount or currency). `NeedsReconciliation`.
- "The loop does not inspect variants" was true and still is; two handlers
  do, and the skill now names them.
- "Retiring or repairing a webhook endpoint (no working `DELETE`, no `PUT` in
  the client)" as a Notch Pay gap. It is an operator's task in Notch Pay's
  dashboard, not code vpay owes.
- "Next `^15.5.27`" (the two vpay apps) and "16.3.8" (`examples/shop`).
  `^15.5.28` and 16.4.2.

Still true: `Malformed` is still an answer the adapter could not read, still
`Warn`, still the ladder. And a real Notch Pay short collection has never been
seen reaching `unresolved`: the proof is WireMock and a real Postgres.

## 2026-10-22 — vpay registers Notch Pay's callback endpoint itself (RFC-0013 steps 2–4)

Read against vaam-apps/vpay's `master` at `8504536f`, the merge of
vaam-apps/vpay#340 (RFC-0013 delivery steps 2–4), whose branch was cut from `e8e70ef` (the merge
of vaam-apps/vpay#338). The maintainer asked that "the webhook registration should
be done automatically then, e.g via cli or at startup fo the providers that
supports it". vpay's evidence page is
`docs/status/verification/2026-10-22-rfc-0013-callback-registration.md`.

- `vpay-provider-adapters` (`SKILL.md`, `references/adding-a-rail.md`): the
  port's eleventh method, `ensure_callback_endpoint`, and the capability
  `callback_url_mode` exist, with `callbacks_are_coherent` kept apart from
  `is_coherent`; the RFC-0013 entry says what was built and where it differs
  from the RFC; two boot refusals join the table.
- `vpay-notchpay` (`SKILL.md`): the webhook endpoint is registered by vpay,
  recognised by its description because Notch Pay answers `"url": null`,
  and never modified or deleted; the sandbox profile sets
  `callbacks.register: check`; "Registering the webhook (RFC-0013)" leaves
  the "Not built" list.
- `vpay-ops` (`references/observability.md`, `references/configuration.md`):
  a fourteenth metric, `vpay_callback_endpoint_checks_total`, and the
  `providers[].callbacks.register` setting.
- `vpay-tooling` (`references/recipes.md`): the `callbacks ensure` subcommand
  and its `--check`.
- `vpay-reconciler`, `vpay-troubleshooting` (`references/docker-testcontainers.md`):
  six singletons, not five.
- `vpay-data-layer`: the next migration is `0052`.
- `vpay-merchant-api` (`references/routes.md`), `vpay-conventions`
  (`references/adr-index.md`): dated notes on two "registered by hand" and
  "nothing in it is built" passages.

### Stopped being true

- "RFC-0013 proposes a tenth method, `ensure_callback_endpoint`; it does not
  exist." It exists, as the eleventh.
- "RFC-0013 proposes one more [capability], `callback_url_mode`; it does not
  exist." It does.
- "The [Notch Pay webhook] URL is registered with the account by hand.
  RFC-0013 is not built." vpay registers it.
- "`seed_singletons` enqueues its five singletons." Six.
- "The next change is `0050`." `0050` and `0051` exist; the next is `0052`.

Still true, and worth repeating: vpay has **never** registered an endpoint
with a real Notch Pay account (RFC-0013 step 5 is not done), and it cannot
see an endpoint someone registered by hand at its URL.

## 2026-10-22 — vpay corrects fourteen self-contradictions; four skills repeated one

Read against vaam-apps/vpay's `master` at `e7406dda`, the merge of
vaam-apps/vpay#333 (`v0.10.2-4-ge7406dda`; first released in `v0.10.3`).
`verify-coverage` also passes against `master` at `760fbb9` (`v0.10.4`) and
`e8e70ef` (vaam-apps/vpay#338). Nothing after `e7406dda` touches a passage
cited below. #333 corrects
fourteen places where vpay's docs contradicted its own code at `v0.10.2`, as
dated notes. Its evidence page is
`docs/status/verification/2026-10-22-self-contradictions.md`. That page names
two skills that repeat an old claim. A grep of every skill for the other
twelve found two more.

- `vpay-tooling` (`references/ci.md`): "`notify-docs.yml` has not run on a
  real tag" is struck. It has run on every `v*` tag from `v0.5.0` to
  `v0.10.2`, and every run succeeded. Each was followed by a vpay-docs
  `release-parity` run whose event is `repository_dispatch`. The note cites
  vpay's `docs/runbooks/release.md` § 9 and the evidence page, which has the
  run ids, and does not repeat them. `v0.10.3` and `v0.10.4`, both cut after
  #333, ran green as well, each followed by a dispatch run within twenty
  seconds (`gh run list`, 2026-10-22).
- `vpay-dashboard` (`references/read-seam-and-bff.md`, `SKILL.md`): the
  README's "There is no current password field" is marked fixed by #333, the
  way the "Sign out" entry is marked fixed by #318. Four README passages are
  stale now, not five. The other four and the "22 files, 243 tests" count are
  still in the README at `e7406dda`.
- `vpay-orange-money` (`references/unverified.md`): "one assumption among
  ten" is struck. The list has had eleven items since vaam-apps/vpay#319
  (2026-10-21). #333 made the same correction in vpay's
  `docs/flows/adapter-orange-money.md` Status.
- `vpay-ops` (`SKILL.md`): "What has run" named only MTN's sandbox. A dated
  note adds Notch Pay's, run on 2026-10-21 through
  `docs/runbooks/notchpay-sandbox-test.md`, and the three B3b payments of
  2026-10-22 (vaam-apps/vpay#338). #333 changed vpay's
  `docs/runbooks/README.md` from "the one page" run against a real rail to
  "the first".

Checked and left alone, because no skill repeats the old claim:

- `vpay-dashboard` already had the SSO and `HOME_PATH` landing on
  `/overview`, the customers list without a filter, the Overview's twelve
  counts, and the payments screens' paging read by the server (all from
  #318).
- `vpay-notchpay` already describes the short collection as read and never
  settled.
- `vpay/SKILL.md` already narrows "every settlement came from WireMock" to
  MTN's and Notch Pay's sandboxes.
- The vpay-docs paragraph in `README.md` does not say `release-parity` opens
  an issue.
- No skill says no Notch Pay status has been read from Notch Pay, or that no
  real rail has declined a charge. `vpay-sdks/references/flutter-plugin.md`
  already says every walk in its lane used WireMock, and that MTN's sandbox
  evidence does not prove a checkout through the plugin.
- `vpay-provider-adapters/SKILL.md`'s "the poll ladder settles a charge whose
  callback never came" is not the claim #333 corrected ("settles **every**
  charge"). It cites RFC-0013 § "What a missed callback costs", which says a
  missed callback costs one rung and never an outcome.

The `SKILL.md` stamps did not move. Each edited passage carries its own date
and cites #333 or the vpay file it now matches.

**Stopped being true.**

- "`notify-docs.yml` … has not run on a real tag" (`vpay-tooling`,
  `references/ci.md`). It has run on eleven as of 2026-10-22, `v0.5.0` to
  `v0.10.4`.
- "There is no "current password" field" in the dashboard README, listed as
  still wrong (`vpay-dashboard`, `references/read-seam-and-bff.md`). The
  README strikes it since #333.
- "~~seven~~ five of its passages are stale" (`vpay-dashboard`, `SKILL.md`
  and `references/read-seam-and-bff.md`). Four are stale.
- "not one assumption among ten" (`vpay-orange-money`,
  `references/unverified.md`). There are eleven.
- "What _has_ run", naming MTN's sandbox alone (`vpay-ops`, `SKILL.md`).
  Notch Pay's ran too, on 2026-10-21.

## 2026-10-22 — Notch Pay's timeout, canceled and Orange numbers through vpay (runbook B3b)

Read against vaam-apps/vpay's `master` at `e8e70ef`, the merge of
vaam-apps/vpay#338.

- `vpay-notchpay` gains a bullet on what runbook B3b showed. It ran on the
  maintainer's MacBook and approval, through `vpay-server`, its worker and
  Postgres, with `livemode=false` and test keys:
  - `+237670000003` (MTN timeout, 240 XAF) read `expired`: charge `failed`,
    `payer_timeout`.
  - `+237670000004` (MTN canceled, 250 XAF) read `canceled`: charge
    `failed`, `payer_declined`.
  - `+237690000000` (Orange success, 260 XAF) `succeeded`, with a balanced
    260 XAF capture.
  - Each was terminal on the worker's second read, 11 to 12 s after create.
    vpay sent 3 submits and 6 reads. Nothing differed from `mapping.rs`.
- `vpay-notchpay`'s "Still open" list now follows step 5 and B3b. It drops
  Orange, timeout and canceled through vpay, and names Orange's failure
  path, which Notch Pay documents no test number for.
- `vpay-notchpay`'s Part B bullets gain B3b's amounts and timing, and runbook
  B0's loopback-only Postgres.
- `vpay-notchpay`'s `providers.md` paragraph: the cells' evidence covers an
  Orange payer, and the cell wording is unchanged.

Checked on `e8e70ef`:

- `docs/status/verification/2026-10-22-notchpay-b3b.md` exists.
- `adapter-notchpay.md` § Status has the B3b bullet, and its "To confirm"
  item 5 carries the through-vpay result.
- The runbook's B0 publishes `127.0.0.1:5434:5432`.
- `mapping.rs` is unchanged since `e7406dd`.

**Stopped being true.**

- "Orange, timeout and canceled through vpay (step 1's probe ran them; step 5
  did not; runbook B3b is written for them)" in `vpay-notchpay`'s "Still
  open after step 5" list. All three ran through vpay on 2026-10-22.
- "Orange through vpay" in the list of what `providers.md`'s `notchpay` cells
  do not cover.
- Runbook B0's `docker run … -p 5434:5432`, as a command to copy: it
  published Postgres on every interface. It is `-p 127.0.0.1:5434:5432` now.

## 2026-10-22 — RFC-0009 revised: one company first, connected accounts later

Read against vaam-apps/vpay's `master` at `facbb67`, the merge of
vaam-apps/vpay#336.

- `vpay-provider-adapters`, § "`ProviderConfig` is per deployment today":
  - a new paragraph: what decides aggregation is whose money it is, not how
    many `merchant_id`s there are;
  - the operator's rule is now one deployment per legal entity, on the
    maintainer's 2026-10-22 decision that the merchants are projects of one
    company;
  - nothing in code enforces it yet, and counsel has not been asked.
- `vpay-provider-adapters`, the RFC-0009 bullet under "Proposed port changes":
  - phase A: `deployment.legal_entity`, a `legal_entity` per merchant, a boot
    refusal for a second company, and `charges.provider_account_id` set to
    `default`;
  - phase B, on Stripe Connect's shape: accounts, the platform client and
    `Stripe-Account`, a separate onboarding application, per-account
    callbacks, sealed secrets, and application fees (the one trait change,
    blocked on an ADR amending ADR-0029 D1 rule 1 and on counsel).

Checked on `facbb67`: nothing of RFC-0009 is built.

- `Stripe-Account` is still accepted and ignored.
- `application_fee_amount`, `transfer_data` and `on_behalf_of` are still
  refused with a `400`.
- No source file names a provider account or a legal entity.

**Stopped being true.**

- The RFC-0009 bullet's "**No trait method changes**", as a description of
  the whole RFC. It still holds for phase A and for § 2. Phase B's
  application fee would change the submit request.
- "Run one deployment per merchant" (vaam-apps/vpay `docs/status/backend.md`).
  It is now one deployment per legal entity.

## 2026-10-22 — Notch Pay's sandbox answers `403` to `POST /refunds`; refused reads are recorded as answered

Read against vaam-apps/vpay's `master` at `9086ba4`, the merge of
vaam-apps/vpay#334.

- `vpay-notchpay` gains a bullet: the sandbox refuses `POST /refunds`. On
  2026-10-22 `sandbox_probe refunds` sent it four times against a
  `complete` 200 XAF payment, through the probe and not the adapter. Every
  answer was `403` `{"code":"403","status":"Forbidden"}`, with no message,
  and nothing was refunded.
  - A hand-made `GET /refunds` answered `200` with an empty list.
  - `GET /payments/{trx}/refunds` answered `404`.
  - So RFC-0012 § 10 items 6 and 12 stay open. The bullet says not to build
    refunds on this evidence.
  - The "Not built" section and the "Still open after step 5" list now
    carry the same date. That list also names the timeout and canceled
    numbers beside Orange, and runbook B3b, written for all three.
- `vpay-notchpay`'s `provider_requests` bullet now says a refused status
  read is recorded as answered, `status_code` 0 with its `error_kind`, since
  vaam-apps/vpay#314 (`6d9dfbf`). A refused submit stays `NULL`.

Checked on `9086ba4`:

- `notchpay::refund` is still declared in `docs/status.md`.
- `docs/status/verification/2026-10-22-notchpay-refunds.md` and its report
  exist.
- `adapter-notchpay.md` § Status carries the same #314 correction.

**Stopped being true.**

- "`provider_requests` records that refused read as "no answer"
  (`status_code` and `responded_at` NULL)" (`vpay-notchpay`). It stopped
  being true when #314 merged on 2026-10-21, a day before this skill caught
  up. vpay's own `adapter-notchpay.md` said the same until #334.

## 2026-10-22 — RFC-0013 step 0 complete: `scan:live` is not hourly, and MTN names no callback allowlist

Read against vaam-apps/vpay's `master` at `1b2543b`, the merge of
vaam-apps/vpay#326. This entry stamps #59, which merged without one. #55's
entry (2026-10-21, read against `90fe1b9`) still says items 4 and 5 are open,
as they were then.

- `vpay-troubleshooting` (`references/known-wrong-docs.md`): a dated update
  says vaam-apps/vpay#326 corrects RFC-0013 § 11 items 4 and 5, which completes
  delivery step 0.
- `vpay-ops` and `vpay-troubleshooting` (`references/deployment.md`): the
  legitimate reason to turn the `/provider` Ingress off is now a callback host
  of the deployment's own. "MTN allows an IP-allowlisted callback host of its
  own" is struck. For MTN the host must be on the domain registered as the API
  user's `providerCallbackHost`.
- `vpay-conventions` (`references/adr-index.md`): items 4 and 5 are corrected
  too.

Checked on `1b2543b`:

- `SCAN_INTERVAL` is ten minutes and `SWEEP_INTERVAL` an hour
  (`vpay-worker/src/handlers.rs`). `reconciler.md`, `crash-safety.md` and the
  worker-queue runbook each say ten minutes for `scan:live`, with a dated
  correction.
- The allowlist reason is corrected, with a dated note, in every place it
  appeared:
  - `vpay-config`'s `callback_url` comment;
  - the chart's `values.yaml`, `ingress-provider.yaml`, `_validate.tpl` and
    `README.md`;
  - `docs/reference/rails.md`;
  - the MTN `requesttopay` WireMock mapping.

  Outside dated corrections and RFC-0013 § 11's own text, no tracked file says
  it any more.

- `mtn_momo` still declares `requires_ip_allowlist: true`. Its comment gives
  MTN's best-practices entry for Disbursements (what `refund` calls), and says
  no MTN page names an allowlisted callback host.

**Stopped being true.**

- "Items 4 and 5 are still uncorrected on `90fe1b9`" (this file, #55's
  2026-10-21 entry). Both are corrected on `1b2543b`.
- "The IP-allowlist comment and the "hourly" are still as described above"
  (`vpay-troubleshooting`, `references/known-wrong-docs.md`). The dated update
  that follows it supersedes it.

## 2026-10-22 — the Flutter example's lockfile sync is confirmed by three releases

Read against vaam-apps/vpay's `master` at `90fe1b9` (release `v0.10.2`). The
vpay side is vaam-apps/vpay#329, which flips
`docs/status/infrastructure.md`'s row to ✅.

- `vpay-sdks` (`references/flutter-plugin.md`): the example's `pubspec.lock`
  bullet gains an observed note. `v0.10.0`, `v0.10.1` and `v0.10.2` each
  carried release-please's lockfile sync commit, and the line moved with
  `pubspec.yaml` every time with no hand edit.

**Stopped being true.** vpay's "no real release has run it yet" on the
`release-please.yml` step that #309 added. No skill said it; this entry
records that the claim the 2026-10-21 entry rests on is now proven.

## 2026-10-21 — MTN sends a callback once; `PUT`/`POST` and Orange's `order_id` are open

Read against vaam-apps/vpay's `master` at `90fe1b9` (release v0.10.2), which
holds vaam-apps/vpay#319 (`0706cd5`). No commit after `0706cd5` touches the
vpay files these notes cite.

- `vpay-mtn-momo`: MTN sends each callback once and never retries it, and
  recommends polling, which the poll ladder already does. MTN's pages disagree
  on the callback's method (`POST` or `PUT`). The route mounts `POST` only, and
  that is "To confirm with MTN" item 1 in `docs/flows/adapter-mtn-momo.md`,
  not something to change from the documentation.
- `vpay-orange-money`: whether Orange's notification carries `order_id` at all
  is unconfirmed (`references/unverified.md` row 11, with pointers from
  `SKILL.md` and `references/the-three-calls.md`). If it does not, every real
  Orange callback is a `400` and the poll ladder settles the charge anyway.
  The repair is a design decision, not relaxing `parse_callback`'s check.
- `vpay-merchant-api` (`references/routes.md`): the callback route is `POST`
  only, and a `PUT` would get axum's bare `405`. The vpay texts that said a
  rail retries a callback now carry a dated correction.
- `vpay-troubleshooting` (`references/known-wrong-docs.md`) and
  `vpay-conventions` (`references/adr-index.md`): vaam-apps/vpay#319 is
  RFC-0013's delivery step 0 for § 11 items 1–3. Items 4 and 5 are still
  uncorrected on `90fe1b9`: `docs/flows/reconciler.md` still calls `scan:live`
  "hourly" (line 120), and `vpay-config/src/config.rs` still says MTN requires
  an IP-allowlisted callback host (line 149).

Checked on `90fe1b9`: both "To confirm" items exist with the numbers cited;
`provider_callback.rs` documents `POST` only; and #319 changed every place
`known-wrong-docs.md` lists (`docs/api/README.md`, `docs/status/backend.md`,
`values.yaml`, `vpay-db` and `vpay-worker`'s `jobs.rs`, the callback
integration test). RFC-0013's dated note says step 0 landed for items 1–3 and
not for 4 and 5.

**Stopped being true.**

- "vpay has not yet corrected them" (`vpay-conventions`, of RFC-0013 § 11's
  findings). vaam-apps/vpay#319 corrects items 1–3.
- Any reading that MTN, or "both rails", retries a callback that is not
  answered with a 2xx. MTN documents that it does not, and no Orange source
  says either way.

## 2026-10-21 — a declined confirm settles before it records the attempt

Read against vaam-apps/vpay's `master` at `b7cc6c9`, the merge of
vaam-apps/vpay#322.

- `vpay-reconciler`: the `provider_requests` paragraph now says `vpay-api`'s
  `finish_confirm` settles a decline (`persist_decline`) **before** it
  completes the `submit` row, as the worker's resubmit already did. If
  completing the row fails after that, it logs and still answers `409`
  `charge_declined`. It also says why the accepted arm keeps the row first:
  there `submitting` + answered is `Advance`'s input. It warns against making
  the two arms match. Checked on `b7cc6c9`: in `payment_intents.rs` the
  `Rejected` arm calls `persist_decline` before `record_response`, and logs
  with `tracing::error!` when the second write fails.

**Stopped being true.**

- "`vpay-api`'s confirm does the opposite order (row, then `persist_decline`,
  two commits), so a crash between them can" (`vpay-reconciler`). Since
  vaam-apps/vpay#322, a refused or interrupted settlement leaves the charge
  `submitting` with the row unanswered. Recovery already resolves that pair.

## 2026-10-21 — the dashboard's theme script runs first in `<body>`, not in `<head>`

Read against vaam-apps/vpay's `master` at `63c395d`, the merge of
vaam-apps/vpay#323.

- `vpay-dashboard` (`references/what-a-screen-may-show.md`): `app/layout.tsx`
  runs `themeInitScript` as the first child of `<body>` and renders nothing
  in `<head>`. The page says why: in `<head>` the script was the one node
  React expected there, Cypress's document puts a whitespace text node in
  front of it, and hydration failed (React #418). `dashboard.cy.ts` was 17 of
  18 red on vpay `master` at `0706cd5`. It names `src/layout.test.tsx` as the
  guard, and it says not to move the script back or add a React-rendered
  child to `<head>`.

**Stopped being true.**

- "`app/layout.tsx` runs the package's `themeInitScript` in `<head>`"
  (`vpay-dashboard`, `references/what-a-screen-may-show.md`). It runs first
  in `<body>`.

## 2026-10-21 — restamps: three entries merged before the vpay change they describe

Read against vaam-apps/vpay's `master` at `e393116`, the merge of
vaam-apps/vpay#307 (release v0.10.0). It holds vaam-apps/vpay#309
(`f554f87`), vaam-apps/vpay#314 with vaam-apps/vpay#316 (`6d9dfbf`) and
vaam-apps/vpay#315 (`8796705`).

vaam-apps/vpay-skills#50, #52 and #53 merged between 13:05 and 13:07 UTC on
2026-10-21, each before the vpay pull request it describes: #309 merged at
13:07:46, #314 at 13:19:46 and #315 at 13:32:25. For those minutes these
skills described code that `master` did not have. For #53 that lasted 27
minutes, and the code was a `VpayInvalidParamsError` class that `sdks/nodejs`
did not export. `tools/verify-coverage.mjs` passed throughout, because it
checks paths and names and not a class name. #52 and #53 also merged with no
entry in this file. No claim changes here. Each one is now stamped against
the `master` commit that makes it true, and one record that #50 said it made
is made here.

- `vpay-reconciler` (#52): what a `provider_requests` row says after an error
  is chosen by whoever writes it. Read against `6d9dfbf`, the merge of
  vaam-apps/vpay#314. #314 carries vaam-apps/vpay#316, which merged into
  #314's branch rather than into `master`. Spot-checked on that commit:
  `record_read_failure`, `read_failure_status` and `settle_failed_with` are in
  `backends/crates/vpay-worker/src/handlers.rs`.
- `vpay-sdks` (#53): Node's local refusals are a `VpayInvalidParamsError`.
  Read against `8796705`, the merge of vaam-apps/vpay#315. Spot-checked on
  that commit: `sdks/nodejs/src/errors.ts` exports
  `VpayInvalidParamsError extends VpayError`.
- `vpay-sdks` (`references/parity.md`): records `verify-sdk-parity` as
  `master` prints it. It was **778** proving tests at 45 dated gaps on
  `73a5b5a`, **788** at 45 on `f554f87` (#309) and **790** at **44** on
  `8796705` (#315) and on `e393116`, with 35 methods across 41 rows
  throughout. Its "778 to 780" was measured on #315's branch, which did not
  hold #309.
- `vpay-sdks` (#50): restamped from #309's branch to `f554f87` in its own
  entry below. That entry's claim that `references/parity.md` "records the two
  new payer rows and the 778 → 788 move" was not true of the merged change,
  which never touched that file. It is corrected in place.

**Stopped being true.**

- "Read against vaam-apps/vpay#309's branch … Merge this after #309" (this
  file, the dot-segment entry). It merged before #309, and it is read against
  `f554f87` now.
- "`references/parity.md` records the two new payer rows and the 778 → 788
  move" (the same entry). It did not, until this entry.
- `references/parity.md`'s "from 778 to **780** proving tests and from 45 to
  **44** dated gaps, measured against vpay `master` `73a5b5af` and the
  branch", read as the current count. That was #315's branch. `master` prints
  790 and 44.

## 2026-10-21 — the dashboard's Material 3 Expressive redesign, Overview home, and no more reloads

Read against vaam-apps/vpay's `master` at `73c3a7cf`, the merge of
vaam-apps/vpay#318. Its measurements ran against a fake of the `/dash/v1`
contract, not a real vpay, and `dashboard.cy.ts` was updated and not run; this
release repeats those numbers and re-ran none of them.

- `vpay-dashboard`: restamped `73c3a7cf`. `/` now redirects to `/overview`
  (`HOME_PATH`), a new route of twelve exact procedure counts that each link
  to the filtered list; new sections on why navigation no longer reloads the
  document (the rail's plain anchors routed client-side, `authProvider.check`
  without a request, `staleTime: Infinity` on server-seeded queries, the shell
  mounted once in `app/(dash)/layout.tsx` with `requireStaff` memoised per
  request) and on the Overview; the rail's Overview top item and sections;
  the merchant always on screen in the top app bar; Sign out as an outlined
  button; `@vaam-apps/ui` `^0.5.0`.
- `vpay-dashboard` (`references/what-a-screen-may-show.md`): the three new
  status systems, `themeInitScript` and the self-hosted fonts, the 0.5.0
  import list, filter chips that are links, the date filter's new width and
  Save-to-apply, the top app bar, six rail destinations, a `loading.tsx` on
  every route, `ProcedureListScreen`, 36 stories, and three new decisive
  mutations from #318's own record.
- `vpay-dashboard` (`references/read-seam-and-bff.md`): `searchPaymentIntents`
  has a caller (the Overview, twelve parallel procedure calls per render);
  `formatIsoInstant` moved to `src/format.ts`; the README's stale passages are
  five, not seven, and its latest suite figure is 455 passed, 1 skipped, in 35
  files.
- `vpay-frontend`: restamped `73c3a7cf`. `@vaam-apps/ui ^0.5.0` in both apps,
  the `minimumReleaseAgeExclude` entry for 0.5.0, the dashboard's self-hosted
  fonts, the `@vpay/tokens` vocabularies, and the re-pinned styling gates. Also
  pnpm `11.18.0`, which this skill had as `9.15.0` since vaam-apps/vpay#217
  (2026-09-19).
- `vpay-checkout` (`references/testing-and-gates.md`): the checkout moved to
  `^0.5.0` and was not redesigned; its styling gate, contrast helper and
  forward-button reference ratio (16.34 → 6.109) changed, and merchant
  branding on 0.5.0 is unmeasured.

**Stopped being true.**

- "`/` redirects to `/payments`", and a signed-in staff member landing on
  `/payments`. They land on `/overview`.
- "The payments list and the payment detail … later reads, paging and
  refetches, go through `dashDataProvider("/api/dash")`", and "deleting the
  route files and `src/server/bff.ts` now breaks the payments screens'
  paging". The server reads every page and #318 measured zero BFF fetches. The
  provider still points at the BFF, so deleting it is still not shown safe.
- "`/refunds`, `/deliveries`, `/customers`, `/checkouts`: … no filters".
  Refunds and checkouts filter by `status`, deliveries by `state`.
- "`payments-filters.tsx` uses a native `<select>`" and its Apply button. The
  status filter is chips that are links, and the date range applies on Save.
- "Below 640px [the merchant] has been one tap away". It is in the top app bar
  at every width.
- "Its `secondary` flag hides a column … as a static class per column". It is
  `Table`'s `hideBelow` prop.
- `w-[calc(23ch+16px+3.5rem)]` for the date trigger (now `25ch`), `pb-20` on
  `<main>` below 640px (now `pb-24`), "the phone bar's sheet holds … (Customers)"
  (now Deliveries and Customers), and "the Storybook has 33 stories" (now 36).
- The `#0a0b0d` dark `base-100` the styling gates pinned. 0.5.0 has no such
  literal.
- "pnpm 9.15.0" (`vpay-frontend`), stale since 2026-09-19.

## 2026-10-21 — the payer surfaces' dot-segment safety is pinned by test

Read against vaam-apps/vpay's `master` at `f554f87`, the merge of
vaam-apps/vpay#309, whose tree is identical to the branch head `93c5eb8` this
was first read on. _(This said it was read against #309's branch
(`claude/admiring-swirles-9b668f`, `93c5eb8`, with vpay `master` merged in at
`73a5b5a`) and "Merge this after #309" until 2026-10-21. It merged 51 seconds
before #309 did, so for those seconds it cited
`sdks/stripe-js/src/testing/unroutable-secrets.ts` before `master` had it.)_

- `vpay-sdks`: the payer-surface bullet now says the stripe-js and Flutter
  client-secret parsers are pinned by tests on every route that builds a path.
  It names the shared list of sixteen refused secrets and says where a new
  method or a new refused shape goes. ~~`references/parity.md` records the
  two new payer rows and the 778 → 788 move.~~ It did not: the merged change
  never touched `references/parity.md`. The 778 → 788 move is recorded there
  since the restamp entry of the same date, above.
- `vpay-sdks` (`references/flutter-plugin.md`): the example's `pubspec.lock`
  now follows the plugin's version on release, because #309's
  `release-please.yml` step rewrites it. A hand bump still has to refresh it.

**Stopped being true.** "No test there names a dot segment, so loosening
that prefix check would reopen it silently." Loosening it now fails four
stripe-js tests and three Flutter tests. Five mutations were checked on
#309's verification page.
Also: "the Flutter example's `pubspec.lock` is deliberately not bumped"
(vpay's `AGENTS.md`). It recorded `0.1.1` against a `0.9.1` plugin.

## 2026-10-21 — RFC-0012 step 5: the notchpay adapter met Notch Pay's sandbox

Read against vaam-apps/vpay's `master` at `e28292d`, the merge of
vaam-apps/vpay#310.

- `vpay-notchpay`: step 5 ran on 2026-10-21. vpay itself (server, worker,
  Postgres) took three test-mode payments through the adapter against Notch
  Pay's real sandbox, from the maintainer's MacBook in a Claude Code desktop
  session, after the maintainer approved the calls in that chat.
  `+237670000000` (210 XAF) `succeeded` with `provider_txn_id` the
  `trx.test_…` and one balanced ledger capture. `+237670000002` (220 XAF)
  `failed`: charge `failed`, `provider_error`, intent
  `requires_payment_method`. `+237670000001` (230 XAF) read `complete` with a
  short `amounts.total`; the first read answered `pending`, and the eleven
  after it (reads 2 to 12, in 15 minutes) were refused as `Malformed`. At 15
  minutes the charge was `pending` and the intent `processing`: never a
  success. Becoming `unresolved`, with an alert, at the 24 h horizon is read
  from the code, not observed. #304's addressing worked against the real
  rail: every charge's `ref_extra` held `notchpay_reference`. The skill
  states what is still open (webhooks, production, a live key's prefix,
  refunds, Orange through vpay, the 24 h escalation, the short
  `amounts.total` value), records two observations (`provider_requests`
  keeps a refused read as "no answer"; "needs an operator" wakes nobody for
  24 h), and carries the corrected Part B expectations: `…000002` is the
  failure number, `…000001` must never be `succeeded`, set `VPAY_AMOUNT` on
  every payment. `providers.md`'s `notchpay` Collect and Status read cells
  say "proven; sandbox 2026-10-21", and the skill says that is the whole
  claim.
- `vpay-provider-adapters`: the RFC-0012 entry and the client-crate note say
  the adapter has called Notch Pay's sandbox, and nothing else.
- `vpay-conventions`: `references/adr-index.md`'s RFC-0012 row says the same.
- `vpay`, `vpay-mtn-momo`: MTN is no longer the only rail called for real;
  both strike the old claim and date the correction.
- `README.md`: the `vpay-notchpay` row no longer says "never called".

**Stopped being true.**

- "The adapter has never called Notch Pay. Step 5 (the first test-mode call
  through it) has not happened" (`vpay-notchpay`). It ran on 2026-10-21,
  against the sandbox only. The same claim, reworded, in
  `vpay-provider-adapters` ("It has never called Notch Pay", "Nothing has
  called Notch Pay through it", "built on a vpay branch, never called"), in
  `vpay-conventions`' `references/adr-index.md` ("has never called Notch
  Pay") and in `README.md` ("never called").
- "That fix has not met Notch Pay (step 5)" and "never yet met Notch Pay:
  step 5". #304's addressing met the sandbox and worked.
- "The probe ran on 2026-10-21; step 5 has never run". Both ran.
- `docs/status/providers.md`'s "stub-only; never called" for `notchpay`'s
  Collect and Status read cells. They say "proven; sandbox 2026-10-21", and
  that is the whole claim.
- Runbook B3's "`…000001` reaches `requires_payment_method` with a
  `failure_code`". It reaches neither that state nor a success; `…000002` is
  the failure number. Also its "about two minutes" (about 15 s) and a ladder
  that "asks at 10, 20, 30, 45, 60 and 90 s" (it also reads about 0.5 s after
  submit, then every 120 s).
- `vpay` and `vpay-mtn-momo`: "no rail other than MTN's sandbox", "the one
  rail vpay has ever called for real" and "Everything else in this
  repository's history settled against a WireMock host". Notch Pay's sandbox
  is a second rail called for real; Orange Money's rail has still never been
  called.

## 2026-10-21 — an empty or all-dots path id is refused by both merchant SDKs

Read against vaam-apps/vpay's `master` at `1ffd148`, the merge of
vaam-apps/vpay#306. The skill change is vaam-apps/vpay-skills#48 (`c535b6c`),
which merged without this entry.

- `vpay-sdks`: both SDKs put a caller's id into a URL path through one helper
  each (`path_segment(id)?` in `sdks/rust`, `pathSegment(id)` in
  `sdks/nodejs`). It refuses `""` and any id made only of dots before any
  request, the token request included: `Error::InvalidParams` in Rust, a
  `TypeError` in Node. `...` and longer runs are refused by decision, as the
  server-side port's `path_segment` does. Both test suites pin the 21
  id-taking methods.

**Stopped being true.** "The merchant SDKs pass an id through
`encodeURIComponent`'s rule and nothing else." A `""`, `.` or `..` id no
longer reaches vpay: before #306, `customers.del("..")` was a `DELETE /v1/`.

## 2026-10-21 — the `notchpay` adapter addresses payments by Notch Pay's reference

Read against vaam-apps/vpay's `master` at `5b4c583`, the merge of
vaam-apps/vpay#304.

- `vpay-notchpay`: process and read go to `/payments/{notchpay_reference}`,
  the `trx.…` kept in `ref_extra`; every answer's `trxref` must be ours; a
  read with none held is `Config`; an unresolvable `409` is `Malformed` (the
  maintainer's decision); callbacks prefer `data.trxref`. The crash window
  is stated as a residual risk. The conformance suite selects key material
  per rail.

**Stopped being true.** "Every call addresses the payment by vpay's
reference, and `ref_extra` is always empty". "The duplicate-submit case
requires a push rail's `ref_extra` to be empty" (it requires the charge's
own key material back). "The 409 answer is undecided" (fail closed).

## 2026-10-21 — Notch Pay's sandbox answered step 1, and contradicted the adapter

Read against vaam-apps/vpay's `master` at `f7c7ae3`, the merge of
vaam-apps/vpay#303.

- `vpay-notchpay`: step 1 ran on 2026-10-21, through the probe, on the
  maintainer's approval in that session. The path takes Notch Pay's `trx.…`
  reference only, so the adapter's `submit` fails closed against the real
  rail (not yet fixed). `search` does not find our reference, so the fix
  will keep a residual crash window. The "insufficient funds" number read
  `complete` for less than asked: a success now also needs `amounts.total`
  to be the charge's amount. `canceled` is `payer_declined`. A payment
  answer has no `id`. The probe gained a `search` phase.

**Stopped being true.** "Nothing has ever called Notch Pay" (the probe has;
the adapter still has not). "`canceled` is not `payer_declined`".
"`PRODUCED_FAILURE_CODES` is three codes" (four). "Every mapping is To
confirm" (items 3, 4, 5, 15 and 19 are answered). "The read addressed by
vpay's own reference" is still what the code does, but it is now known to be
wrong against Notch Pay.

## 2026-10-20 — the sandbox probe's `--ca-bundle`, and who may run step 1

Read against vaam-apps/vpay's `master` at `d14cbe9`, the merge of
vaam-apps/vpay#302.

- `vpay-notchpay`: the probe, and only the probe, takes a cloud-only
  `--ca-bundle <path>`, so that step 1 can run from a Claude Code cloud
  session. Step 5 cannot, because vpay itself has no such flag and must never
  gain one (ADR-0003). An agent may run step 1 only after the maintainer has
  approved the calls in that session's own chat. Nothing has called Notch Pay
  yet.
- `vpay-tooling`: `expected_suites` is 53 on `master`, where the probe's unit
  tests became a test binary.

**Stopped being true.** "Do not run the probe yourself" was unconditional. It
now holds only without the maintainer's approval in the session's own chat.
"The probe is an example, not a test" is false: since #302 its
`[[example]]` says `test = true`. "Do not add a CA or a proxy to vpay" now
reads "to vpay's shipping code", because the probe has one.

## 2026-10-20 — a runbook for Notch Pay's sandbox, run by a person

Read against vaam-apps/vpay's `claude/notchpay-sandbox-runbook` branch.

- `vpay-notchpay` names the runbook, the `sandbox_probe` example and the
  `notchpay-sandbox` profile, and says the run is a person's, with real
  egress, and not an agent's. Nothing has called Notch Pay yet.
- Not added to `coverage.json` yet: the three paths exist only on that
  branch, and the gate reads vpay's `master`. Claim them once it merges.
  _(Done the same day, once vaam-apps/vpay#299 merged as `c5b0a80`:
  `vpay-notchpay` claims the runbook, the profile and the probe
  (`backends/crates/vpay-notchpay/examples/sandbox_probe.rs`).)_

## 2026-10-20 — Notch Pay's real test keys carry `test.`

Read against vaam-apps/vpay's `claude/notchpay-sandbox-key-shape` branch, cut
from `357c825`. The first Notch Pay sandbox session made no request, but it
measured the maintainer's test-mode keys. They carry `test` followed by `.`
and not the documented `test_`. The boot rule now reads either.

- **Claims that stopped being true:** "a sandbox key must contain `test_`"
  and "a real Notch Pay test key carries `test_`" (`vpay-notchpay`,
  `vpay-troubleshooting`, `vpay-ops`, `vpay-provider-adapters`). The rule
  is `test.` or `test_`.
- **Still true, and restated:** nothing has called Notch Pay. Steps 1 and 5
  are not done, and `providers.md`'s cells still say "stub-only; never
  called".
- `vpay-notchpay` now says why that session could not run: a permission
  policy, and an egress that re-terminates TLS, which vpay's vendored roots
  correctly refuse.

## 2026-10-20 — who ran the 2026-09-15 MTN sandbox tests

Read against vaam-apps/vpay#297. No stamp moves.

**A claim that stopped being true.** Seven passages credited "the
tester/maintainer" with the 2026-09-15 MTN Collections sandbox tests and
with confirming the over-seven count. Asked on 2026-10-20, the maintainer
answered: "It was Valsuh", that is Valantine Suh (@Valsuh45). Each passage
now names them, in `vpay`, `vpay-payments`, `vpay-mtn-momo`, `vpay-ops`,
`vpay-troubleshooting` (`demo-compose`) and `vpay-sdks` (`tauri-plugin`,
`flutter-plugin`). The counts and the evidence are unchanged. The older
entries below keep their wording, as they were written.

## 2026-10-20 — the Notch Pay adapter, admitted under ADR-0031

Read against vaam-apps/vpay#295 at `bf96e50`, merged to `master` as `357c825`
the same day. `vpay-notchpay` is stamped `357c825`; every other skill's stamp
stays where it was, because moving one means re-verifying the whole page
(VERSIONING.md).

- **New skill: `vpay-notchpay`.** The third rail. A push rail through an
  aggregator, admitted under ADR-0031 because its status read takes a key
  Notch Pay calls public. **It has never called Notch Pay.** It covers
  `docs/flows/adapter-notchpay.md`, the threat model, the adapter crate and
  its stub.
- **Claims that stopped being true** on that branch: RFC-0012 is "gated, not
  buildable" (`vpay-provider-adapters`, `vpay-conventions`); the port has
  "nine methods" (ten: `accepted_exposure`); `verify-status` reports "exactly
  one" token (two: `notchpay::refund`); "neither rail is `Origin`"
  (`vpay-sdks`, `vpay-merchant-api`); `vpay-notchpay` is "linked by nothing";
  `expected_suites` (52); the version-reference count (26);
  `REQUIRED_RAIL_KEYS` checks presence only (it gained three columns:
  `vpay-ops`, `vpay-troubleshooting`); `callback_url_pattern` returns a
  `String` (an `Option` now: `references/conformance.md`).
- **Livemode acknowledgement** (`bf96e50`, same day): the maintainer answered
  the question the adapter PR asked, "Livemode acknowledgement: YES, require
  it." `vpay-notchpay` said boot "does **not** refuse" and "do not build one
  unasked"; in livemode boot now refuses `notchpay` without
  `settings.accept_public_key_exposure: true`
  (`PublicKeyExposureNotAcknowledged`), still warning when it is set.
  `REQUIRED_RAIL_KEYS` gained a fourth column, not three (`vpay-ops`,
  `vpay-troubleshooting`, `vpay-provider-adapters`' `adding-a-rail.md`).

## 2026-10-20 — MTN sandbox testing evidence correction

~~Exactly one payment had settled against a real rail.~~ **Corrected
2026-10-20 from the tester/maintainer's firsthand confirmation:** the testing
recorded on 2026-09-15 included **over seven successful payments against MTN's
real sandbox**. The three attempts in the original documented session were two
token-mint failures followed by one successful settlement; that session was not
the total of the testing.

The orientation, payments, MTN, operations and troubleshooting briefings now
distinguish the aggregate from that session. The Flutter and Tauri briefings
still describe WireMock-only testing in their own lanes, without extending that
claim to all vpay testing. No production rail or real payer is claimed, and the
Orange, Disbursements/refund and account-holder lookup limitations are unchanged.

This corrects the testing record, not the verified code baseline. Skill stamps
and `coverage.json`'s baseline are unchanged; no full re-verification or release
is claimed.

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

- **The banner.** ~~Exactly one real rail call has ever been made (MTN's sandbox,
  2026-09-15).~~ **Count corrected 2026-10-20:** over seven payments succeeded
  against MTN's real sandbox; see the firsthand correction above. No rail has
  returned money, no cluster has run vpay. Nothing in the
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

- **The banner.** ~~Exactly one real rail call has ever been made (MTN's sandbox,
  2026-09-15).~~ **Count corrected 2026-10-20:** over seven payments succeeded
  against MTN's real sandbox; see the firsthand correction above. Orange has
  never been called, no webhook has reached a merchant
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

- ~~**One real rail call has ever been made.**~~ **Corrected 2026-10-20:**
  the tester/maintainer confirms firsthand **over seven successful payments
  against MTN's real sandbox** in the testing recorded on 2026-09-15. The
  original documented session had three attempts, two token-mint failures and
  one settlement; it was not the overall total. Orange has never been called. Before
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
