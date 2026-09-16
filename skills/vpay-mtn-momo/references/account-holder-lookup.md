# `account_holder_name` — real code, never called against the real rail

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

`ProviderAdapter::account_holder_name(msisdn, config) ->
Result<Option<AccountHolder>, ProviderError>`. MTN implements it, under the
**Collections** subscription key and token scope `submit` already holds;
Orange declares `supports_account_holder_lookup: false` and inherits the
port's `Unsupported` (see the `vpay-orange-money` skill — that is still true
of the lookup, and is no longer true of Orange's `refund`). Issue #47. Policy:
`docs/flows/account-holder-lookup.md`.

**As of 2026-09-16 this endpoint has never been called against MTN's real
sandbox.** The 2026-09-15 live run exercised the token mint, `requesttopay`
and the status query, and nothing else. Everything below the first section is
therefore proven against WireMock only.

## Since 2026-09-16 there are TWO callers, and neither may reach this method directly

`GET /v1/account_holders` was the only caller until 2026-09-16. Now
`POST /v1/refunds` asks the same question about a nominated payee before it
instructs a transfer (`vpay_api::v1::refunds::verify_registered_holder`), on
the **capability** and never on a rail code — so it runs on `mtn_momo` and is
skipped entirely on `orange_money`, where RFC-0003 § 1 accepts the destination
unverified.

**Both go through `vpay_api::v1::account_holders::ask_rail`, not through the
adapter.** That is a fix from the review of 2026-09-16, and the two properties
it restored are the reason to keep it that way:

- `vpay_account_holder_lookups_total` is **every** lookup vpay makes again.
  The flow doc asks an operator to alert on a sustained `not_found` rate near
  1.0 — which is what the mis-cased path segment below looks like from outside
  — and a caller missing from the series makes that alarm read a fraction of
  the traffic;
- a refund refused for an unregistered payee is refused **before any row, any
  attempt and any rail instruction**, so without `ask_rail`'s line it left _no
  trace at all_: a third party looked up by phone number with nothing written
  anywhere in vpay.

Two consequences follow for this method, and they are the sharp ones:

1. **A `payments:write` credential can now probe registration and read the
   answer off a `400`.** The reserved rate-limit decision in issue #47 § 3 is
   therefore about _lookups_, not about a route — a limit that covered only
   `GET /v1/account_holders` would not be the control it reads as.
2. **The refund caller never compares a name.** It branches on
   `Some`/`None` only: `Some(_)` means the number is a registered account and
   the transfer may be attempted, `None` is a `400` naming `destination`, and
   an `Err` keeps its own `502`/`500`. vpay holds no verified buyer name to
   compare against, so matching the holder's name is the _merchant's_ job —
   which is what the route exists for. The name is not stored, not logged and
   not compared.

## The three answers, and why the middle one is the sharp one

| answer             | means                                           |
| ------------------ | ----------------------------------------------- |
| `Ok(Some(holder))` | the rail knows this number and named its holder |
| `Ok(None)`         | **the rail has no record, and nothing else**    |
| `Err(..)`          | everything else                                 |

`Ok(None)` is **not** "we could not ask", not "the rail was down" and not
"we have no credential". Every one of those is an `Err`, classified through
ADR-0011 so the boundary answers 502/500 rather than a 200 a caller would
read as "no such holder".

The distinction is the whole point of the method, and since 2026-09-16 it is
load-bearing in vpay's own money path and not only in an integrator's.
`verify_registered_holder` must be able to tell a number that is not
registered from a lookup that never happened — the first is the merchant's to
fix and answers `400`, the second is ours and keeps its `502`/`500`.
Collapsing them would tell an integrator that a real person's real account
does not exist, and would refuse a refund over an outage.

## The two unverified assumptions, and why they compound

Both are written into the code with the uncertainty stated, rather than
smoothed over.

### 1. The case of the path segment

```rust
const ACCOUNT_HOLDER_ID_TYPE: &str = "msisdn";
```

**Lower-case, and MTN's own portal declares the enum upper-case.** The APIM
operation `GetBasicUserinfo` lists the parameter's values as
`MSISDN | Email | Alias | ID`, while every published example of the endpoint
— and issue #47's own citation — spells the segment `msisdn`. Both cannot be
right about a case-sensitive backend.

A single constant is what makes changing the answer one edit. The
`capabilities()` comment deliberately does **not** write the spelling out
again: it said `MSISDN` until 2026-09-06, a second and silently contradicting
spelling of the one thing the constant exists to keep in a single place.

### 2. `404 → Ok(None)`

**MTN documents no 404 for this operation.** The portal lists 200, 401 and
500 and nothing else — unlike `RequesttoPayTransactionStatus`, which
documents "404 Resource not found" explicitly.

It is mapped anyway, as a stated assumption: a 404 is the only status a REST
resource has for "no such thing", vpay must not turn one into a 502 the
merchant reads as an outage, and the assumption is safe in the direction that
matters. If MTN never sends a 404, the arm is dead code. If it sends one for
some _other_ reason, the caller's fail-closed rule (`Ok(None)` refuses the
nomination) still holds.

### Why together they are worse than either alone

> Either assumption alone fails loudly; **the two together fail quietly.**

