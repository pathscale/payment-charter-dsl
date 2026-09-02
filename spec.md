# Payment Charter DSL — specification

Date 2026-09-02. **Normative.** This is the document a parser is written against.

Design rationale lives in [`design.md`](design.md); where the two disagree, this file
wins. Requirement levels are RFC 2119.

## 0 · Name

The formal name is **Payment Charter DSL**. After first mention, **Charter**.

A single document is **a charter**. A charter grants spending authority from a higher body to a
lower one and can never exceed the charter it extends (§8A).

The opening keyword is `charter`, matching the format and the file
extension. A reader never has to reconcile a `policy` keyword inside a `.charter` file with
a format called Charter.

Crates: `pays-charter` compiles, `pays-policy` evaluates. Files: `.charter`, `.charter.json`.

## 1 · Scope and forms

A charter has two interchangeable forms:

- **Text form** — what a human authors, reviews and diffs. UTF-8, extension `.charter`.
- **JSON form** — what the browser builds and the wire carries. Extension `.charter.json`.

They MUST be isomorphic: parsing text and serialising to JSON, or the reverse, MUST round-trip
without loss of anything semantic. Comments, whitespace and declaration order are not
semantic and MAY be lost.

An implementation MAY provide only one direction. The backend MUST accept both. A browser
client builds JSON and MUST be able to emit text (§1.1) so a controller can read and diff what
they are authorising; it need not parse text.

**Compilation** turns either form into the *compiled form* (§9), which is what the engine
evaluates. A document that does not compile is not a charter.

### 1.1 Canonical text form

Semantic isomorphism is not enough to test an emitter against. Two emitters can agree on every
meaning and still disagree on every character a human reads, and "semantically identical" is not
a comparison a test suite can make cheaply or a reviewer can trust.

There is therefore exactly one **canonical text form** for a given charter. An emitter MUST
produce it, and `roundtrip/` conformance is a **byte comparison**.

A parser MUST accept any conforming document, canonical or not; §2.1 is unchanged and layout
remains insignificant on input. Canonicality constrains *output* only.

**Encoding and layout.** UTF-8 with no BOM. `LF` line endings. No trailing whitespace on any
line. Exactly one `LF` at end of input. Indentation is spaces only.

**Comments are not emitted.** They are not semantic (§2.2) and do not survive the JSON form, so
a canonical document has none. An implementation that round-trips text→text MAY preserve them,
but the result is then not canonical and MUST NOT be compared byte-wise.

**Line structure.** No line is ever wrapped, regardless of length. A `mint://` or `unit://`
reference is never broken. Wrapping would need a column budget, and a column budget makes the
canonical form depend on a setting.

**Ordering.** The header comes first, in grammar order — `charter`, `extends` if present,
`resolver`, `timezone` — one per line, unindented, with no blank line between them. Then a blank
line. Then declarations, grouped by kind in this order:

1. `asset`
2. `group`
3. `approvers`
4. `limit`

Within each kind, declarations are sorted by identifier, ascending by byte value. Declaration
order is not semantic (§5), so sorting is what makes the output a function of the meaning rather
than of the author's typing. A blank line separates one kind from the next, and separates each
`limit` from the next; consecutive `asset`, `group` and `approvers` declarations are not
separated.

**Within a limit,** clauses appear in grammar order: dimension, its `except` clauses, `per`,
`scope`, then `escalate`. `except` clauses are sorted ascending by byte value of their emitted
text — S4 forces them disjoint, so their order carries no meaning. `escalate` clauses are sorted
with `above` triggers first, ascending by threshold, and `when exhausted` last; §8.4 admits at
most one of the latter.

**Indentation.** Header at column 0. Declarations at 2 spaces. Clauses within a `limit` at 4.
`except` clauses at 6.

**Spacing.** Exactly one space between adjacent tokens. One space either side of `=`. No space
either side of `@`. **No alignment padding** — the example in §4 is aligned for reading, and an
emitter MUST NOT reproduce that, because alignment depends on the widest name in the block and
so changes unrelated lines whenever a name changes.

**Sets** are emitted as `{ a, b, c }`: one space inside each brace, `, ` between items, no
trailing comma. Items keep source order; set membership is unordered, but reordering them buys
nothing and loses the author's grouping.

