# Migrations and the database

_Verified against vpay `7a79684e` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

## The rule that will bite you: a shipped migration is never edited

`sqlx::migrate!` stores a **SHA-384 of each migration file's whole bytes,
comments included**, in `_sqlx_migrations.checksum`. At every boot it re-hashes
the files on disk and refuses to run if any applied migration's hash has moved.

So editing a shipped migration **does not change a database that already
applied it — it stops that database booting the new binary.** Not for a typo,
not for a comment, not for a rename.

**To correct an applied migration, write a new migration that corrects it.**

### The story, because it is what makes it stick

PR #39 — an npm scope rename — rewrote **one line of comment** inside
`backends/migrations/0028_create-checkout-sessions.sql` after it had shipped.
It was an old package name inside a note about `ui_mode`. **No SQL changed.**
The whole edit was two lines.

sqlx hashes the file, comments included, so that one line is a different
migration as far as every database is concerned. Every stack brought up between
#37 and #39 then exited 78 on boot with

```
migration 28 was previously applied but has been modified
```

**Every job in CI was green.** That is the point: nothing in the build could
see it, because the build creates fresh databases.

The decision (issue #76) was **not** to edit the file back — databases created
after #39, including every one a new deployment will create, carry the new
checksum, and reverting would break those instead.

### The gate, and the hole it deliberately leaves

`backends/migrations/MANIFEST.sha256` plus `cargo xtask verify-migrations` (the
twelfth gate, in `just verify` and CI's `self-checks`): a migration file whose
SHA-256 has moved fails the build, and `just migrations-manifest` **refuses to
rewrite an existing line**, so the gate cannot be silenced by regenerating.

The runbook states the hole plainly: a contributor who edits a migration **and**
hand-edits its `MANIFEST.sha256` line passes the gate. Hashing the manifest
would not fix it — "whoever can edit two files can edit three."

> The manifest was not added to make the edit impossible — it was added to make
> it **visible**, as a one-line diff on a file whose only purpose is to be
> reviewed. A comment reflowed inside a 292-line `.sql` file is not visible.
> **Review the manifest line.**

## Repairing a database already in the broken state

Follow `docs/runbooks/migrations.md`. Two things to know before you start.

**§3 first: ask the database which state it is in.** There are two SHA-384
values in play and they are 96 hex characters of noise each. "Do not eyeball
them; ask the database."

**The first draft of that repair was inverted, and this is the cautionary
tale.** It told the operator to `UPDATE … SET checksum = decode('f4d1a8e1…')`
— correctly describing that value as the checksum of the file "in its original,
unedited state". That is the value **a broken database already holds**. Run as
drafted, the statement reports `UPDATE 1`, changes nothing, and the binary keeps
exiting 78 — **with the page telling the operator it had worked.**

The repair must write the **current** file's checksum. The page now states both
values side by side with a `psql` query, and the fix is executable rather than
asserted: `the_0028_repair_in_the_runbook_fixes_a_database_that_applied_the_original`
in `backends/tests/integration/tests/postgres_smoke.rs` migrates a fresh
container, rewinds version 28's checksum to the original, confirms
`sqlx::migrate!` refuses with the message the runbook quotes, **parses the
`UPDATE` out of the markdown file**, runs it, and confirms the migrator then
runs clean.

That is the only runbook in the repository whose central SQL is executed by the
test suite. Its `kubectl` commands, like every other runbook's, have been run
nowhere.

## Adding a migration

The rule and the mechanics live in `backends/migrations/README.md`;
`docs/runbooks/migrations.md` is the operational half. Add the file, run
`just migrations-manifest` to append its line, and **review that line in the
diff.**

## `vpay-shop` dies in `zen migrate deploy`

**Cause:** a stale `pgdata` volume. The demo shop's database is created once,
from Postgres's entrypoint, **on an empty data directory** (see
`deploy/dev/postgres-init/10-shop-database.sql`). A volume from before the shop
landed has no `shop` database.

**Fix:** `just demo-down`, which removes volumes.

## Things about the schema that are true and surprising

- ~~**`refunds` rows are never written.** Reading one works — issue #45 landed
  `vpay_db::Refunds::get_for_merchant` and `GET /v1/refunds/{id}` — but there is
  no `POST /v1/refunds` route, no `create` in the repository, no adapter that
  can execute a refund, and no writer for `charge.refunded` /
  `charge.refund.updated`. Both event types are in the documented vocabulary and
  **neither has ever been emitted.**~~ **Corrected 2026-09-16 (RFC-0003,
  vpay#178) — every clause above was true until then and all four stopped
  being true at once.** `vpay_db::Refunds::create` writes `refunds` rows and
  reserves the amount against the intent in the same transaction; five `/v1`
  refund methods are mounted across three paths; `mtn_momo::refund` is a real
  `POST /disbursement/v1_0/transfer`; and `charge.refunded` /
  `charge.refund.updated` are emitted for the first time.

  **What is still true, and is the part that matters at 3 a.m.:** a `refunds`
  row does not mean money moved. **No rail in this repository has ever returned
  money to anyone**, and MTN's Disbursements product has never been called
  outside WireMock — there is no real Disbursements credential in the project.
  Nothing settles a pending refund (there is no refund poll ladder, RFC-0003
  open question 8), so a refund the rail accepted — or one whose outcome is
  unknown — stays `pending` indefinitely, `invoices.amount_refunded` never
  moves, and `refunds.fee` is written by nothing. Only a *refusal* moves a
  refund, to `failed`; nothing reaches `succeeded` that a merchant can cause. If you are looking at a stuck `pending` refund, that is the designed
  state, not your bug.
- **`refunds.fee` is nullable with no `DEFAULT` on purpose.** `None` means "the
  rail did not report a fee" and `Some(0)` means "the rail said it was free";
  collapsing them is the defect the integrator issue reported. An adapter must
  not invent one. `amount` is the payer's money and **is never net of the fee** —
  `a_reported_fee_never_moves_the_payers_amount` exists because
  `amount: row.amount - row.fee.unwrap_or(0)` once passed all 244 of
  `vpay-api`'s tests.
- **`charges.payer_ref_masked` is never written by anything.** The dashboard
  renders it as null rather than deriving a mask. See
  [dashboard.md](dashboard.md).
- **Configuration reconcile disables rather than deletes.** A rail absent from
  the seed is set `enabled = false`, "because a rail that has ever taken money
  must stay nameable forever."
