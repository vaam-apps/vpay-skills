# CrateStack in vpay: the private module, the traps, and the drift

`vpay-db` compiles `schemas/vpay.cstack` with CrateStack's
`include_server_schema!` macro
(`cratestack = { package = "cratestack-pg", version = "=0.12.0" }`).

**`backends/migrations/*.sql` remains the authoritative schema.** Nothing
generates DDL from the `.cstack` file and it drives no migration. What the
file buys is a second, machine-checked description of the same database and
a query layer for the parts of it that fit.

## What actually runs through it, as of 2026-09-16

Nineteen `model` declarations, **five** of which carry real queries:

| model             | statements                                                                        |
| ----------------- | --------------------------------------------------------------------------------- |
| `DisabledClient`  | `find_unique`, `upsert`, `delete_many().where_(..)`                               |
| `Currency`        | `find_unique(..).for_update()` then `upsert(..)`, both `run_in_tx` on boot step 4 |
| `Provider`        | `upsert(..).run_in_tx` — migration `0033` is what made it possible                |
| `Event`           | the outbox: `update_many(..)` on the fan-out transaction                          |
| `WebhookDelivery` | the outbox: `upsert(..).do_nothing()` on the same transaction                     |

Plus three whole tables whose **every** repository method runs through the
generated layer — `staff_members` (7), `staff_sessions` (6),
`oauth_authorization_codes` (3), sixteen of the thirty-two statements the
reference page counts, with no raw `sqlx` between them.

That is a property of migration `0035` rather than of ambition. Everything
that keeps other tables on raw `sqlx` was designed out of those three before
they were created: no `jsonb`, no `bytea`, no native enum, no `DEFAULT` on
any column a writer names, no `seq` cursor.

**Every other model is a design sketch.** `model PaymentIntent`,
`model Charge` and `model Refund` in particular carry **no `@@allow` arm at
all**, so every generated read on them answers zero rows — and that is
pinned in that direction on purpose by
`the_three_money_models_answer_no_rows_to_every_action`. The usual hazard is
an `@@allow` going missing; here it is the reverse. An arm added to one of
those three would be a standing permission over `payment_intents`, `charges`
and `refunds` with no caller asking for it, and nothing else in `just ci`
would say a word.

## Why `mod schema` is private, and what keeps it that way

`include_server_schema!` expands to `pub mod cratestack_schema { … }` **at
its invocation site**. The invocation therefore lives in a private
`mod schema;` in `lib.rs` — never `pub mod` — and nothing below it is
re-exported, ever.

At the crate root the same expansion would publish
`vpay_db::cratestack_schema::*` — every model struct, every query delegate
and a whole generated `pub mod axum` — to every consumer of `vpay-db`, **a
wholesale reversal of ADR-0016 standard 5 in a diff that looks like one
line**.

`cargo xtask verify-repositories` has three signals. The first two read
declarations and `impl` headers:

1. a declaration whose body carries a `DB_HANDLE_TYPES` field — `PgPool`,
   `Transaction`, **`Cratestack`**, **`SqlxRuntime`**. The last two were
   added 2026-09-06 and neither contains the word `PgPool`: a
   `struct CsChargeStore { cs: Cratestack }` would have been invisible.
   Matching is on identifier boundaries, which keeps `Cratestack` out of
   `CratestackError` and `CratestackContext`.
2. a type on the right of `impl <Trait> for <T>` where `<Trait>` is a `pub`
   trait `vpay-db` declares.

The third is **textual, and that is not laziness**:

> The module the macro creates **does not exist in any source file**, so the
> two signals above cannot see it, and neither can `cargo doc`'s intra-doc
> link resolution or any lint. `pub use schema::cratestack_schema;` would
> hand `vpay_db::cratestack_schema::Charge` to every consumer, **and this
> gate said `ok` for it until this check landed** — with both original
> signals in place, `pub mod schema;` _and_ `pub use
schema::cratestack_schema;` both printed `ok`.

So the gate also greps for `include_server_schema!` and refuses the module
being made `pub` or named in a `pub use`.

`verify-repositories` strips comments (unlike `verify-no-mocks`), because
`PgRepositories` is a real type with a real reason to be discussed and an
intra-doc link is documentation, not a reach.

## `procedure_router`, never `router()`

