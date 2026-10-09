# Deployment: images, the chart, and what it deliberately does not do

_Verified against vpay `a33aac61` (2026-09-29), section "Publishing the chart" re-read in full; the rest of this page was last read at `9653ee94` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

Everything here renders and validates. **Nothing here has run on a cluster.**
(~~Nothing here has run.~~ The release workflow's chart-publishing job has —
see "Publishing the chart" below. No cluster has installed the chart.) See the
`SKILL.md` preamble.

## Images

`.github/workflows/release.yml` is the only thing in this repository that pushes
an image; `ci.yml` builds the backend image for the e2e stack and throws it
away. arm64 is built on native `ubuntu-24.04-arm` runners rather than under QEMU
(step-6 decision (8)).

### `backends/Dockerfile`

cargo-chef, three stages. **Only one `FROM` names the rust image** (the `chef`
stage); `planner` and `builder` are both `FROM chef` — "so the three stages
cannot drift apart by construction. **Do not turn that into three literals.**"
Runtime is `FROM scratch AS server`, `USER 65532:65532`, `EXPOSE 8080`,
`ENTRYPOINT ["/vpay-server"]`.

It passes `--target` set to the builder's **own host triple**, read from
`rustc -vV` at build time (ADR-0014), so it is never a cross-compile. It bakes
the whole `config/` directory into the image at `/config` and sets
`VPAY_CONFIG`.

`ARG VPAY_GIT_SHA=unknown` — `release.yml` passes `github.sha`; **every local
build and every `compose*.yml` do not, and nothing ever shells out to `git`.**

There was a second `FROM scratch AS worker` stage until 2026-09-07. It is gone:
"the two binaries were always one `cargo` invocation over one dependency graph,
so the second image bought a second pull and a second signature for the same
code."

### `frontends/Dockerfile`

`base` → `dashboard-builder` → `runner`
(`CMD ["node", "frontends/apps/dashboard/server.js"]`), and
`checkout-builder` → `checkout` (`USER node`, `EXPOSE 3000`,
`CMD ["node", "frontends/apps/checkout/server.js"]`).

The checkout image runs as `node` (uid 1000) with a read-only root filesystem
and a memory-backed `emptyDir` on `/tmp`, **holds no merchant credential, no
rail credential and no signing key, and reads no YAML** — its whole
configuration is two environment variables.

**Since 2026-09-16 (ADR-0022) the `runner` (dashboard) stage matches it**:
`USER node`, `HOME`/`XDG_CACHE_HOME` at `/tmp`, `HOSTNAME=0.0.0.0`, and a
`HEALTHCHECK` against `/healthz` — a route that did not exist in the dashboard
app before that commit. This page previously described the dashboard image as
builder-plus-`CMD` only, which was accurate: the stage declared no `USER` and
its behaviour under `readOnlyRootFilesystem` had never been observed, which is
why the chart refused to template it (the retired `dashboard-not-templated`
guard).

`HOSTNAME=0.0.0.0` is load-bearing, not cosmetic. Docker injects
`HOSTNAME=<container id>` into every container and Next's standalone server
binds whatever that resolves to — the bridge IP. Without the explicit value,
`docker run -p` works while **every `127.0.0.1` probe is refused**, which is
what the `HEALTHCHECK` and a kubelet's probe both dial.

### Passing `worker` as an argument

Two platforms, two spellings, and the difference matters:

- **Compose:** `command: ["worker"]`. Compose's `command:` _is_ the Docker CMD,
  and the image's `ENTRYPOINT` is `["/vpay-server"]`.
- **Kubernetes:** `args: ["worker"]`. Kubernetes `args` _is_ the CMD; its
  `command` would **replace** the entrypoint — "so the chart must not use it and
  does not."

## What the chart renders

`deploy/helm/vpay/`. Both backend workloads run **one image**; the worker
Deployment passes `args: ["worker"]`.

