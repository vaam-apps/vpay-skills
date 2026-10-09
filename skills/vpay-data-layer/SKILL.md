---
name: vpay-data-layer
description: vpay's persistence layer — `backends/migrations/*.sql` as the authoritative schema and the rule that a shipped migration is never edited, the CrateStack `.cstack` file that compiles and runs real queries for fourteen of its twenty models while six stay a type-checked design sketch, the repository traits and why their implementations may never be named outside `vpay-db`, sqlx with no offline mode and no query macros, and the drift and testcontainer machinery. Load before adding a migration, touching `backends/crates/vpay-db`, editing `schemas/vpay.cstack`, or writing any SQL.
---

# The data layer

> **Verified against vpay `87166eaf` (2026-10-09).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

Postgres. One crate owns it: `backends/crates/vpay-db`. Nothing else in the
workspace may name a `PgPool`, a `sqlx::Transaction`, or a CrateStack handle.

## Two schemas, one of them authoritative

**`backends/migrations/*.sql` is the schema.** 48 files as of 2026-09-16 —
this page said **44** until `0045`–`0048` landed the same day (RFC-0003,
vpay#178); `0045` adds `ledger_entries.merchant_id`, `0046` widens the ledger
id, and `0047`–`0048` are comment-only corrections to `refunds`. **49 since
vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251)**:
`0049_manual-payments.sql` adds `invoices.paid_out_of_band` and the
`manual_payments` table — see below. Counted
off the directory, which is the only authority (`cargo xtask
verify-migrations` printed 49 on `b747e5d5`): they are applied in filename
order by `sqlx::migrate!("../../migrations")` from
`vpay_db::migrations::Migrations::run_migrations`, which every mode of the one
`vpay-server` binary — `serve`, `worker`, `staff add` — calls at boot through
`vpay_api::boot::open_migrated_database`, and every container-backed suite
runs. _(This said "which both binaries call" until 2026-09-23; there has been
one binary since 2026-09-07, issue #77. `open_migrated_database`'s doc
comment was fixed by vaam-apps/vpay#255 on 2026-09-24; other doc comments and
comments in shipped files still say "both binaries" as of `a33aac61` —
`vpay-db`'s `config_reconcile.rs`, `lock_keys.rs` and `repository.rs`, and
`0032`'s SQL comment, which is shipped and may not be edited. I found no
"both binaries" in `backends/migrations/README.md`, which this page named
alongside them until 2026-09-29.)_

`schemas/vpay.cstack` is a **second, partial** description of the same
database. It compiles into `vpay-db` on every build, so a syntax or type
error there is a build failure. But:

> **Compiled is not used — for six of its twenty models.** As of vpay
> `b747e5d5` (2026-09-23; re-confirmed identical at `a33aac61`, 2026-09-29),
> **fourteen** models carry production statements,
> thirty-six between them: `DisabledClient`, `Currency`, `Provider`, `Event`,
> `WebhookDelivery` (all since 2026-09-06), `Customer` (2026-09-06),
> `CheckoutSession`, `Invoice`, `InvoiceItem`, `StaffMember`, `StaffSession`,
> `OauthAuthorizationCode` (2026-09-07), `Credential` (2026-09-13) and
> `ManualPayment` (one read, 2026-09-23, step A). **Six carry none** —
> `PaymentIntent`, `Charge`, `Refund`, `LedgerTransaction`, `LedgerEntry`,
> `RateLimitWindow` — and those are the design sketch: type-checked by a
> compiler as well as by the CLI, read and written by no code, and several
> **do not match the live table at all**.
>
> ~~Of its nineteen models … **five** carry real queries … Every other model is
> a design sketch.~~ **Corrected 2026-09-23.** "Five" had been wrong since
> 2026-09-06, when `Customer` joined the first five the same day — it was the
> figure in `schemas/vpay.cstack`'s own header box, which said it on
> `b747e5d5` and ~~still says it~~ says "FOURTEEN OF TWENTY MODELS ARE QUERIED"
> since vaam-apps/vpay#255 (2026-09-24); vpay's `docs/status/infrastructure.md`
> corrected it to "twelve of seventeen" on 2026-09-10 (issue #87), before
> `Credential` and `ManualPayment` arrived. **Re-derive it rather than trust
> any of these numbers**: count the `find_unique`/`find_many`/`create`/
> `upsert`/`update_many`/`delete_many` chains in
> `backends/crates/vpay-db/src/*.rs` outside `#[cfg(test)]` that end in one
> `run(..)` or `run_in_tx(..)`, attributed to the accessor's model. The five
> `procedure search*` bodies under `src/schema/` are hand-written SQL and do
> not count.

Nothing generates DDL from the `.cstack` file and it drives no migration. The
gap between the two is **counted, not closed** — ~~190 pending changes over 25
relations as of migration 0044~~, which this page said until 2026-09-23 and
which had been stale since 2026-09-15: `0045` and `0046` moved it to 192 and
then **194 over 25**, the value on `master` before step A. **201 over 26
relations since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251)**,
19 unmappable columns unmoved — `EXPECTED_DRIFT_CHANGES`,
`EXPECTED_DRIFTED_RELATIONS` and `EXPECTED_UNMAPPABLE_COLUMNS` in
`postgres_smoke.rs`, ~~measured at cratestack 0.12.0 and unchanged on vpay
`master` at `b747e5d5`~~ **re-measured at cratestack 0.15.0 on 2026-09-29 and
unmoved** (vaam-apps/vpay#259: the CLI reports 201 changes across 26 tables,
the constant the test already asserted). See
[references/cratestack.md](references/cratestack.md) — in particular for which
0.12.0 claims about _why_ the totals look like that are **unverified at 0.15.0**.

