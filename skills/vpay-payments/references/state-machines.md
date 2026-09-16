# The state machines

_Verified against vpay `7a79684e` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Every enum below is in `backends/crates/vpay-core/src/state.rs` except
`FailureCode`, which is `failure.rs`. Verified **2026-09-16**.

Each enum carries the same three things, and the pattern is worth copying if
you add one: `ALL` (so an exhaustive test can be written once over the list
rather than twice over the table), `as_wire_str` with the labels **written out**
beside the `serde` rename, and `from_wire` that reads `as_wire_str` back rather
than repeating the table — so the two directions cannot disagree.

The wire labels are written out on purpose. They are simultaneously the
`serde` form, a Postgres CHECK's vocabulary, `schemas/vpay.cstack`'s enum and
both SDKs' enum. Nothing ties those four together automatically, so the literal
strings are kept visible where a reviewer will see a mismatch.

## `IntentStatus` — the merchant-facing one

| Value                     | Meaning                                                                                            |
| ------------------------- | -------------------------------------------------------------------------------------------------- |
| `requires_payment_method` | created, not yet confirmed. `INITIAL`. The only status from which `confirm` and `cancel` are legal |
| `requires_action`         | redirect rails only; carries `next_action.redirect_to_url`                                         |
| `processing`              | the rail has the charge; the reconciler owns what happens next                                     |
| `succeeded`               | the money moved                                                                                    |
| `canceled`                | cancelled **before any rail saw it**                                                               |

**There is no `failed`.** A rail-reported failure returns the intent to
`requires_payment_method` and sets `last_payment_error`. `canceled` is not a
failure state.

### `next_status`

```
                      Create   Confirm(Push)  Confirm(Redirect)  Cancel
requires_payment_method  –      processing     requires_action    canceled
requires_action          –      –              –                  –
processing               –      –              –                  –
succeeded                –      –              –                  –
canceled                 –      –              –                  –
```

`–` is `None`, which means **"that request is not legal from that status"** —
it does **not** mean the intent is stuck. `processing → succeeded` is a real
edge; it is just not a _merchant verb_, so it is not in this table.

Two `None`s that surprise people:

- `requires_payment_method + Create` is `None`, because `Create` has no source
  status at all. A new intent is born at `INITIAL`.
- `requires_action + Confirm(Redirect)` is `None`. **Once a rail has the
  request it cannot be recalled, and there is no second confirm** — one charge
  per intent, forever.

`Cancel` is legal only from `requires_payment_method`. Cancelling a
`processing` intent is a `409`, not a rail call.

## `ChargeState` — operator-facing, internal, never on the wire

| Value        | Meaning                                         | Still polled? |
| ------------ | ----------------------------------------------- | ------------- |
| `submitting` | the row exists; the rail has not been asked yet | yes           |
| `submitted`  | submitted, no answer yet                        | yes           |
| `pending`    | the rail acknowledged and is working on it      | yes           |
| `unresolved` | past the 24 h horizon with no terminal answer   | **yes**       |
| `succeeded`  | the rail says the money moved                   | no            |
| `failed`     | the rail refused                                | no            |

`ChargeState::is_live()` is the reconciler's predicate and
`is_terminal()` its complement.

**`unresolved` is escalated, not abandoned.** It is still polled — once an hour
(`vpay_worker::UNRESOLVED_POLL_INTERVAL`), with an alert raised for a human.
"Polled" is literal: each hourly run asks the rail again, and a terminal answer
settles the charge exactly as it would have on the first rung. A charge nobody
asks about is one whose late success is lost. Never make `unresolved` terminal.

There is **no `ChargeObject`** in `vpay_api::model`. A charge is internal;
merchants see the intent.

## The crash-safety ordering rule

`docs/flows/crash-safety.md`, and it is the invariant the whole confirm path is
built around:

> **Never let a payer act on a transaction you cannot name.**

- **Push (MTN):** the prompt reaches the payer's handset, so the payer can act
  _before_ we learn whether submission succeeded. The reference must be durable
  **before** submitting. That is what `submitting` is for.
- **Redirect (Orange):** the payer cannot act until we hand them a URL, so the
  rail's token must be durable **before** redirecting.

`vpay_worker::recovery` (`RecoveryPolicy`, `SubmitAttempt`, `recovery_step`) is
what decides whether an interrupted submit may be retried. A rail timeout on a
push rail after the payer may already have acted must **not** be retried
blindly — that is why `Classify::retry` can be overridden per leaf rather than
derived from the category alone.

## `RefundStatus`

