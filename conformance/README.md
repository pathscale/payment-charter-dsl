# The conformance corpus

An implementation is conforming if it agrees on every case here. Both
[`payment-charter-dsl-rs`](https://github.com/pathscale/payment-charter-dsl-rs) and
[`payment-charter-dsl-ts`](https://github.com/pathscale/payment-charter-dsl-ts) are judged
against this directory rather than against their own tests.

```
conformance/
  parse/accept/*.charter    compiles cleanly
  parse/reject/*.charter    leading comment block carries `# expect: E304`
  roundtrip/*.charter       text → JSON → text, byte-identical in canonical form (§1.1)
  canonical/*.charter       + expected compiled bytes
  eval/*.json               charter + request sequence → expected decisions
  asset-ref/                the mint:// and unit:// sub-parser, on its own
  authenticity/             signature, key, digest, version and validity cases (spec §12)
```

## Reject cases name their error

A reject case carries its expected code in the **leading comment block** — not necessarily the
first line, because a provenance header may come first. The harness scans that block for
`# expect:`.

This is the point of the directory. A test that passes because the *wrong* error fired is worse
than no test, so the suite asserts the identity of a refusal rather than merely that one
occurred.

## Eval vector shape

An `eval/*.json` file names a charter, a starting clock, and a sequence of requests with the
decision each MUST produce:

```json
{
  "description": "what this vector is for",
  "charter": "parse/accept/petty-cash-at-least.charter",
  "clock": "2026-09-01T09:00:00-04:00",
  "requests": [
    { "amount": "50.00", "asset": "USDC", "agent": "a1", "expect": "escalate",
      "escalation": { "limit": "petty_cash", "trigger": "at least" } }
  ]
}
```

`expect` is `allow`, `escalate` or `deny`. A request may carry `at` to advance the clock, and a
sequence entry may be an `install` rather than a request, which replaces the charter mid-run —
that is how the §8.4A window-continuity cases are written. `note` is for the reader and carries
no assertion.

A denial SHOULD assert `denied_by` naming every rule that refused, since §8.3 requires reporting
all of them rather than the first. An escalation SHOULD assert its trigger, because "escalated"
for the wrong reason is the same class of false pass as the wrong error code on a reject case.

## Where the cases come from

Two sources, and the second is where the volume is.

### Ported from upstream suites — about 30–40 cases

Measured, not estimated. The upstream suites do **not** contain hundreds of portable tests.

| Source | Portable | Note |
|---|---|---|
| Catala `tests/exception/good` | 14 | Port all. This is §8.2 exactly, and these are the highest-value cases in existence for our conflict semantics. |
| Catala `tests/default/good` | 3 | Port all. |
| Catala `tests/date/good` | 6 | Window boundaries and DST. |
| Catala `tests/money`, `tests/dec` | 7 | Minor-unit and rounding edges. |
| Catala `tests/bool` | 3 | Condition precedence. |
| Catala `arithmetic`, `func`, `array`, `enum`, `modules`, `io` | — | Not portable. Charter has no arithmetic, functions, collections or modules, by design. |
| Cedar `cedar-integration-tests` | few | Entity-hierarchy authorization is a different problem; `corpus-tests.tar.gz` is large and almost entirely inapplicable. |
| OPA/Rego suite | few | Datalog over documents. |
| DMN TCK | unverified | The `dmn-tck/dmn-tck` path 404s. Hit-policy cases would be genuinely relevant to §8.2/§8.3 if the suite still exists — find its current home. |

Each of Catala's exception tests is worth ten written from scratch. Port them. Do not expect
volume from them.

### Derived from the specification — several hundred cases

Systematic, not creative. This is the coverage.

- **Every error code needs a case.** E1xx–E4xx as catalogued in §10. Roughly 30 codes.
- **Every static rule needs a reject case naming it.** S1–S16 and H1–H6, several with more than
  one distinct violation. Roughly 22 rules.
- **The type table is a cross product.** §6 is 10 fields × 7 operators = 70 combinations, most
  invalid and required to produce E301. Mechanical, and roughly 70 cases on its own.
- **Literal edge cases.** Fractional digits at, one under and one over the asset's decimals;
  minor-unit overflow at `u64::MAX`; durations at 9, 10, 2678400 and 2678401 seconds;
  `2026-02-29` against `2028-02-29`; every malformed `mint://` segment; base58 containing
  `0`, `O`, `I` or `l`.
- **Every dynamic clause in §8 needs an eval vector.** The ten named in §11 are the minimum, not
  the target.
- **Hierarchy.** Depths 1 through 4; `unlimited` at each depth; escalation authority at each
  level (H3); a child above its parent (W2); an asset permitted nowhere (H5).

Every one of these traces to a numbered rule rather than to taste.

### `asset-ref/` gets its own directory

The `mint://` and `unit://` sub-parser is where a large slice of the corpus lives, and cheaply:
every malformed segment, every namespace mismatch, base58 with the excluded characters, a
CAIP-19 form that normalises and one that does not, a `unit://` where a `mint://` is required.
Keep these here rather than scattered through `parse/reject/`.

## Attribution — copied and derived fixtures carry a header

Catala, Cedar and OPA are Apache-2.0. **Copy freely and attribute properly.** Do not contort a
test to avoid attribution; adding a header is mechanical and is the honest thing to do.

Every fixture copied verbatim **or derived from** one of those suites carries a header naming its
source:

```
# expect: E304
# Derived from CatalaLang/catala tests/exception/exceptions_squared.catala_en
# Copyright (c) Inria and the Catala contributors. Apache-2.0.
# Modified: re-expressed in Payment Charter DSL.
```

"Derived" is read broadly. If the case exists because someone read theirs, it is derived — even
where every line was rewritten. The header costs nothing and settles the question.

**This is a deliberate carve-out from the house rule against copyright banners in source.** That
rule exists so our own files stay clean; it was never meant to dodge attribution for third-party
work. An agent applying `AGENTS.md` literally will try to strip these headers as violations.
They are not violations, and they stay.

Fixtures written from the specification rather than from an upstream suite carry no header, as
usual.

[`NOTICE`](NOTICE) lists each upstream suite and its licence.
