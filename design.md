# Payment Charter DSL — design notes

Date 2026-09-02. Syntax and semantics for the authoring layer. Companion to
[`docs/language-comparison.md`](docs/language-comparison.md), which argues for taking
Catala's default-logic model without taking Catala.

**The normative document is [`spec.md`](spec.md)** — the one a parser is written
against. This file is the rationale: why the semantics are what they are. Where the two
disagree, the spec wins.

One idea is borrowed: **a value has a base definition and prioritised exceptions, and
ambiguity is an error rather than a choice.** Everything else here is ours, because the
vocabulary has to stay closed for the engine's invariant to remain provable.

## Two compositions, not one

The distinction that shapes the grammar:

**Limits are conjunctive.** Five hundred a day *and* a hundred in any five minutes *and*
twenty transactions a day are three limits that must all hold. There is no precedence between
them; each is checked, any one can refuse.

**A limit's value has exceptions.** "Five hundred, except five thousand for this supplier" is
one limit whose ceiling depends on the request. This is where default logic applies, and it
applies to *the value*, not to whether the rule runs.

Conflating these is how policy languages get confusing. Velocity controls compose by
conjunction; allowances compose by precedence.

## Grammar

```
charter     ::= "charter" ident "version" integer resolver declaration*

resolver    ::= "resolver" ("common" | "full") "@" integer

declaration ::= asset-def | limit | group | approvers

asset-def   ::= "asset" ident "=" asset-ref

limit       ::= "limit" ident
                  dimension
                  window
                  [ scope ]
                  [ escalation ]

dimension   ::= ("amount" money | "count" integer) exception*
exception   ::= "except" (money | integer | "deny") "when" condition

money       ::= decimal ident              // ident must be a declared asset

window      ::= "per" ( "rolling" duration
                      | "fixed" ("day"|"week"|"month"|"year") [ "in" timezone ] )

scope       ::= "scope" ("account" | "agent" | "instrument" | "counterparty")

escalation  ::= "above" money "require" integer "of" ident

group       ::= "group" ident "=" "{" literal ("," literal)* "}"
approvers   ::= "approvers" ident "=" "{" ident ("," ident)* "}"

condition   ::= term (("and" | "or") term)*
term        ::= field operator value | "not" term | "(" condition ")"
field       ::= "counterparty" | "category" | "asset" | "asset.class"
              | "provenance" | "date"
operator    ::= "is" | "is not" | "in" | "not in" | "before" | "after"
```

**The vocabulary is closed.** `field` is an enumeration, not an open set. Adding one is a
change to the engine, reviewed, not something an author can do.

## A charter

```
charter acme-treasury version 7
resolver common@41

  asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp

  group trusted_suppliers = { 0xA1B2…, 0xC3D4… }
  group hardware          = { mcc:5045, mcc:5732 }
  approvers finance       = { alice, bob, carol }

  limit daily_spend
    amount 500.00 USDC
      except 5000.00 USDC when counterparty in trusted_suppliers
      except deny         when date after 2026-12-20 and date before 2027-01-02
    per fixed day in Europe/London
    scope agent
    above 200.00 USDC require 2 of finance

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

Read the last one carefully: it caps spend *per counterparty* at fifty, except where the
principal stated the details themselves, in which case this limit does not bind. That is the
tainted-tier rule from the existing engine, expressed as an exception rather than as a second
hardcoded field.

## Asset references

### The ban is on undeclared tickers, not on short names

`USDC` appears throughout the charter above, and that is fine, because **the document binds it
in the document**:

```
asset USDC = mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp
```

A bare ticker is the one thing in a charter that looks like a control and is not one. USDC
exists on several chains under different mints, bridged and native variants both read as
"USDC", and anyone can mint a token called USDC on Solana for a few cents. A charter saying
`500 USDC` with no binding looks tight and admits an attacker's mint.

A binding declared in the same file has none of that problem: the full reference is in the
artefact, verbatim, and the short name is unambiguous within it. So the rule is narrow and
absolute — **an alias must be declared in the document that uses it, and there is no global
alias table, ever.** A global table is the hole; a local binding is just readability, and
readability matters here because the controller and the auditor read the same file the engine
compiled.

### The reference

```
mint://USDC/Circle/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/solana:5eykt4Us…
         │     │      │                                            │
         │     │      │                                            └─ network
         │     │      └─ the mint: the actual identity
         │     └─ issuer
         └─ symbol
