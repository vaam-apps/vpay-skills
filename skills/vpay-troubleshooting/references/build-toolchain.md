# Build and toolchain failures

## Three version numbers that are not each other

Getting these confused is the most common toolchain mistake here.

| Where | Value (2026-09-16) | What it is |
| --- | --- | --- |
| `rust-toolchain.toml` `channel` | `1.98.0` | the compiler this workspace is built and tested with. CI reads the pin **from the file**, never `@stable` |
| `backends/Dockerfile` `FROM rust:` | `1.98.0-alpine3.22` | the one place that **cannot** read the file |
| `Cargo.toml` `rust-version` | `1.88` | `cargo metadata`'s max `rust_version` across the *resolved dependency graph* — "a theoretical floor nobody has actually compiled this workspace with" |

`rust-toolchain.toml` was `1.95.0` until 2026-09-05. `CLAUDE.md` said `1.95.0`
for a while after it moved, and a review finding exists whose entire content is
that sentence being stale.

### `verify-toolchain` fails

**Cause:** `backends/Dockerfile`'s `FROM rust:` version and
`rust-toolchain.toml`'s `channel` disagree. This is the tenth gate, added
2026-09-05 out of the 1.95.0 → 1.98.0 bump review, because a mismatch there
"was measured to pass every other gate".

**Fix:** bump both together. The gate checks **the compiler version only** —
the Alpine suffix is deliberately outside its subject.

**Do not bump the Alpine base as a rider.** `rust:1.98.0-alpine3.23` exists and
was not taken: "an Alpine major bump changes musl and gcc under a static build
and is its own decision with its own evidence."

Only **one** `FROM` names the rust image (the `chef` stage); `planner` and
`builder` are both `FROM chef`, "so the three stages cannot drift apart by
construction. **Do not turn that into three literals.**"

## `cannot produce proc-macro for 'async-trait' as the target 'x86_64-unknown-linux-musl' does not support these crate types`

**Cause:** `.cargo/config.toml` sets `-C target-feature=+crt-static` for
`x86_64-unknown-linux-musl`. When host and target are the same triple and **no
`--target` is given**, cargo applies target rustflags to *host* artifacts too —
build scripts and proc-macros — and a proc-macro cannot be a static executable.

**Fix:** always pass an explicit `--target`. `just build-dist` does;
`backends/Dockerfile` does. With one, cargo keeps host artifacts free of target
rustflags, and output lands under `target/<triple>/dist/`.

This is not hypothetical: the first CI build of that Dockerfile (run
`33646048616`, 2026-09-02) failed exactly this way.

## musl target missing

**Symptom:** `just build-dist` fails.

**Fix:** `rustup target add x86_64-unknown-linux-musl`.

**But note the Dockerfile does not use that triple.** Per ADR-0014 it passes
`--target` set to the **builder's own host triple**, read from `rustc -vV` at
build time, because `rust:*-alpine` is a multi-arch image whose toolchain is
already musl-native for whichever architecture Docker pulled — and hardcoding
`x86_64-unknown-linux-musl` on an arm64 host forces a cross-compile needing a
GNU-compatible cross-linker no `rust:*-alpine` ships. It "fails outright".

## `check-schema: FAIL — needs the 'cratestack' CLI on PATH, and it is not there.`

The recipe follows that with: "this is a failure, not a skip: nothing checked
`schemas/vpay.cstack` in this run."

**Fix:**

```bash
cargo install cratestack-cli --locked --version 0.12.0
```

Three sub-traps the recipe's own message covers:

- Installing **from inside the checkout** used to fail with an MSRV error while
  `rust-toolchain.toml` pinned 1.95.0 (`cratestack-cli` declares
  `rust-version = 1.98.0`). The instruction then was to `cd` out of the tree
  first. The pin is 1.98.0 now, so it works in place — **that bump is why.**
