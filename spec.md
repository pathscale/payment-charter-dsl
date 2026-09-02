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
4. `prohibit`
5. `limit`

Within each kind, declarations are sorted by identifier, ascending by byte value. Declaration
order is not semantic (§5), so sorting is what makes the output a function of the meaning rather
than of the author's typing. A blank line separates one kind from the next, and separates each
`limit` from the next; consecutive `asset`, `group` and `approvers` declarations are not
separated.

**Within a limit,** clauses appear in grammar order: dimension, its `except` clauses, `per`,
`scope`, then `escalate`. `except` clauses are sorted ascending by byte value of their emitted
text — S4 forces them disjoint, so their order carries no meaning. `escalate` clauses are sorted
with threshold triggers first, ascending by threshold, and `when exhausted` last. S17 makes
`above` and `at least` one trigger kind and S15 admits at most one of each kind per limit, so no
two threshold triggers can coexist on a limit and the sort never has to break a tie between
them.

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
extends  unlimited  policy  prohibit
```

`policy` is reserved but is not a keyword: it was the opening keyword in an earlier draft, and
reserving it turns a stale document into a clear error rather than a confusing one.

`deny` is reserved and is **no longer a value**. It was an exception value in an earlier draft
(`except deny when …`); prohibition is now its own declaration (§3, §8.2.1). Reserving the word
turns a stale document into E103 rather than a parse error at an unhelpful position.

Reserved words are lowercase and MUST NOT be used as identifiers. Matching is
case-sensitive; `Limit` is not a keyword and is a valid identifier.

Multi-word operators — `is not`, `not in`, `is at least`, `at least`, `up to`, `when exhausted`
— SHOULD be recognised by the lexer as single tokens. A parser that does this has no ambiguity
between the `not` of negation and the `not` of an operator. `is at least` and `at least` are
distinct tokens: the first is a comparison operator on a field (§6), the second is an escalation
trigger (§3). Longest match wins, so `is at least` is never lexed as `is` followed by
`at least`.

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

declaration    = asset-decl | group-decl | approvers-decl
               | prohibit-decl | limit-decl ;

asset-decl     = "asset" ident "=" ( mint-ref | unit-ref ) ;
group-decl     = "group" ident "=" "{" literal { "," literal } [ "," ] "}" ;
approvers-decl = "approvers" ident "=" "{" ident { "," ident } [ "," ] "}" ;

prohibit-decl  = "prohibit" ident "when" condition ;

limit-decl     = "limit" ident dimension window [ scope ] { escalation } ;

dimension      = "amount" money { amount-exc }
               | "count"  uint  { count-exc } ;

amount-exc     = "except" ( money | "unlimited" ) "when" condition ;
count-exc      = "except" uint "when" condition ;

window         = "per" ( "rolling" duration
                       | "fixed" cal-unit [ "in" tz ] ) ;
cal-unit       = "day" | "week" | "month" | "year" ;

scope          = "scope" ( "account" | "agent" | "instrument" | "counterparty" ) ;

escalation     = "escalate" trigger
                 "require" uint "of" ident
                 "up to" ( money | uint )
                 [ "within" duration ] ;

trigger        = ( "above" | "at least" ) ( money | uint )
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
override. A parser MUST NOT rely on evaluation order for meaning (§8.2.2).

## 4 · Example

```
charter acme-treasury version 7
resolver common@41
timezone Europe/London

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group trusted_suppliers = { 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU }
  group hardware          = { mcc:5045, mcc:5732 }
  approvers finance       = { alice, bob, carol }

  prohibit holiday_freeze when date after 2026-12-20 and date before 2027-01-02

  limit daily_spend
    amount 500.00 USDC
      except 5000.00 USDC when counterparty in trusted_suppliers
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