```

`mint://` names the on-chain identity of a fungible token, whatever the chain calls it — a mint
on Solana, an ERC-20 contract on Ethereum, a coin type on Sui. The word is ours to define.

**Every segment is verified against the resolver, and any mismatch is a compile error naming
the disagreement.** That is what makes the redundancy worth having: the author writes what they
believe they are authorising, and the compiler checks the belief. A bare address can only be
wrong silently; this can be wrong loudly.

**CAIP-19 is accepted as an input form** and normalised on the way in, so a mint pasted from a
wallet or a cross-chain tool resolves rather than being rejected on syntax:

```
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/token:EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

### The network segment is a genesis identity, never a name

This is the segment most likely to be got wrong, and writing it as `solana/mainnet` is how it
gets got wrong. A name is a label somebody chose; the question a charter actually needs answered
is *which ledger*.

So the network is identified by its **genesis identity**, in CAIP-2 form — for Solana, the
first 32 characters of the base58 genesis hash:

```
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp     mainnet-beta
solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1      devnet
eip155:1                                     Ethereum mainnet
```

Three consequences fall straight out, and each is the behaviour we want:

**A runtime upgrade changes nothing.** Solana ships a new validator version, Ethereum executes
a hard fork — the genesis is the same, the ledger is the same, the mint is the same, and the
charter keeps meaning exactly what it meant. Nothing to re-author, nothing to re-approve. A
version number in this segment would have forced a migration for an event that changed none of
the facts the charter depends on.

**A chain split does not carry over silently.** A contentious fork is a new chain with a new
identity — a new genesis hash, or on EVM a new chain ID, which is precisely the mechanism that
exists to keep the two apart. The old reference does not resolve on the new chain, so the
charter fails closed rather than authorising spend on a ledger its author never considered.
This is the one case where refusing to work is the whole point.

**devnet is a different network, so the entire thing can be rehearsed.** A charter authored
against `solana:EtWTRABZ…` cannot resolve on mainnet and vice versa. Real policies, real
limits, real escalations, real signatures, against a devnet that is structurally incapable of
being confused for production. The failure mode this closes — a charter tested on devnet that
resolves identically on mainnet — is the expensive kind.

Canonicalise on the way in: `solana:5eykt4Us…`, never `Solana`, `SOL` or `mainnet`. The
comparison is security-relevant, so it is a byte comparison over a normalised form. The UI
displays "Solana mainnet"; the document stores the identity.

### Units of account look different on purpose

A dollar is not a token, and the format should make that visible without reading carefully:

```
unit://USD/ISO-4217
```

`mint://` moves. `unit://` only measures. The original question — *500 what?* — is answered
by the shape of the string before anyone parses it.

Denominating a limit in dollars requires a rate, and the enclave has no network, so the rate
arrives from the host — where an understated rate is an inflated cap. It also breaks the bound
argument, since a ceiling of "$500" is unbounded in settled units as the rate moves.

- **Asset-denominated limits are the primitive.** Exact, no oracle, and the natural expression
  for crypto-native users, who think in tokens rather than dollars.
- **A `unit://` limit is opt-in**, requires a **signed rate** with a staleness bound, and
  **denies when stale**. Never a 1:1 assumption: the depeg record in
  the synthetic-stablecoin survey (internal) — USDe at $0.65 on Binance, October
  2025 — is the case where 1:1 fails exactly when the cap matters.