- There is **no prebuilt binary for linux MUSL.** Five triples only:
  x86_64/aarch64 linux-gnu and apple-darwin, x86_64-pc-windows-msvc.
- A CLI on `PATH` at a **different version** prints a WARNING and **still runs
  the check against the wrong grammar.** `docs/status.md` records exactly that:
  pinned `0.12.0`, CLI on the authoring machine `0.11.1`, check ran in full
  against the 0.11.1 grammar. "A gate that ran against a different grammar than
  CI will is a gate whose green means less than it looks."

A further hazard inside the gate: an **emptied or truncated** `.cstack`, or one
with the `datasource` block deleted, **type-checks vacuously and exits 0**.
`cratestack check` is right to accept both (a client-only schema is a real
thing) and there is no CLI flag that refuses it — so the recipe asserts a
declaration floor itself, next to the claim it protects.

## Editing `schemas/vpay.cstack` breaks `cargo build`

**`schemas/vpay.cstack` IS wired into the build.** `CLAUDE.md`'s bullet said
the opposite — "not wired into the build, its syntax is unverified, do not try
to make it compile" — and had been wrong **in two stages**: `just check-schema`
began verifying the syntax 2026-09-05, and `vpay-db`'s private `mod schema`
began *compiling* the file 2026-09-06. A syntax error in it is now a
`cargo build` failure.

Three consequences:

1. **The CLI and the library must stay on one version.** `justfile`'s
   `cratestack_version` and `Cargo.toml`'s `cratestack = "=0.12.0"` — bump them
   together.
2. **The generated module is private to `vpay-db` on purpose.**
   `cargo xtask verify-repositories` fails if `mod schema` is made `pub` or
   re-exported, "because the module the macro creates exists in no source file
   and nothing else would object."
3. **Adding a `model` is not free.** It must match the live table, and
   `postgres_smoke.rs`'s drift test pins the exact gap. See
   `docs/reference/vpay-db/cratestack.md`. (`vpay-db.md` § CrateStack is a
   pointer to it and keeps that heading on purpose — four doc comments in
   `backends/crates/vpay-db/src/` link to `vpay-db.md#cratestack`, and
   `verify-links` does not check anchors.)

## `cargo deny` — RUSTSEC-2023-0071

**Do not try to fix this.** It is in `deny.toml`'s `[advisories] ignore` list
with its reasoning in full, accepted deliberately by the maintainer on
2026-08-09.

- The "Marvin Attack": a timing side-channel in the `rsa` crate's PKCS#1 v1.5
  **decryption**.
- **There is no patched release.** The advisory has carried no fixed version
  since 2023, and `rsa` is an unconditional, non-optional dependency of
  `authkestra-engine`, which vpay uses to run its own OpenID Provider. It
  cannot be feature-gated away, and `authkestra-op` signs RS256 only.
- The exposure is on-topic, not incidental — it is the crate that signs every
  token vpay issues. What limits it is that the attack needs a *decryption*
  oracle and vpay's use is signing and verification.
- The entry genuinely fires: `cargo deny -L info check advisories` reports
  `note[advisory-ignored]` against `rsa v0.9.10`.

Revisit if a fixed `rsa` appears, if authkestra gains non-RSA signing, or if
the Keycloak/ZITADEL comparison ADR-0009 leaves open is carried out.

## Adding a dependency that needs OpenSSL

**It will fail `cargo deny`, and that is the intended friction.** `openssl`,
`openssl-sys` and `native-tls` are banned outright (ADR-0005) so a *transitive*
dependency that pulls one in fails CI rather than silently linking and
defeating the static-musl goal. Adopting one needs a new ADR.

## Slow first build

Expect it. `just test` needs `postgres:16-alpine` and `wiremock/wiremock`
pulled; the conformance suite starts **a real WireMock container per rail per
test**. `just demo` builds every image it needs in Docker. Docker Compose
**v2.24+** is required — the demo overlay uses `!reset`.