## Migration `0049` (step A): the database says what a CHECK cannot

Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged in vaam-apps/vpay#251). The pattern is
reusable, and the traps are specific:

- **A cross-table invariant is a composite foreign key, not a CHECK.** A CHECK
  sees one row of one table. `manual_payments_agree_with_their_invoice` points
  `manual_payments (invoice_id, merchant_id, livemode, currency_code, amount,
paid_out_of_band)` at `invoices_payment_record_key`, a six-column `UNIQUE`
  on `invoices` that exists only to be its target; `manual_payments.paid_out_of_band`
  is pinned `true` by `records_an_out_of_band_payment`. So a record exists only
  for an invoice flagged paid out of band, with that invoice's amount, tenant,
  mode and currency, and `NO ACTION` freezes those invoice columns once a
  record exists. The writer never needs the key — it copies the values off the
  invoice inside `INSERT … SELECT` — and the key is the guard that survives a
  writer who does not.
- **CrateStack 0.12.0 introspects no foreign key** (**unverified at 0.15.0** —
  vpay's docs and comments still say 0.12.0 here, and the drift total did not
  move across the bump, which is consistent with it and does not prove it), so
  the drift report cannot see that key at all, and a six-column `@@unique` would generate an index
  name past Postgres's 63-byte limit, so `invoices_payment_record_key` cannot
  be declared: it is one permanent `[safe] index` line in the drift.
  `the_out_of_band_invariants_are_enforced_by_the_database_itself` writes each
  refused row straight past the API — that test, not the drift count, is the
  guard for every multi-column constraint here.
- **`model ManualPayment` was born with the table**, in `invoice_items`'
  shape (no `jsonb`, `bytea`, native enum, `int4`, or writer-named DEFAULT),
  so every column is compared. Its CHECK and unique index are created under
  CrateStack's generated names (`manual_payments_method_enum_check`,
  `manual_payments_invoice_id_key`) so declaration and live object are one
  object to the diff engine. One read runs through it
  (`Invoices::manual_payment_for_invoice`, a generated `find_many` behind
  `@@allow("read", auth().isSystem())`); the write is hand-written, because a
  generated `create` takes values and the amount must be copied in the
  statement. Losing that `@@allow` arm makes every render of an out-of-band
  invoice a `500` (`every_action_this_module_calls_has_an_allow_arm`).