The dashboard's read surface mounts `cratestack_schema::axum::procedure_router`.

The generated `router()` **merges `model_router(...)`** — the CRUD CrateStack
generates for every one of the nineteen models, creates, updates and deletes
included, whether or not anything routes it. `procedure_router` is the same
generated function with that merge removed. **"Reads only" is expressed by
calling a different generated function rather than by trusting a route
table**, and `no_generated_model_route_is_mounted_only_the_one_procedure_is`
probes all nineteen model paths to prove none matches.

`dashboard_procedure_router` never calls `crate::persistence::system_context`
— the only place in the crate that can produce a context for which
`is_system()` is true. `resolvers: ()` because the schema declares no
`@computed` field.

**Known-stale prose.** That test's name, its inline comment ("the one real
procedure"), and `schema.rs`'s header ("the bodies of this schema's
`procedure` declarations — one today") all say **one**. The schema declares
**five**: `searchPaymentIntents`, `searchRefunds`, `searchWebhookDeliveries`,
`searchCustomers`, `searchCheckoutSessions`, with five body modules under
`src/schema/`. **The code wins.** If you touch that file, fix the sentences.

## Traps that no gate catches

### 1. The table name is derived from the model name — and there is no `@@map`

0.12.0 derives it with
`cratestack_core::route_naming::pluralize(to_snake_case(model))`.

> **`model Staff` reads and writes a table called `staffs`.**

The first draft of migration `0035` created `staff`. Every query answered
`relation "staffs" does not exist`, and **no gate said anything** — not
`cargo build`, not `just check-schema`, not clippy, not any of the `just
verify` gates — until a container-backed test ran.

The model is `StaffMember` and the table is `staff_members`. The Rust trait
is still `vpay_db::Staff`, because it is a trait about staff and not about a
table.

**If you add a model, run a container-backed test against it before you
believe the name.**

### 2. A `procedure` name that collides with a generated method is a compile error

`procedure listPaymentIntents` collides with the generated `list`:

```
Error: procedure `listPaymentIntents` collides with the generated ...
```

and fails as `error[E0428]`. That is why all five are named `search*`, not
`list*` — a naming convention with a compiler behind it, not a preference.
(`validate/procedure_handler_collisions.rs` upstream.)

### 3. A `procedure` body is hand-written, and that is the point

The macro emits the signature into a `ProcedureRegistry` trait **inside the
private expansion**; vpay writes the body in `src/schema/*.rs`. That is why a
procedure can read `payment_intents` when a generated delegate cannot. It is
also why every body module must be a **child of `mod schema`**, not a
sibling: the trait it implements is private there.

The trait requires exactly **one implementer per schema** (`Payments`, in
`search_payment_intents.rs`). Other body modules contribute a `pub(super)`
free function that `Payments` delegates to — not a second impl.

Tenancy predicates CrateStack's policy language cannot express (it cannot
name the caller's own tenant) live in the **procedure body's own `WHERE`**.

### 4. The grammar has no `@@check(expr)` at 0.12.0

`@db_enforce` promotes only a _single field's_ `@range`/`@length`/`@iso4217`
validator to a CHECK. A cross-column constraint — such as
`supports_partial_refunds ⇒ supports_refunds` — cannot be expressed, and the
`.cstack` file carries a GAP note saying so. The real CHECK lives in
`backends/migrations/0002_create-providers.sql`.

## The version lockstep

Three pins that must move together, because they are one release:

| where                | what                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------- |
| `justfile`           | `cratestack_version := "0.12.0"` — what CI installs and `just check-schema` compares against |
| root `Cargo.toml`    | `cratestack = { package = "cratestack-pg", version = "=0.12.0" }`                            |
| `vpay-db/Cargo.toml` | `cratestack-codec-json = { version = "=0.12.0" }`                                            |

The CLI and the library being the same version is why the two checks cannot
answer about different grammars. The twelve `cratestack-*` packages all
declare `rust-version = "1.98.0"` and are the sole maximum over the whole
graph — which is why the workspace's `rust-version` moved to 1.98 when they
entered it.

`just check-schema` is the seventh gate in `just verify`. It:

