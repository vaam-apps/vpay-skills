---
name: vpay-data-layer
description: vpay's persistence layer — `backends/migrations/*.sql` as the authoritative schema and the rule that a shipped migration is never edited, the CrateStack `.cstack` file that compiles but is mostly a type-checked design sketch, the repository traits and why their implementations may never be named outside `vpay-db`, sqlx with no offline mode and no query macros, and the drift and testcontainer machinery. Load before adding a migration, touching `backends/crates/vpay-db`, editing `schemas/vpay.cstack`, or writing any SQL.
---

# The data layer

> **Verified against vpay `93c6dfd0` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

Postgres. One crate owns it: `backends/crates/vpay-db`. Nothing else in the
workspace may name a `PgPool`, a `sqlx::Transaction`, or a CrateStack handle.

## Two schemas, one of them authoritative

**`backends/migrations/*.sql` is the schema.** 44 files as of 2026-09-16,
applied in filename order by `sqlx::migrate!("../../migrations")` from
`vpay_db::migrations::Migrations::run_migrations`, which both binaries call
at boot and every container-backed suite runs.

`schemas/vpay.cstack` is a **second, partial** description of the same
database. It compiles into `vpay-db` on every build, so a syntax or type
error there is a build failure. But:

> **Compiled is not used.** Of its nineteen models, **five** carry real
> queries — `DisabledClient`, `Currency`, `Provider`, `Event`,
> `WebhookDelivery`. Every other model is a design sketch that happens to be
> type-checked by a compiler as well as by the CLI. No code reads or writes
> through any of them, and several **do not match the live table at all**.

Nothing generates DDL from the `.cstack` file and it drives no migration. The
gap between the two is **counted, not closed** — 190 pending changes over 25
relations as of migration 0044. See
[references/cratestack.md](references/cratestack.md).

## The migration rule: a shipped migration is never edited

`sqlx::migrate!` records a **SHA-384 of each file's whole bytes, comments
included**, in `_sqlx_migrations.checksum`, and refuses to run against a
database whose recorded checksum no longer matches:

```
migration 28 was previously applied but has been modified
```