**Money literals** are emitted with exactly the asset's declared minor-unit digits — `500.00`
for a 2-decimal asset, never `500` or `500.000`. Counts are emitted with no leading zeros.
Durations and calendar units are emitted in the unit the author used; the unit is significant
(§8.3) and is not normalised.

An example of the canonical form of §4's charter is in
[`examples.md`](examples.md), which marks it as such.

## 2 · Lexical structure

### 2.1 Encoding and whitespace

Input is UTF-8. A leading U+FEFF MUST be ignored. All syntactically significant characters are
ASCII; non-ASCII MAY appear only inside comments, and a parser MUST NOT reject a document for
non-ASCII in a comment.

Whitespace is `SP`, `HT`, `CR`, `LF`. It separates tokens and is otherwise insignificant:
**indentation carries no meaning and newlines are not terminators.** The examples are formatted
for reading only. A conforming document MAY be written on one line.

### 2.2 Comments

`#` begins a comment that runs to the next `LF` or end of input. There is no block comment.

### 2.3 Reserved words

```
above  account  agent  amount  approvers  asset  at  count  day  days
deny   escalate except  exhausted  fixed  full  group  hour  hours  in
instrument  is  least  limit  minute  minutes  month  not  of  or  and
per   charter require  resolver  rolling  scope  second  seconds  timezone
to    up  version  week  when  within  year  common  counterparty  before
after  date  category  provenance  principal  merchant  network
extends  unlimited  policy
```

`policy` is reserved but is not a keyword: it was the opening keyword in an earlier draft, and
reserving it turns a stale document into a clear error rather than a confusing one.

Reserved words are lowercase and MUST NOT be used as identifiers. Matching is
case-sensitive; `Limit` is not a keyword and is a valid identifier.

Multi-word operators — `is not`, `not in`, `is at least`, `up to`, `when exhausted` — SHOULD be
recognised by the lexer as single tokens. A parser that does this has no ambiguity between the
`not` of negation and the `not` of an operator.

### 2.4 Identifiers

```
ident = ( ALPHA | "_" ) { ALPHA | DIGIT | "_" | "-" } ;
```

Maximum 64 characters. An identifier that equals a reserved word is a lexical error (E101).

### 2.5 Numbers

```
uint    = DIGIT { DIGIT | "_" } ;
decimal = uint [ "." DIGIT { DIGIT } ] ;
```

`_` is a readability separator and is stripped before interpretation. It MUST NOT lead, trail,
or follow `.`. There is no sign, no exponent, and no hexadecimal number. Leading zeros are
permitted and insignificant.

`uint` MUST fit in an unsigned 64-bit integer (E102).

### 2.6 Money

```
money = decimal ident ;
```

The identifier MUST name an asset declared in this document (E201). The number of fractional
digits MUST NOT exceed the declared decimals of that asset as given by the resolver (E202).

A money literal denotes an exact integer count of the asset's minor units:
`value × 10^decimals`. The conversion MUST be exact — a literal that cannot be represented
exactly in minor units is an error, never a rounding (E202). The result MUST fit in an unsigned
64-bit integer (E203).

There are no negative amounts. `0` is a valid amount.

### 2.7 Durations

```
duration = uint time-unit ;
time-unit = "second" | "seconds" | "minute" | "minutes"
          | "hour" | "hours" | "day" | "days" ;
```

Singular and plural are interchangeable and carry no agreement requirement: `1 seconds` is
legal. A duration denotes a whole number of seconds and MUST be in the closed range
**[10, 2678400]** — ten seconds to thirty-one days (E204).

### 2.8 Dates

```
date-lit = 4DIGIT "-" 2DIGIT "-" 2DIGIT ;
```

A proleptic Gregorian calendar date. It MUST be a real date (E205); `2026-02-30` is an error.
It denotes a local calendar date in the charter's timezone (§5.4), not an instant.

### 2.9 Timezones

```
tz = "UTC" | tz-name ;
tz-name = tz-part { "/" tz-part } ;
tz-part = ( ALPHA | DIGIT ) { ALPHA | DIGIT | "_" | "+" | "-" } ;
```

MUST be a name present in the IANA time zone database (E206). Implementations MUST agree on a
tzdata version, which the compiled form records (§9). Aliases MUST NOT be silently
canonicalised: a charter naming a link that later retargets is a change of meaning, so the name
is recorded as written and resolved at compile time.

### 2.10 Asset references

