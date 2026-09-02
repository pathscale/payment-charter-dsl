# Working agreement — payment-charter-dsl

The operating contract for **any** coding agent working in this repository. This file is the
single source of truth for the rules: Codex, Cursor and Gemini CLI read `AGENTS.md` natively,
and Claude Code loads it through the `@AGENTS.md` import in [`CLAUDE.md`](CLAUDE.md). **Never
fork these rules into a per-vendor file.**

**This repo holds the specification, the examples and the conformance corpus. No code.**
Implementations live in
[`payment-charter-dsl-rs`](https://github.com/pathscale/payment-charter-dsl-rs) (Rust, the
reference) and
[`payment-charter-dsl-ts`](https://github.com/pathscale/payment-charter-dsl-ts) (TypeScript).

## Invariants (don't break these)

- **The maximum a charter can ever authorise must stay computable by reading it.** This is the
  whole value of the language. Any proposal — a new field, a new operator, a new clause form —
  has to answer this question first, and the answer goes in [`design.md`](design.md). Charter is
  not a general policy language and must never become one.

- **No arithmetic, no functions, no collections, no modules.** Not as a limitation to work
  around; as the thing being protected. If a task seems to need one of them, the task is wrong,
  or it is a different project.

- **[`spec.md`](spec.md) is normative.** Where it and [`design.md`](design.md) disagree, the spec
  wins. A change to the language changes the spec **and** lands fixtures in
  [`conformance/`](conformance/) in the same commit — a rule with no reject case naming it is
  not enforced by anything.

- **No Python.** Not a script, not `python3 -c`, not a heredoc. Do not swap it for another
  parser either, and do not assume `jq` is present: it does not ship with macOS. A fixed-shape
  field is one `sed -nE` line; anything needing real parsing belongs in an implementation repo,
  where it can be tested.

- **Docs describe what is true now.** If you change the language, update the spec, the examples
  and the README in the same change.

## The attribution carve-out — read this before "cleaning up" a fixture

The house rule is that our own source files carry no copyright, licence or SPDX banners.

**Fixtures in [`conformance/`](conformance/) are a deliberate exception.** Anything copied from
or derived from Catala, Cedar or OPA carries a header naming its source and licence:

```
# expect: E304
# Derived from CatalaLang/catala tests/exception/exceptions_squared.catala_en
# Copyright (c) Inria and the Catala contributors. Apache-2.0.
# Modified: re-expressed in Payment Charter DSL.
```

**Do not strip these.** They are not a violation of the house rule — that rule exists so our own
files stay clean, and was never meant to dodge attribution for third-party work. "Derived" is
read broadly: if the case exists because someone read theirs, it is derived, even where every
line was rewritten.

Fixtures written from the spec rather than from an upstream suite carry no header, as usual.
Every upstream suite is indexed in [`conformance/NOTICE`](conformance/NOTICE).

## Fixture conventions

- A reject case names its expected error code as `# expect: E304` **in the leading comment
  block**, not necessarily on the first line — a provenance header may come first.
- Coverage is derived from the spec, not invented. Every error code, every static rule, every
  cell of the §6 type table, every dynamic clause of §8. See
  [`conformance/README.md`](conformance/README.md).
- Asset-reference cases go in `conformance/asset-ref/`, not scattered through `parse/reject/`.

## Git

- **`master`, never `main`.**
- One change per commit. Don't batch unrelated work into a "round" commit.
- **No AI attribution.** No `Co-Authored-By` trailers, no "Generated with" lines, anywhere.

## Licence

Dual [Apache-2.0](LICENSE-APACHE) / [MIT](LICENSE-MIT). Contributions are taken under both.