> **Prohibitions are exempt from S4** and are exempt in both directions: two prohibitions may
> overlap each other, and a prohibition may overlap any exception. Disjointness exists because
> two *ceilings* covering one request have no defined composition. A prohibition is not a
> ceiling and composes with everything by dominating it (§8.2.1), so there is nothing to
> resolve. Requiring disjointness here would be actively harmful: it would force an author who
> wants "5000 for trusted suppliers, but never during the freeze" to thread `and not
> <freeze-condition>` through every exception, and a prohibition that has to be repeated in
> every clause it constrains is one that will eventually be forgotten in one of them.

> Forcing disjointness rather than resolving by priority is the strict choice, and it is
> reversible: a later version can relax it without invalidating any charter written under it,
> whereas the reverse breaks documents already deployed.

**S5 · Every limit MUST have a computable static ceiling.** For `amount`, the maximum over the
base value and every exception value, in minor units of the limit's asset. For `count`, the
maximum integer. An escalation raises this to its `up to` value, which MUST be a literal (E305)
and MUST be greater than or equal to the base (E306).

> Prohibitions do not enter the calculation at all. They can only remove authority, never grant
> it, so the bound computed while ignoring every prohibition is still a sound upper bound — and
> a bound that is sound when you ignore a construct is a bound nobody has to reason about that
> construct to trust. This is the structural reason prohibition is a declaration and not a value
> in the ceiling slot: as a value it needed a special case here, in §8.4 and in H6, and three
> special cases for one production is the language telling you the shape is wrong.

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
impossible to express by omission. Prohibitions do not satisfy S16: a document consisting only
of prohibitions permits everything it did not think to forbid, which is the deny-list posture
this language exists to refuse.

**S17 · At most one escalation per trigger kind per limit, and `above` and `at least` are the
same kind** (E310). A limit carrying both `escalate above 50.00 USDC` and `escalate at least
50.00 USDC` has two thresholds meeting at a boundary and no defined composition on it. S15
states the rule; this fixes which triggers collide under it.

**S18 · A prohibition MUST be reachable** (E317). A prohibition whose condition the compiler can
prove unsatisfiable — an empty date range, a group with no members, a conjunction of a field
comparison and its own negation — is rejected rather than compiled to nothing. Every other
construct in this language fails towards refusing; a prohibition that silently never fires fails
towards permitting, and it looks exactly like protection while providing none.

> Note the deliberate asymmetry with S4. A prohibition may overlap anything, but it may not
> overlap *nothing*. Overlap is composition, which is defined; unreachability is a mistake,
> which is not.

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

### 8.2 Evaluating a request

Prohibitions are evaluated first and independently of every limit. Only if none holds is any
limit consulted.

#### 8.2.1 Prohibitions

Evaluate every prohibition in the document, and in every document in the chain (§8A). If any
condition holds, the request is **denied**, the outcome names every prohibition that held, and
evaluation stops. No limit is examined, no accumulator moves, and no quorum is offered.

A prohibition is **final**. It is not a ceiling of zero and MUST NOT be modelled as one:

- A ceiling of zero is exhaustion, and §8.4 turns exhaustion into an escalation where one is
  declared. A prohibition must never become escalatable, because "you may not pay this
  counterparty" and "you have run out of money for this month" are different sentences and only
  the second is a reasonable thing to ask a human to override at 2am.
- A ceiling participates in H1's minimum. A prohibition does not participate in anything; it
  short-circuits.

Because prohibitions never touch an accumulator, a prohibited request costs nothing. An agent
repeatedly attempting a prohibited payment is refused every time and consumes no allowance —
including no `count`, which §8.1.3 otherwise never releases. This is deliberate: `count` exists
to bound retry against a *permitted* control, and letting a prohibition burn it would let an
attacker exhaust the legitimate rate budget with requests that were never going to be paid.

#### 8.2.2 Evaluating one limit

1. Collect every exception whose condition holds for this request.
2. **None** — the base value applies.
3. **Exactly one** — its value applies.
4. **More than one** — a **conflict**. The limit denies and the charter is flagged as broken.

