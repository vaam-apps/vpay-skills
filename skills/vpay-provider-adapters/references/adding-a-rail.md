# Adding a rail

_Verified against vpay `b747e5d5` (2026-09-23). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

The canonical checklist is `docs/flows/provider-port.md` § "Adding a rail".
**Two of its steps are wrong and are corrected in place there** — the
corrections are dated 2026-09-06 and are reproduced below, because the wrong
version is what a reader of an older checkout has.

## 0. Answer the preconditions first — before writing code

`docs/flows/provider-port.md` § "Preconditions, per flow shape". These are
commercial questions, not engineering ones:

**A push rail must satisfy both:**

- you can supply your own idempotent reference on submit;
- you can query final status by that reference, **indefinitely**.

Both are load-bearing because the payer's phone starts buzzing before you
learn whether your request succeeded.

**A redirect rail must satisfy:**

- the submit response is persistable before the payer can act;
- status is queryable by material you hold after that persist.

> Ask these **during commercial negotiation**, not after signing. If either
> fails for a push rail, **stop and renegotiate** before writing code.

## 1. A `providers[]` entry in the deployment's YAML — not an INSERT

**CORRECTION (2026-09-06).** `docs/flows/provider-port.md` step 2 said
`INSERT INTO providers`. In a running deployment nobody writes that table by
hand: `providers` is reconciled from `config.yaml` at **boot step 4** by
`vpay_db::ConfigReconcile::reconcile`, which is the **only writer**. So the
real step is a `providers[]` entry in the deployment's YAML plus the adapter
of step 3.

The sentence still holds where it matters: **adding a rail needs no schema
migration.** Providers are rows in a table, never enum variants (ADR-0002).

A hand-written `INSERT` at a psql prompt is still possible, but since
migration `0033` it must name **all eight** columns — the five capability
booleans have no column default, so an omitted one is a `23502` rather than
an invented capability. Pinned by
`a_hand_written_provider_insert_must_now_name_every_capability_column` in
`backends/tests/integration/tests/postgres_smoke.rs`.

## 2. Hosts are configuration — there is no `provider_hosts` table

**CORRECTION (2026-09-06).** `docs/flows/provider-port.md` step 3 said
`INSERT INTO provider_hosts`. **There is no such table and there never has
been** — no migration under `backends/migrations` creates one. The only
other mention in the tree is a `vpay-testkit` doc comment that inherited the
error from that page.

A rail's hosts are `providers[].host.{url,label}` in the deployment's YAML
(`vpay_config::ProviderHost`), one entry per deployment. That is what makes a
sandbox, a production rail and a WireMock stub **three profiles rather than
three rows** (ADR-0003: a profile selects a FILE, never a code path — same
binary, same image, different YAML). See `docs/flows/configuration.md`.

`ProviderHost::effective_callback_url` derives
`{public_base_url}/provider/{code}/callback` unless the YAML overrides it.
The adapter must send that value **verbatim**; the conformance suite asserts
equality, not presence, because an adapter sending its `base_url`, an empty
string or a hard-coded constant would satisfy a presence check and be exactly
as broken.

## 3. `backends/crates/vpay-adapter-<rail>/`

Mirror the two existing crates' module split — it is not cosmetic:

| module       | holds                                                  | rule                                                                                                                                                 |
| ------------ | ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lib.rs`     | the `Adapter` struct, the trait impl, the transport    | the outcome of each response is a **pure function** of `(status, body)`, so it is provable without a network                                         |
| `wire.rs`    | only the JSON shapes the rail exchanges                | **never** `#[serde(rename_all = "snake_case")]` — that attribute is the workspace convention for _vpay's own_ wire; these types model the **rail's** |
| `mapping.rs` | the rail's error vocabulary → `vpay_core::FailureCode` | a table a reviewer can diff against the rail's documentation                                                                                         |
| `token.rs`   | the rail's half of the token cache                     | the endpoint, grant, body shape and refresh margin are the rail's alone                                                                              |

Required in `Cargo.toml`: add the crate to the workspace `members` list in
the root `Cargo.toml` and to `[workspace.dependencies]`.

The pure-function split matters because it is what the crate's own unit tests
assert row by row, and the conformance suite then proves the same rows arrive
over a real socket.

## 3b. `payer_fields` — what the payer types, declared once

