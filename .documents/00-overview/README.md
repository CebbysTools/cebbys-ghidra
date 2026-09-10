# Decompiler Refactor — Documentation Index

This directory is the human-readable knowledge base for the **Fully Refactored C-Code Decompiler**
initiative (root ticket [`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md)). It
covers Ghidra's decompiler back end — the P-code → pseudo-C pipeline living in
`Ghidra/Features/Decompiler/src/decompile/cpp`.

Every document here is produced by, and links back to, a specific ticket in `.tasks/`. Read the root
epic ticket first if you haven't — it defines scope, goals, and non-goals.

## How to read this

The initiative is organized as four sequential phases (a fifth and sixth exist as stubs for future
work, intentionally not started yet):

1. **Review** — what exists today, precisely and neutrally.
2. **Flaws** — what's wrong with it, and candidate improvement directions (not yet decided).
3. **Requirements** — what the refactored system must do, including rich language features
   (templates/generics) called out explicitly by the initiative's sponsor.
4. **Solution research & selection** — survey of prior art + internal options, ending in one
   explicit, accepted decision (an ADR).
5. *(stub)* Iterative implementation of basic decompiler features — not started.
6. *(stub)* Preparation for the next milestone — not started.

## 1. Review — [`01-review/`](../01-review/)

Epic: [`.tasks/0002`](../../.tasks/0002-epic-review-current-implementation.md)

| Document | Ticket | Subsystem |
|---|---|---|
| [`00-index.md`](../01-review/00-index.md) | 0002 | End-to-end pipeline narrative, ties everything below together |
| [`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) | 0003 | Process model, `Architecture`, wire protocol, extension points |
| [`ssa-varnode-heritage.md`](../01-review/ssa-varnode-heritage.md) | 0004 | `Funcdata`, `Varnode`, `Heritage`, SSA construction, `Cover`/merge |
| [`type-system.md`](../01-review/type-system.md) | 0005 | `Datatype` hierarchy, type propagation, Ghidra type bridge |
| [`action-rule-engine.md`](../01-review/action-rule-engine.md) | 0006 | `Action`/`Rule` simplification engine, ~500-rule catalog |
| [`control-flow-structuring.md`](../01-review/control-flow-structuring.md) | 0007 | `BlockGraph` structuring, jump tables, goto fallback |
| [`signature-calling-conventions.md`](../01-review/signature-calling-conventions.md) | 0008 | `FuncProto`, `ProtoModel`, parameter/ABI recovery |
| [`expression-cast-printer.md`](../01-review/expression-cast-printer.md) | 0009 | `PrintC`/`PrintLanguage`, `CastStrategy`, token emission |
| [`java-integration-testing.md`](../01-review/java-integration-testing.md) | 0010 | Java↔C++ boundary, `datatests`/`unittests` inventory |

## 2. Flaws — [`02-flaws/`](../02-flaws/)

Epic: [`.tasks/0011`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md)

| Document | Ticket | Focus |
|---|---|---|
| [`00-index.md`](../02-flaws/00-index.md) | 0011 | Consolidated, prioritized flaw list |
| [`varnode-ssa-type-system.md`](../02-flaws/varnode-ssa-type-system.md) | 0012 | Datatype overlap/merge candidates, type-system expressiveness gaps |
| [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md) | 0013 | Rule ordering/interaction risk, structuring algorithm limits |
| [`printer-output-quality.md`](../02-flaws/printer-output-quality.md) | 0014 | Cast correctness/readability, printer abstraction quality |
| [`architecture-extensibility.md`](../02-flaws/architecture-extensibility.md) | 0015 | Extension model, test coverage gaps, refactor sequencing risk |

## 3. Requirements — [`03-requirements/`](../03-requirements/)

Epic: [`.tasks/0016`](../../.tasks/0016-epic-requirements-definition.md)

| Document | Ticket | Focus |
|---|---|---|
| [`00-index.md`](../03-requirements/00-index.md) | 0016 | Master requirements table (all REQ-IDs) |
| [`language-feature-support.md`](../03-requirements/language-feature-support.md) | 0017 | REQ-LANG-*: templates/generics, qualifier fidelity, C++ constructs |
| [`output-fidelity-ux.md`](../03-requirements/output-fidelity-ux.md) | 0018 | REQ-OUT-*: cast minimality, determinism, diagnosability |
| [`extensibility-integration-api.md`](../03-requirements/extensibility-integration-api.md) | 0019 | REQ-ARCH-*: wire-protocol compatibility, testability bars |

## 4. Solution Research & Selection — [`04-solution-research/`](../04-solution-research/)

Epic: [`.tasks/0020`](../../.tasks/0020-epic-solution-research-and-selection.md)

| Document | Ticket | Focus |
|---|---|---|
| [`prior-art-external-decompilers.md`](../04-solution-research/prior-art-external-decompilers.md) | 0021 | Other decompilers + academic literature survey |
| [`architecture-options.md`](../04-solution-research/architecture-options.md) | 0022 | ≥3 concrete internal architecture options, compared against requirements/flaws |
| [`decision-record.md`](../04-solution-research/decision-record.md) | 0023 | **The ADR** — selected approach, rationale, consequences, scope hand-off to epics 5/6 |

## 5–6. Future work (stubs)

- [`.tasks/0024`](../../.tasks/0024-epic-iterative-implementation-phase.md) — not started; entry
  criteria is an **Accepted** decision record.
- [`.tasks/0025`](../../.tasks/0025-epic-next-milestone-preparation.md) — not started; depends on 0024.

## Conventions used throughout

- **Ticket IDs**: one incrementing sequence across all of `.tasks/`, `NNNN-{epic|task}-slug.md`.
- **Citations**: every non-obvious claim in a review/flaw/requirements document cites a concrete
  `path/to/file.cc:123`-style location, or another document in this tree.
- **Requirement IDs**: `REQ-<DOMAIN>-<n>` with an RFC-2119 MUST/SHOULD/MAY priority.
- **Diagrams**: Mermaid, inline in the Markdown.
- **Glossary**: shared terminology lives in [`glossary.md`](glossary.md) — add to it rather than
  redefining terms locally.

## Status

See [`.tasks/0001-epic-decompiler-full-refactor.md`](../../.tasks/0001-epic-decompiler-full-refactor.md)
for the live checklist of sub-epics. This README is updated as documents land.