- **Convert conservatively**, always rounding to tighten. Rate manipulation can then only
  refuse payments that should have passed, which is a liveness failure rather than a safety
  one.

### Asset class is a predicate

The five-class taxonomy in the synthetic-stablecoin survey (internal) becomes
something a controller can state rather than something hardcoded:

```
except deny when asset.class is not fiat_reserve
```

That separates two questions currently conflated: what **we** custody, which is class A only,
and what a **user's** charter may reference, since a crypto-native user may knowingly pay a
merchant in an asset we never hold.

## The resolver

Every segment above is checked against something, and that something is published.

**Per chain we support, a resolver maps a mint to its facts:** symbol, issuer, decimals, token
program, asset class. It is versioned and monotonic — entries are added and revoked, never
silently rewritten.

It lives in the same on-chain allowlist as the signer keys, for the same reasons: one source,
revocable, visible to every party, and not editable by a compromised host. Symbol-to-mint and
mint-to-class carry the same weight as a signer key — **mislabel a mint and the control
silently inverts**, which makes the resolver part of the trusted set rather than a convenience
lookup.

Two rules follow, and both fail in the safe direction. **An ambiguous symbol is a compile
error** naming every candidate, never a default to the popular one. And **an unknown mint fails
closed** — "we have never seen this token" and "this token is fine" must not produce the same
outcome.

Decimals come from the resolver, not from the charter and not from an assumption, which also
settles where `500.00` gets turned into minor units.

### Common and full

The full list is meant to be **generous**. Anybody may petition to have a mint listed, and the
bar for listing a mint is much lower than the bar for us to custody it — listing says "this is
what this mint is", not "this is a good idea to hold". Being generous is what makes the format
usable by people paying in assets we would never touch.

But a generous list is a large one, and every client does not need to carry it.

- **`common`** is small: the assets nearly every charter references. It ships with the client
  and is cached, so the ordinary charter compiles with no round trip.
- **`full`** is everything. It is fetched only when a charter names something outside `common`.

A charter declares which it needs, in its header:

```
resolver common@41
```

**The security property that makes this safe is that `common` is a strict subset of `full`,
never a different answer.** Where a mint appears in both, every field must be identical.
That is a checkable property of the published pair — verified when each version is cut, and
re-verified by anything that consumes both — not a promise. If the two ever disagree about a
mint, both fail closed for that mint.

That distinction is the whole justification. A cache that can disagree with its source is an
attack surface, and the cheapest attack on this design would be to get a mint into `common`
with a different class than it has in `full`. A cache that is provably a subset is just
bandwidth.

### Compatibility, and what happens when the resolver moves

A charter pins the resolver version it was authored against, because the author consented to the
facts as they stood: this mint, this issuer, this class.

Evaluating under a **later** version is allowed, on one condition: **every asset the charter
actually names still resolves identically.** Re-verification is cheap — a charter references a
handful of assets, not the whole list — and it gets the behaviour right in both directions.
A resolver update that adds a thousand mints does not disturb a charter that references three.
An update that reclassifies one of those three stops that charter, loudly, naming the field that
changed.

Refusing outright on any version bump would be simpler and worse: it would make routine
additions break every charter in the system, which trains people to re-pin without reading.
Accepting silently would be worse still — it would let a reclassification change what a charter
authorises without anyone having agreed to it.

**A revoked mint always stops the policies that name it.** Revocation is the mechanism for
"this turned out to be something else", so it is exactly the case that must not be
absorbed quietly.

## Resolution, and why ambiguity is an error

Evaluating a limit's ceiling:

1. Collect every exception whose condition holds.
2. **None hold** — the base value applies.
3. **Exactly one holds** — its value applies.
4. **More than one holds** — this is a **conflict**, and it is an error, never a silent pick.

That fourth case is the whole reason to borrow from Catala. A language that resolves overlap
by declaration order will happily let an author write two exceptions they believe are
exclusive, and pay whichever the parser reached first. For money the correct response is to
refuse.