| Object                             | Notes                                                                                                                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `Deployment` server                | `server.replicaCount` (2) — **or no `spec.replicas` at all** when `server.autoscaling.enabled`, so the HPA governs                      |
| `Deployment` management            | optional, `management.enabled` **false** by default; fixed `replicaCount` (2), **never autoscaled**                                     |
| `Deployment` worker                | 1 replica, `strategy: Recreate`, server image + `args: ["worker"]`                                                                      |
| `Deployment` checkout              | optional, `checkout.enabled` **false** by default                                                                                       |
| `Deployment` dashboard             | optional, `dashboard.enabled` **false** by default; fixed 2 replicas, **never autoscaled**                                              |
| `HorizontalPodAutoscaler`          | optional (`server.autoscaling.enabled`), **server only**                                                                                |
| `Service`                          | ClusterIP, ports `http` (8080) and `metrics` (9090)                                                                                     |
| `Service` worker                   | **headless, `metrics` only** — exists so the worker can be scraped                                                                      |
| `Service` management, dashboard    | with their Deployments                                                                                                                  |
| `ServiceAccount`                   | `automountServiceAccountToken: false`                                                                                                   |
| `PodDisruptionBudget`              | `minAvailable: 1`; server, **and management when enabled**                                                                              |
| `ConfigMap` overlay                | optional; mounted with `subPath`. **Splits into two** when `management.enabled` — each tier needs its own `deployment.surfaces`         |
| `Ingress` ×4                       | `-api` (`/v1`), `-token` (`/v1/oauth/token`, tighter `limit-rps`), `-provider` (`/provider`, **on by default**), `-checkout` (optional) |
| `HTTPRoute`                        | optional (`route.enabled`); **one** object, **four rules** since ADR-0022 added `/dash/v1`                                              |
| `NetworkPolicy`                    | optional, default-deny both directions                                                                                                  |
| `ServiceMonitor`, `PrometheusRule` | optional; every threshold proposed. Scrapes server, worker **and management**; the dashboard exports no metrics and gets none           |

**It renders no Secret and no database.**

## Publishing the chart to GHCR — added 2026-09-19; ran for real on 2026-09-20

~~**Read this whole section as a description of code, not of an event.** … As of
this writing it has never executed: no `v*` tag has triggered it, no chart has
ever been pushed to a registry, nothing has been signed.~~ **Corrected
2026-09-29 (vpay `a33aac61`):** it has executed twice, and the first execution
found three defects. What vpay's own pages (`docs/flows/deployment.md` § 2a and
§ Status, `docs/status/infrastructure.md`'s "Helm chart publishing" row,
`docs/runbooks/release.md` § 8) record:

| Date       | Tag / run                   | Outcome                                                                                                                                                 |
| ---------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-09-20 | `v0.2.2`, run `35491807158` | thirteen jobs green, `publish-chart` red: the chart **was pushed** (as `0.2.1` — see below) and `cosign sign` then died `UNAUTHORIZED: unauthenticated` |
| 2026-09-20 | `v0.3.0`, run `35492982589` | all fourteen jobs green **including `publish-chart`**: packaged, pushed, signed. The first end-to-end success                                           |

`release.yml`'s `publish-chart` job is at lines 416-650 as of `a33aac61`
(~~392-527 as of 2026-09-19~~; the file grew). It pushes this chart to the same
registry the images use, as its own OCI artifact:
`oci://ghcr.io/vaam-apps/charts/vpay`. Helm has needed no separate
chart-repository format since 3.8, so this is a publish path, not new hosting.

**Three defects the first run found, each fixed on its own PR:**

