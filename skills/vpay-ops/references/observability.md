# Observability: two ports, thirteen names, and no scraper

_Verified against vpay `a33aac61` (2026-09-29) for the queue-age gauge note; the rest of this page was last read at `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

**Nothing has ever scraped any of this.** The series exist; the alerts on them
have never been evaluated. Every claim below is about what a scrape _would_
find.

## Two listeners, and why that way round

| Path           | Port                                           | Answers                                       | Probe role                     |
| -------------- | ---------------------------------------------- | --------------------------------------------- | ------------------------------ |
| `GET /livez`   | `--observability-bind`, default `0.0.0.0:9090` | a static `ok` — no state, no database         | **liveness**, both modes       |
| `GET /metrics` | same                                           | Prometheus text (`text/plain; version=0.0.4`) | —                              |
| `GET /healthz` | `--bind`, 8080                                 | `SELECT 1` against Postgres                   | **readiness**, serve mode only |

**Why liveness is the stateless one:**

> A liveness probe that fails on a database outage restarts every pod in the
> deployment, repeatedly, and **a restart cannot fix a database**. A readiness
> probe that fails on one correctly takes the pod out of service.

**Why a second port rather than two more routes on 8080:** `/metrics` "names
every rail this deployment talks to, every route pattern it serves and every
error code it has produced", and `--bind` is the port an Ingress fronts. The
chart's NetworkPolicy admits 9090 from the monitoring namespace only — **a
policy it can only express because the two are different ports**, where a
path-based exclusion would depend on an ingress controller's rule ordering
staying correct forever.

`--observability-bind` is on **both** modes. The worker had no HTTP listener of
any kind before it, which is why the chart renders a **headless worker Service
with `metrics` only** — it exists so the worker can be scraped.

## The thirteen metrics

Owned by `vpay_core::metrics`, which **describes them and installs no
recorder**; each binary installs exactly one recorder, beside its rustls
provider. All thirteen have a live seam; `docs/status.md` names the seam for
each. **Fourteen since 2026-10-22**: RFC-0013 added
`vpay_callback_endpoint_checks_total`. **Fifteen since 2026-10-23**: RFC-0014
added `vpay_provider_callbacks_total` (last row). The heading keeps
"thirteen", as vpay's `docs/flows/deployment.md` § 6a does, so links to it
keep working.

| Metric                                                                   | Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `vpay_build_info`                                                        | see the `git_sha` caveat below                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `vpay_http_requests_total`, `vpay_http_request_duration_seconds`         | labelled by matched route **pattern**, never a concrete path, and by one of nine methods or `other`. **Both label sets are closed, because an unauthenticated caller controls both the path and the method** — an open label set there is a cardinality bomb anyone can fire                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `vpay_provider_requests_total`, `vpay_provider_request_duration_seconds` | per **port call**, not per HTTP request                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `vpay_charge_transitions_total`                                          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| three `vpay_jobs_*`                                                      | including `vpay_jobs_oldest_claimable_age_seconds` — see below                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `vpay_error_events_total`, `vpay_alert_events_total`                     | **incremented in the same statements that write `alert = true` to the log, so the two cannot diverge**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `vpay_webhook_deliveries_total`                                          | emitted at the two points a delivery attempt's outcome becomes durable                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `vpay_account_holder_lookups_total`                                      | once per `GET /v1/account_holders`, on **every** path including refusals. Its `outcome` label is the only thing that tells `found` from `not_found`, which are both `200` — and **no label on it carries the number looked up or the name returned**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `vpay_callback_endpoint_checks_total`                                    | once per rail per `ensure_callbacks` pass (worker job or `vpay-server callbacks ensure`), labelled `provider` and `state`: `present`, `created`, `drifted`, `foreign`, `missing`, `locked`, `failed`. **A counter, not a gauge**: the singleton runs on whichever worker claims it, so a per-process gauge would keep each process's last answer. `present` and `created` are healthy, and so is an occasional `locked`. `missing` and `failed` mean callbacks may not be arriving, and the rail's charges settle by polling alone; `drifted` means some event types are not subscribed; `foreign` means they arrive through an endpoint vpay did not create                                                                                                                                                       |
| `vpay_provider_callbacks_total`                                          | once per rail callback that names a charge this deployment holds, by `vpay_api::provider_callback`, labelled `provider` and `verdict`: `verified`, `verified_previous` (matched only the previous signing secret: a rotation in flight; remove the previous secret when it stops increasing), `invalid`, `not_checked`, `no_secret`, `unavailable` (RFC-0014, 2026-10-23; `verified_previous` from its security review the same day). `invalid` and `unavailable` did not pull the poll forward. **`invalid` with no `verified` beside it is this deployment's own signing secret gone stale** (rotated in the provider's dashboard, not in the Secret); `invalid` beside `verified` is someone sending forgeries. A body that does not parse and an unknown reference are not counted. No alert rule reads it yet |

That last constraint generalises: if you add a metric on a path that touches
personal data, the label set is part of the privacy review
(**ADR-0020**, privacy-controls-and-evidence — that document was a second
`0018` until vpay#172 renumbered it on 2026-09-16; `0018` now means
cross-tenant admin reads and nothing else).

## Three things to know before trusting a dashboard

**`vpay_jobs_oldest_claimable_age_seconds` goes negative on a healthy idle
queue.** It is `now - min(run_at)` over unleased rows **including future ones**,
so a deployment whose only queued work is the hourly sweep reports around
`-3500`. Read it as "seconds until (negative) or since (positive) the next
queued work was due".

> A `> 300` alert is unaffected; **an `abs()` applied to make the graph tidy
> would hide the case it exists for.**

_Added 2026-09-29 (ADR-0026, 2026-09-23):_ the subtraction is done by Postgres
now. `Jobs::oldest_runnable_age()` (it was `oldest_runnable_run_at()`, which the
worker then subtracted from its own `now_utc()`) returns
`now() - min(run_at)` computed in SQL over the same unleased, unparked rows, so a
worker host whose clock disagrees with the database no longer skews the gauge.
The sign convention — negative while the next job is in the future — did not
change, and neither did the metric name or its `abs()` warning above.

**`vpay_build_info{git_sha}` is `unknown` unless the image was built with
`--build-arg VPAY_GIT_SHA=…`.** `release.yml` passes `github.sha`; **every local
build and every `compose*.yml` do not, and nothing ever shells out to `git`.**
So `unknown` in a local or compose stack is correct, not a bug.

**`VpayProviderErrorRateHigh` will fire on ordinary declines.** Its numerator is
`error_kind!=""` — every failed port call, which is what makes it able to fire
during a rail outage (`provider_unavailable`) at all — and that set includes
`charge_declined`, **a rail decision rather than a rail failure**. On mobile
money that is a large, normal share of traffic. Whether to exclude declines is a
maintainer decision to be made with the threshold itself, against measured
traffic. See `docs/runbooks/provider-error-rate.md`. Since 2026-10-23 the set
also includes `charge_needs_reconciliation` (vaam-apps/vpay#313). That one is
not a rate problem: each escalates one charge to `unresolved`, so
`VpayUnresolvedChargesRising` and `docs/runbooks/unresolved-charges.md` cover
it, and an `unresolved` charge is no longer always 24 hours old.

## Alert rules

`deploy/helm/vpay/templates/prometheusrule.yaml`. **Every threshold is proposed,
not derived** (step-6 decision (5)): the two runbooks that predate the chart
contained no numbers to transcribe ("crosses its threshold", "more than one
hour"), so the figures were invented against a system that has never taken a
real payment. Each rule carries `provisional: "true"` and the `PrometheusRule`
is off by default.

The `Alert` column in `docs/runbooks/README.md` maps each rule to the runbook
its `runbook_url` points at: `VpayUnresolvedChargesRising`,
`VpayProviderErrorRateHigh`, `VpayJobQueueBehind`, `VpayJobsDeadLettered`,
and since 2026-10-23 `VpayCallbackEndpointUnconfirmed`
(`docs/runbooks/callback-endpoints.md`).

**`VpayCallbackEndpointUnconfirmed`** reads `vpay_callback_endpoint_checks_total`.
It fires when a rail's `ensure_callbacks` passes ran across
`metrics.prometheusRule.callbackEndpointWindow` (default `3h`, three hourly
passes) and none counted `present`, `created` or `foreign`: every pass said
`missing`, `drifted`, `failed` or `locked`. Three things to know before you
change it:

- It is written with `unless`, not `== 0`. A rail never once confirmed has no
  confirming series at all, and `sum(...) == 0` over a series that does not
  exist returns nothing. The comparison form cannot fire for a `check`
  deployment that was never registered, or for a key refused from the first
  pass.
- It is a `warning`, in `check` mode too. Callbacks are hints, so a missing
  endpoint costs latency and never money. The metric has no `registration`
  label, and in `check` mode `missing` is what the operator asked to be told.
- It cannot fire before a worker's second pass (a counter's first sample shows
  no increase), and it cannot fire for a pass that never runs. The job rules
  cover that case.

**Every rule has unit tests (2026-10-23).** `just test-alert-rules` renders the
rules with `helm template`, runs `promtool check rules`, and runs
`promtool test rules` over `deploy/helm/vpay/ci/prometheusrule.test.yaml`. It
**refuses a rendered rule that lacks a firing case or a silent case**, and a test
of a rule the chart no longer renders. promtool compares annotations exactly,
so changing a rule's description, a threshold or a window fails until the test
file says the same. CI's `deploy (helm chart)` job runs it, with promtool
3.13.5 pinned by SHA-256. The series are synthetic: this proves the
expressions, and nothing about vpay's real series, which no Prometheus has
scraped.

`VpayPageableErrorEvents` has no runbook of its own: it fires on any error
ADR-0011 classifies `Severity::Page`, and **the classification _is_ the alert —
there is no threshold to tune.** That is the design paying off: if you want
something paged, give its leaf error `Severity::Page`; do not add a rule.

## Logging

`tracing_subscriber::fmt()`, **`.json()` by default**
(`--log-format json|text`), to **stdout**. `--log-filter` / `RUST_LOG` builds an
`EnvFilter`; an unparseable directive falls back to `info` rather than failing.

On a startup failure `main` prints the full error chain to **stderr** with
`eprintln!("{e:#}")`, "because `tracing` may not be initialised yet when
configuration fails". So a boot failure is on stderr and everything afterwards
is JSON on stdout.

**The error/log contract** (`ApiError::into_response`):

```text
status = err.category().http_status()
body   = { "error": { "type": …, "code": …, "message": err.public_message(), "param"?: … } }
log    = at err.severity(), alert=true when Page, with the full Display + source chain
```

**The full chain goes to the log; only `public_message()` goes to the
merchant.** A storage error's leaf text reaches the log and never the body; an
idempotency key is never echoed past an 8-character hint, in the log only.

## Request correlation

`tower_http`'s `MakeRequestUuid` plus `SetRequestIdLayer` /
`PropagateRequestIdLayer`. An inbound proxy-supplied id is vetted by
`discard_unusable_request_id` — a caller cannot inject an arbitrary
`request_id` into your logs. The id is emitted as a Stripe-compatible
`request-id` response header.

`request_id` is recorded **on the span**, not as a field on each event, so every
error logged while serving a request carries it
(`an_error_logged_while_serving_a_request_carries_the_request_id`).

## Traces are deliberately absent

Step-6 decision (6) chose `metrics` + `metrics-exporter-prometheus` over the
OpenTelemetry SDK and **deleted the unused `opentelemetry` pin rather than
keeping it "for later"**.

> A slow request can be seen in `vpay_http_request_duration_seconds` and
> **cannot be decomposed**; JSON logs carrying a request id on every event are
> the correlation mechanism until an OTLP decision is made.

Adding tracing is an ADR-level decision, not a dependency you add in passing.
