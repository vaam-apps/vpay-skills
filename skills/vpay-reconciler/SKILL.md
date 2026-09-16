---
name: vpay-reconciler
description: The vpay worker — the job loop, the poll and delivery ladders, lease reaping, crash recovery, SIGTERM draining, and the worker-concurrency bound. Load this when changing anything in vpay-worker, adding a job kind, touching the retry ladders, debugging a stuck or re-run job, reasoning about what happens when a worker is killed mid-charge, or working on settlement and crash safety.
---

# The vpay worker

> **Verified against vpay `7a79684e` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`vpay-server worker` runs the job loop. It is what actually moves money: the
API's confirm submits a charge and queues a job; **the worker's authenticated
status query is the only thing that settles it.**

Crate: `backends/crates/vpay-worker/`. Binary entry:
`backends/apps/vpay-server/src/worker.rs`. Flow docs:
`docs/flows/reconciler.md`, `docs/flows/crash-safety.md`.

## The rule everything else follows from

> **Never let a payer act on a transaction you cannot name.**

Push rails: persist the reference _before_ submitting. Redirect rails: persist
the rail's token _before_ redirecting. A payer who has been debited against a
reference we never wrote down is a reconciliation we cannot perform.

And its corollary: **callbacks are hints.** `parse_callback` returns identifiers
only — the port has nowhere to put a status, deliberately. A callback can do
exactly one thing: pull an already-queued `poll_charge` job forward. It never
writes charge or intent state.

## `vpay-worker` cannot reach the database directly

It has **no `sqlx` dependency and cannot grow one**. Every statement goes
through `&dyn Repositories`; two writes that must commit together go through
`UnitOfWork::transaction`. No `sqlx` type is nameable from here, so "forgot to
commit" is not expressible.

The handlers are deliberately **not** unit-tested. Their proofs are the
integration suites (`worker_kill9.rs`, `worker_recovery.rs`,
`worker_claim_latency.rs`, `worker_e2e.rs`).

## Two ladders, and they are not the same ladder

|       | `poll_delay` — charge polling                                                       | `delivery_delay` — webhook delivery   |
| ----- | ----------------------------------------------------------------------------------- | ------------------------------------- |
| Shape | 10, 20, 30, 45, 60, 90 s, then 120 s to the 30-minute mark, then **15 min forever** | 10 s, 30 s, 2 m, 10 m, 1 h, 6 h, 24 h |
| Ends? | **Never runs out**                                                                  | Seven rungs, then `None` = exhausted  |

The asymmetry is the point. A charge's true status is knowable indefinitely and
giving up on it means losing a payer's money, so polling never stops. A webhook
has a receiver that may simply be gone, so delivery does. Do not "unify" them.

`UNRESOLVED_POLL_INTERVAL` is 1 h and is deliberately _not_ the last rung of
`poll_delay`.

Fan-out has its own ceiling: `FANOUT_MAX_ATTEMPTS = 5`, after which the event is
`fanout_state = 'failed'` and **nothing retries it**.

## Lease discipline — the part that bites

Every write that ends a lease is **guarded on `locked_by`**. A worker whose
lease was reaped mid-run therefore _discards its answer_ rather than stamping it
over whoever holds the job now.

That outcome has its own name and its own counter: `Disposition::Lost` — "not an
error; the honest name for 'we were too slow'". It is counted separately from
`Finished`, `Rescheduled` and `DeadLettered` because any `Lost` means a lease
shorter than a handler, which is a real defect and invisible if folded into
either neighbour. **If you see `Lost` climbing, the lease is too short — do not
widen it away by folding the counter.**

Only `Retry::Never` reaches `DeadLettered` (parked at `run_at = 'infinity'`).

## Crash recovery