An asset reference is lexed as a **single token** and decomposed by a dedicated sub-parser
(§2.10.6). It is never handled by splitting on `/`.

#### 2.10.1 Surface grammar

```
mint-ref = "mint://" symbol "/" issuer "/" mint-id "/" caip2 ;
unit-ref = "unit://" unit-code "/" authority ;

symbol    = ( ALPHA | DIGIT ) { ALPHA | DIGIT | "." | "-" } ;        (* 1..32 *)
issuer    = ( ALPHA | DIGIT ) { ALPHA | DIGIT | "." | "-" | "_" } ; (* 1..64 *)
mint-id   = ( ALPHA | DIGIT ) { ALPHA | DIGIT } ;                     (* 1..128, opaque *)
unit-code = 3ALPHA ;                                                  (* uppercase *)
authority = ( ALPHA | DIGIT ) { ALPHA | DIGIT | "." | "-" } ;        (* 1..32 *)

caip2     = namespace ":" reference ;
namespace = LOWER-ALNUM ;                                             (* 3..8, CAIP-2 *)
reference = ( ALPHA | DIGIT | "-" | "_" ) ;                          (* 1..32, CAIP-2 *)
```

Exactly four segments after `mint://` and two after `unit://`. No trailing slash, no query,
no fragment, no port, no userinfo, no percent-encoding (E410). A reference with the wrong
segment count is E411, naming the count found.

#### 2.10.2 The parsed object

```
AssetRef = Mint { symbol, issuer, mint_id, chain }   |   Unit { code, authority }
chain    = Caip2 { namespace, reference }
```

Each field carries a **span into the original token**, so a diagnostic underlines the
offending segment rather than the whole reference. This is why it is a parser: a lexer has
already discarded the offsets a segment-level error needs.

#### 2.10.3 Identity versus assertion — the distinction that matters

**The identity of an asset is `(chain, mint_id)` and nothing else.**

`symbol` and `issuer` are **assertions**: what the author believes they are authorising.
They are verified against the resolver and never used to identify anything (S7). Two
references with the same `(chain, mint_id)` and different symbols are **not two assets** —
one of them is wrong, and it is E401.

This is why the redundancy is worth its length. A bare address can only be wrong silently;
a reference carrying the author's belief can be wrong loudly.

It is also the rule for deciding what belongs in a reference at all: **a segment exists
only if an author holds a belief about it.** Decimals, token program and asset class are
facts about the mint that an author has no independent belief about, so they come from the
resolver and MUST NOT appear in the reference (E412). Putting a fact in the reference means
two references to one mint could disagree about it.

#### 2.10.4 Per-namespace validation

`mint-id` and the CAIP-2 `reference` are opaque to the grammar and validated by a table
keyed on `namespace` (S9). An unknown namespace is E403, never accepted unchecked.

| namespace | chain reference | mint-id | CAIP-19 asset ns |
|---|---|---|---|
| `solana` | 32 base58 chars, the genesis-hash prefix | 32–44 base58, excluding `0` `O` `I` `l` | `token` |
| `eip155` | decimal chain id, no leading zero | `0x` + 40 lowercase hex | `erc20` |

Adding a chain is an entry in this table plus resolver support, reviewed — not an
authoring-time capability.

#### 2.10.5 CAIP-19 as an input form

A parser MUST accept CAIP-19 wherever a `mint-ref` is expected:

```
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/token:EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

`<namespace>:<reference>/<asset-ns>:<asset-ref>`. The asset namespace MUST match the table
above for that chain (E413). `symbol` and `issuer` are filled from the resolver, so a
CAIP-19 input **cannot fail S7** — there is no author belief to contradict. Emission is
always the native four-segment form; CAIP-19 is accepted, never produced.

#### 2.10.6 Canonical form and equality

Before any comparison: the scheme and the CAIP-2 namespace are lowercased. `symbol`,
`issuer`, `mint_id` and the CAIP-2 `reference` are preserved **byte for byte** — no case
folding, no Unicode normalisation, no percent-decoding, no alias expansion.

Two references are equal iff every segment is byte-equal after that normalisation. Two
references are the **same asset** iff `(chain, mint_id)` are byte-equal, which is the
weaker test and the one the engine uses.

#### 2.10.7 What a reference resolves to

The resolver record, carried verbatim into the compiled form's `resolved_assets` (§9) so
evaluation performs no lookup:

```
MintRecord { chain, mint_id, symbol, issuer, decimals, class,
             token_program, status: active | revoked, since_version }