| Value       | Meaning                                   |
| ----------- | ----------------------------------------- |
| `pending`   | submitted to the rail; money not back yet |
| `succeeded` | the rail returned the funds               |
| `failed`    | the rail refused. **Terminal**            |
| `canceled`  | withdrawn before the rail acted           |

Deliberately not `IntentStatus`, and deliberately carrying a `failed` the
intent has none of: a refused refund _is_ failed and stays that way, whereas a
declined charge falls back to `requires_payment_method` so another rail can be
tried. **Refunds do not change their intent's status at all**, so sharing one
type would invite exactly the assignment the flow doc forbids.

~~`canceled` is in the vocabulary because migration `0017` put it there, not
because anything reaches it, and nothing writes a refund at all.~~
**Corrected 2026-09-16 (RFC-0003 §§ 2-3).** A merchant creates refunds now,
and the reachability of each value is uneven in a way worth spelling out:

| Value       | Reached by                                                                                           |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| `pending`   | `POST /v1/refunds` — and **nothing moves a refund out of it except a refusal**, the row below        |
| `failed`    | the rail declining, or a `Config`/`NotImplemented` on the way to it (`fail_with_event`)              |
| `canceled`  | `POST /v1/refunds/{id}/cancel`, which **refuses every refund whose transfer was already instructed** |
| `succeeded` | `Settlement::apply_refund_succeeded` only — **reached by nothing a merchant can cause**              |

Read those four rows together: a refund the rail **accepted**, and a refund
whose outcome is **unknown** (`Transport`/`Malformed` — the handler returns
`201` with the pending row and logs that nothing will move it), both stay
`pending` indefinitely. Only a refusal moves a refund, and it moves it to
`failed`. "Nothing settles a pending refund" is the exact claim; "every refund
stays pending" is the loose one, and it is wrong about Orange, where a refund
create fails on `NotImplemented("orange_money::refund")`.

**There is still no transition function for this enum, and none should be
invented.** Every move is a compare-and-swap in the statement — the cancel
carries `AND status = 'pending'` plus
`NOT EXISTS (… provider_requests … 'refund')`, and a Rust guard beside it is
the thing a future writer calls _instead of_ taking the lock. That `NOT EXISTS`
closed a measured double-payout hole on 2026-09-16 (a cancel released the
reservation `no_over_refund` is computed from, and the same money was refunded
twice); since nothing settles refunds, it means **no refund a merchant creates
is cancellable** — the attempt row is written before the rail call, so by the
time they hold the `re_…` there is one.

`succeeded` is the one to be careful with. An `Ok` from `ProviderAdapter::refund`
is the rail **accepting an instruction**, not money moving — `Refunded` has no
status field and the port has no refund status read (RFC-0003 open question 8)
— so writing `succeeded` on one would tell a merchant something no response
said. `pending` is what the handler writes, deliberately.

The over-refund guard is in the database, not in Rust:
`no_over_refund CHECK (amount_refunded + amount_refund_pending <= amount)` on
`payment_intents` (migration `0003`), plus `fee_non_negative` on `refunds`
(migration `0031`). It is **reachable from a merchant request since
2026-09-16**: `Refunds::create` reserves the amount in the transaction that
writes the row, so the CHECK is what answers `409 over_refund`.

## `InvoiceStatus`

```
draft ──finalize──> open ──> paid | void | uncollectible
  └────void────> void        (also: DELETE, which removes it entirely)
```

| Value           | Meaning                                                                                                                                  |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `draft`         | being edited; lines mutable, no number, cannot be paid                                                                                   |
| `open`          | issued; has a number, lines frozen, waiting to be paid                                                                                   |
| `paid`          | terminal; only reachable with `amount_remaining = 0`                                                                                     |
| `void`          | cancelled by the merchant. Terminal. **Keeps its number** — a number that vanished is a hole an accountant reads as a destroyed document |
| `uncollectible` | written off: still owed, never expected. Terminal                                                                                        |

An invoice is a _document_; an intent is an _attempt to move money_. They share
not one state, which is why overloading `IntentStatus` here would put "somebody
wrote this off" in the same type as "the rail is thinking about it".

**No `can_transition_to` method, deliberately** — see the SKILL page. Each
transition is its own `POST` with its own preconditions, its own event and its
own refusal, rather than a `PATCH` with a `status` field: a `status=void`
parameter would make "which transitions exist" something a merchant discovers
by trying.

