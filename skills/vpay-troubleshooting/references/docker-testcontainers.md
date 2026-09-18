# Docker and testcontainers

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Everything Postgres-backed in this workspace starts a real container. Nothing
skips when Docker is absent — `README.md`: the suites "**fail loudly** without a
reachable daemon — they never skip, so a green run is a real one." The adapter
conformance suite needs Docker too, because a stub rail is a host reached over
HTTP (ADR-0006), not an in-process double.

## Rootless Docker

**Symptom:** `Error: postgres:16-alpine container starts … Caused by: … (Connect)`.

**Cause:** `testcontainers` reads `/var/run/docker.sock` by default and reads a
rootless socket path **only** from `DOCKER_HOST`. Your `docker` CLI works
because it uses a context; the test harness does not.

**Fix:**

```bash
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
```

On the authoring machine that is literally `unix:///run/user/1000/docker.sock`,
and it appears in the header of nearly every `docs/plans/*/opus-review.md` as
part of the recorded run environment. Copy that habit: state the daemon you ran
against.

You also need `postgres:16-alpine` pulled.

## `failed to create a container: Timeout error` at ~120 s

The 120 s figure is testcontainers' container-create deadline. A failure at
`120.0xx s` is that deadline, not your code.

**How to tell it is the host and not the tree,** the four signals the reviews
use:

1. It fails on a **different, untouched** test each run.
2. It is never an assertion — always a container-start error.
3. The same suite passes in isolation, or on the same tree at lower load.
4. `docker ps -a` shows `Created`-state debris and/or the load average is
   above ~9.

**Fix:**

```bash
docker ps -a --filter label=org.testcontainers.managed-by=testcontainers
# remove Created-state containers older than ~30 minutes, then re-run
```

Recorded instances, all on the authoring host, all resolved without a code
change:

| Evidence                                                                                                     | Outcome                                                 |
| ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------- |
| 250 containers stuck in `Created`, accumulating for days; another agent running the same suite at load 13–24 | 240 removed, then **1290/1290 passed**                  |
| load 19, `fs.inotify.max_user_instances` 128, 24 `created`-state containers                                  | debris removed, re-run **0**                            |
| five consecutive failures at 1151/1151/1151/1225/1270 of 1696, load 25, another worktree's e2e stack up      | same suite **1696/1696 at load 3**, Rust byte-identical |
| an abandoned `vpay-demo` compose stack crash-looping on the daemon                                           | stack torn down, suite green                            |

**Report it, do not absorb it.** The house phrasing for a run that died this way
is "an environment failure, not a code one… **Not counted as a result**", with
the load average and container count stated. A reviewer who silently re-ran
until green has hidden a signal.

## `RootlessKit PortManager.AddPort(): bind: address already in use`

**Cause:** testcontainers' random host-port mapping racing another container
start under rootlesskit's port manager. Also seen as testcontainers' own
"container port" error.

**Fix: already structural — do not re-solve it.** `.config/nextest.toml` puts
every container-starting package into a `postgres-containers` test group with
`max-threads = 1`, serialising container _starts_ while each test keeps its own
container and its own `Drop`-based cleanup.

**If you add a package whose tests start a container, add it to that `filter`.**
The current list is `vpay-tests-integration`, `vpay-tests-conformance`,
`vpay-db`, `vpay-server`, `vpay-testkit`. The group is "our tests that ask
Docker for a port", not "our tests that use Postgres" — WireMock starts count.
A `package(...)` naming a package that does not exist is a **hard nextest
config error, not a no-op**, so the list cannot silently rot.

**Do not "optimise" this into a shared container.** That was tried and
rejected, with evidence: a `ContainerAsync` held in a `static` never runs `Drop`
at process exit (Rust does not drop statics), so a shared container leaks the
underlying Docker container forever. Verified by running that version 8+ times
and finding hundreds of orphaned `postgres:16-alpine` containers still `Up`,
which drove the Docker VM into memory pressure and OOM-killed an unrelated
container. `max-threads = 3` was also tried and still failed roughly 1 run in 14. The tradeoff taken is wall-clock, not correctness.

## `OCI runtime exec failed: … current working directory is outside of container mount namespace root` / `possible container breakout detected`

**Cause:** a known **rootless-Docker shim fault on the host**. It hits the
container's `HEALTHCHECK` exec, so Docker reports the container `unhealthy`
while the container itself keeps answering real requests correctly.

**Symptoms it produces upstream:** `docker compose … up -d --wait` fails with
`container <name> is unhealthy`; one recorded instance had **157 consecutive**
such health-log entries while `/healthz` on the server that stack brought up
answered `200` throughout.

**Fix:** `docker restart <that one container>` (not the daemon) was enough to
get the Orange-redirect e2e tests passing without retry. Or check the
container's own logs and `/healthz` and proceed.

**It is not a defect in the recipe or the stack.** Record it as a host fault.

## Debris and teardown discipline

Two habits the reviews enforce on themselves and you should copy:

