---
name: vpay-sdks
description: The five packages under sdks/ and the parity rule that binds the two merchant SDKs — add a capability to one and you add it to all, or you write a dated gap row. Covers what each SDK is and which are published, how a parity row is written and all three directions of cargo xtask verify-sdk-parity with the measured holes that motivated each, the Stripe SDK compatibility story, the Flutter payer plugin, and the generated code that exists (pigeon, ZenStack) versus the OpenAPI codegen that does not. Load before adding, renaming or removing any SDK method or test.
---

# vpay SDKs

> **Verified against vpay `93c6dfd0` (2026-09-16).** Version-sensitive claims below
> carry the date they became true — a feature in vpay's `master` may be absent
> from the tree you are editing. On an older or newer vpay, trust the
> repository over this page. See [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md).

`sdks/` holds five packages. Two are merchant SDKs, two are payer surfaces, and
one is evidence rather than an SDK.

## The rule, before anything else

> **Add a capability to one SDK and you add it to all of them — or you write a
> dated `⛔` row in `docs/sdks/parity.md` for the ones that do not have it.**

That is ADR-0015, and `cargo xtask verify-sdk-parity` enforces it on every
`just verify`, which is in `just ci`. It is not advisory and it is not a
reviewer's job.

Three things follow that catch people out:

- **Renaming a test breaks the build.** A `✅` cell names the test(s) that prove
  the capability; the gate looks each one up by exact name under that column's
  directory. Rename `a_cached_token_is_reused_across_calls` and `just verify`
  fails naming the cell.
- **A method with no row breaks the build**, naming the `file:line` it is
  declared on.
- **An `#[ignore]`d, `it.skip`ped or `skip: true` test does not count.** Nor
  does a commented-out declaration, nor a title that only appears inside a
  string.

Full mechanics, including how to write a row and what each direction of the
gate refuses: `references/parity.md`. Read it before editing
`docs/sdks/parity.md`.

## The packages

| Path                                 | Name                            | What it is                                                                                                                                                                                                                                   | Published                                                                                                                                            |
| ------------------------------------ | ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sdks/nodejs`                        | `@vaam-apps/vpay-sdk`           | **Merchant** SDK for `/v1`. `private_key_jwt` auth, the single-401 re-auth, the form encoder, `verifyWebhook`, and eight resources. Second entry point `./stripe` exports `createStripeAuthenticator`, with `stripe` as an **optional** peer | yes, public                                                                                                                                          |
| `sdks/rust`                          | `vpay-sdk`                      | The Rust twin. Same eight resources, same wire                                                                                                                                                                                               | **`publish = false`, deliberately** — "publishing a client for an API nobody can reach would be actively misleading". Flip it once `/v1` is deployed |
| `sdks/stripe-js`                     | `@vaam-apps/vpay-stripe-js`     | The browser **payer** surface, Stripe.js-shaped. `loadStripe`, `initEmbeddedCheckout`, `openCheckoutPopup`, `notifyCheckoutOpener`. **Zero runtime dependencies**                                                                            | yes, public                                                                                                                                          |
| `sdks/flutter/vpay_checkout_flutter` | `vpay_checkout_flutter`         | The mobile **payer** surface. Opens the hosted page in a native window and reports a typed result once the intent actually settles                                                                                                           | **`publish_to: none`** (2026-09-13)                                                                                                                  |
| `sdks/stripe-compat`                 | `@vaam-apps/vpay-stripe-compat` | **Evidence, not an SDK.** Drives the real `stripe@22.6.1` package against a live compose stack. `private: true`, ships nothing, and **gets no rows in the parity matrix** — "the compat suite proves claims rather than making them"         | never                                                                                                                                                |

The two payer surfaces are **not** a third and fourth merchant SDK. They
authenticate a payer's _device_ with a publishable key and a per-session
`client_secret`, speak `/v1/browser`, and share **no capability row** with the
merchant tables. Each has its own table in `docs/sdks/parity.md`.

The eight merchant resources, in both languages: `payment_intents`,
`checkout.sessions`, `customers`, `invoices`, `invoice_items`, `refunds`,
`events`, `account_holders`, `balance`.

## Parity is per capability, not per method name or per shape

ADR-0015 decision 1. Two deliberate asymmetries you will meet:

- **One spelling divergence exists and it is a table, not a rule.**
  `PARITY_COLUMN_SPELLINGS` in `.xtask/src/main.rs` carries exactly one entry:
  Node's `invoices.markUncollectible` is the row's `invoices.mark_uncollectible`.
  Rust cannot spell the first (`non_snake_case`) and the Node package should
  not spell the second (Stripe's own Node SDK says `markUncollectible`).
  **An entry in that table that exempts nothing fails the build**, the way
  ADR-0016's serde exemption table does. A _rule_ ("TypeScript may camel-case
  any row") was rejected: it would make every future divergence silent.
- **Types diverge where the languages do.** A patch field is three-state in
  both (`Option<Option<String>>` / `string | null | undefined`) because
  `description=` clears; an _invoice item_ patch field is two-state in both,
  because its columns are `NOT NULL` and `description=` is a `400`. `refund.fee`
  is `Option<i64>` in Rust and `fee?: number | null` in TypeScript. Each cell's
  proving test asserts the resulting **body**, never the type.

## The live suites fail rather than skip

`sdks/rust/tests/live_invoices.rs` (behind the `live-stack` cargo feature) and
`sdks/nodejs/src/invoices.live.test.ts` (its own vitest project), both run by
`just sdk-live` and CI's `e2e` job. `sdks/stripe-compat` works the same way:
`src/preflight.ts` is a vitest `globalSetup` that fails the run when no stack
answers `/healthz` or the merchant handshake does not complete.

Run with no `VPAY_BASE_URL` they **fail, naming the variable**. They never
skip, because `AGENTS.md` forbids `#[ignore]` for exactly this reason: a suite
that skipped itself would print `ok` with zero cases and be indistinguishable,
in a CI summary, from one that passed.

It earned its keep immediately: the first live run found that `currency` is
**required** on `POST /v1/invoices`, which both SDKs had documented as optional
with a deployment default that does not exist.

## There is no OpenAPI spec and no OpenAPI codegen

Both merchant SDKs are **hand-written** against the wire contract in
`docs/flows/merchant-auth.md` and the flow docs. `verify-sdk-parity` is what
keeps them in step — there is no generator to re-run and no spec to edit.
Generated code that _does_ exist (pigeon, ZenStack) and how to regenerate it is
in `references/generated-code.md`.

## More

- `references/parity.md` — how to write a row, all three directions of the
  gate, and the measured holes each direction closed. **Read this before
  editing the matrix.**
- `references/stripe-compat.md` — using the official Stripe SDKs against vpay:
  the one seam, and the divergences an integration actually hits.
- `references/flutter-plugin.md` — the payer plugin, its two e2e suites, and
  D8's external browser.
- `references/generated-code.md` — pigeon and ZenStack, and the hand-added line
  that regeneration will silently drop.