`amount_refunded` on an invoice is **gross** and does not reopen a `paid`
invoice (D5, migration `0042`): a fully refunded invoice stays `paid` with
`amount_remaining` at `0`. **Its stored value is `0` in every deployment**, and
the reason changed on 2026-09-16: it used to be that nothing created a refund;
now a merchant can, but the single statement that writes this column runs only
inside the settlement that moves a refund to `succeeded`, and nothing settles a
`pending` refund.

## `ProviderFlow`

Two values, `push` and `redirect`, and the core branches on **this value**,
never on a provider code (ADR-0002). `flow.status_after_confirm()` is the only
place the fork is written.

`if provider == "mtn_momo"` outside `backends/crates/vpay-adapter-*` is a
defect, and **nothing greps for it** — it is caught by review or not at all.

## `FailureCode` — a closed vocabulary owned by the core

Eleven codes. Every adapter maps its rail's error strings into this list;
merchants integrate against it once and **it does not grow when a rail is
added**.

| Code                       | Payer can act | Merchant can act |
| -------------------------- | ------------- | ---------------- |
| `insufficient_funds`       | ✅            |                  |
| `payer_timeout`            | ✅            |                  |
| `payer_declined`           | ✅            |                  |
| `payer_limit_reached`      | ✅            |                  |
| `invalid_payer`            |               |                  |
| `payer_account_blocked`    |               |                  |
| `invalid_payee`            |               | ✅               |
| `payee_account_blocked`    |               | ✅               |
| `provider_account_blocked` |               |                  |
| `provider_unavailable`     |               |                  |
| `provider_error`           |               |                  |

`payer_actionable` means "the payer could plausibly succeed on a **fresh**
PaymentIntent" — never on the same one. The two predicates are **never true at
once**, and a code that is neither is the _operator's_:
`provider_account_blocked` means your own partner account is blocked, and it
pages.

`provider_error` is the unmapped case and always arrives with the raw reason
attached. **A rising rate of it means an adapter's mapping table has drifted
behind the rail — alert on it, do not tolerate it.**

`LastPaymentErrorObject.code` on the wire is a `String`, not this enum: the
vocabulary is enforced where it is _written_, not on the read path, so a code
this build cannot name never turns a merchant's `GET` into a `500`.

## Where the state lives, versus where it is enforced

| Invariant                                        | Enforced by                                                                                                 |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| legal intent transition                          | `next_status` **and** a compare-and-swap `UPDATE ... WHERE status = …`                                      |
| one charge per intent                            | `CREATE UNIQUE INDEX one_charge_per_intent` (migration `0004`)                                              |
| one invoice per intent                           | the same device, migration `0036`                                                                           |
| `paid` implies nothing remaining                 | `paid_means_nothing_remaining` CHECK (migration `0036`)                                                     |
| no over-refund                                   | `no_over_refund` CHECK (migration `0003`) — reachable from `POST /v1/refunds` since 2026-09-16              |
| a refund the rail already has is not cancellable | `NOT EXISTS (… provider_requests … 'refund')` in the cancel statement (2026-09-16)                          |
| a ledger transaction balances, per currency      | `vpay_ledger::Transaction::validate()`, called by `vpay_db::ledger::post_in_tx` before any statement        |
| a `merchant_id` iff `merchant_payable`           | `ledger_entries_merchant_id_iff_merchant_payable` CHECK (migration `0045`) **and** `AccountKind`'s sum type |
| non-negative amounts                             | four CHECKs on `payment_intents`, plus `Money::new`                                                         |
| `supports_partial_refunds ⇒ supports_refunds`    | `Capabilities::is_coherent` in Rust **and** `partial_refunds_imply_refunds` CHECK (migration `0002`)        |
| event type is in the vocabulary                  | `type_is_a_documented_event` CHECK (migration `0039`)                                                       |

The pattern: a rule that a concurrent writer could violate lives in the
**statement or the index**, and a Rust guard beside it is at best a nicer error
message and at worst the thing a future writer calls instead of taking the
lock.

## A note on `schemas/vpay.cstack`

It transcribes `IntentStatus`, `ChargeState`, `RefundStatus`, `InvoiceStatus`,
`ProviderFlow` and `FailureCode` variant-for-variant in their snake_case wire
form, and it **compiles into `vpay-db`** — a mismatch is a build failure, not a
gate failure. But `model PaymentIntent`, `model Charge` and `model Refund` are
type-checked **sketches that nothing queries**; the money tables are read
through hand-written sqlx. `backends/migrations/*.sql` is the authoritative
schema, and the drift between the two is _measured_, not closed
(`docs/status/cratestack.md`).
