# The confirm/poll race

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

A defect worth knowing in full, because the shape of it recurs: **a recovery
path whose precondition ("the process died") was never actually checked.**

Fixed 2026-09-04 (Step 8, lane G). Recorded in
`docs/runbooks/demo/known-flake.md`.

## The symptom

`just demo` returned `500` with:

```text
write_matched_no_row
no row in charges matched ch_… or it was no longer in the required state
```

`alert: true` — it paged.

Four of six demo runs hit it on a loaded machine.

## The mechanism

`insert_charge` commits, in one transaction:

- the charge, in state `submitting`, and
- its `poll_charge` job, with `run_at = now()`.

The worker's `IDLE_SLEEP` is 1 s. So within about a second of the merchant's
confirm being _sent_, the worker claims that job, finds a charge in
`submitting`, and applies the crash-recovery table — whose precondition is that
the process handling the confirm **died**.

It had not died. It was mid-flight, holding the rail's token.

Measured window: **3.7 seconds** on a loaded machine.

## Two distinct bad outcomes

On **MTN** (push): the merchant received a `500`, _and_ a
`payment_intent.succeeded` webhook was delivered. The two disagreed about the
same payment.

On **Orange** (redirect): a **live order was killed** and mislabelled
`provider_unavailable`, while the confirm that owned it was still in flight
holding the `pay_token`.

The second is the worse one, and it is the reason the fix is a predicate rather
than a retry.

## The fix

`recovery_step` now answers `RecoveryAction::Wait` for any `submitting` charge
younger than `RecoveryPolicy::not_found_window` (60 s), measured from
`charges.created_at` — **by Postgres, not by the worker's host clock.**

Deleting that one predicate reproduces the failure, including the merchant's own
error text. That is the mutation that proves the fix; if you touch
`recovery_step`, run it.

## What this teaches

**A recovery path must check its own precondition.** "The process died" is not
established by "the row is in the state a dying process would have left it in" —
a row mid-write looks identical to a row abandoned mid-write. Something has to
distinguish them, and here it is elapsed time measured by the one clock both
parties share.

## If you see this today

**Report it.** The tree carries the fix, so an occurrence would be the first
observation of the defect on a fixed tree — which is a finding, not a flake.

## The neighbouring symptom that is _not_ this

`just demo` can also fail on a slow or loaded host with settlement landing after
the demo's budget (120 s / 30 s). The tell is the worker log: it shows the
settlement and the webhook arriving, just late, with Postgres answering single
statements in 14–36 s under host I/O pressure above 50%.

That is host load. Reduce it and re-run. Do not go looking for the race.
