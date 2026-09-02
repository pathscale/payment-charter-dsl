# Payment Charter DSL

A policy language for **bounded spending authority** — what a financial controller writes so an
autonomous agent can spend within limits that are provable rather than hoped for.

A single document is **a charter**. Files are `.charter` (text) and `.charter.json` (wire).
After first mention the language is just **Charter**.

```
charter "research-assistant" version 1

  limit 500.00 asset mint://solana/EPjFWdd5.../USDC per rolling 7 days
  limit 50.00  asset mint://solana/EPjFWdd5.../USDC per payment

  when exhausted escalate to group finance up to 2000.00
```

## The one property everything protects

**The maximum a charter can ever authorise is computable by reading it.**

Charter is deliberately *not* a general policy language and must never become one. There is no
arithmetic, there are no functions, no collections and no modules. Every restriction in the
specification exists so that a static bound can be read off the source, and every proposal has
to answer whether it preserves that.

## The documents

| File | Role |
|---|---|
| [`spec.md`](spec.md) | **Normative.** Write a parser against this and nothing else. |
| [`examples.md`](examples.md) | Worked examples and a refusal gallery. Seeds the corpus. |
| [`design.md`](design.md) | Rationale. Why the semantics are what they are. |
| [`docs/authoring-background.md`](docs/authoring-background.md) | What a controller expects, drawn from Lithic/Marqeta field sets. |
| [`docs/language-comparison.md`](docs/language-comparison.md) | Why not Catala, Rego, DMN, DAML or Rebel. |

The specification covers lexical structure, a complete EBNF, the field × operator × value type
table, sixteen static rules (S1–S16), hierarchy (H1–H6), dynamic semantics, the compiled form, a
stable error catalogue (E1xx–E4xx), and the conformance layout.

## Implementations

| Repo | Contents |
|---|---|
| **`payment-charter-dsl`** (this one) | The spec, the examples, and the conformance corpus. No code. |
| [`payment-charter-dsl-rs`](https://github.com/pathscale/payment-charter-dsl-rs) | Rust: lexer, parser, AST, emitter, static rules, compiled form. |
| [`payment-charter-dsl-ts`](https://github.com/pathscale/payment-charter-dsl-ts) | TypeScript: types, JSON Schema, emitter; a parser last, if it comes out cheap. |

Rust is the reference implementation. TypeScript leads with the types and the emitter, which is
the half the browser actually needs, and takes a parser only once the emitter is done and the
cost is known to be small — a second full parser means a second implementation of the whole
error catalogue, which is the expensive part.

Whatever ships is held to the corpus in this repo. That is the only thing keeping two
implementations from drifting, so a change to the language lands here first, as fixtures, and in
the implementations second.

## The conformance corpus

```
conformance/
  parse/accept/*.charter    compiles cleanly
  parse/reject/*.charter    leading comment block carries `# expect: E304`
  roundtrip/*.charter       text → JSON → text, byte-identical in canonical form (§1.1)
  canonical/*.charter       + expected compiled bytes
  eval/*.json               charter + request sequence → expected decisions
```

A reject case names the error code it expects, so the suite tests the **identity** of a refusal
rather than merely that one occurred. A test that passes because the wrong error fired is worse
than no test.

Coverage is derived from the specification rather than invented: every error code, every static
rule, every cell of the type table, the literal edge cases, every dynamic clause, and hierarchy
at each depth. See [`conformance/README.md`](conformance/README.md).

## Licence

Dual [Apache-2.0](LICENSE-APACHE) / [MIT](LICENSE-MIT), at your option.

Fixtures copied from or derived from upstream suites keep their own attribution headers and are
listed in [`conformance/NOTICE`](conformance/NOTICE).