UnitRecord { code, authority, decimals, rate_sources[] }
```

`class` is a closed enumeration matching the taxonomy in `synthetic-stablecoins.md`:
`fiat_reserve`, `over_collateralised`, `delta_neutral`, `algorithmic`, `tokenised_treasury`.
It is what `asset.class` compares against (§6).

`decimals` is the sole authority for money conversion (§2.6). `token_program` distinguishes
Token from Token-2022 and is a resolver fact for the reasons in §2.10.3. `status: revoked`
always stops a charter naming that mint (E406) — revocation is the mechanism for "this
turned out to be something else", so it MUST NOT be absorbed quietly.

### 2.11 Group literals

```
literal = address | tagged ;
address = ( ALPHA | DIGIT ) { ALPHA | DIGIT } ;         (* 1..128, opaque *)
tagged  = tag ":" tag-value ;
tag     = LOWER-ALPHA { LOWER-ALPHA } ;                  (* 1..16 *)
tag-value = ( ALPHA | DIGIT ) { ALPHA | DIGIT | "-" } ;  (* 1..32 *)
```

Defined tags are `mcc` and `country`. `mcc` values MUST be exactly four digits (E207);
`country` values MUST be ISO 3166-1 alpha-3, uppercase (E208). An unknown tag is an error
(E209) — the tag set is closed for the same reason the field set is.

## 3 · Grammar

```
charter        = "charter" ident "version" uint
                 [ "extends" ident "@" uint ]
                 "resolver" resolver-tier "@" uint
                 "timezone" tz
                 { declaration } ;

resolver-tier  = "common" | "full" ;

declaration    = asset-decl | group-decl | approvers-decl | limit-decl ;

asset-decl     = "asset" ident "=" ( mint-ref | unit-ref ) ;
group-decl     = "group" ident "=" "{" literal { "," literal } [ "," ] "}" ;
approvers-decl = "approvers" ident "=" "{" ident { "," ident } [ "," ] "}" ;

limit-decl     = "limit" ident dimension window [ scope ] { escalation } ;

dimension      = "amount" money { amount-exc }
               | "count"  uint  { count-exc } ;

amount-exc     = "except" ( money | "deny" | "unlimited" ) "when" condition ;
count-exc      = "except" ( uint  | "deny" ) "when" condition ;

window         = "per" ( "rolling" duration
                       | "fixed" cal-unit [ "in" tz ] ) ;
cal-unit       = "day" | "week" | "month" | "year" ;

scope          = "scope" ( "account" | "agent" | "instrument" | "counterparty" ) ;

escalation     = "escalate" trigger
                 "require" uint "of" ident
                 "up to" ( money | uint )
                 [ "within" duration ] ;

trigger        = "above" ( money | uint )
               | "when exhausted" ;

condition      = disjunction ;
disjunction    = conjunction { "or" conjunction } ;
conjunction    = negation { "and" negation } ;
negation       = "not" negation | primary ;
primary        = "(" condition ")" | comparison ;
comparison     = field operator value ;

field          = "counterparty" | "category" | "asset" | "asset.class"
               | "provenance" | "provenance.recipient" | "provenance.amount"
               | "provenance.asset" | "provenance.venue"
               | "date" ;

operator       = "is" | "is not" | "in" | "not in"
               | "before" | "after" | "is at least" ;

value          = ident | literal | date-lit | plane | asset-class
               | "{" value-item { "," value-item } [ "," ] "}" ;
value-item     = ident | literal | plane | asset-class ;