S4 rejects provable overlap at compile time; case 4 exists for overlap that could not be decided
statically. Resolving by declaration order is forbidden: an author who writes two exceptions
believing them exclusive would otherwise be paid by whichever the parser reached first.

The limit then compares `reserved + requested` against the resolved value. Every resolved value
is a ceiling; there is no longer a value that means refusal, because refusal is §8.2.1.

#### 8.2.3 Escalation triggers

A limit's escalations are tested against the **requested** amount, before accumulation:

- `above V` fires when `requested > V`, strictly.
- `at least V` fires when `requested >= V`.
- `when exhausted` fires when `reserved + requested` exceeds the resolved ceiling (§8.4).

Both threshold forms exist because English does not agree with itself here and the difference is
one payment. "Anything of fifty dollars or more needs my approval" is `at least 50.00`; written
as `above 50.00` it lets a payment of exactly 50.00 through unattended, which is the single
transaction the controller was most clearly thinking about when they wrote the rule. A language
that offers only `above` guarantees that class of off-by-one, and guarantees it silently —
nothing is malformed, the charter compiles, and the boundary payment is simply not the one the
author meant. Neither form is a default and neither is spelled `>=`; the author has to say which
edge they mean.

### 8.3 Composing limits

Every limit is evaluated. The outcome is the join over this order:

```
allow  <  escalate  <  deny
```

Any limit denying denies. Otherwise any limit escalating escalates. A denial reports every
limit that denied, not the first.

A prohibition (§8.2.1) does not enter this join. It is decided before the join runs and there is
nothing for it to lose to.

### 8.4 Exhaustion is not prohibition

Two refusals that MUST behave differently:

- **Exhaustion** — `reserved + requested` exceeds the resolved value. If the limit declares a
  `when exhausted` escalation, the outcome is **escalate**, bounded by its `up to` ceiling.
  Otherwise **deny**.
- **Prohibition** — the author wrote `prohibit` (§8.2.1). **Final.** No escalation lifts it, and
  a quorum MUST NOT be offered.

Collapsing these makes an empty allowance unappealable and an explicit prohibition negotiable,
both backwards. Keeping them in separate constructs rather than separate values of one construct
is what makes the distinction survive a careless edit: an author cannot accidentally turn a
prohibition into a ceiling by changing a number, because there is no number there to change.

### 8.4A Replacing a charter mid-window

A charter is edited while its windows are open. A CFO tightening the company cap on the 14th
does so with two weeks of the month already spent, and what happens to that spending is a
semantic decision, not an implementation detail.

**The rule: an edit changes the ceiling. It never changes the meter.**

A limit's accumulator is keyed `(limit id, scope value, asset, window instance)` (§8.1.1) and a
window instance is identified by the wall-clock interval it covers, not by the charter version
that created it. Installing a new charter version therefore re-points every limit at a new
ceiling and leaves every accumulator exactly where it was.

Consider a `100.00 USDC per fixed month` limit with `80.00` already reserved:

| Edit | Result | Why this is the only safe answer |
|---|---|---|
| Lowered to `50.00` | Everything denies until the window rolls | The controller *reduced* authority. Any rule under which they instead handed the agent a fresh allowance is a rule where tightening a limit increases spending. |
| Raised to `200.00` | `120.00` remains this month | What the controller meant. Not `200.00` more. |
| Unchanged | `20.00` remains | Editing an unrelated limit must not disturb this one. |

The rejected alternative is restarting the window on edit. It fails in the first row and fails
catastrophically: the agent spends `130.00` in a calendar month in which the controller twice
said `100.00` and then said `50.00`. It also hands anyone who can trigger a charter update a
general-purpose allowance reset, which is a spending exploit that requires no signature forgery
at all — just the ability to make the controller save the file.