- If you kill a run, tear the stack down explicitly (`docker compose … down -v`)
  and **confirm** containers, network, volume and every overridden port are
  gone. Orphaned stacks are the cause of the next person's timeout.
- `just demo-down` removes containers **and their volumes**. It is also the fix
  for a stale `pgdata` — see
  [migrations-database.md](migrations-database.md).

## CI does not look like your machine

`.github/workflows/ci.yml`'s `rust` job has **no WireMock services** and none
should be added — the conformance suite starts its own containers via
testcontainers. CI's `rust` job also installs Node and builds the Node SDK
before running tests, which is why one Rust test fails on a fresh local worktree
and never in CI (see [node-web.md](node-web.md)).

### macOS has one loopback address; Linux has sixteen million (2026-09-18)

Four cases in `backends/tests/integration/tests/staff_sign_in.rs` simulate
distinct client source addresses with `reqwest::ClientBuilder::local_address` —
`the_sign_in_rate_limit_is_per_source_address`,
`the_second_factor_is_rate_limited_and_not_only_the_password`,
`a_forwarded_for_header_from_an_untrusted_peer_buys_no_fresh_budget` and
`two_replicas_share_one_sign_in_budget`. They are the only coverage of the
per-source-address half of the sign-in rate limit, and the address is the
thing under test, so none of it can be faked with a header: two of the four
exist precisely to prove `X-Forwarded-For` is **not** honoured from an
untrusted peer.

Linux assigns the whole `127.0.0.0/8` to `lo`, so those binds succeed and CI is
green. macOS assigns only `127.0.0.1` to `lo0`, so `bind()` returns
`EADDRNOTAVAIL` and all four fail — deterministically, not flakily, and
identically whether run in the full workspace or alone. The bind happens at
**connect** time rather than at `ClientBuilder::build()`, so before 2026-09-18
this surfaced as an opaque reqwest transport error:

```text
error sending request for url (http://127.0.0.1:PORT/dash/v1/staff/login)
  1: client error (Connect)
  2: tcp bind local error
  3: Can't assign requested address (os error 49)
```

The fix is host setup, once per boot, not a change to the tests:

```bash
just loopback-aliases
```

It aliases every address the suite uses (`127.0.0.2`, `.3`, `.4`, `.20` and
`.30`–`.35`) onto `lo0` and is a no-op on Linux. The aliases do **not** survive
a reboot. Since 2026-09-18 the suite preflights the bind through a shared
helper, so an unaliased machine fails with a message naming this command
instead of `os error 49`.

**Do not "fix" this by ignoring or gating the four cases.** A macOS developer
seeing red here is correct; a green run that skipped them would claim the
per-address limiter is covered when nothing measured it.

### `/__admin/health` proves nothing about WireMock's stubs (2026-09-18)

A conformance or integration case that reaches a WireMock host and gets a
**404 from a live server** — e.g.
`Config("orange_money: the token endpoint answered HTTP 404; check
providers[].host.url")` — is not a misconfigured `host.url`. It is a readiness
gap.

`HealthCheckTask.execute()` at the pinned `wiremock/wiremock:3.9.2` tag never
touches the `Admin` it is handed: it returns a hardcoded HTTP 200 on every
call. A passing Docker healthcheck therefore proves the JVM started and
`/__admin/*` is routed, and **nothing about whether the bind-mounted mappings
are being served on the mapped host port**. `compose.yml`'s comment claimed
otherwise until 2026-09-18 and is now struck through in place.

`vpay_testkit::containers::start_wiremock` gained a second gate the same day:
it polls `GET /__admin/mappings` on the mapped host port until WireMock's own
`meta.total` is positive, bounded by a 5s deadline. Every caller gets it, so a
rail request can no longer be issued against a host reporting zero mappings.
If you add a `start_wiremock` caller, point it at a **non-empty** mappings
directory or that gate will time out by design.

### A bounded retry over `run_once` is a flake, not a test (2026-09-18)

`vpay_worker::seed_singletons` stamps a job's `run_at` from the **test
process's** clock (`OffsetDateTime::now_utc()`), while `Jobs::claim` selects
`WHERE run_at <= now()` — evaluated by the **Postgres server** inside its own
testcontainer. Those are two clocks. A container reading milliseconds behind
the host makes a just-seeded job unclaimable, and `run_once` answers `None`.

`None` means "nothing claimable _this instant_", never "nothing left to
claim". The shipping `run_loop` knows this — `Ok(None) => idle(…)` waits out
`IDLE_SLEEP` and asks again — so this is invisible in production and shows up
only in a test that gives up. A `for _ in 0..N { … else { break } }` loop
around `run_once` will pass on a quiet machine and fail under a full-workspace
run. **Poll on a wall-clock deadline instead**, re-entering `run_once` on
`None`; `checkout_sessions.rs`'s `run_until` is the worked example, and
`worker_kill9.rs`'s `wait_for_settlement` is the older one.
