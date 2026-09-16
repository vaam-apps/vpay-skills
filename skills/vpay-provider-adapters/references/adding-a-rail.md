# Adding a rail

_Verified against vpay `f063ee96` (2026-09-15). Version-sensitive claims
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
  `RailUnderTest` variant plus rows in the five table functions:
  `mappings_dir`, `start` (adapter + `ProviderConfig` + the credentials the
  stub answers 401 to), `documented_declines`, `declared_failure_codes`,
  `documented_callback_body`, `callback_url_pattern`, `return_url_pattern`.
  **Nothing else in that file may learn the rail's name.**
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
fails the build. The gate fails in **both** directions: a token with no
bullet, and a bullet naming a token no shipping code carries any more.

## The test of whether you did it right

> **Nothing in the core changes. If step 9 is "and also patch the
> reconciler", the port leaked.**