plane          = "principal" | "agent" | "merchant" | "network" ;
asset-class    = ident ;   (* validated against the resolver's class set *)
```

The grammar is LL(1) given §2.3's multi-word tokens. Every declaration and every clause within
a limit is keyword-led, so a limit ends at the first token that cannot continue it. There is no
terminator and no significant layout.

**Operator precedence** in conditions is `not` > `and` > `or`, left-associative. Parentheses
override. A parser MUST NOT rely on evaluation order for meaning (§8.2).

## 4 · Example

```
charter acme-treasury version 7
resolver common@41
timezone Europe/London

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group trusted_suppliers = { 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU }
  group hardware          = { mcc:5045, mcc:5732 }
  approvers finance       = { alice, bob, carol }

  limit daily_spend
    amount 500.00 USDC
      except 5000.00 USDC when counterparty in trusted_suppliers
      except deny         when date after 2026-12-20 and date before 2027-01-02
    per fixed day in Europe/London
    scope agent
    escalate above 200.00 USDC  require 2 of finance up to 5000.00 USDC within 1 days
    escalate when exhausted     require 2 of finance up to 5000.00 USDC within 1 days

  limit burst
    amount 100.00 USDC
    per rolling 5 minutes
    scope agent

  limit transaction_count
    count 20
    per fixed day in Europe/London
    scope agent

  limit untrusted_counterparty
    amount 50.00 USDC
      except 0 USDC when provenance is principal
    per rolling 24 hours
    scope counterparty
```

## 5 · Names and scope

**5.1** All declarations share one flat namespace per document. Redeclaring a name is an error
(E210), including across kinds — an asset and a group MUST NOT share a name.

**5.2** Declarations are **order-independent**. A limit MAY reference an asset declared below
it. Implementations MUST resolve names in a pass separate from parsing.

**5.3** There are no forward-declaration rules, no imports, no includes, and no inheritance
between policies. A document is self-contained apart from the resolver.

**5.4** The charter-level `timezone` is **mandatory** and is the zone in which every `date`
condition is interpreted. A `fixed` window MAY override it with `in tz` for its own alignment
only; that override MUST NOT affect date conditions. This decoupling is deliberate: the
alternative makes a condition's meaning depend on which limit encloses it.

## 6 · Types

Each field admits a fixed set of operators and value shapes. Anything else is a type error
(E301).

| Field | Operators | Value |
|---|---|---|
| `counterparty` | `is`, `is not`, `in`, `not in` | address, or group of addresses |
| `category` | `is`, `is not`, `in`, `not in` | `mcc:` or `country:` tagged, or group thereof |
| `asset` | `is`, `is not`, `in`, `not in` | declared asset ident, or set thereof |
| `asset.class` | `is`, `is not`, `in`, `not in` | class name, or set thereof |
| `provenance` | `is`, `is not`, `in`, `not in`, `is at least` | plane, or set of planes |
| `provenance.recipient` | as `provenance` | as `provenance` |
| `provenance.amount` | as `provenance` | as `provenance` |
| `provenance.asset` | as `provenance` | as `provenance` |
| `provenance.venue` | as `provenance` | as `provenance` |
| `date` | `before`, `after` | date literal |

`in` and `not in` take either a group identifier or an inline `{ … }` set. A group's members
MUST be homogeneous (E302) and its kind MUST match the field (E303).

### 6.1 Provenance is ordered

`Plane` is a **total order by increasing taint**:

```
principal  <  agent  <  merchant  <  network
```

`provenance` alone denotes the intent's *tier* — the maximum over the four fields, because an
intent is exactly as trustworthy as its worst field. The dotted forms denote one field each.

`is at least` compares on that order: `provenance is at least merchant` holds for `merchant`
and `network`.

**Prefer `is at least` to an enumerated set in security predicates.** `provenance in { merchant,
network }` silently excludes any plane added later, so a rule meant to catch untrusted input
stops catching the newest untrusted input. `is at least merchant` continues to hold. Both are
legal; a compiler SHOULD warn (W1) on an enumerated set over `provenance` that contains the
maximum plane.

`date before X` is strict (`< X`), as is `date after X` (`> X`). Neither includes the boundary
day, so `after 2026-12-20 and before 2027-01-02` spans the 21st through the 1st inclusive.

## 7 · Static semantics

A conforming compiler MUST reject a document unless all of the following hold. Each rule has a
stable number so a conformance case can name what it tests.

**S1 · Exception values are literals.** Never an expression, an arithmetic form, or a reference
to another value. This is what makes the ceiling set finite and readable from the source.

**S2 · Conditions MUST NOT reference accumulated state.** Guaranteed structurally by §6's field
table, which contains no such field. A compiler MUST NOT provide an extension that adds one.

**S3 · No construct removes or disables a limit.** Adding a limit can only tighten a charter.

**S4 · Exceptions within one dimension MUST be provably disjoint.** Where two conditions can be
shown to overlap — intersecting date ranges, groups sharing a member, one condition subsuming
another, or any pair a compiler cannot separate — the document is rejected naming both (E304).

> Forcing disjointness rather than resolving by priority is the strict choice, and it is
> reversible: a later version can relax it without invalidating any charter written under it,
> whereas the reverse breaks documents already deployed.

**S5 · Every limit MUST have a computable static ceiling.** For `amount`, the maximum over the
base value and every exception value, in minor units of the limit's asset. For `count`, the
maximum integer. `deny` contributes nothing. An escalation raises this to its `up to` value,
which MUST be a literal (E305) and MUST be greater than or equal to the base (E306).

**S6 · A limit's asset is fixed.** Every money literal in one limit — base, exceptions and
escalation ceilings — MUST name the same declared asset (E307). Limits over different assets
are different limits. A cap without an asset is not a bound, because summing across assets sums
incommensurable units.

**S7 · Every asset reference MUST resolve** in the pinned resolver, with **every segment
matching** (E401). A `mint-ref` whose symbol, issuer or network disagrees with the resolver's
record for that mint is rejected naming the disagreeing segment. An unknown mint fails closed
(E402); "never seen" and "fine" MUST NOT produce the same outcome.

**S8 · References are normalised before comparison.** The scheme and the CAIP-2 namespace are
lowercased; the CAIP-2 reference, the mint id, the symbol and the issuer are preserved
byte-for-byte. Comparison is then exact. No case-folding, no Unicode normalisation, no
percent-decoding.

**S9 · The mint id MUST match its namespace's shape.** `solana` requires 32–44 base58
characters excluding `0`, `O`, `I`, `l`; `eip155` requires `0x` and 40 hex digits. An unknown
namespace is rejected (E403) rather than accepted unchecked.

**S10 · Issuer MUST come from the resolver's curation, never from on-chain metadata.** Token
metadata is self-asserted, so an issuer copied from it is worthless as a discriminator — and
the issuer segment is what separates native from bridged. A resolver that derives issuer from
metadata is non-conforming.

**S11 · A `unit://` limit MUST declare a rate source and a staleness bound**, and MUST deny when
the rate is stale (E404). Conversion MUST round in the direction that tightens the limit.

**S12 · The resolver pin MUST be satisfied.** Evaluating under a version later than the pin is
permitted only if every asset the document names still resolves identically, field for field
(E405). A revoked mint always stops the documents that name it (E406).

**S13 · `common` MUST be a strict subset of `full`.** Where a mint appears in both, every field
MUST be identical. If they disagree for a mint, both fail closed for that mint. This is
verified when a resolver version is cut and re-verified by anything consuming both; it is what
makes the tiering a bandwidth optimisation rather than a second source of truth.

**S14 · Every approver ident in an escalation MUST name a declared approver set** (E308), and
the required quorum MUST be at least 1 and no greater than that set's size (E309).

**S15 · At most one escalation per trigger kind per limit** (E310). Two `when exhausted`
escalations on one limit have no defined composition.

**S16 · A document MUST declare at least one limit** (E311). An empty charter is more likely a
truncated file than an intent to permit everything, and permitting everything MUST be
impossible to express by omission.

**Emitted, not checked:** the static ceiling per limit per path (§9). A document whose bound
cannot be computed does not compile.

## 8 · Dynamic semantics

### 8.1 Accumulation

The model is **petty cash**: a limit's allowance is drawn down when a payment is committed to,
not when it settles.

**8.1.1** Each limit owns its own accumulator, keyed `(limit id, scope value, asset)`. Two
limits with the same scope do **not** share a counter.

**8.1.2 · `amount` is a balance.** Reserved at the moment of decision. Released only on
**definite failure**.

**8.1.3 · `count` is a rate.** Consumed on attempt and **never** released — including on
failure. Releasing it would let an agent with a total failure rate retry without bound, which
is the case the control exists for.

**8.1.4 · Release requires proof of death, not a timeout.** An amount returns to the allowance
only when the transaction provably can never land — on Solana, blockhash expiry, roughly 150
slots. Releasing on a timer while a transaction is still in flight double-spends the allowance.
This trigger is a chain fact and MUST NOT be author-configurable.

**8.1.5 · A pending escalation holds its reservation.** Otherwise thirty approvals queue under
the ceiling and release together. It expires after the escalation's `within` duration, or a
default of 1 day; on expiry the reservation is released and the request is denied.

**8.1.6 · A payment belongs to the window in which it was reserved**, not the one in which it
settled.

**8.1.7 · Fixed windows** align to local midnight in the window's timezone. Under a DST
transition an ambiguous local time resolves to its **first** occurrence and a nonexistent one to
the instant the offset changes; a 23-hour and a 25-hour day each receive one allowance.

### 8.2 Evaluating one limit

1. Collect every exception whose condition holds for this request.
2. **None** — the base value applies.
3. **Exactly one** — its value applies.
4. **More than one** — a **conflict**. The limit denies and the charter is flagged as broken.

S4 rejects provable overlap at compile time; case 4 exists for overlap that could not be decided
statically. Resolving by declaration order is forbidden: an author who writes two exceptions
believing them exclusive would otherwise be paid by whichever the parser reached first.

If the resolved value is `deny`, the limit denies. Otherwise the limit compares
`reserved + requested` against the value.

### 8.3 Composing limits

Every limit is evaluated. The outcome is the join over this order:

```
allow  <  escalate  <  deny
```

Any limit denying denies. Otherwise any limit escalating escalates. A denial reports every
limit that denied, not the first.

### 8.4 Exhaustion is not prohibition

Two refusals that MUST behave differently:

- **Exhaustion** — `reserved + requested` exceeds the resolved value. If the limit declares a
  `when exhausted` escalation, the outcome is **escalate**, bounded by its `up to` ceiling.
  Otherwise **deny**.
- **`deny` as a resolved value** — the author wrote a prohibition. **Final.** No escalation
  lifts it, and a quorum MUST NOT be offered.

Collapsing these makes an empty allowance unappealable and an explicit prohibition negotiable,
both backwards.

### 8.5 What human authorization means for the bound

An approved escalation may release a signature above the autonomous ceiling and below the
escalation ceiling. The guarantee is therefore:

> No execution releases signatures whose aggregate exposure exceeds the charter's limits over
> any window, **absent explicit human authorization of the exact payment digest.**

The qualifier is the thesis rather than a weakening: the bound is on what an agent does
unaccompanied. A person approving a digest is a different party making an attributable
decision. Because every escalation names a finite `up to` (S5), the ceiling on the
human-authorized path is still a literal in the source, and the static bound remains
computable — two numbers instead of one.

## 8A · Hierarchy

A charter MAY extend exactly one parent, forming a chain — company, department, manager,
agent. There is no multiple inheritance (E312): two parents give a limit two ceilings with
no defined composition.

```
charter dept-a version 3
extends company-wide@7
resolver common@41
timezone Europe/London
```

The parent pin is a version, not a name alone. A parent that changes is subject to the same
re-verification rule as a resolver (S12): a child MAY evaluate under a later parent version
only if every limit the child references still resolves compatibly, and a parent that
*tightens* always propagates (E407).

**H1 · A child may only tighten.** The effective ceiling for a request is the **minimum**
over every level in the chain, evaluated per request after each level resolves its own
exceptions. A child declaring a higher number does not raise anything; the minimum still
binds. A compiler SHOULD warn (W2) on a child ceiling above its parent's static maximum,
since it is dead text rather than an error.

This is §3's conjunctive rule lifted one level: limits within a document compose by
conjunction, and documents in a chain compose the same way. The bound argument extends by
induction — the root is finite by S5, and every child is min(own, parent), which is finite.

**H2 · Each level accumulates over its whole subtree.** A payment by the leaf draws down the
leaf's allowance, its manager's, its department's and the company's. That is what a
company-wide limit means. Accumulator keys gain the level: `(level, limit id, scope value,
asset)`. `scope` is orthogonal — a limit at the department with `scope agent` means per agent
within that department.

**H3 · Escalation is answered by the level that imposed the constraint.** If the binding
ceiling came from COMPANY_WIDE, a department's approvers MUST NOT satisfy it (E313). A
manager cannot convene a quorum of their own reports to spend past a company limit. Where
several levels bind, the highest imposing level's approvers are required.

**H4 · `unlimited` means *this level adds no constraint*, never *no constraint exists*.**

It is legal only in a charter that extends another (E314), and its value is the parent's
effective ceiling — finite by induction, so S5 still holds. In a root charter it is
unbounded and MUST be rejected.

An author writing `unlimited` under a parent capped at $200 a week is capped at $200 a week.
The interface MUST show the effective ceiling alongside the declared one, because the word
means something narrower than it reads.

**H5 · An asset the chain does not permit is denied.** There is no inheritance of permission
by omission: an intent naming an asset with no declared cap at every level is denied (E315),
not permitted by default. A child cannot introduce an asset its parent never allowed.

**H6 · `deny` propagates downward and cannot be lifted.** A prohibition at any level is final
for every level beneath it, and no child exception and no quorum reaches it (§8.4).

## 9 · Compiled form

Compilation emits, per limit:

```
limit_id · dimension · asset · base · exceptions[] · ceiling_static
         · window_kind · window_params · tz · scope_key
         · escalations[] · selector_program
```

and per document a header:

```
charter_id · version · resolver_tier · resolver_version · tzdata_version
         · timezone · resolved_assets[]
```

`resolved_assets` carries the full resolver record for every asset named — mint, symbol,
issuer, network, decimals, class, token program — so evaluation performs no lookup and S12's
re-verification has something to compare against.

`selector_program` is the compiled condition: a tree over §6's closed field set, with no loops,
no recursion, and bounded depth. `ceiling_static` is the S5 maximum, recorded per path
(autonomous, escalated) so the invariant check reads it rather than recomputing it.

The compiled form MUST be canonical: the same document compiles to byte-identical output under
the same resolver and tzdata versions. This is what makes a decision reproducible in a dispute.

## 10 · Errors

Errors are stable identifiers so two implementations report the same thing. Every error MUST
name a source span; every error over two rules MUST name both.

```
E1xx lexical    101 reserved word as identifier   102 integer overflow
E2xx literal    201 unknown asset name            202 fractional digits exceed decimals
                203 minor-unit overflow           204 duration out of range
                205 invalid calendar date         206 unknown timezone
                207 malformed mcc                 208 malformed country
                209 unknown tag                   210 duplicate declaration
E3xx structure  301 operator not valid for field  302 heterogeneous group
                303 group kind mismatch           304 overlapping exceptions
                305 non-literal escalation ceiling 306 escalation below base
                307 mixed assets in one limit     308 unknown approver set
                309 quorum out of range           310 duplicate escalation trigger
                311 no limits declared
E4xx resolver   401 segment disagrees             402 unknown mint
                403 unknown chain namespace       404 missing or stale rate source
                405 pinned asset changed          406 mint revoked
                410 malformed reference syntax   411 wrong segment count
                412 fact segment in a reference  413 caip-19 asset ns mismatch
W1   warning    enumerated provenance set includes the maximum plane
```

## 11 · Conformance

An implementation is conforming if it agrees on every case in the suite.

```
conformance/
  parse/accept/*.charter         parse and compile cleanly
  parse/reject/*.charter         leading comment block: # expect: E304
  roundtrip/*.charter            text → JSON → text, byte-identical in canonical form (§1.1)
  canonical/*.charter            + expected compiled bytes
  eval/*.json                    charter + request sequence → expected decisions
  asset-ref/                     the mint:// and unit:// sub-parser, on its own
```

Every rule in §7 MUST have at least one `reject` case naming it. Every clause in §8 MUST have at
least one `eval` case. Reject cases carry the expected error code so the suite tests the
*identity* of the refusal, not merely that one occurred.

A reject case's expected code appears **somewhere in the leading comment block** — the run of
comment lines before the first non-comment line — and not necessarily on the first line, because
a fixture copied from or derived from an upstream suite carries a provenance header. A harness
MUST scan that whole block for `# expect:` rather than reading line one. A fixture with no
`# expect:` in its leading block is a malformed fixture, not a wildcard.

`roundtrip/` is a **byte comparison** against the canonical text form of §1.1, not a semantic
one. Two emitters that agree semantically and disagree on layout are two emitters that will
drift.

`eval` vectors MUST include: the 101st unit against a 100-unit allowance escalating rather than
failing; a reservation released on blockhash expiry and not on a timer; a `count` not released
on failure; a payment reserved before and settling after a window boundary; and a DST
transition in both directions.

## 12 · Deferred

Not in this version, and each would be additive:

- Explicit exception priority (S4 forces disjointness instead).
- `deny` as its own rule form rather than a value.
- Time-of-day conditions.
- Cross-limit references; conjunction is the only composition.
- Cross-asset ceilings — these need a rate, which drags every §S11 objection into the general
  case rather than the opt-in one.
- Fixed-window realignment when a charter is edited mid-window.
