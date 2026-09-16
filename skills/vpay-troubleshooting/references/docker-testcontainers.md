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
