# The outbox, the ladder, and the deliverer

_Verified against vpay `93c6dfd0` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

All of it in `vpay_worker::webhooks`, run by the job loop in `vpay-server`'s
worker mode. `docs/reference/vpay-worker.md` §"The outbox drain" is the long
form; `docs/runbooks/webhook-delivery-failures.md` is what an operator does.

## Two steps, not one

**Step 1 — `handle_fan_out`.** Claims a page of the `events` backlog and, in
**one transaction per event**, writes a `webhook_deliveries` row per configured
endpoint and enqueues a `deliver_webhook` job. It is a singleton job that
reschedules itself, keyed `vpay_worker::jobs::FANOUT_DEDUPE_KEY`
(`"fanout:events"`).

- `FAN_OUT_PAGE` = 100 events per pass; a pass that comes back full reschedules
  immediately, so a backlog drains over several passes rather than in one
  enormous read.
- `FAN_OUT_IDLE` = 5 s when the backlog is empty. That is the whole latency
  budget between a payment settling and its webhook being _enqueued_.
- A **poll**, not `LISTEN`/`NOTIFY`, deliberately: the drain must also pick up
  events written by a process that has since died, which a notification would
  not deliver.
- `FANOUT_MAX_ATTEMPTS` = **5**. After five failed passes the event is
  `fanout_state = 'failed'`, and **nothing retries it** — that is a webhook the
  merchant will never receive. Five is large enough that ~25 seconds of
  Postgres being unavailable abandons nothing, small enough that a poisoned
  page clears in under a minute. Re-arming a `failed` event is a runbook, not
  an automatic behaviour.

**Step 2 — `handle_deliver`.** Renders one delivery through
`vpay_api::model::EventObject`, signs the exact bytes it is about to send, and
POSTs them.

**Step 3 — `handle_scan_deliveries`.** The backstop behind both: every 10
minutes it re-enqueues up to 500 outstanding deliveries. A healthy deployment's
pass finds nothing.

## Two retry ladders, and they do not share

|             | `poll_delay`                                                               | `delivery_delay`                                              |
| ----------- | -------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Asks        | a **rail**, what happened to money                                         | a **merchant**, telling them what already happened            |
| Rungs       | 10s, 20s, 30s, 45s, 60s, 90s, then 120s to the 30-minute mark, then 15 min | 10s, 30s, 2m, 10m, 1h, 6h, 24h                                |
| Runs out    | **never** — always another rung                                            | **yes** — `None` after seven, and the delivery is `exhausted` |
| Return type | `Duration`                                                                 | `Option<Duration>`                                            |

Both are in `vpay_worker` (crate root). The `Option` is the point: "the ladder
ran out" is the `exhausted` transition of a `webhook_deliveries` row and must
not be expressible as another delay. A live charge, by contrast, is always owed
another question — a charge nobody asks about is one whose late success is
lost. `UNRESOLVED_POLL_INTERVAL` (1 hour) is the escalated rung once the
24-hour ladder is spent; it changes the interval and adds an alert, it never
stops the question being asked.

The delivery index is the **pre-increment** `attempt`, which counts failures so
far: after the first failure the wait is `delivery_delay(0)`.

**Delivery never consults `Classify`.** A merchant's `500` is not a
`ProviderError` and has no place in the rail failure vocabulary. Do not route a
delivery failure through `JobError::decision`.

## Budgets and caps

| Constant                   | Value     | Why                                                                      |
| -------------------------- | --------- | ------------------------------------------------------------------------ |
| `WEBHOOK_CONNECT_TIMEOUT`  | 5 s       | shorter than a rail's; an unreachable receiver is retried in seconds     |
| `WEBHOOK_REQUEST_TIMEOUT`  | 10 s      | a handler slower than this has acknowledged nothing a sender can rely on |
| `MAX_ACK_BODY_BYTES`       | 8 KiB     | nothing parses the ack; the cap stops an unbounded response              |
| excerpt stored             | 512 chars | enough of an error page to recognise it in a runbook                     |
| `SCAN_DELIVERIES_INTERVAL` | 10 min    | backstop                                                                 |
| `SCAN_DELIVERIES_BATCH`    | 500       | backstop                                                                 |

