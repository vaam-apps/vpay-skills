# Deployment failures

_Verified against vpay `7a79684e` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

**No pod has ever run this chart.** Everything on this page is reasoned from the
templates, the binaries' own boot code and their tests — not observed in a
cluster. `deploy/helm/vpay/README.md`'s Status section is the authority and says
so at length. For the model see **vpay-ops**.

## The pod exits 78 complaining about a file it cannot see

**Cause:** the ConfigMap overlay was mounted **at** `/config`.
`backends/Dockerfile` bakes the whole `config/` directory into the image at
`/config`; mounting a ConfigMap there replaces the directory and the baked
`application.yml` disappears.

**Fix:** mount a single file with **`subPath`**, at
`/config/application-<profile>.yml`. The chart does this.

**The consequence you must know:** a `subPath` mount does **not** update when
the ConfigMap changes. That is fine — ADR-0003 has no hot reload — and the chart
puts a `checksum/config-overlay` annotation on both pod templates so an overlay
edit becomes a rolling restart instead of a silent no-op. If you add a workload
that mounts the overlay, add that annotation too.

## The pod boots happily on placeholder credentials and says nothing

**Cause:** a typo in `config.profile`. `Config::load_with_env` merges the
overlay only `if overlay_path.is_file()`, so a deployment that typos the profile
**boots on the image's baked sandbox configuration — placeholder merchant keys,
WireMock rail hosts — and says nothing.**

**There is no gate for this.** The `overlay-empty` guard catches "you asked for
a ConfigMap and gave it no content". Nothing can catch a profile typo, and there
is **no shell in the image** to `kubectl exec` into and check.

**Fix:** check the rendered mount path against `VPAY_PROFILE` **before you
install**:

```bash
helm template … | grep -A2 'mountPath: /config'
```

## The pod exits 78 naming a signing-key file it can see and cannot open

**Cause:** the Secret's `defaultMode`. A Secret volume in a pod with `fsGroup`
set is owned by `root:<fsGroup>`, and these pods run as **UID 65532**. `0400`
leaves the file readable by root alone — unreadable by the only process in the
image.

**Fix:** `signingKey.defaultMode` is **`0440`**, not `0400`. The group read bit
is what makes it work. The step-6 design document says `0400`; the chart is a
deliberate departure from it.

_Reasoned from Kubernetes' documented ownership rule for projected Secret
volumes. It has not been observed in a running pod, because no pod has run._

## A server crash-loops with exit 78 after a rollback

**Cause:** you rolled **back** to a retired `kid`. `DbError::SigningKeyRetired`
is `Category::Configuration`, so `78`, not `69`.

**Rotation is restart-based:** `TokenManager` holds one key for the life of the
process. Update the Secret, then
`kubectl rollout restart deploy/<release>-server`.

**Roll forward, never back.**

## Both pods CrashLoopBackOff against the liveness probe

**Cause:** an image older than the `--observability-bind` listener has nothing
on port 9090, and the kubelet restarts both pods in a loop against one.

**Fix:** pin `images.*.digest`.

## Every MTN and Orange callback answered with a 404, while every object reports healthy

**The best failure story in the deployment surface, because nothing could
catch it.**

Three facts, none of which lives in the same crate as the others:

| Fact                                                                                   | Where                                                               |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| The route is mounted at the **root**, beside `/v1` and not inside it                   | `vpay-api/src/provider_callback.rs` — `PROVIDER_NEST = "/provider"` |
| The URL each rail is handed is `{deployment.public_base_url}/provider/{code}/callback` | `vpay_config::ProviderHost::effective_callback_url`                 |
| The chart's `ingress.api.path` / `route.api.path` are `/v1`                            | `deploy/helm/vpay/values.yaml`                                      |

