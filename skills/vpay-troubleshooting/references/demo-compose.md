# `just demo`, compose, and two stacks on one machine

_Verified against vpay `7a79684e` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`just demo` is `just demo-up` then `just demo-walk`. Both exist separately so
the walkthrough is re-runnable against a stack that is already up.
`just demo-status` says what is running and under which project; `just
demo-down` removes the containers **and their volumes**.

The full procedure, with real pasted output, is `docs/runbooks/demo.md` — an
overview plus a directory since 2026-09-11. Its §7, §8 and §9 are the pages this
one summarises.

## `invalid_client: Client authentication failed` at step 2 of a walkthrough

**Cause: the `just` variables isolate everything Compose owns, and not `.e2e/`.**
`demo_project`, `demo_port`, `demo_receiver_port`, `demo_orange_port`,
`demo_checkout_port`, `demo_shop_port` and `demo_dashboard_port` give you
different containers, a different network and a different `pgdata` volume. But
`.e2e/` holds **one** merchant key pair and **one** profile overlay for the
whole checkout.

Bringing a second stack up with a different `demo_port` **regenerates the shared
merchant key pair**:

```
gen-demo-keys: .e2e/application-demo.yml was generated for a different demo_port than 18088 — regenerating the pair
```

The first stack's server still holds the _old_ public JWK in memory, so its
walkthrough then fails with `{"error":"invalid_client"}`.

**The rule, stated exactly:** two demos brought up in sequence coexist and both
serve; **the older one's `demo-walk` stops working from the moment the newer
one's `demo-up` runs**, and the older one's _shop_ stops authenticating too.

**Fix:** bring the second stack up _before_ you start walking the first, or
accept that only the most recently generated key pair authenticates.

**Why it has not been fixed:** `.e2e/demo-merchant/oauth-signing-key.pem` is a
literal in `.github/workflows/ci.yml` (twice), in `just stripe-compat`, in
`examples/merchant-stripe-node/index.mjs`, in `sdks/stripe-compat`, and as the
default of `examples/merchant-demo`'s `VPAY_PRIVATE_KEY_FILE` — "**and a mistake
there fails _silently_ as `invalid_client`.**"

## The same error from a port mismatch

**Cause:** `gen-demo-keys` was run with a `demo_port` that does not match the
port actually in use, writing `.e2e/application-demo.yml`'s OAuth audience for
the wrong port. Surfaces in Cypress as `invalid_client` / `InvalidAudience`.

**Fix:** re-run `gen-demo-keys` with the matching `demo_port`, **and restart the
affected containers** — the config is bind-mounted and re-read on start, not
baked into the image.

## Two stacks colliding on the Orange stub's port

**Cause, historical and worth knowing because the fix is a file nobody looks
at:** the Orange stub's `payment_url` comes from a WireMock mapping that
templates a literal `http://localhost:8082`. WireMock renders a response from
the current request alone, and vpay's submit arrives over the compose network as
`wiremock-orange:8080`, so **the stub cannot learn what the host published it
on**. `gen-demo-keys` therefore used to _check_ the pair and refuse any
`demo_orange_port` but 8082 — correct, and it meant two demos collided with no
way out but editing a committed file.

**Fixed:** `gen-demo-keys` now writes a per-project **copy** of those mappings
with the port substituted, under `.e2e/<demo_project>/wiremock-orange/`, and
`compose.demo.yml` mounts the copy (Compose merges `volumes:` by target path).
The committed mapping is untouched and stays the CI/e2e default.

Also structural: Postgres and the rail stubs publish nothing
(`ports: !reset []` in `compose.demo.yml`), because 5432, 8081 and 8082 are
fixed literals in `compose.yml` and two stacks would otherwise collide on them
however the variables were set. This is why Compose **v2.24+** is required.

## `500` / `write_matched_no_row` during a confirm

**As of 2026-09-16 this is fixed. If you see it, report it — it would be the
first observation on a tree that carries the fix.**

**Symptom:**