**8.4A.1 · A changed window specification is a superposition.** If the new version alters a
limit's `per` clause — `fixed month` to `fixed week`, `rolling 7 days` to `rolling 24 hours`,
or the timezone — there is no corresponding accumulator to carry forward. The superseded limit
therefore continues to be enforced against its own accumulator until its final window closes,
alongside the new one. Both are live; §8.3's join means the most restrictive wins.

Without this, changing `fixed month` to `fixed week` is a window reset, and the exploit closed
in the table above reopens through a different clause.

**8.4A.2 · A removed or renamed limit keeps enforcing until its window closes.** A limit present
in version *n* and absent in version *n+1* is not deleted; it stops accruing new authority and
continues to deny against what it already accumulated, until its final window ends.

This is the same mechanism, and it closes the same hole. A limit id is a name, so renaming
`petty_cash` to `petty_cash_v2` would otherwise produce a fresh accumulator and a fresh
allowance — a rename as an allowance reset. Under this rule the retired `petty_cash` still holds
its `80.00` against its `100.00` for the rest of the month, so the rename buys nothing.

Note that S3 ("no construct removes or disables a limit") governs what a *document* can say.
8.4A.2 governs what a *replacement* can do, which is the same principle applied across versions:
authority already spent is not recoverable by editing the thing that measured it.

**8.4A.3 · Hierarchy inherits this unchanged.** Each level's accumulator is its own (H2), so a
parent tightening mid-window binds every child immediately through H1's minimum, and no child
gains anything from the parent's edit. A department that has spent its `80.00` under a company
cap the CFO has just lowered is over its cap, at once, everywhere.

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

**H6 · Prohibitions propagate downward and cannot be lifted.** A `prohibit` declaration at any
level holds for every level beneath it. No child exception and no quorum reaches it (§8.4), and
a child MUST NOT declare a prohibition that narrows a parent's — a prohibition is not a ceiling,
so there is no minimum to take and nothing for a child to tighten.

The set of prohibitions in force for a request is the **union** over the whole chain, evaluated
together in §8.2.1. This is the only construct in the language that unions rather than takes a
minimum, and it is consistent: H1 minimises ceilings because the tightest grant wins, and H6
unions prohibitions for the same reason — every refusal in the chain is in force at once.

A child may of course add prohibitions of its own. That is tightening, which H1 permits in
general and S3 requires to always be available.

## 9 · Compiled form

Compilation emits, per limit:

```
limit_id · dimension · asset · base · exceptions[] · ceiling_static
         · window_kind · window_params · tz · scope_key
         · escalations[] · selector_program
```

per prohibition:

```
prohibition_id · selector_program
```

and per document a header:

```
charter_id · version · resolver_tier · resolver_version · tzdata_version
         · timezone · resolved_assets[] · prohibitions[]
```

Prohibitions are document-level and carry no dimension, asset, window, scope or ceiling. A
prohibition has nothing but a name and a condition, which is the compiled form saying the same
thing S5 and §8.2.1 say: it is not a limit, it holds no state, and it takes no part in any
arithmetic. `prohibition_id` exists so a denial can name which one refused, as §8.2.1 requires.

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
                103 `deny` used as a value
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
                311 no limits declared            312 multiple inheritance
                313 escalation answered below the level that imposed it
                314 `unlimited` in a root charter
                315 asset with no cap at any level
                316 prohibitions but no limit     317 unreachable prohibition
E4xx resolver   401 segment disagrees             402 unknown mint
                403 unknown chain namespace       404 missing or stale rate source
                405 pinned asset changed          406 mint revoked
                407 parent tightened               410 malformed reference syntax
                411 wrong segment count           412 fact segment in a reference
                413 caip-19 asset ns mismatch
E5xx authenticity
                501 signature invalid             502 unknown controller key
                503 compiled digest mismatch      504 version not monotonic
                505 charter outside validity window
                506 charter name mismatch