- **fails, never skips**, if the `cratestack` CLI is absent ("this is a
  failure, not a skip: nothing checked $schema in this run");
- **warns** on a version mismatch rather than failing, because CI installs
  the pin exactly and CI is the gate of record;
- asserts a `datasource` block exists — **without one, cratestack treats the
  file as client-only, stops applying every database-backed-model rule, and
  still prints `schema OK`**;
- asserts at least `cratestack_min_declarations` (15) model/enum
  declarations — an emptied file type-checks vacuously;
- then runs `cratestack check --schema schemas/vpay.cstack`.

`cargo build` is the second, independent check: the macro parses the file
with `cratestack-parser` and fails the build on anything the CLI would
reject, **and on some things it would not** (composite `@@id`, list-arity
scalars on a database-backed model).

## The drift test — exact, not a floor

`the_cstack_schema_drifts_from_the_migrations_by_a_measured_amount` in
`backends/tests/integration/tests/postgres_smoke.rs` runs
`cratestack migrate baseline --strict` against a real, fully migrated vpay
Postgres. `--strict` inverts the exit condition (fail non-zero, no writes, if
any drift is found), which is what makes an adoption tool usable as a
measurement. vpay is **not** adopting anything.

It asserts `exit code == 1` — a zero exit would mean schema and migrations
agree, the one outcome it must not silently accept — and then three
`assert_eq!`s, exact and not floors:

| constant                      | value (2026-09-16) | what it pins                                    |
| ----------------------------- | ------------------ | ----------------------------------------------- |
| `EXPECTED_DRIFT_CHANGES`      | **190**            | total pending changes                           |
| `EXPECTED_DRIFTED_RELATIONS`  | **25**             | tables/views the drift is spread over           |
| `EXPECTED_UNMAPPABLE_COLUMNS` | (pinned)           | live columns cratestack **declines to compare** |

Plus the **exact sorted set** of tables the live database has and the schema
does not.

Why all four:

- 85 changes in three relations and 85 over sixteen are different facts, and
  only one is true. The relation count moving and the change count moving
  mean **different things**.
- The table-set assertion exists because the count alone was proven
  insufficient: adding a `DisabledClient` model swapped one change for
  another ("table … is not declared" out, "column default value differs" in)
  and the total stayed at exactly 86. **A whole table entering the schema was
  invisible to the change count.** This assertion caught it.
- `EXPECTED_UNMAPPABLE_COLUMNS` is a **blind spot in the measurement
  itself**: those columns are excluded from the comparison, so the drift on
  them is unmeasured. Every one is a `jsonb`, a `bytea` or an `int2`/`int4`.
  `jsonb` and `bytea` do not round-trip at 0.12.0, so it cannot reach zero by
  schema work alone. **If it grows, the report is comparing less than it was,
  and the change count can fall for a reason that has nothing to do with the
  schema improving.**

> **A commit that grows the schema and leaves these constants alone has
> changed the schema without measuring the change.** The test failing is the
> intended way to find that out.

Migration `0033` moved the count by **zero, and had to**: a column default
costs drift only when the two sides disagree, so dropping the five
`@default(...)` from the `.cstack` file _without_ that migration would have
made it 89.

The test prints the CLI version it ran under and **warns** (does not fail) on
a mismatch with the justfile pin — measured 2026-09-05, a shim printing
`cratestack 9.9.9-review-fake` ran the whole measurement and said nothing.
Whether that should fail instead is a maintainer's call, left open.

It also writes to an `--out-dir` **outside the checkout**, created empty
first, so "`--strict` wrote nothing" is an assertion about a directory that
exists rather than one that may never have been reached.

## The reference pages

`docs/reference/vpay-db/cratestack.md` plus four siblings
(`cratestack-what-runs-through-it.md`,
`cratestack-currencies-and-providers.md`, `cratestack-money-tables.md`,
`cratestack-procedure.md`). Moved out of a 3 330-line
`docs/reference/vpay-db.md` on 2026-09-11 and **deliberately left unedited**
— every dated measurement, struck-through claim and correction is there as
written, because "this said X until date Y and was wrong" is the only signal
a reader has about which sentences have been checked recently.

Its headline number, "thirty-two statements over twelve tables", was itself
corrected on 2026-09-10 (issue #87) from two contradictory figures written in
consecutive sentences on branches that did not see each other. Read the
running totals in that page as the history of what each change _said_, not as
arithmetic that adds up to today.