`recover_and_seed` **reaps before it seeds, unconditionally.** A SIGKILLed
worker leaves every held job with `locked_at` set, and `Jobs::claim` matches
only `locked_at IS NULL`. If the dead worker held `sweep:expired`, the job that
would have reaped it is itself stranded — a deadlock the queue cannot leave on
its own. The reap uses `policy.lease`, so a co-running worker's fresh leases are
untouched. There are **two lease reapers on purpose**: the hourly `sweep_expired`
job and the in-process `reaper_loop`.

`recovery_step` is `docs/flows/crash-safety.md`'s recovery table as one pure
function. It disambiguates "we crashed before the POST" from "the POST went out
and the answer was lost", using `provider_requests` as the only evidence. Two
properties to preserve if you touch it:

- **Every duration is measured by Postgres, never the worker's host clock** —
  there is no `Instant` in the signature.
- **It branches on `ProviderFlow`, a capability value, never a rail code.**

`RecoveryPolicy`: `not_found_streak: 3`, `not_found_window: 60 s`,
`lease: ≥ 4 × DEFAULT_REQUEST_TIMEOUT`, `unresolved_after: 24 h`.

The `not_found_window` predicate is load-bearing and was added for a measured
defect — see `references/the-confirm-poll-race.md`.

## SIGTERM, and the race that was actually there

`ShutdownSignals::install()` is **the very first thing in `main`**, right after
CLI parsing and before subcommand dispatch. This is not tidiness.

`tokio::signal::unix::signal(kind)` registers the OS handler _synchronously
inside the call_. `tokio::signal::ctrl_c()` is an `async fn` that registers on
**first poll**. Both binaries used to build their shutdown future as an argument
to `with_graceful_shutdown(..)` — so until that future was first polled, SIGTERM
kept its default disposition: immediate termination, in-flight requests dropped.
The window was tens of milliseconds, longer under load. SIGINT is handled the
same way to get the same guarantee.

**The grace clock starts when shutdown is signalled, not at boot.** Racing
`timeout(grace, …)` from process start would abort every in-flight job `grace`
seconds after startup, forever. A clean drain hands back no lease by
construction; a timed-out drain aborts the tasks and releases their leases, and
`LoopReport::drain` distinguishes the two so the binary can log and exit
differently.

Note `--shutdown-grace-seconds` is honoured by the **serve** mode only. The
worker accepts it, logs it, and does nothing with it. Neither binary's timeout
case is covered by a test.

## The concurrency bound

```
pool_max = vpay_db::MAX_CONNECTIONS   // 10, not configurable anywhere
max_safe = pool_max / 2               // 5
concurrency > max_safe  =>  StartupError::WorkerConcurrencyExceedsPoolSize
```

Checked **at boot, before the pool is opened and before the database is
touched** — so a bad knob costs milliseconds rather than a connection and a
migration run.

Why halved: `create_in_tx` on the already-exists branch holds **two**
connections at once — its own transaction's, and the one CrateStack's
update-policy re-check takes from the same pool.

Raising `MAX_CONNECTIONS` is a code change that moves this ceiling with it.

## Job kinds

`PollCharge`, `ResubmitCharge`, `SweepExpired`, `FanOutEvents`,
`DeliverWebhook`, `ScanDeliveries`, `SweepIdleCustomers`, and the eighth in
`JobKind::EVERY`. `IDLE_SLEEP` is a plain 1 s sleep, not `LISTEN`/`NOTIFY` — the
fastest poll rung is 10 s, so a job is never waiting on the loop.

## Also here

- `references/the-confirm-poll-race.md` — the 3.7-second window in which the
  worker applied crash recovery to a charge that had not crashed, what it did to
  a live Orange order, and the one predicate that closes it.
- Retry policy is `Classify::retry` and nothing else. There is deliberately **no
  `ProviderError::retryable()`**; a second oracle beside `Classify` is what
  ADR-0011 exists to prevent. See `vpay-conventions`.
- Webhook signing, the outbox and SSRF vetting live in `vpay-webhooks`.
