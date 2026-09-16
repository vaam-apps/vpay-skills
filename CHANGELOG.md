# Changelog

Each release records the vpay range it was verified against, which skills
changed, and — the section worth reading — **any claim that stopped being true**,
so someone upgrading can find the thing that will break them.

Releases are named for the date of verification and the vpay commit verified
against, because vpay publishes no release tags and its workspace version has
never moved off `0.1.0`. See [VERSIONING.md](VERSIONING.md).

## Unreleased — pending [vaam-apps/vpay#180](https://github.com/vaam-apps/vpay/pull/180)

**Do not merge before that PR does, and bump `coverage.json`'s `baseline` to its
merge commit in the same change.** Until then these skills describe a state that
exists only on a branch, which is the exact failure this repository's versioning
policy exists to prevent.

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

Twenty skills, covering all 23 pages in vpay's `docs/flows/`.

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