**This is not hypothetical.** The `@vpay` → `@vaam-apps` npm rename (PR #39)
rewrote **one comment** inside `0028_create-checkout-sessions.sql` after it
had shipped, and **every stack brought up before it exited 78 on the next
boot** (issue #76).

**To fix a mistake in an applied migration, write a new migration that
corrects it. Never touch the old file.** Not a word of a comment. Not
whitespace. Not a line reflow by a formatter.

`backends/migrations/MANIFEST.sha256` makes that enforceable at review time
rather than at production boot. Adding a migration:

1. Write `NNNN_short-name.sql`, numbered one above the current highest.
2. `just migrations-manifest` — **appends** its SHA-256 line.
3. Commit the `.sql` **and** `MANIFEST.sha256` in the same commit.

`just migrations-manifest` **refuses to rewrite an existing line** (it errors
if a listed file is missing from disk, or if a listed file's hash has moved),
so the gate cannot be silenced by regenerating. `cargo xtask
verify-migrations` (gate in `just verify` and in CI's `self-checks`) then
fails if any hash moved, if a `.sql` file has no manifest line, or if a
manifest line names a file that is gone. It hashes **bytes, not text** — a
CRLF checkout is a different migration from an LF one.

What the gate does **not** stop, stated rather than hidden: someone who edits
a migration _and_ hand-edits its manifest line passes. That cannot be fixed
by hashing harder. What the manifest buys is that the edit becomes **visible
in the diff** — a one-line change to `MANIFEST.sha256` is exactly what review
is for, where a comment reflowed inside a 292-line `.sql` file is not.

## The repository seam

One `#[async_trait]` trait per table family (`Charges`, `Jobs`,
`PaymentIntents`, `Customers`, `Invoices`, `Events`, `WebhookDeliveries`,
`Refunds`, `Settlement`, `Staff`, `Credentials`, …). `Repositories` is the
umbrella every consumer holds as `&dyn Repositories`. `PgRepositories` is
`pub(crate)` and is the only implementation.

**Nothing in this crate takes or returns a `PgPool`.** `vpay_db::connect`
returns `Arc<dyn Repositories>`.

Two statements that must commit together go through
`UnitOfWork::transaction`, which hands a closure a `&mut dyn TxRepositories`
and decides `COMMIT`/`ROLLBACK` from what it returns:

```rust
pub enum TxOutcome<T> { Commit(T), Abandon(T) }
```

**"Forgot to commit" is not expressible.** `Abandon` is how a caller returns
what it learned from a transaction it then rolled back — it is not an error,
and it is what lets the confirm path's "re-read on the plain pool" recovery
be written without holding a `sqlx` handle across the decision.

`cargo xtask verify-repositories` refuses a concrete implementation being
named anywhere outside `vpay-db` (ADR-0016 standard 5). It derives the set of
"concrete" types from `vpay-db`'s own source rather than from a list, so a
type nobody has written yet is caught the day it is added. Details, including
the check that is textual because the thing it guards exists in no source
file, are in [references/cratestack.md](references/cratestack.md).

## sqlx: no offline mode, no query macros, no compile-time SQL check

There is **no `.sqlx/` directory**, no `sqlx-data.json`, no `SQLX_OFFLINE`,
and **zero uses of `sqlx::query!` / `query_as!` / `query_scalar!`** anywhere
in the workspace. Everything is the runtime API (`sqlx::query(..)`,
`sqlx::query_as::<_, T>(..)`). There is no `cargo sqlx prepare` step, and
**nothing verifies your SQL against the schema at compile time.** A column
name typo is a runtime error a container test finds, or nothing finds.

sqlx 0.9 accepts a statement only as a `&'static str` or wrapped in
`sqlx::AssertSqlSafe`. `vpay-db` wraps at **61** call sites
(`EXPECTED_ASSERT_SITES`, asserted exactly) — and a wrapper whose contract is
discharged by a comment is discharged by whoever last read the comment. So
the contract is a test, `src/sql_audit.rs`:

> **Every `format!` whose result reaches `AssertSqlSafe` interpolates a
> `const … : &str` declared in this crate, and nothing else.** Not a merchant
> id, not a cursor, not a limit, not a status — every one of those is already
> a bind parameter.

"Interpolates a `const`" means **captured by name** (`{COLUMNS}`, not `{}`).
A positional capture takes its value from an argument list the audit does not
resolve, so it is a violation on sight — that was the module's own blind
spot, walked straight through by a review mutation on 2026-09-05. There are
exactly two allowed non-constants, in a closed list, so a third is a
deliberate edit to that file.

`EXPECTED_ASSERT_SITES` is asserted with `assert_eq!`, not a floor: a
_falling_ count means the scanner stopped matching, which is the failure that
would make the whole audit pass vacuously. Its doc comment is the ledger of
every move. **Known-stale prose:** that module's own header still says "this
crate has 37 statements built by `format!`, so it wraps 37 times" — true on
2026-09-05, and the constant has moved several times since. **The constant
wins**; fix the header if you touch the file.

Pool: `MAX_CONNECTIONS = 10`, not configurable anywhere. That constant is
also the worker's concurrency ceiling (`MAX_CONNECTIONS / 2`, refused at boot
with exit 78) — raising it is a code change that moves the ceiling with it.

## Tests get a real Postgres, from one place

`vpay_testkit::containers::start_postgres_with_retry()`. It replaced eight
byte-identical copies.

- **`postgres:16-alpine`, pinned.** `testcontainers-modules` 0.15 defaults to
  `postgres:11-alpine`, which is not cached on the machines this runs on —
  and `compose.yml` runs Postgres 16, so testing against 11 would itself be a
  version mismatch.
- **The retry is narrow on purpose**: four attempts, 250 ms × attempt, and
  **only** when the error chain contains `address already in use`. That
  failure is testcontainers asking for a random free host port and
  rootlesskit racing whatever else on the host grabbed it. Any other failure
  — daemon unreachable, image missing, wait-strategy timeout — returns
  immediately and unwrapped, because a broken daemon retried four times still
  fails, just more slowly and with the cause four levels deep in a log.
- **The caller owns the container**; `Drop` stops and removes it. A shared
  `static` container was tried and rejected: a `ContainerAsync` in a `static`
  never runs `Drop` at process exit, which leaked hundreds of live
  containers.
- **Migrations are the caller's**, deliberately — the integration suite runs
  `sqlx::migrate!` and `vpay-db` runs `vpay_db::run_migrations`, and folding
  either into the helper would make it pick a side.
- `.config/nextest.toml`'s `postgres-containers` group caps these at
  `max-threads = 1` across `vpay-tests-integration`, `vpay-tests-conformance`,
  `vpay-db`, `vpay-server` and `vpay-testkit`.

**Known-stale prose:** the module header of
`backends/crates/vpay-db/tests/repositories.rs` still says this crate "is not
covered by `.config/nextest.toml`'s `postgres-containers` concurrency cap".
It is, and has been since 2026-09-02 — `tests/postgres.rs` carries the dated
correction. **`.config/nextest.toml` wins.**

## Where to go next

- [references/cratestack.md](references/cratestack.md) — the private
  `mod schema`, the table-naming trap that no gate catches, procedures, and
  the drift test's exact-not-floor constants.
- `vpay-tooling` skill — the `just verify` gates in full.
- `docs/reference/vpay-db.md` and `docs/reference/vpay-db/`,
  `backends/migrations/README.md`, `docs/runbooks/migrations.md`.
