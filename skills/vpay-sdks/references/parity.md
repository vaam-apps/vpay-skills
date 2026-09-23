# `docs/sdks/parity.md` and `cargo xtask verify-sdk-parity`

_Verified against vpay `d3a8810b` (2026-09-16). Version-sensitive claims
carry the date they became true — see [VERSIONING.md](https://github.com/vaam-apps/vpay-skills/blob/main/VERSIONING.md)._

The record ADR-0015 requires: one row per capability, one column per SDK, and a
cell that is either a `✅` naming the proving test(s) **in that SDK** or a `⛔`
with a date, a reason and an owner.

The gate is `cargo xtask verify-sdk-parity` (`just verify-sdk-parity`),
implemented in `.xtask/src/main.rs` (`verify_sdk_parity`, `parity_outcome`,
`check_parity_cell`, `parity_tables`). It runs inside `just verify`, which runs
inside `just ci`.

Success line:
`verify-sdk-parity: ok — N proving test(s) … all exist, M dated gap(s), K SDK
method(s) enumerated across R row(s)`.

## How a table is recognised

A markdown table is a parity matrix **iff its first header cell is literally
`Capability`** (`PARITY_TABLE_MARKER`). That is a marker rather than "every
table in the file", because the document also carries a gap ledger and a
legend, and a check that tried to read those as matrices would either fail on
prose or force the prose out of the document.

The remaining header cells are the SDK roots the table compares, written as
**code spans holding repo-relative paths** — `` `sdks/rust` ``,
`` `sdks/nodejs` ``, `` `sdks/stripe-js` ``,
`` `sdks/flutter/vpay_checkout_flutter` ``. Each must be a real directory or
the gate fails naming the line.

## How a capability row is named — and why the leading span is load-bearing

A row is a **capability row** when its first cell **opens** with a code span
holding `<resource>.<method>`, in the SDKs' own spelling:

| Source                                                                                  | Read as                     |
| --------------------------------------------------------------------------------------- | --------------------------- |
| Rust `impl <Resource>Resource { pub async fn <method>(` in `sdks/rust/src/resources.rs` | `<resource_snake>.<method>` |
| Node exported class methods in `sdks/nodejs/src/resources/<resource>.ts`                | `<resource_snake>.<method>` |

Resources map by snake_case in both languages: `PaymentIntentsResource` and
`client.paymentIntents` are both `payment_intents`. The one nested resource
keeps the spelling a merchant reads — `checkout.sessions`, not
`checkout_sessions` (`PARITY_NESTED_RESOURCES`). Private helpers, constructors
and namespace accessors (`CheckoutResource::sessions`) are **not**
capabilities.

**Opening with the span is load-bearing, not tidiness.** Two consequences:

- Rows that describe a behaviour spanning several methods carry **no leading
  span** and are checked by the cell rules alone. Most of the document is that
  shape.
- Rows that _mention_ a dotted code span mid-sentence must not be read as
  naming a method. The `checkout.session.expired` event-type rows are the
  example: there is no such method and there must not be one.

So: if your row is about one method, lead with it. If it is not, do not put a
dotted span first.

## Cell rules

| Cell          | Rule                                                                                                             |
| ------------- | ---------------------------------------------------------------------------------------------------------------- |
| `✅ …`        | Names the test(s) that prove the capability **in that SDK**, each in a code span, and **nothing else**           |
| `⛔ …`        | A `YYYY-MM-DD` date, the reason, and who owns closing it                                                         |
| blank         | **Always fails.** A blank cell is the only answer that says nothing, and it is what an unfinished row looks like |
| anything else | Fails — the cell must begin `✅` or `⛔`                                                                         |

A named test must exist under that column's directory as a live:

- Rust `#[test]` / `#[tokio::test]` function, or
- TypeScript `it("…")` / `test("…")`, or
- Dart `test('…')` / `testWidgets('…')` / `group('…')` (since 2026-09-13).

**Not counted:** anything `#[ignore]`d, `it.skip`ped, carrying `skip: true` or
a string `skip:` reason, a test inside a skipped group, a commented-out
declaration, or a title that only appears quoted inside a string.

**An undated `⛔` fails.** ADR-0015 allows a capability to be missing from one
SDK; it does not allow the absence to be undated, "because an undated gap is
indistinguishable from one nobody has looked at since it was written".

Directory walks skip `node_modules`, `dist`, `target`, `.git`, `coverage`,
`.dart_tool` and `build` (`PARITY_SKIPPED_DIRS`) — so a vendored dependency's
test cannot satisfy a `✅` nobody wrote. Method scanning additionally skips
`tests`, `test`, `testing`, `examples` and `benches`.

## All three directions, and the hole each one closed

Every direction below exists because the gate was measured passing while
something was wrong. That history is the reason to trust the current shape and
the reason not to simplify it.

### 1. Doc → tests (original)

Every name in a `✅` cell must exist. This is the direction that stops a rename
from quietly retiring a proof.

### 2. Code → doc, and doc → code (2026-09-06)

- **code → doc.** Every `<resource>.<method>` either SDK declares must have a
  row. A method with no row fails, naming the `file:line` it is declared on.
- **doc → code.** Every `<resource>.<method>` row must name a method at least
  one SDK declares — **unless every one of its cells is a dated `⛔`**, which is
  how a capability written down before it exists is recorded. (`events.retrieve`
  has been ⛔/⛔ since 2026-09-03.) A stale row fails, naming its own line.

**The hole:** until this landed the gate only ever read the file, and could
therefore only check whether what the file _said_ was true. **Deleting a whole
capability row was measured to pass** — 350 proving tests dropped to 347 and
`just verify` stayed green — and an SDK method with no row at all was
invisible. ADR-0015's rule is a claim about the SDKs, and a check that starts
from the document can only ever verify the document's own footnotes.

### 3. Per column (2026-09-08)

A `✅` on a capability row may only appear in a column whose **own tree**
declares that method. Checked **before** the test names.

**The hole:** directions 2 are both satisfied by _either_ SDK declaring the
method. Measured on the exp33 head — deleting `invoices.void` from `sdks/rust`
alone, leaving `sdks/nodejs` untouched — **`verify-sdk-parity` exited 0** and
still reported 443 proving tests. doc→code was answered by the Node
declaration, and the Rust cell's named test is **source text that goes on
existing whether or not the method it calls does**. What caught it in practice
was `cargo nextest -p vpay-sdk`, because the test no longer compiled; the gate
is what is supposed to say so first.

The failure message says it plainly: _a `✅` is a claim about THIS SDK, and the
other column declaring it is what the row's two cells exist to tell apart. Ship
the method here, or make this cell a dated `⛔`._

Rows that name no `<resource>.<method>` — most of the document — are untouched
by this rule.

## What to do when you add or change a method

1. **Add it to both merchant SDKs**, or decide not to.
2. Write the proving test(s) in each SDK that ships it. They must be real
   tests, not `skip`ped.
3. Add or edit the row in `docs/sdks/parity.md`. Lead the first cell with the
   `` `<resource>.<method>` `` span if the row is about that one method.
4. For any SDK that does **not** ship it, write
   `⛔ YYYY-MM-DD — <reason>. Owner: <who>`.
5. Add a line to the gap ledger at the bottom of the file if it is a standing
   gap.
6. Run `cargo xtask verify-sdk-parity`. Then `just verify`.

When you **rename** a test, edit the cell in the same commit. When you
**delete** a method, delete or re-cell the row in the same commit.

## When a capability is in neither SDK

> A capability absent from _both_ SDKs is still recorded ⛔/⛔ — the SDKs are at
> parity with each other and both short of the server. That is a different
> statement from "done", and the rule that keeps this table honest is that
> neither shape may be silently omitted.

## Deliberate non-parity worth knowing before you "fix" it

- **Neither SDK validates an MSISDN or a phone number locally**, identically
  and on purpose: a phone-number rule is a _market_ rule vpay owns and may
  widen, so an SDK copy would refuse offline a number a later server version
  accepts.
- **The Node accessor is `client.accountHolders` (camelCase) while its request
  field is `payment_method_type` (snake_case)**, like every other params type
  in that package and like the wire. The issue's original sketch said
  `paymentMethodType`; following it would have made that the only camelCase
  request field in the SDK. **Corrected 2026-09-23:** since vaam-apps/vpay
  step A (RFC-0004 §§ 5–6, merged <pending>) there **are** camelCase request
  fields — `invoices.pay`'s `outOfBand` object and its `receivedAt` — by the
  maintainer's decision, recorded in `docs/sdks/parity.md` under the
  `/v1` resource table. Rust spells them `out_of_band` / `received_at`; the
  wire bytes are identical. It is a decided exception, not a new rule: every
  other Node params field still follows the wire.
- **`del`, not `delete`** — in both SDKs, because `delete` is a reserved word
  in older JavaScript object literals and that is why Stripe's own SDKs spell
  it that way. Rust matches even though `delete` is legal in Rust: the _name_
  is what a merchant looks up when they read one SDK's docs and write against
  the other.
- **The event union grew by four and deliberately not by six.**
  `invoice.marked_uncollectible` and `invoice.payment_failed` are real Stripe
  types vpay does not write; an entry for either would be a claim. Both SDKs'
  proving tests assert the four are known **and** that those two are not.
- **`invoice_items` is the route and `InvoiceLine` is the type.** Stripe has
  two objects where vpay has one. Both spellings are the wire's.

## Step A: the rows it moved

Since vaam-apps/vpay step A (RFC-0004 §§ 5–6, merged <pending>). No SDK
method was added, so the method count did not move: `verify-sdk-parity`
printed **757 proving tests, 45 dated gaps, 35 methods across 40 rows** on the
branch (`docs/status/verification/2026-09-23-manual-payments.md`) — one new
row, the out-of-band `invoices.pay`, and `customer` added to the three
`*.list` rows.

- **`customer`.** Rust: `ListPaymentIntentsParams`,
  `ListCheckoutSessionsParams`, `ListRefundsParams` each gain
  `customer: Option<String>`. Node: `ListCheckoutSessionsParams` and
  `ListRefundsParams` gain `customer?`, and `paymentIntents.list` takes a new
  exported `ListPaymentIntentsParams` (`ListParams` stays, unchanged). Proven
  by exact-query-string cases in both; the Rust integration case for intents
  also drives it through `vpay-sdk` against a real server. No live-suite case.
- **Out of band.** Rust `PayInvoiceParams.out_of_band` or
  `PayInvoiceParams::out_of_band(…)`; Node
  `pay(id, { outOfBand: { method?, reference?, receivedAt? } })`, where
  `PayInvoiceParams` became a union whose hosted arm is the old shape. An
  empty object is Stripe's bare flag. A URL beside it is refused before any
  request (`Error::InvalidParams` / `TypeError`): the Node type makes it
  unrepresentable, the Rust struct cannot, so Rust relies on the runtime check.
  Both `Invoice` types decode `paid_out_of_band` and `out_of_band_payment`
  with a default, so an older server still decodes. Each live suite gained a
  case and ran green against a compose stack (`just sdk-live`, 2026-09-23:
  Rust 5 passed, Node 6).
- **Counts on the branch, 2026-09-23:** `sdks/rust` 183 passed, 0 skipped;
  `sdks/nodejs` 226 passed, 0 skipped.

## What the matrix does not claim

- Not that a `✅` row is bug-free. It claims a named test in that SDK fails when
  the capability breaks. Nothing more.
- Not that a capability works against a deployed vpay. Both merchant SDKs test
  against in-process stubs; `docs/status.md` is where "has this ever run" is
  answered.
- Not that the server offers every capability. ~~`refunds.create` and
  `balance.retrieve` are SDK methods with no route — they reach the nest's
  `404`.~~ **Corrected 2026-09-16:** `refunds.create` **is routed** since
  2026-09-16 (RFC-0003 § 2, vpay#178), along with update, list and cancel.
  `balance.retrieve` is the only one left with no route — a ledger read path
  exists (`vpay_db::Ledger::merchant_payable_balance`) and nothing mounts it.
  That remains a **server** gap tracked in `docs/status.md`, not a parity gap.
  A routed `refunds.create` still settles nothing: an accepted refund stays
  `pending` indefinitely — only a refusal moves it, to `failed` — and no rail
  has ever returned money.

## The gap ledger

The bottom of `docs/sdks/parity.md` is a list of standing gaps, with closed
ones **struck through and dated** rather than deleted. Open entries as of
2026-09-16 include: no CI-gated real-OP conformance for the Node assertion;
`token_type` not validated on the Rust token response; `invalidate()` with no
compare-and-swap; no retry policy beyond the single 401; `request-id` not
surfaced; `stripe-should-retry` not read; the browser package never run against
a live stack; the popup surface never driven by a real browser; and, for the
Flutter plugin, that none of its `just` recipes is in `just ci` and that iOS
and macOS are compiled by nobody.

Keep the strikethroughs. They are the only signal a reader has about which
sentences on the page have been checked recently.
