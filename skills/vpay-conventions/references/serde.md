# serde: `rename_all` is for _our_ wire, never a rail's

_Verified against vpay `b747e5d5` (2026-09-23). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

ADR-0016 standard 3. Gate: `cargo xtask verify-serde`, in `just verify` and in
CI's `self-checks` job.

## The rule

Every type deriving `Serialize` or `Deserialize` under `backends/crates/*/src`
**either**:

1. carries `#[serde(rename_all = "snake_case")]`, **or**
2. renames every field/variant itself with `#[serde(rename = "…")]`, **or**
3. is listed in ADR-0016's exemption table **with a reason**.

## What "spelling the wire convention" means concretely

ADR-0016, and this is the sentence every exemption is an instance of:

> `rename_all` is a statement about **our** wire. Where the names belong to
> somebody else — a rail's JSON, an `untagged` union whose variant names never
> appear at all — the attribute is either inert or **actively dangerous,
> because the day the other party sends a name that is not already snake_case
> the attribute renames it away from their spelling.**

So the attribute is not decoration and it is not a lint about characters. On a
type vpay owns it is a **promise** about the bytes vpay emits. On a type
modelling somebody else's bytes it is a **claim about their roadmap**, which is
not yours to make.

## Four things that catch people

**Visibility is not part of the rule.** `verify-errors` scans `pub` types only,
because a `pub(crate)` error reaches no boundary. `verify-serde` does not,
because **a `pub(crate)` type with a `Serialize` derive reaches a rail** — and
both adapters' entire wire modules are `pub(crate)`. "A wire does not care what
Rust thinks of a type's visibility."

The measurement that settled it: of the 28 pre-ADR violations, only 8 were
`pub`. The other 20 were `pub(crate)`, `pub(super)` or private, and 13 of those
20 were the two adapters' `wire.rs`/`token.rs` — "the types where getting a
field name wrong costs a real payment."

**A tuple struct and a unit struct are compliant by construction.** Neither
serialises a name, so the attribute would rename nothing and requiring it would
be a rule about characters.

**"Already snake_case by coincidence" is the trap, not the excuse.** MTN's
`TokenResponse` and `BasicUserInfo` are snake_case today. That is precisely
what makes the attribute a _promise_ rather than a no-op: it would do nothing
today and become a silent claim about MTN's wire tomorrow. Both are exempt for
that reason, and the exemption table says so in those words.

**A registry is not a rail.** `vpay_api::resource_auth::RawClaims` decodes RFC
7519's registered `sub` and RFC 6749's `scope`. Those names are snake_case and
the IANA registry cannot retroactively rename them, so the attribute **states
the truth rather than making a promise** — `RawClaims` is not exempt. ADR-0016
records the rejected proposal to exempt it as "the exemption most likely to be
proposed again". A rail's product roadmap is not a registry.

## The exemption table

It lives in `docs/adr/0016-engineering-standards.md`, under standard 3.
~~Sixteen rows as of 2026-09-16 (`verify-serde` last printed "90 types, 16
exemptions").~~ **Corrected 2026-09-23: seventeen rows** — the seventeenth,
`Transfer` in `vpay-adapter-mtn-momo/src/wire.rs` (MTN's camelCase
Disbursements wire), arrived with the refund path
([vpay#178](https://github.com/vaam-apps/vpay/pull/178)) on 2026-09-16, so the
old figure was already stale on the tree it was stamped against. Measured on
vpay `b747e5d5`, `cargo xtask verify-serde` prints **104 serialisable types,
17 exempted** (it was 90 and 16 on 2026-09-12). Re-run it rather than trusting
either number. Each row is `Type | File | Reason`. Adding a row is the only
sanctioned escape, and:

**It is read in both directions.** A row naming a type that now complies, or a
type that no longer exists, **fails the build** — "a stale exemption describes
a decision the code has already reversed, and the next person reads it as
current."

**The gate cannot judge a reason.** ADR-0016 is explicit: "'models MTN's
camelCase Collections wire' and 'too many to fix' are both non-empty strings —
so it only refuses a blank one. The table exists to put the sentence where a
reviewer will see it." Honesty is the reviewer's job, and it is the _only_
part of this standard left to a reviewer.

**Why a table in an ADR rather than a constant in `.xtask`**: "a constant in a
Rust file is not where a reviewer looks for the reason an exception exists, and
a reason nobody reads is the same as no reason." The same instinct puts
`verify-status`' token list in `docs/status.md` and the parity matrix in
`docs/sdks/parity.md`.

**Editing the table is expected.** ADR-0016 is immutable like every other ADR,
but it says of itself: "Adding or removing a row is a change to an accepted
decision's _data_, not to the decision; it is expected, and the gate is what
keeps it honest."

## The shape of an existing row, to copy

| Type           | File                                                 | Reason                                                                                                                                                       |
| -------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `RequestToPay` | `backends/crates/vpay-adapter-mtn-momo/src/wire.rs`  | Models MTN's camelCase Collections wire (`externalId`); the per-field `rename`s are what make it exact.                                                      |
| `ExpiresIn`    | `backends/crates/vpay-adapter-mtn-momo/src/token.rs` | `#[serde(untagged)]` — variant names never reach the wire, so there is nothing for `rename_all` to rename.                                                   |
| `Currency`     | `backends/crates/vpay-core/src/money.rs`             | `rename_all = "UPPERCASE"`: ISO-4217 codes, not vpay field names. `"XAF"` is the spelling the database, both adapters and `Currency::code` already agree on. |

Three exemption _shapes_, and they are the only three that have ever been
accepted: **it models a foreign wire**, **it is `untagged` so no variant name
reaches the wire**, or **it is a non-snake_case vocabulary vpay does not own**
(ISO-4217).

## What to do when the gate fails on your change

You added a serialisable type, or touched a file with one. In order:

1. Is it vpay's own wire? Add `#[serde(rename_all = "snake_case")]`. Done.
2. Does it model a rail's JSON, an OAuth response, a callback body? Add a row
   to ADR-0016's table naming the rail and the field that proves it (an
   `externalId`, a `financialTransactionId`). Do not write "foreign wire" and
   stop.
3. Is it `#[serde(untagged)]`? Add a row saying so — the reason is that no
   variant name reaches the wire.
4. Is the type you are exempting one you did not have to touch? **Stop.**
   ADR-0016's migration rule: "Do not, in particular, add an exemption row to
   make a gate pass on code you did not have to touch."

## Where the rule used to live

Before ADR-0016 the convention was written in `docs/reference/rails.md`, in the
module doc of `vpay-adapter-mtn-momo/src/wire.rs`, in the module doc of
`vpay-adapter-orange-money/src/wire.rs`, and in a comment above
`vpay_core::Currency` — "four correct statements of the same rule, in four
files, checked by nobody", while 28 of 64 serialisable types did not carry the
attribute and only 15 of the 28 had a reason. `docs/reference/rails.md`'s
section is still there and is still correct; the gate is what makes it
load-bearing.