1. **`helm` and `cosign` do not share a credential store.** `helm registry
login` writes `$XDG_CONFIG_HOME/helm/registry/config.json`; cosign reads
   `~/.docker/config.json`. The job now logs in twice (`docker/login-action`
   beside `helm registry login`; vaam-apps/vpay#223). The signature step had
   been what failed.
2. **`Chart.yaml`'s `version:` was hand-bumped, and nobody bumped it.** At
   `v0.2.2` it still read `0.2.1`, so the pushed chart was numbered `0.2.1`
   while `appVersion` said `0.2.2`, and defaulted `images.*.tag` to a release
   it was not named for. ~~**The artifact's tag is `Chart.yaml`'s hand-bumped
   `version:` field**, `version:` is deliberately not release-please's to
   manage.~~ **Since 2026-09-20 (vaam-apps/vpay#225) release-please owns
   `version:`** through an `x-release-please-version` annotation on that line,
   and `publish-chart` asserts **`version == appVersion == the tag`**
   (`version` is `0.5.0` on `a33aac61`, with `appVersion`). The cost, stated
   in `Chart.yaml`: **the chart can no longer be released independently of the
   application** — a chart-only fix waits for the next app release or gets one
   cut for it. Do not name the annotation token in prose anywhere in
   `Chart.yaml`: `verify-versions` scans every line for the literal string and
   a comment that mentions it fails the gate.
3. **The republish guard was wrong about what a hit means.** Push-then-sign is
   not atomic: after the first run pushed and failed to sign, the guard refused
   every re-run because it saw the chart that same run had pushed. It now has
   **four** answers, not three (below).

**Tags only. There is no `edge` chart** — the one place this job does not
mirror what §"Images" above describes for the three images. An OCI chart's
tag is not a label a workflow step can choose independently of
`Chart.yaml`; an `edge` chart would mean either republishing one version
over itself on every push to `master`, or inventing accumulating
pre-release strings that `helm search` would then offer as real releases.
Between releases the chart stays exactly what it has always been — a
directory in a clone, installed with `helm upgrade --install vpay
deploy/helm/vpay`, which is what this page and `just helm-check` have always
exercised. So the job runs only `if: startsWith(github.ref, 'refs/tags/v')`.

**`needs: [namespace, merge]`**, not the per-architecture `build` matrix:
`values.yaml` defaults `images.*.tag` to `.Chart.AppVersion`, so a chart
published before all three images are merged, signed and pushed would
resolve to tags GHCR does not have yet. The chart cannot exist before the
thing it points at.

**The republish guard reads four outcomes, not three.** `helm push` to an
OCI registry overwrites an existing tag with no error and no warning —
measured against a throwaway local `registry:2` (2026-09-19): pushing the
same chart twice returned exit 0 both times. Before pushing, the job runs
`helm show chart oci://ghcr.io/vaam-apps/charts/vpay --version <chart
version>` and reads it:

| `helm show chart`                                      | Reads as                                    | Job does                                                              |
| ------------------------------------------------------ | ------------------------------------------- | --------------------------------------------------------------------- |
| exit 0, and a signature exists for its digest          | published **and signed** — a real collision | fails the release                                                     |
| exit 0, **no signature**                               | a previous run died between push and sign   | **resumes**: skips the push, signs the digest already in the registry |
| non-zero, `not found`                                  | never published                             | packages and pushes                                                   |
| non-zero, anything else (connection refused, TLS, 5xx) | the registry did not answer                 | fails rather than pushing past an unanswered question                 |

~~exit 0 → "already published" → fails, naming the fix: bump `Chart.yaml`'s
`version:`.~~ That was the three-outcome design and it is the one this page
described until 2026-09-29. Resume re-signs the existing digest rather than
re-packaging, because `helm package` is not byte-reproducible: a second package
of identical content hashes differently and the signature would attach to a
copy nobody pulls. **Proven against the real registry:** the "not found → push"
branch (`v0.3.0`) and the "published AND signed" branch (checked against a
signed `0.3.0`, confirmed to report a collision and refuse). **Never fired in a
real run: the resume branch** — `v0.3.0` was a clean first-time publish, so
nothing was stranded.

A missing chart **name** and a missing **version** return the textually
identical `not found` message (measured against the local registry), which is
why the guard greps for that string instead of trying to tell the two apart.

**Signed exactly like the images: keyless cosign, GitHub OIDC, the same
certificate identity** (`release.yml` at that ref). A chart manifest is an
ordinary OCI artifact to cosign, so the same verification shape
`docs/runbooks/release.md` §3 gives for the images works on the chart with
only the reference changed:

```bash
cosign verify \
  --certificate-identity-regexp '^https://github\.com/vaam-apps/vpay/\.github/workflows/release\.yml@refs/tags/v.*$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/vaam-apps/charts/vpay:<version>
```

**What is established, and what is not.** `0.3.0`'s signature exists and
`cosign download signature` finds it. **`cosign verify` has still never been
run** against a chart manifest or against anything else this repository
produced: it was attempted from an authoring machine and could not reach
sigstore's TUF CDN (`tuf-repo-cdn.sigstore.dev`, connection refused, twice).
So the command above is written from Fulcio's documented identity format, not
from a certificate somebody read. And **no cluster has ever installed this
chart**, from a registry or any other way.

**Visibility.** ~~GHCR creates a package **private** on first push; making it
public is a one-time change a human makes in package settings.~~ **Corrected
2026-09-29:** vpay measured on 2026-09-20 that `ghcr.io/vaam-apps/charts/vpay`
answered an anonymous manifest pull with HTTP 200, i.e. the chart package is
**public**, and says this corrects "the standing assumption elsewhere that a
first push leaves a private package". That measurement was of `0.2.1`, which
was deleted from GHCR the same day; no vpay page I found repeats it for
`0.3.0` or later. The three **images** are still unmeasured and the evidence
(an anonymous token request answering `UNAUTHORIZED`) points at not
anonymously pullable.

**What to hand a user who wants to install a released chart:**

```bash
helm show chart  oci://ghcr.io/vaam-apps/charts/vpay --version <version>
helm show values oci://ghcr.io/vaam-apps/charts/vpay --version <version> > my-values.yaml

helm upgrade --install vpay oci://ghcr.io/vaam-apps/charts/vpay \
  --version <version> -f my-values.yaml
```

`<version>` is `Chart.yaml`'s `version:`. ~~It was hand-bumped, was `0.2.1` when
this page was written, and was **not** the `v*` git tag — "say so explicitly if
the person reaches for the tag from a release notification, because it will
not resolve".~~ **Corrected 2026-09-29:** since 2026-09-20 `version:` equals the
release tag without its `v` (`0.5.0` on `a33aac61`; the job refuses a tag it
disagrees with), so `v0.5.0` still will not resolve but `0.5.0` is the right
argument. `charts/vpay:0.2.1` **no longer exists**: the maintainer deleted that
mislabelled, unsigned version from GHCR on 2026-09-20. Which versions after
`0.3.0` were actually published is not recorded in the vpay pages this skill
was checked against; `helm show chart` answers it.

## Surfaces: one image, two tiers (ADR-0022, 2026-09-16)

`deployment.surfaces` in the YAML selects which surfaces a process mounts.
**Absent means every surface**, so a deployment that predates this key upgrades
unchanged; an empty list or an unknown value is a boot refusal (exit 78).

| Surface      | Mounts                                                   |
| ------------ | -------------------------------------------------------- |
| `business`   | `/v1`, `/v1/browser`, `/provider`, and `/v1/oauth/token` |
| `management` | `/dash/v1` and the `staff` sign-in routes                |

`/v1/oauth` is **split rather than assigned**: `/token` is business-only
because it is the _merchant_ private-key-JWT grant, while `jwks.json` and the
discovery document serve on both, so the management tier does not depend on the
business tier being up to validate a staff token. Do not "simplify" this by
mounting the whole nest on either surface — ADR-0017's staff grant terminates
at `/dash/v1/oauth/token`, **not** at `/v1/oauth/token`, and the two are
different grants.

It cannot be inferred from `merchant_clients`: `validate_dashboard_binding`
requires the dashboard client's `merchant_id` to exist in that list, so a
management-only process still needs it populated.

### The ceiling that binds before CPU does

`vpay_db::pool::MAX_CONNECTIONS` is a **compile-time `10` per process**, not a
chart value. Under an HPA that is the binding constraint, and the
`connection-budget` guard enforces it:

```text
(server.autoscaling.maxReplicas + management.replicaCount + worker.replicaCount) × 10
  ≤ database.maxConnections − database.reservedConnections
```

A business tier scaling to 8 with management at 2 and worker at 1 wants 110
connections against Postgres' default budget of 100 — it exhausts connections
**before** it reaches any CPU target, and the symptom is `acquire_timeout` on
whichever path asks next. `database.maxConnections` is required once
autoscaling is on; there is deliberately no default, because guessing 100 on
behalf of a managed instance is how this becomes an incident.

The `10` is duplicated into the chart on purpose — a Helm template cannot read
a Rust constant — and both sides name each other. **If you change one, change
both.**

## The three Secrets you must create first

| Value                                            | Default name                                       | Shape                                                                                             | If wrong                            |
| ------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------- |
| `database.existingSecret` / `.existingSecretKey` | `vpay-database` / `url`                            | one key holding a full `postgres://` URL                                                          | both Deployments fail to start      |
| `signingKey.existingSecret` / `.key`             | `vpay-oauth-signing-key` / `oauth-signing-key.pem` | PEM RSA private key (PKCS#8 or PKCS#1)                                                            | the server Deployment exits **78**  |
| `rails.existingSecret`                           | `vpay-rails`                                       | one key per `${VAR}` in the deployed image's config — **read it at upgrade time, the list grows** | exit **78** on **both** Deployments |

Use **`--from-env-file`, not `--from-literal`**: "a credential on a command line
is in your shell history and in `ps` output."

The signing key is mounted on the **server Deployment only**. Both _flag_
spellings of handing it to the worker are refused; `VPAY_OAUTH_SIGNING_KEY_FILE`
in the environment is ignored rather than refused, deliberately — "a shared env
block must not `CrashLoopBackOff` a worker". **So the guarantee is the volume
list in `deployment-worker.yaml`, not the flag check.**

`signingKey.defaultMode` is **`0440`, not `0400`** — with `fsGroup` set, a
Secret volume is owned by `root:<fsGroup>` and these pods run as UID 65532, so
`0400` is unreadable by the only process in the image, which exits 78 naming a
file it can see and cannot open. _Reasoned from Kubernetes' documented ownership
rule; not observed, because no pod has run._

The rail Secret is projected with `envFrom.secretRef`, so `kubectl describe pod`
shows the variable **names** and never the values.

## Postgres is deliberately outside the chart

Step-6 decision (9). `DATABASE_URL` comes from `database.existingSecret` and
from nowhere else. CloudNativePG is the documented in-cluster alternative. This
is also what makes ADR-0013 necessary in its current shape: it can state backup
obligations it cannot enforce, "because the machinery that would meet them
belongs to whoever operates the database."

## The two configuration facts that will bite you

**1. The overlay is mounted with `subPath`, and it must be.** Mounting a
ConfigMap _at_ `/config` replaces the baked directory and the process exits 78
complaining about a file it can no longer see. The consequence: the mounted file
does **not** update when the ConfigMap changes — which is fine, ADR-0003 has no
hot reload — and the chart puts a `checksum/config-overlay` annotation on both
pod templates so an overlay edit becomes a rolling restart rather than a silent
no-op.

**2. A missing or wrong-named overlay is not an error to the process.**
`Config::load_with_env` merges the overlay only `if overlay_path.is_file()`, so
a deployment that typos `config.profile` **boots happily on the image's baked
sandbox configuration — placeholder merchant keys, WireMock rail hosts — and
says nothing.** The `overlay-empty` guard catches an empty ConfigMap; **nothing
can catch a profile typo**, and there is no shell in the image to check from
inside. Compare the rendered mount path against `VPAY_PROFILE` before you
install.

**The OP's issuer comes from the overlay**, as `deployment.public_base_url`.
There is no environment variable for it.

## Rotation and rollback

Signing-key rotation is **restart-based**: `TokenManager` holds one key for the
life of the process. Update the Secret, then
`kubectl rollout restart deploy/<release>-server`.

**Rolling back to a retired `kid` crash-loops with exit 78**
(`DbError::SigningKeyRetired`), not 69. **Roll forward, never back.**

Runbooks: `docs/runbooks/deploy-and-rollback.md`,
`docs/runbooks/rotate-signing-key.md`, `docs/runbooks/rotate-rail-credentials.md`,
`docs/runbooks/release.md`. None of their `kubectl` or `helm` commands has been
run against a cluster.

## Routing

**Two Ingress objects for `/v1` and `/v1/oauth/token`** because ingress-nginx
applies `limit-rps` per Ingress _object_, and a token request costs an RSA
verification plus a database write — "the expensive unauthenticated surface".
nginx enforces the limit **per controller replica**, so the effective global
limit is roughly `limitRps × replicas`; an exact one needs Gateway API's
`BackendTrafficPolicy` and a rate-limit service, i.e. a second component to
operate.

**`/provider` is a fourth object and is on by default** — "a rail callback is
not optional for any deployment that takes money". Before 2026-09-11 no shape of
this chart routed it, and a deployment that enabled the chart's routing and
nothing else answered **every MTN and Orange callback with the controller's
404**: vpay never received the request, wrote no log line, and settlement
degraded to the poll ladder **while every object reported healthy**. Its rate
limit is tighter than `/v1`'s (10 vs 20) because `/provider` takes no bearer
token and `provider_callback.rs`'s own header says of the route that "nothing
else here is a rate limit: there is none". Its body cap is `16k`, matching the
handler's `CALLBACK_BODY_LIMIT_BYTES`.

Turning it off is **opt-out with a sentence**: `enabled: false` needs one line
in `servedElsewhere` naming what serves the prefix instead. Legitimate case:
MTN allows an IP-allowlisted callback host of its own, reached through
`providers[].callback_url`.

**Gateway API** (`route.enabled: true`) renders `HTTPRoute`s instead of
`Ingress`es, for a cluster running a Gateway rather than ingress-nginx. It is an
**alternative**, not a migration — enabling one does not disable the other, and
a cluster that turns on both gets two controllers answering for the same
hostname. One object, three rules. Rendering is gated on
`.Capabilities.APIVersions.Has`, and `helm-check` asserts that gate in both
directions.

`route.rateLimitedBy` is a free-text assertion the guard refuses only when
**empty**: "it cannot tell a true sentence from a false one, and neither can
CI." The `ExtensionRef` spelling is no better off — "the chart renders the
reference and never looks for the object it names, so a typo'd `Middleware` name
is an unmetered token endpoint that renders, validates and reports healthy."

## The checkout page needs two API URLs, and a third thing outside the chart

- **`checkout.apiUrl`** — _this pod's_ view of vpay. `middleware.ts` calls
  `GET {apiUrl}/v1/browser/checkout/origins?key=…` server-side to build the
  embedded page's `frame-ancestors`. Empty means this release's own server
  Service. **Missing entirely is not an error**: the CSP becomes
  `frame-ancestors 'none'` — correct, fail-closed, and no merchant can embed.
- **`checkout.publicApiUrl`** — _a payer's browser's_ view. Required when the
  page is enabled, enforced by a named guard rather than defaulted, "because the
  app throws on a missing `NEXT_PUBLIC_VPAY_API_URL` and a default would be a
  pod that starts, fails readiness and never says why."
- **`checkout.public_base_url` in vpay's own profile overlay** — the origin every
  payer link vpay mints is built on. **The chart cannot check it** (the overlay
  is opaque YAML to it) and a disagreement is a `session.url` that resolves to
  nothing, **with no log anywhere naming a port**.

## Verifying the chart

`just helm-check` — exactly what CI's `deploy` job runs. It lints and templates
three value sets, renders every file under `ci/guards/` requiring each to
**fail with its own guard name**, checks the expected guard names are exactly
the files on disk, greps the rendered Ingress annotations for rate-limit
ordering, asserts the Gateway API capability gate in both directions, and runs
`kubeconform -strict` over all three renders (a **skipped** schema would mean
nothing was checked, so zero-skipped is asserted).

**It is not in `just ci`** — kubeconform downloads schemas. Guards are proven
negatively: neutering a `fail` in `templates/_validate.tpl` makes the recipe
report that the guard "did not fire" and name it.

Open follow-ups the chart names itself: a kind smoke test ("deferred by decision
(9), worth doing"); `helm unittest` for object shapes; a dashboard workload once
someone has run that image with a non-root UID; a cluster run of the checkout
path-prefix shape on either mechanism; a rendered example of the rate-limit
object `route.token.filters` is meant to reference; an HPA "once anything has
measured what would drive it".