```
✘ orange_money · the payer completes the hosted page — confirm: … vpay API error (500)
{"level":"ERROR","fields":{"alert":true,"category":"Internal","code":"write_matched_no_row",
 "error":"no row in charges matched ch_… , or it was no longer in the required state"}}
```

**Cause — a race the demo found, not a demo bug:**

1. `insert_charge` commits the charge in `submitting` **and** its `poll_charge`
   job in one transaction, with `run_at = now()` — immediately runnable.
2. The confirm then calls the rail and finally CASes the charge
   `submitting → submitted` (`WHERE id = $1 AND state = 'submitting'`).
3. The worker is entitled to claim that job at once (`IDLE_SLEEP` is 1 s, zero
   if already busy). It finds a charge in `submitting` and applies the
   crash-recovery table, **whose precondition is "the process died". Nothing
   distinguished that from a confirm still in flight.**
4. Whichever branch it takes moves the charge, so the confirm's CAS matches no
   row and the merchant gets a `500` — with `alert: true`, so it pages.

The window is the rail call plus two commits. Normally tens of milliseconds;
**measured at 3.7 s** on a loaded machine, which is where four of six demo runs
lost it.

**The two bad outcomes, and the second is the serious one:** on a push rail the
merchant is told the confirm failed and is then sent a `succeeded` webhook; on a
redirect rail a **live order is killed** and mis-labelled `provider_unavailable`
— "having never been unreachable" — while the confirm was in flight holding
exactly that token.

**The fix:** `recovery_step` now answers `RecoveryAction::Wait` — reschedule on
the ladder's first rung, write nothing, ask nothing — for any `submitting`
charge younger than `RecoveryPolicy::not_found_window` (60 s), measured from
`charges.created_at`. **Deleting that one predicate reproduces the failure,
including the merchant's own error text.**

The lesson generalises: the `Never` branch of that recovery table already had a
60-second `not_found_window` guarding this exact class of mistake, with a
comment saying a count alone "would look identical to [a rail] that never got
it". The `Answered` and `Redirect` branches had no equivalent minimum age. If
you add a recovery branch, **ask what distinguishes "the process died" from "a
request is still in flight".** See `docs/flows/crash-safety.md`.

## `just demo` fails with the settlement landing after the budget

**Not the race.** Recorded cause: the VM's Postgres answering single statements
in 14–36 s while host I/O pressure was above 50 %, with the worker's log showing
the settlement and the webhook landing _after_ the demo's 120 s / 30 s budgets.
`write_matched_no_row` appeared in no such run. Reduce host load and re-run.

## `ClientBuilder::build()` panics on a missing rustls `CryptoProvider`

**Cause:** the workspace pins reqwest with `rustls-no-provider`, under which
`ClientBuilder::build()` **panics** if no process default was installed. An
application may install one; a library may not.

**Already structural — copy it if you write a new binary.** Both shipping
binaries call `rustls::crypto::ring::default_provider().install_default()` as
the **second** thing in `run()`, before tracing init, so no client construction
can precede it. A unit test per binary asserts `CryptoProvider::get_default()` is
`Some` afterwards and that a second call does not panic; emptying the function's
body fails both. `examples/merchant-demo` does the same in its own `main()`.

## What `just demo` does and does not prove

Six payments on both rails to every outcome each rail documents. **Every outcome
is chosen at the rail stub, never in the demo** — MTN's by the payer's MSISDN,
Orange's by the amount, "because those are the only fields of each rail's
protocol a merchant actually controls". Nothing rewrites stored state.

The three MSISDNs (`237600000ce0`, `237600000f01`, `237600000f02`) are **not
phone numbers**: the last three characters are a hex steering code the stub keys
its scenario on.

**A `succeeded` here means `vpay-worker` asked a stub and the stub said
`SUCCESSFUL`. It does not mean anyone paid.** As of 2026-09-15 exactly one real
rail call has ever been made — an MTN sandbox push, `docs/runbooks/live-sandbox-test.md`.
Orange's redirect rail has still never been called.