- **`0049` was edited in place, and that was legal**: review added the
  composite key while the migration was unshipped, and the manifest line was
  re-derived. The rule is "a **shipped** migration is never edited".
  ~~Once step A merges, `0049` is shipped.~~ **It merged on 2026-09-23
  (#251): `0049` is shipped, and not a byte of it may change.** The next
  change is `0050`.
- **The ADD COLUMN backfills with a DEFAULT and drops it in the next
  statement** (`0042`'s device), which is why `model Invoice.paid_out_of_band`
  carries no `@default` and costs the drift nothing.

## The migration rule: a shipped migration is never edited

`sqlx::migrate!` records a **SHA-384 of each file's whole bytes, comments
included**, in `_sqlx_migrations.checksum`, and refuses to run against a
database whose recorded checksum no longer matches:

```text
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
2. `just migrations-manifest` — **appends** its SHA-256 line. ~~**It fails on
   macOS** as of vpay `b747e5d5` (`find: -printf: unknown primary or
operator`): the recipe uses GNU `find -printf`… [vaam-apps/vpay#252] makes
   the recipe run on macOS; it is **open and not merged** as of 2026-09-23.~~
   **Corrected 2026-09-29:** #252 merged on 2026-09-23 (`9184e42`). On vpay
   `a33aac61` the recipe lists files with `find … -exec basename {} \;` and
   takes the digest from `sha256sum` or else `shasum -a 256`, so it runs on
   Linux and macOS alike and the line is not appended by hand. **On a tree
   older than `9184e42` it still fails on macOS** (`find: -printf: unknown
primary or operator`); step A's `0049` line was appended by hand with
   `shasum -a 256` for exactly that reason, and `verify-migrations` accepted it
   (vpay's `docs/status/infrastructure.md`, 2026-09-23).
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
`sqlx::AssertSqlSafe`. `vpay-db` wraps at **74** call sites as of 2026-09-29
(`EXPECTED_ASSERT_SITES`, asserted exactly). _(This said **71** until
2026-09-29 — ADR-0027 moved it 71 → 74 on 2026-09-23, because the three erasure
statements that were literals now interpolate the one `PAYERS_INTENTS` const; and
**61** before that, the value from 2026-09-11 to 2026-09-16; #178 moved it to
69 and step A to 71. ADR-0026 and the cratestack 0.15.0 bump left it alone.)_ — and a wrapper whose contract is
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
**The reason for the halving is a 0.12.0 reason**: `.do_nothing()` held two
connections on the already-exists branch. Since cratestack 0.15.0 (2026-09-29,
#1117) it holds one, so `MAX_CONNECTIONS / 2` is now a conservative ceiling
rather than a tight one. vpay left it alone on purpose — the boot guard, its
integration test and the chart's `"worker-concurrency-pool"` literal move
together, in a PR of their own. Do not loosen it in passing.

## Which clock writes an instant Postgres will compare (ADR-0026, 2026-09-23)

_Added 2026-09-29, verified against vpay `a33aac61`; the reference page is `docs/reference/vpay-db/jobs.md` § "One clock: the database's"._ **A repository method that
writes an instant which Postgres will later compare with its own `now()` takes
a `std::time::Duration`, and the statement computes `now() + $delay`. It never
takes an `OffsetDateTime`.** Applied as of `a33aac61`:

| Method                                                                   | Was (until 2026-09-23)                                                           | Is                                                                                                                |
| ------------------------------------------------------------------------ | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `TxRepositories::enqueue_in_tx(kind, dedupe_key, payload, delay)`        | `run_at: OffsetDateTime`                                                         | `delay: Duration` → `run_at = now() + delay`; `Duration::ZERO` is "due now"                                       |
| `WebhookDeliveries::record_attempt(…, retry_after: Option<Duration>, …)` | `next_attempt_at: Option<OffsetDateTime>`                                        | `now() + retry_after` in the statement that stamps `sent_at`; `None` writes `NULL`                                |
| `Settlement::live_charges_stale_since(stale_after: Duration, limit)`     | `cutoff: OffsetDateTime`                                                         | `updated_at < now() - $stale_after` (`charges.updated_at` is written by `now()`)                                  |
| `Jobs::oldest_runnable_age() -> Option<time::Duration>`                  | `oldest_runnable_run_at() -> Option<OffsetDateTime>` (renamed, not just retyped) | `now() - min(run_at)` subtracted in SQL; sign unchanged, so the gauge still goes negative on a healthy idle queue |

Why it is a signature rule and not a habit: a method with no instant in its
signature cannot be handed the wrong clock, and the compiler is the reviewer.
The alternative — read `SELECT now()` first and pass it — keeps an instant
parameter and costs a round trip. The rest you will trip on:

- **Unsigned on purpose (D3).** `Duration` cannot be negative, so nothing in
  production can enqueue an already-overdue job. A test that needs rows at
  chosen points around the database's clock inserts through the shipping
  enqueue at `ZERO` and then writes `run_at` directly from `SELECT now()`: that
  is a fixture's privilege, not an API.
- **`now()`, not `clock_timestamp()` (D4)**: `now()` is the transaction's start,
  so a job enqueued late in a long fan-out transaction is due from the start —
  only ever earlier than the commit that makes it visible, so it cannot delay
  work; and it makes `run_at` equal the row's `created_at` plus the delay
  exactly.
- **Facts keep the application's clock (D6–D8).** `checkout_sessions.created_at`
  and `expires_at`, `customers.last_used_at` and `anonymized_at`, the invoice
  lifecycle stamps, `due_date`, the `t=` in a webhook signature and every
  `events` body are instants vpay _observed_ and handed to a merchant, a payer
  or a rail. The sweeps over them (`due_for_expiry(now, …)`, `expire_due(now, …)`,
  `Customers::idle_since(horizon, …)`) still take an instant, because both sides
  of each comparison are the application's clock. **Do not move one side alone.**
  The residual is stated by the ADR: such comparisons can still span two
  application hosts. Two more same-class columns were named and left out (D8):
  `oauth_signing_keys.expires_at` and `oauth_client_assertion_jtis.expires_at`.
  _(Updated 2026-10-09, verified against vpay `87166eaf`: the second is concluded.
  ADR-0028, vaam-apps/vpay#271, made `ClientAssertions::delete_expired_client_assertion_jtis`
  take `retain_after_exp: Duration`, with the statement
  `expires_at < now() - ($1::BIGINT * INTERVAL '1 microsecond')` on the database's
  clock, and the worker passes `vpay_worker::CLIENT_ASSERTION_JTI_RETENTION`,
  5 minutes. The column still stores the client's raw `exp`, a fact under D6; the
  horizon is policy, applied once, at deletion. `record_jti` and the migrations did
  not change, and `sql_audit`'s `EXPECTED_ASSERT_SITES` did not move. Why 5 minutes:
  `vpay-merchant-api`. `oauth_signing_keys.expires_at` is untouched and still
  open.)_
- ~~**ADR-0026's status is "Accepted in part"** on vpay `master` at `a33aac61`:
  the rule is the maintainer's, D1–D8 are the implementing agent's and are marked
  as needing confirmation.~~ **Corrected 2026-10-09:** the status is
  `Accepted (2026-10-08)`. The maintainer confirmed D1–D8 on 2026-10-08 and the text
  of D1–D8 is unchanged; the Status block is the only edit, and it reached `master`
  with vaam-apps/vpay#271. D6's list of comparisons against a fact omits one site,
  the checkout confirm gate (`SessionGate::admit_confirm` in
  `backends/crates/vpay-api/src/v1/return_trip.rs`), which compares the
  application's clock with `checkout_sessions.expires_at`: it is on the right side of
  D6's rule, nothing about it changes, and ADR-0028 § "Related findings" records the
  omission in the list, not in the code. See `vpay-conventions` → `adr-index.md`.
- **Tests read "now" off the database too (#254, `d08dafd`).** Fixtures that
  stamp a job and then claim it use `support::db_now(pool)` (integration) or a
  private `db_now` in `vpay-db/tests/repositories.rs`, both `SELECT now()`.
  A host clock in a fixture that `Jobs::claim` then judges is the flake #254
  removed; see `vpay-troubleshooting`.
- **What the tests cannot show:** nothing can set a testcontainer's clock apart
  from the host's, so no test runs a _skewed_ host. The guarantee rests on the
  signatures. `a_job_is_due_at_the_databases_now_plus_its_delay_and_a_zero_delay_claims_at_once`
  proves `run_at - created_at` equals the delay to the microsecond.

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