W1   warning    enumerated provenance set includes the maximum plane
W2   warning    a child limit above its parent's ceiling is dead text
```

E103 exists because `deny` was an exception value before prohibition became a declaration
(§2.3). A document written against the earlier draft must fail at the `deny` token saying so,
rather than at whatever the parser tripped over next.

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
  authenticity/                  signature, key, digest, version and validity cases (§12)
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
on failure; a payment reserved before and settling after a window boundary; a DST transition in
both directions; a request exactly equal to an `at least` threshold and the same request against
an `above` one; a prohibited request consuming no accumulator and being offered no quorum; a
limit lowered mid-window; and a limit renamed mid-window.

An implementation MUST also have cases for §12: an unsigned charter, an invalid signature, an
unknown key, a compiled digest that does not match its commitment, a replayed lower version, a
commitment whose `charter` names a different document, and one outside its validity window. An
engine that enforces every rule above on a charter an attacker wrote is not conforming, and
authenticity is the only part of this specification whose absence is invisible from inside a
correct evaluation.

## 12 · Authenticity

Everything above describes what a charter *means*. This section is about whether the charter the
engine enforces is the one the controller wrote, and it is normative.

### 12.1 The problem

The host is untrusted by design. Today the host hands the engine a compiled charter and the
engine enforces it faithfully. Faithful enforcement of an attacker's charter is not enforcement:
**a compromised host can choose its own limits**, and every bound in this specification becomes
a statement about a document nobody authorised.

This is not a residual risk to note. It is the whole guarantee, and it is currently missing.

### 12.2 The rule

A conforming engine MUST reject any charter that does not arrive with a valid controller
signature. There is no unsigned path, no development bypass reachable in a production build, and
no "trusted host" configuration.

### 12.3 What is signed

Not the charter. A **commitment**:

```json
{
  "charter":       "acme-treasury",
  "version":       7,
  "text_digest":   "sha256:…",
  "compiled_digest": "sha256:…",
  "key_id":        "…",
  "not_before":    "2026-09-02T00:00:00Z",
  "not_after":     "2027-09-02T00:00:00Z"
}
```

Serialised with JCS (RFC 8785) and signed. Two digests, and the reason for each is the reason
this design is not obvious.

**`text_digest` covers the canonical text form (§1.1).** It is the only field that binds the
signature to something a human read. A controller reviews text; if the signature covered JSON
alone they would be attesting to bytes they never saw, in a different notation, which is
precisely the display-one-sign-another gap this system exists to close for payments. Applying
that standard to payments and not to the document that authorises them would be incoherent.

**`compiled_digest` covers the compiled form (§9).** It is what the engine actually evaluates.
The engine verifies the signature, then verifies that `sha256(received compiled form)` equals
`compiled_digest`, and evaluates. **It never parses text** — `text_digest` is an opaque 32 bytes
to it — so the crate split holds and the enclave-side implementation stays dependency-free.

The controller's own client compiles and computes both digests. That client is trusted, and it
is the only component that can be: the controller has to be trusted to read their own charter.
The host is not in that path, which is the point. Anyone may independently recompile the text
and check that the two digests agree — a divergence is evidence, publicly checkable, and it
makes a lying client a detectable event rather than an undetectable one.

### 12.4 What the engine checks

In order, failing closed at the first failure:

1. `key_id` names a key in the engine's trust root (§12.5), else **E502**.
2. The signature over the JCS-canonicalised commitment verifies, else **E501**.
3. The current time is within `[not_before, not_after)`, else **E505**.
4. `charter` matches the charter being installed, else **E506**. Without this, a signed
   commitment for a permissive charter can be replayed against a restrictive one.
5. `version` is strictly greater than the highest version yet installed for this charter name,
   else **E504**.
6. `sha256(compiled form)` equals `compiled_digest`, else **E503**.

**Step 5 is anti-rollback and it costs nothing**, because the header already carries
`charter <name> version <uint>` (§3). That field was there for humans; it is exactly the
monotonic counter the engine needs, and the engine already has anti-rollback machinery for
accumulator state. Reusing one number for both means an attacker cannot replay last quarter's
generous charter, and an operator cannot do it by accident either.

Note the interaction with §8.4A: rejecting a stale version is not the same as ignoring a valid
one. A newly installed charter takes effect immediately and does not reset any meter.

### 12.5 The trust root and rotation

The engine holds a **controller key set**, established at provisioning and part of what the
attestation covers. A host that can change the trust root can mint charters, so:

- Adding or removing a controller key is itself a signed operation, requiring a quorum of the
  *current* set. A single compromised controller key cannot rotate itself into sole control.
- Removing the last key is refused. An engine with no controller keys accepts nothing and is
  unrecoverable, and an unrecoverable engine holding funds is worse than a compromised one.
- A revoked key invalidates future installations only. Charters already installed under it stay
  in force until replaced, because the alternative — spontaneously voiding live limits — means
  either everything denies or, far worse, nothing does.
- `not_after` bounds the damage from a key compromise nobody noticed. A charter that outlives
  its validity window stops (E505) rather than continuing indefinitely.

### 12.6 What this does not solve

A compromised host still controls *availability*: it can refuse to forward a new charter, or
keep presenting an older valid one within its validity window. Signing bounds what a host can
make the engine *do*; it cannot make a host cooperate.

That residual is why `not_after` is mandatory rather than optional. Staleness becomes a
liveness failure with a deadline instead of a silent indefinite one, and a controller who sees
their new charter not taking effect learns something is wrong from the charter expiring rather
than from an invoice.

## 13 · Deferred

Not in this version, and each would be additive:

- Explicit exception priority (S4 forces disjointness instead).
- Time-of-day conditions.
- Cross-limit references; conjunction is the only composition.
- Cross-asset ceilings — these need a rate, which drags every §S11 objection into the general
  case rather than the opt-in one.

### 13.1 `prices` and `scope item` — deferred, with the analysis corrected

Proposed in [`examples.md`](examples.md) §2c: a `prices` table naming a per-item ceiling the
principal wrote down, `amount from <table>` taking the ceiling per item, and `scope item`
accumulating per item so buying one thing does not consume another's allowance.

The motivation is sound and the shape is close to right. It is deferred for one hard reason and
two soft ones.

**The hard one: the proposal's stated bound is wrong.** §2c claims the static ceiling is the
maximum entry, 45.00. It is not. Under `scope item` each item accumulates separately, so the
window exposure is the **sum** over the table — 105.00 for three items — and the maximum is only
the per-item ceiling. A feature whose published bound is off by a factor of the table length is
not finished, and S5 is the rule this language is built around.

**`scope item` alone is unbounded.** Accumulating per item keys the allowance on an identifier
the merchant supplies. Absent an enumeration, a merchant that varies the item id draws a fresh
allowance every time, and the aggregate is unbounded — the §2b failure exactly, moved one level
down: the merchant can no longer manufacture the *condition*, but can manufacture an unlimited
supply of fresh *accumulators*. The `prices` table is what closes this, by enumerating every id
that can resolve to money at all. So the two are not separable features; `scope item` MUST be
legal only in a limit whose dimension is `amount from <prices>`, and an item absent from the
table has no ceiling and is refused.

**And it needs a vocabulary extension.** §6's field table has no item identity. Adding one is an
engine contract change, which S2 says is a reviewed engine change and not an author's to make.

None of this is fatal, and the corrected version is coherent:

- `scope item` legal only with `amount from <prices>`.
- The S5 contribution is the **sum** of the table's entries, not the maximum.
- An item not in the table is refused; the table is the enumeration that makes the key space
  finite.
- The request vocabulary gains an item identity, as a reviewed change.

It is deferred rather than adopted because nothing in the core mandate needs it, and because
grammar is one-way: syntax added can be removed only by breaking documents already written,
while syntax deferred costs nothing but a later version number. When a price table is wanted,
this is the design — with the sum.