If the segment's case is wrong, MTN's gateway answers 404 to _every_ lookup
and assumption 2 renders every one as "the rail has no record" — a total
misconfiguration that looks exactly like an empty subscriber base, with no
error and nothing in the metrics but a `not_found` rate of 1.0.

Reversing either costs one constant or one match arm.
`docs/flows/account-holder-lookup.md` carries what to check on the first real
sandbox call. **Check both on that call, and date what you find** — and note
that since 2026-09-16 the cost of being wrong includes refusing every MTN
refund whose payee is perfectly real.

## `account_holder_outcome` — the full table

| status              | outcome                                                                                                                                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `200` with a name   | `Ok(Some(AccountHolder::new(name)))`                                                                                                                                                                                      |
| `200` naming nobody | `Malformed` — **not** `Ok(None)`                                                                                                                                                                                          |
| `404`               | `Ok(None)` — assumption 2                                                                                                                                                                                                 |
| `401` / `403`       | `Rejected{ProviderAccountBlocked}`                                                                                                                                                                                        |
| `400`               | `Malformed` — the MSISDN we sent is not one MTN can parse. Not `Config` (nothing in the deployment's configuration produced it; the number came from the caller past `/v1`'s E.164 check) and emphatically not `Ok(None)` |
| `500`               | the same `CONFIGURATION_CODES` table `submit` uses                                                                                                                                                                        |
| other `5xx`         | `Transport`                                                                                                                                                                                                               |
| `3xx`               | `Malformed` — redirects are never followed                                                                                                                                                                                |
| anything else       | `Malformed`                                                                                                                                                                                                               |

A 200 with neither `given_name` nor `family_name` is `Malformed` because
"the rail answered something we cannot act on" and "the rail has no record"
are different facts, and only the second may be `Ok(None)`.

Conversely, **one half is a name.** MTN's schema documents both fields as
plain strings with no `required` list, and its own note on `family_name` says
some cultures have none. `{"given_name":"David"}` is `Some("David")`;
`{"given_name":"David","family_name":"   "}` is `Some("David")` (joined with
one space, then trimmed); `{}` is `None` at the wire-type level, which the
caller turns into `Malformed`.

## The escaping, and the mutation that made it a function

```rust
fn account_holder_url(base: &str, msisdn: &str) -> String {
    format!("{base}/collection/v1_0/accountholder/{ACCOUNT_HOLDER_ID_TYPE}/{}/basicuserinfo",
        vpay_provider::http::path_segment(msisdn))
}
```

The `path_segment` call is the only thing standing between a caller and an
arbitrary endpoint on MTN's API **under this deployment's own subscription
key and bearer token**: an unescaped `/` moves the request, a `?` or `#`
truncates the path.

It was inlined until 2026-09-06, and **a mutation that deleted the call left
113 tests green**. The test that was supposed to hold it exercised
`vpay_provider::http::path_segment` itself and never the adapter's _use_ of
it, and every stubbed MSISDN is digits-only, where escaping is a no-op.
Extracting a pure function is what lets
`the_lookup_url_escapes_the_payer_reference_it_interpolates` assert on the
string the adapter would actually put on the wire.

`ACCOUNT_HOLDER_ID_TYPE` is interpolated verbatim and **not** escaped: it is
a constant in the file, not caller data.

## Privacy: the projection happens at the port, not downstream

`AccountHolder` holds **a name, and nothing else** — one private `String`,
readable only through `AccountHolder::name()`, with a `Debug` that renders
`"[redacted]"`. There is deliberately no constructor taking a whole rail
response, so the projection happens in the adapter that knows the rail's
shape and the discarded fields never cross the port boundary.

MTN's `basicuserinfo` answers an OIDC-shaped body — `given_name`,
`family_name`, `birthdate`, `locale`, `gender`, `status`. Every field but the
two names is personal data vpay has no use for. An adapter that deserialised
the whole body into a type the port exposed would put a third party's date of
birth one `{:?}` away from a log line, and no amount of care at the call
sites would take it back out.

**Nothing in this method logs the number or the name.** The `debug!` carries
the HTTP status and the rail's code only; the masked MSISDN that reaches an
operator's log is written once by `ask_rail`, which is where the counter and
the mask live for **both** callers — `info` with `found = true/false`, `warn`
with the rail's code when the lookup failed, plus a `caller` field
(`v1_account_holders` or `v1_refunds_create`) that is deliberately a `tracing`
field and **never a metric label**, because the counter is pinned at one
label. The obligation the type
cannot enforce, stated in `AccountHolder::name`'s own `# Errors` section: the
value is a third party's name and **must not be logged, stored, or counted as
a metric label**.

The conformance case
`an_account_holder_body_of_personal_data_yields_a_name_and_leaks_nothing`
installs a scoped `tracing` subscriber over the port call and greps the
captured output for eight pieces of personal data, the holder's name **and
the two halves the rail sends it in**. Asserting on the joined name alone was
a hole found by mutation — see the `vpay-mtn-momo` SKILL.md.

## The capability flag

`supports_account_holder_lookup: true` here is a claim about the **rail** and
about **this code**: MTN Collections exposes the endpoint under the
subscription key and token scope `submit` already holds, and this method
calls it. If it were declared `true` with no implementation, the method would
have to be overridden with a `NotImplemented` token so `verify-status` could
see the gap. That flag is deliberately **not persisted** in the `providers`
table — see the `vpay-provider-adapters` skill.