The timeouts live **beside the handler that spends them**, not in the binary.
They used to live in `vpay-worker-bin` and be written out again by two test
helpers — three copies of a number nothing pinned together, so changing the
binary's pair would have left the whole suite exercising a client that no
longer shipped, and staying green while it said so. There is exactly one reader
now: `vpay_worker::ssrf::pinned_client`.

This is why merchants are told to **acknowledge first and work afterwards**.

## SSRF vetting is per delivery, and the client is pinned

`vpay_worker::ssrf::vet` resolves and classifies the target before the request
(`AddressClass`, `EgressPolicy`, `EgressRefusal`, `VettedTarget`), and
`pinned_client` builds a client for _that_ delivery pinned to the vetted
address. A shared client cannot be pinned, which is why there is no shared
webhook client anywhere in the process.

There is a boot-time half too: `vpay_config::validate_webhook_url` refuses a
non-https URL under livemode and refuses stub markers. Both halves are required — boot
validation cannot see a hostname that starts resolving to a private address
later.

## Headers on every delivery

| Header             | Value                                     |
| ------------------ | ----------------------------------------- |
| `Vpay-Signature`   | `t=…,v1=…[,v1=…]`                         |
| `Stripe-Signature` | the same value, so `constructEvent` works |
| `Vpay-Event-Id`    | the event's `evt_…`, **not signed**       |
| `Content-Type`     | `application/json`                        |

`Vpay-Event-Id` is a convenience for a merchant deduping in an access log or a
proxy. **Only the body is signed**, so a value in that header is not evidence
of anything and a receiver must still read `event.id` out of the verified body.

`vpay_worker::signing::signature_header` is the only producer in the workspace,
and `vpay_worker::webhooks::event_bytes` produces the bytes that are both
signed and sent. Keep it that way: the scheme's whole guarantee is that the
bytes signed are the bytes on the wire, and a second serialisation between the
two would break every verifier for a reason nobody could see.

An endpoint with **no secrets is not sent to** — `handle_deliver` records a
failed attempt instead, because a receiver refusing a bare `t=` is a much worse
diagnostic than a log line naming the endpoint.

## The verifiers are the specification

`sdks/rust/src/webhooks.rs` and `sdks/nodejs/src/webhooks.ts` are the two
verifiers this sender is held to, and `vpay-worker`'s own doctests verify
against the real `vpay_sdk::webhooks::verify_at`. If you change the header
grammar, you are changing four things: the signer, both verifiers, and
`docs/flows/webhooks.md` — plus the parity row, because
`cargo xtask verify-sdk-parity` reads `docs/sdks/parity.md`.

`DEFAULT_TOLERANCE` in the SDKs is the 5-minute timestamp window.

## What an operator can and cannot recover

| Situation                                  | Recovery                                                               |
| ------------------------------------------ | ---------------------------------------------------------------------- |
| delivery failed, ladder not spent          | automatic — next rung                                                  |
| delivery `exhausted` (7 attempts)          | operator action against `webhook_deliveries`; the merchant is not told |
| event `fanout_state = 'failed'` (5 passes) | operator re-arms it; **nothing** retries automatically                 |
| merchant simply missed one                 | they poll `GET /v1/events` — same renderer, same bytes                 |

`webhook_deliveries` (migration `0022`, `vpay_db::webhook_deliveries`) is a
**state row read by humans**, not a log. That is why the response excerpt is
512 characters of something recognisable rather than a full body.

## Status, as of 2026-09-16

- Signing, both ladders, the outbox, `webhook_deliveries`, the SSRF guard and
  the SDK verifiers: real, tested, with the Node SDK's signature parity checked
  under `VPAY_REQUIRE_NODE=1`.
- **No merchant endpoint outside this repository has ever been POSTed to.**
  Every delivery in this repository's history went to a test receiver
  (`backends/tests/webhook-receiver`, `examples/webhook-receiver`).
- `docs/status/backend.md` marks the outbox drain and the delivery ladder 🟡,
  and those markers are there because a test says so.