**Nothing compiles those against each other.** Together they meant a deployment
that enabled the chart's routing and nothing else answered every rail callback
with the ingress controller's 404: **vpay never received the request, wrote no
log line, and settlement degraded to the poll ladder while every object reported
healthy.** `docs/reference/rails.md` already recorded that the callback _path_
is the half that drifts silently, "because the route lives in `vpay-api` and the
derivation in `vpay-config`, and neither crate compiles against the other".

**Fixed 2026-09-11:** `ingress.provider` and `route.provider` are **on by
default** — "a rail callback is not optional for any deployment that takes
money" — and turning it off is **opt-out with a sentence**: `enabled: false`
needs one line in `servedElsewhere` naming what serves the prefix instead. That
guard has an arm that parses `config.overlay` and refuses a `servedElsewhere`
sentence while an **enabled** provider carries no `callback_url` override. A
third arm refuses a `path` that is neither `/provider` nor `/` — "the literal
belongs to `vpay-api`, and this chart does not get to rename it."

Legitimate reason to turn it off: MTN allows an IP-allowlisted callback host of
its own, reached through `providers[].callback_url`.

## `helm lint` refuses `images.worker`

**Correct.** There is one image since 2026-09-07 (issue #77); `images.worker`
was removed and `values.schema.json` is `additionalProperties: false` — "so that
a pinned-but-unused digest cannot sit in a values file looking load-bearing."
The worker Deployment runs the server image with `args: ["worker"]`.

Argument-passing differs by platform and both spellings matter: compose's
`command:` **is** the Docker CMD, so `command: ["worker"]`; Kubernetes `args`
**is** the CMD, so `args: ["worker"]` — Kubernetes' `command` would _replace_
the entrypoint, "so the chart must not use it and does not."

## `just helm-check` reports that a guard "did not fire"

**Cause:** you changed `templates/_validate.tpl` or a file under `ci/guards/` in
a way that stopped a guard refusing its own values file. The recipe renders
every guard file and **requires each to fail with its own guard name in the
message**, and separately checks that the twenty-two names it expects are
exactly the files on disk — so deleting a guard _and_ its values file fails
rather than passing quietly.

`just helm-check` is **not** part of `just ci`: kubeconform downloads its
schemas and `just ci` is expected to work offline. Run it by hand when you touch
the chart; CI's `deploy` job runs it on every PR either way.

The guard count "**said 15 until 2026-09-10**" and had been wrong since the
sixteenth landed. It is 22.

## Alerts that fire on a healthy system

`VpayProviderErrorRateHigh`'s numerator is `error_kind!=""` — every failed port
call, which is what lets it fire during a rail outage at all — **and that set
includes `charge_declined`, a rail decision rather than a rail failure.** On
mobile money that is a large, normal share of traffic, so expect it to fire.
Whether to exclude declines is a maintainer decision to be made with measured
traffic, together with the threshold.

**Every threshold in `prometheusrule.yaml` is proposed, not derived** — the
runbooks contained no numbers to transcribe, so the figures were invented
against a system that has never taken a real payment. Each rule carries
`provisional: "true"`, and `metrics.prometheusRule.enabled` and
`metrics.serviceMonitor.enabled` are both `false` by default.

**No rule here has ever been evaluated against a real series.** No Prometheus
has polled a vpay process.

## Things the chart's own Status section says have never been verified

Read it before you trust any of these: no cluster has run this (not kind, not
minikube); `readOnlyRootFilesystem: true` is "no observed writer", not proven;
the NetworkPolicy has never been enforced by a CNI ("a cluster whose CNI ignores
NetworkPolicy and one that honours it look identical from here"); the
PodDisruptionBudget's behaviour during a rolling restart or node drain is
untested; **"nothing has verified that ingress-nginx honours `limit-rps` at all
— CI checks that the annotation is present in the rendered YAML. That is the
whole claim"**; no Gateway has ever accepted one of these HTTPRoutes;
`route.rateLimitedBy` is "an assertion by a human, checked by nobody"; resource
requests and limits are placeholders with no profiling behind them; the images
have been pushed and signed but **nobody has ever pulled one**.
