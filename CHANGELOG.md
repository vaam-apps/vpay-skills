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
