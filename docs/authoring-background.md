# The charter a controller writes, and the engine that enforces it

Date 2026-09-02. Companion to [`product-todo.md`](product-todo.md), section C2.

Two layers, and only the second one exists.

**The authoring layer** is what a financial controller sits in front of: limits, velocities,
approval thresholds, expressed in their vocabulary. **The enforcement layer** is the 552-line
state machine that couples durable state to signature release. Today the second one *is* the
interface — a `Charter` struct with eight scalar fields, set from environment variables. There
is no authoring layer at all.

## What a controller actually configures, per the industry

Not a guess. This is the field set that card issuing platforms have converged on, taken from
Lithic's velocity limit rules and matched by Marqeta, Highnote and Stripe Issuing.

**Two dimensions, not one.**
- `limit_amount` — maximum spend
- `limit_count` — maximum number of transactions

**Five window types, fixed as well as rolling.**
- `DAY` — a *fixed* window aligned to midnight in a named timezone
- `WEEK` — fixed, with a configurable `day_of_week`
- `MONTH` — fixed, with a configurable `day_of_month`
- `YEAR` — fixed, with `month` and `day_of_month`
- `CUSTOM` — a rolling window in seconds, from 10 seconds to 31 days

**Scope.**
- `CARD` — each instrument tracked independently
- `ACCOUNT` — combined across every instrument under it

**Filters that select which transactions a rule applies to.**
- `include_mccs` / `exclude_mccs` — merchant category
- `include_countries` / `exclude_countries` — ISO 3166-1 alpha-3
- `include_pan_entry_modes`
- explicit exemptions by instrument

**And several rules coexist**, each matching a different slice of traffic.

Fireblocks' Transaction Authorization Policy is the same idea for digital assets: a rule
engine over outgoing transactions that returns **allow, block, or require additional
approval**, with multi-asset rules, segmented charter views, a guided builder and an API.

### Measured against what we expose

| A controller expects | We have |
|---|---|
| Amount **and count** limits | Amount only |
| Fixed calendar windows — "this month's budget" | One rolling window in seconds |
| Several windows at once — 5 minutes, a day, a month | One |
| Scope hierarchy: account, agent, instrument | One global `Charter` |
| Selection by category, geography, counterparty class | A flat recipient allowlist |
| Many coexisting rules | One struct |
| Three outcomes including *require approval* | Present — this part we do have |

"Max spend in five minutes" is a `CUSTOM` rolling window of 300 seconds. It is the single
most ordinary thing on the list, and it is the one thing our single `window_secs` can express.

## Is there a financial charter DSL?

**Yes, and it is a standard: DMN.**

Decision Model and Notation is an OMG standard for specifying business decisions as tables,
with **FEEL** — Friendly Enough Expression Language — as its expression language. It is built
for exactly this split: authored by business people, executed by an engine. It is used in
fintech and implemented by Drools, Camunda and Red Hat Decision Manager.

The part of DMN that matters most here is **hit charter**: the declared rule for what happens
when several rules match one input — first match, unique match, priority, collect. That is
precisely the precedence question a multi-rule charter has to answer, and DMN answers it as a
property of the table rather than as an accident of evaluation order.

Adjacent, and worth knowing:

- **AWS Cedar** — an authorisation language whose evaluation core is *mechanically verified*.
  Not financial, but it is the standard to hold an engine to if the claim is that limits are
  never exceeded.
- **OPA / Rego** — general charter-as-data, Datalog-descended. Common in authorisation, rare
  in money.
- **XACML** — the older enterprise ABAC standard, largely superseded.

## Should it be hardcoded?

**No. And it should not be a general-purpose language either.**

Nobody in payments ships a Turing-complete policy language for money, and the reason is not
timidity. It is that **an open language destroys the ability to say anything about the
system as a whole.**

That matters here more than anywhere, because of what this codebase claims:

> No execution releases signatures whose aggregate exposure exceeds the charter's limits over
> any window.

`check_invariant` is the executable statement of that, and it is only checkable because the
predicate set is closed and tiny. **Let a controller write arbitrary rules and the invariant
becomes unstatable** — not merely harder to test, but undefined, because "the charter's limits"
no longer names a bounded thing. The paper's central claim and an expressive DSL are in direct
tension, and the tension resolves in exactly one direction.

## The shape that keeps both

Three layers, with the constraint living in the middle.

**1 · Author against a closed vocabulary.** A rule table, DMN-shaped: conditions drawn from a
*fixed* set of predicates — amount, count, window, scope, counterparty class, category,
asset — and outcomes drawn from a fixed set — allow, deny, require approval at quorum N.
Authored in the merchant dashboard, and in an API for the same reason every other surface has
one.

**2 · Compile, and refuse what cannot be bounded.** Rules compile to the engine's primitives.
The compiler enforces the properties the invariant needs: an explicit hit charter so
overlapping rules resolve deterministically, a proof obligation that the effective ceiling is
finite, and refusal of any table whose composition cannot be bounded. **A charter that does not
compile is not a charter** — which is a far better error than one that compiles and quietly
admits more than the author believed.

**3 · Enforce in the small engine.** The 552 lines stay small, stay provable, and gain the
fields they are missing — count limits, fixed calendar windows, several simultaneous windows,
a scope hierarchy, per-asset amounts.

The value of this split is that expressiveness lives where it can be reviewed and rejected,
while the thing making the safety claim stays small enough to reason about. It is the same
argument as compiling the payload inside the enclave rather than accepting caller-supplied
bytes, applied one level up.

## What to do first

1. **Add the missing primitives** — count, fixed windows, multiple windows, scope, per-asset.
   The authoring layer cannot express what the engine cannot enforce, and these are the fields
   every issuing platform has.
2. **Write the predicate vocabulary down** as a closed set, before any UI. It is the contract
   between the two layers and the thing the invariant quantifies over.
3. **Pick a hit charter and make it explicit.** First-match is the common choice and the
   easiest to explain to a controller looking at a table.
4. **Then build the table UI**, which is the easy part once the vocabulary is fixed.
5. **Read Cedar's approach to verified evaluation** before committing to a compiler. If the
   claim is that limits hold, the evaluator is the place that claim is won or lost.
