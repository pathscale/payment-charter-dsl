# Choosing the model, not the library

Date 2026-09-02. A design comparison of the candidate policy languages for the authoring
layer. Companion to [`authoring-background.md`](authoring-background.md).

Assume any of them can be made to run inside the enclave — that is engineering, not a
constraint. The question is which **model** fits the thing being expressed.

## What is actually being expressed

A spending policy, written by a financial controller, has a shape that is worth stating
before comparing anything:

1. A **base allowance** — five hundred a day.
2. **Exceptions**, which are more specific and win — except this supplier, which is five
   thousand.
3. **Exceptions to exceptions** — except in December.
4. An **escalation boundary** — above that, a human approves.
5. Evaluated against **accumulated state** — what has already been spent this window.

Points 1 to 4 are one structure: *a general rule with prioritised exceptions*. Point 5 is a
different thing entirely, and no candidate below addresses it.

## The candidates, by their core abstraction

### Catala — prioritised default logic

Catala's central feature is **default logic as a first-class construct**:
definition-under-conditions, formalised from Sarah Lawsky's *A Logic for Statutes*. A value
has a base definition and a set of exceptions, and the language resolves precedence as a
semantic property rather than as evaluation order.

**This is the closest model fit in the entire comparison**, because legislation and spending
policy are the same shape. "X is allowed, except when Y, unless Z" is how statutes read and
how controllers think. Nothing else here expresses exceptions natively; everywhere else you
encode precedence by hand, and hand-encoded precedence is where policies quietly become
wrong.

It also produces a human-readable rendering alongside executable code, which maps directly
onto the audit requirement: the controller and the auditor read the same artefact the engine
executes.

**Against it, and it is serious.** The project describes itself as "a research project from
Inria" whose "compiler is yet unstable and lacks some of its features." There is *partial*
certification of the compiler, not full. Adopting an unstable research compiler as the thing
that decides whether money moves is a risk that has nothing to do with whether the model is
right.

### Rego — Datalog over documents

Rego derives facts from input and data, and a decision is a query against them. Regorus makes
it practical here: Rust, `no_std`, tested against musl targets, compliant with the OPA suite.

**As a model it is a poorer fit.** There is no native notion of a default with exceptions, so
"base limit, except for this vendor" is encoded as ordinary rules whose interaction the author
must get right unaided. Rego is excellent at *"is this request permitted given these facts"*
and unremarkable at *"which of several overlapping allowances applies."*

Its real strengths are ecosystem and operational familiarity, and those are not nothing. But
they are arguments about adoption cost, not about fit.

### DMN with FEEL — decision tables with declared hit policy

Rows of conditions mapping to outputs, plus an explicit **hit policy** — unique, first,
priority, collect — declaring what happens when several rows match.

That makes precedence a stated property of the table rather than an emergent one, which is
Catala's insight arrived at from the business-analyst direction rather than the legal one. It
is less expressive: a flat table does not nest exceptions the way default logic does. It is
also an OMG standard with mature implementations and no research risk, and a table is the
artefact a controller is least surprised by.

### DAML — multi-party contracts

Models rights, obligations and workflows between parties, with a ledger authorisation model
built around signatories and observers.

**A different problem.** It answers *who may do what to which contract*, which is
authorisation over a shared ledger, not *how much may be spent against a rolling allowance*.
Powerful, and aimed elsewhere.

### Rebel — state machines for financial products

CWI with ING, deployed in their product development: transactions and the states they pass
through, with finance-native types and tooling for simulation and validation.

**Also a different problem, and the closest neighbour.** It specifies what a financial product
*is* and how its state evolves — which overlaps our reservation state machine far more than
our limit expression. Worth reading precisely because it is the mature, bank-deployed example
of formalising this domain, and because whether it checks properties over *accumulated* values
is the question that decides whether the second paper has a topic at all.

## The thing none of them do

**Every candidate is a stateless evaluator.** Each decides one request against inputs it is
handed. None models accumulation, none bounds aggregate behaviour over a window, and none can
tell you what a set of rules admits in total.

That is not a defect in any of them. It means the division of labour is forced:

> **The language chooses which limit applies. The engine enforces the accumulation.**

Exposure is computed by the engine and passed in as an input; the policy decides *which
ceiling* this request faces. The invariant — no execution releases signatures exceeding the
limits over any window — stays a property of the engine, provable because the engine stays
small, and is not delegated to a language that cannot express it.

This is also why the predicate vocabulary must stay closed regardless of which language wins.
An expressive language does not endanger the invariant by being expressive; it endangers it by
letting an author name a ceiling the engine cannot bound.

## What I would do

**Take Catala's model. Do not take Catala.**

Prioritised default logic is the right abstraction and the reason is not aesthetic: it is the
only one of these that expresses base-plus-exception as semantics rather than as a convention
the author must uphold. Adopt that — a base allowance with prioritised exceptions and declared
resolution — as the semantics of our rule table.

**Express it as a DMN-shaped table with an explicit hit policy.** That gets the precedence
property in a form a controller can read, an OMG standard to point at rather than a bespoke
notation to defend, and no dependency on an unstable compiler.

**Keep the evaluator ours and small.** With a closed vocabulary the evaluator is a few hundred
lines. Rego via regorus stays a plausible fallback if the vocabulary later needs to open up,
and it is a genuinely good escape hatch — but reaching for it first means importing a
general-purpose evaluator into the one component whose value is that it is small enough to
reason about.

**Read Rebel before writing any of it.** It is the mature bank-deployed formalisation of this
domain, and there is no sense rediscovering what ING already paid for.

## Confidence

Catala's instability and default-logic core, and Regorus' `no_std` and musl support, are from
primary sources. Catala's compilation targets, its production deployments, and whether Rebel
checks accumulated properties are **unverified** — the pages read were project descriptions
rather than technical documentation. The recommendation above would survive Catala being more
mature than advertised; it would not survive Rebel turning out to do exactly this already.