Added 2026-09-16 (vpay#186). This page did not have the step until
2026-09-23, and `docs/flows/provider-port.md` still does not. If the payer
types something vpay forwards to the rail, as a push rail's MSISDN is,
override `ProviderAdapter::payer_fields` with a `const &[PayerField]`: name,
kind, `required`, `label_key`. `vpay_adapter_mtn_momo`'s `PAYER_FIELDS` is
the one example. A rail that collects nothing, as a redirect rail does, keeps
the default `&[]`. Do not override it to return `&[]` explicitly.

What reads it, so you know what a wrong declaration breaks:

- `vpay_api::v1::payer_fields` validates every `confirm`'s
  `payment_method_data[<code>][…]` against it. An undeclared key is a `400`
  naming the key. A `Phone` field is parsed and canonicalised for its `region`
  before `submit` runs. **Steering numbers in your integration tests and demo
  mappings must therefore be valid numbers in that region**, or vpay refuses
  them before your rail sees them. `requesttopay.json`'s move from
  `237600000400` to `237670000400` is the precedent.
- `vpay_api::browser::checkout_sessions::build_rails` puts it in the payer
  surface's `RailSpec.fields`, beside `flow` and the deployment's optional
  `providers[].display_name`.
- The value is a fact about the rail's product, never read from
  `ProviderConfig`. `PayerFieldKind::Phone`'s `region` doc says why.

## 4. A mapping table into the failure taxonomy

`docs/flows/failures.md` is the closed taxonomy (eleven `FailureCode`
values). Your `mapping.rs` needs:

- the reason→code table itself, case-insensitive, with an unmapped string
  falling back to `FailureCode::ProviderError` **carrying the raw string**
  (an unmapped reason is still a decline; `provider_error` is an alert, not a
  resting place);
- **`pub const PRODUCED_FAILURE_CODES: [FailureCode; N]`** — every code this
  rail can emit. The conformance suite reads this from the adapter rather
  than keeping its own copy, and asserts the declared list and the stubbed
  decline rows agree. It cannot be derived from the table alone: codes that
  arrive from HTTP 401/403 or from the fallback are not table rows.
- if the rail publishes an error enum, snapshot it as a `#[cfg(test)]` const
  and assert every published code is **either mapped or deliberately
  unmapped**. That is the shape issue #59 established for
  `vpay-adapter-mtn-momo`; it is what caught `payer_declined` being defined
  by the core, promised in buyer copy, and produced by nothing.

## 4b. A `Required` rail: `parse_destination`, and the rules around the number

New on 2026-09-15 (RFC-0003 § 1 and open question 4). If your rail's refund
is an outbound **transfer**, declare
`Capabilities::refund_destination = RefundDestination::Required` and
**override `parse_destination`** — a `Required` rail that takes the port's
default is a bug in that adapter, and
`a_required_rail_parses_its_own_destination` in the conformance suite refuses
it on the adapter's own declaration rather than on a table in the test. A rail
that returns money to the instrument that paid declares `Origin` and writes
nothing; the default `Unsupported` is the permanent, correct answer there.

The merchant sends `destination[<rail_code>][…]`. The core strips its own
envelope — the outer key is a `code()`, core knowledge by definition, and
spelled the same way as the confirm path's `payment_method_data[<code>]` — so
what reaches you is the **rail-scoped inner map**, and a `destination` naming
no rail, or naming one as something that is not an object, has already been
refused by the core. Everything inside is yours; both shipping adapters read
the key `msisdn`.

Four rules, each with a test behind it:

- **You own the key; the port owns the number.** Trim the value and hand it to
  `RefundTarget::mobile_money`, which is **fallible and canonicalising** since
  the maintainer's decision of 2026-09-15 (it was infallible before, and both
  adapters applied the confirm path's `payer_instrument` rule — any
  non-whitespace string, passed to the rail as written). The field is private
  and that constructor is the only way in, so no adapter can build an invalid
  destination and none has to re-spell the rule. Out comes digits only, no
  `+` — `237600000200`, the `partyId` shape both adapters already send on the
  charge path.
- **The `+` is required, and the asymmetry is deliberate.**
  `GET /v1/account_holders` accepts a bare `600000200` because it knows it is
  in Cameroon; `vpay-provider` carries no country code and must not acquire
  one. Read internationally, `600000200` is country code `6` — Malaysia — so
  accepting it would not be lenient, it would be a different payee in a
  different country. A lookup that guesses wrong returns the wrong name; a
  transfer that guesses wrong sends the money and it does not come back.
- **A refusal names the rule, never the number.** Every `InvalidMsisdn`
  variant is a unit variant so nothing can carry the input; `RefundTarget`'s
  `Debug` prints `[redacted]`, including through an enclosing `Some(..)`; and
  there is deliberately no `Serialize`/`Deserialize` derive, because retention
  is RFC-0003 open question 2 and is **undecided** — a serde impl would put
  "persist it" and "put it in an event payload" one derive away.
  `an_invalid_msisdn_never_names_the_number_it_refused` holds the first of
  those shut. One `{input}` in a format string undoes all three.
- **Refuse, never silently succeed.** A missing key, a non-string value (a
  JSON number has already lost a leading `+` or `0`), an empty or
  whitespace-only string, and an empty sub-map are each
  `ProviderError::Malformed`. It is never an `Ok` with no payee: a refund that
  reached the rail with a silently dropped destination is money sent somewhere
  nobody nominated.