Conflicts are caught in two places:

**At compile time, wherever provable.** If two exception conditions can be shown to overlap —
overlapping date ranges, groups with a common member, a condition that subsumes another — the
charter is rejected with both rules named. The author resolves it by narrowing a condition or
declaring precedence explicitly.

**At evaluation, otherwise.** Where overlap cannot be decided statically, a runtime conflict
denies the payment and raises the charter as broken. Denial is the safe direction, and a
charter that produces conflicts in practice is one to fix rather than tolerate.

## Why the bound survives

The engine's invariant says no execution releases signatures exceeding the charter's limits
over any window. That is only provable if "the charter's limits" names something finite, which
this grammar guarantees by four restrictions:

- **Exception values are literals.** `except 5000.00 USDC`, never `except (balance * 0.1)`.
  So each limit's ceiling comes from a finite set written in the source, and the maximum it can
  ever resolve to is the largest literal — computable by reading the file.
- **Conditions cannot reference accumulated state.** A condition selects *which* ceiling
  applies from request attributes. It cannot ask how much has been spent, which would make the
  ceiling a function of the thing it bounds.
- **Limits are conjunctive and never disable each other.** Adding a limit can only tighten a
  charter. There is no construct that removes one.
- **Every ceiling is denominated in a resolved asset.** A bound is a quantity of something. A
  number with no asset is not a bound, because summing across assets sums incommensurable
  units — see the engine gap below, which is a live bug rather than a gap.

Together: the effective ceiling of a charter over a window is at most the smallest of the
per-limit maxima **per asset**, and the compiler computes it. A charter whose bound cannot be
computed does not compile.

## What it compiles to

A table the existing engine can evaluate without a parser:

```
limit_id · dimension · asset · ceiling_set · window_kind · window_params
         · scope_key · selector_program · escalation
```

plus a header carrying the resolver tier, its version, and the resolved facts for every asset
the charter names — so evaluation needs no lookup and re-verification has something to compare
against.

`selector_program` is the compiled condition — a small tree over the closed field set, no
loops, no recursion, bounded depth. `ceiling_set` is the base plus each exception's literal,
which is what makes the static maximum available to `check_invariant`.

The engine gains what it currently lacks: count as well as amount, fixed calendar windows,
several simultaneous limits, a scope key for accumulation, and **an asset on every cap**.
Those are additive and do not disturb the state machine.

## Deliberately absent

- **Arithmetic on values.** See the bound argument.
- **Loops, recursion, function definitions.** This is a charter, not a program.
- **User-defined fields.** The vocabulary is the engine's contract.
- **A global alias table.** Aliases are per-document or they are a ticker again.
- **Time-of-day beyond dates**, for now, because timezone-dependent windows are already the
  subtlest part and two timezone-sensitive constructs is one too many for a first version.
- **Cross-limit references.** A limit cannot mention another limit; conjunction is the only
  composition.

## Open

- **Precedence when the author wants overlap.** Catala labels exceptions and orders them.
  Whether to allow an explicit `priority` on an exception, or force disjoint conditions
  always, is undecided. Forcing disjointness is stricter and probably right at first.
- **Money literals and rounding.** Minor units as integers, with decimals from the resolver.
  The rounding direction must be decided now, and must always tighten.
- **Whether `deny` belongs as an exception value.** It reads well above, but a denial is not a
  ceiling, and it may be cleaner as its own rule form.
- **Fixed-window alignment on charter change.** Editing a monthly limit mid-month: does the
  window restart, or does the new ceiling apply to the accumulated total? The second is safer
  and stranger to explain.
- **Whether a cross-asset ceiling is expressible at all.** "No more than $1000 across
  everything" is a thing controllers ask for, and it requires a rate, which drags every
  objection in the `unit://` section into the general case rather than the opt-in one.