**And tell the handler author:** that `Malformed` must be **translated** by
its caller, not forwarded. It is the only honest variant available here
(RFC-0003 open question 5 left inventing one to the first adapter that makes a
real transfer), but its classification is written for a rail that answered
gibberish, and this is the one call site where no rail answered anything.
Forwarded unchanged it becomes a **502** carrying "The payment rail is
temporarily unavailable. The charge will be retried.", with
`Retry::AfterBackoff` and `Severity::Warn`, counted against the rail's error
budget — for a merchant's typo in their own parameters; and the parameter name
you carefully put in `context` reaches the operator's log and never the
integrator. `vpay_api::v1::refunds` maps it to `invalid_param("destination")`,
a `400`, exactly as the confirm path already answers for
`payment_method_data`. `a_malformed_destination_is_classified_as_a_rail_fault`
pins the classification so the note cannot quietly stop being true.

Finally, `refund`'s `destination` argument is `Some` **exactly when** you
declare `Required` — the core has already refused both broken combinations on
the capability value and never on a rail code, so do not re-check it. If it is
ever `None` anyway, answer `ProviderError::Config` (RFC-0003 open question 6,
settled 2026-09-15 by MTN's `refund`); the reasoning is in
[SKILL.md](../SKILL.md).

## 5. WireMock mappings under `backends/tests/conformance/wiremock/<rail>/mappings/`

One directory per rail; **the shared suite is reused unchanged**. The
contract each directory must satisfy is in
[conformance.md](conformance.md). That same directory is what
`compose.yml` bind-mounts, so a mapping fixed for CI is fixed for local
development.

## 6. Wire it into the binary and the suite

Not in `docs/flows/provider-port.md`, but required:

- `backends/apps/vpay-server/src/lib.rs` — add to **both** `adapters()` and
  `adapter_codes()`. They are two lists on purpose (a boot log line must not
  need an HTTP client), kept honest by
  `the_codes_match_the_adapters_that_are_linked`.
- `backends/tests/conformance/tests/adapter_conformance.rs` — a
  `RailUnderTest` variant plus rows in the **eight** table functions (it was
  seven until `refund_body_pattern` landed on 2026-09-15): `mappings_dir`,
  `start` (adapter + `ProviderConfig` + the credentials the stub answers 401
  to), `documented_declines`, `declared_failure_codes`,
  `documented_callback_body`, `callback_url_pattern`, `refund_body_pattern`
  (`None` for a rail with no transfer to assert on), `return_url_pattern`.
  **Nothing else in that file may learn the rail's name** — the two
  constructor sites, `adapters()` and
  `a_required_rail_parses_its_own_destination`, build every adapter rather
  than branching to one.
- `compose.yml` — a `wiremock-<rail>` service on the next free host port,
  bind-mounting the same mappings directory, with the
  `curl -fsS /__admin/health` healthcheck the other two use.
- The rail code must appear in the documented `payment_method_types` values
  (`docs/api/`, both SDKs).

## 7. Boot refuses the half-done states

`vpay_api::v1::boot::boot_seeds` runs **before the database is touched**, so
these fail in milliseconds rather than after a connection and a migration run:

| condition                                                                  | error                                 | exit   |
| -------------------------------------------------------------------------- | ------------------------------------- | ------ |
| a configured `providers[]` code the binary links no adapter for            | `ConfigError::ProviderWithoutAdapter` | **78** |
| an adapter declaring `supports_partial_refunds` without `supports_refunds` | `ConfigError::IncoherentCapabilities` | **78** |

`a_provider_code_with_no_linked_adapter_is_exit_78` in
`backends/apps/vpay-server/tests/cli.rs` needs no container precisely because
of that ordering — moving the call below `vpay_db::connect` would break it.

## 8. A flow doc recording its quirks

`docs/flows/adapter-<rail>.md`, on the model of the two that exist. It must
carry, explicitly:

- what is **proven, and by what** (unit tests, conformance, a real call);
- a **"To confirm"** list of everything still unverified against the real
  rail, with what each item blocks;
- the mapping table, so `mapping.rs` can be diffed against it by eye.

If any operation lands as `NotImplemented`, it also needs a bullet in
`docs/status.md` under the token heading, or `cargo xtask verify-status`
fails the build. The gate fails in **three** directions since 2026-09-15 (it
was two): a token with no bullet; a bullet naming a token no shipping code
carries any more; and — the new one — **a token whose prefix names a rail
this workspace ships an adapter for, carried by a different rail's crate.**
Spell your tokens `<rail>::<fn>` (AGENTS.md § 2) with _your_ rail's code:
the first two directions compare token strings and cannot tell which adapter
answered one, so a copy-paste between adapters was invisible to both.

## The test of whether you did it right

> **Nothing in the core changes. If step 9 is "and also patch the
> reconciler", the port leaked.**
