# Requirements Index — Master Requirements Table

Epic: [`.tasks/0016`](../../.tasks/0016-epic-requirements-definition.md) · Root:
[`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md)

Every requirement below is `REQ-<DOMAIN>-<n>` with an RFC-2119 priority (MUST/SHOULD/MAY) and traces
to a specific finding in [`../02-flaws/00-index.md`](../02-flaws/00-index.md). Read the linked detail
document for full rationale and acceptance-test sketches — this index is for lookup and cross-domain
comparison, which is exactly what epic [`0020`](../../.tasks/0020-epic-solution-research-and-selection.md)
needs.

## The three requirement documents

| Document | Ticket | Domain | Count |
|---|---|---|---|
| [`language-feature-support.md`](language-feature-support.md) | 0017 | REQ-LANG — templates/generics, qualifier fidelity, C++ constructs | 13 |
| [`output-fidelity-ux.md`](output-fidelity-ux.md) | 0018 | REQ-OUT — cast correctness/readability, determinism, diagnosability | 8 |
| [`extensibility-integration-api.md`](extensibility-integration-api.md) | 0019 | REQ-ARCH — wire protocol, testability, extension model, tooling | 15 (incl. REQ-ARCH-8a) |

## REQ-LANG — Rich Language Feature Support ([detail](language-feature-support.md), ticket 0017)

| ID | Priority | Statement | Traces to |
|---|---|---|---|
| REQ-LANG-1 | MUST | Print C++ template instantiations with correct `<...>` syntax using existing demangler metadata | [printer-output-quality.md](../02-flaws/printer-output-quality.md) Flaw 5 |
| REQ-LANG-2 | SHOULD | First-class `Datatype`-level template/generic representation (not just printer-level text) | [varnode-ssa-type-system.md](../02-flaws/varnode-ssa-type-system.md) §3.1 |
| REQ-LANG-3 | MAY | Detect and group monomorphized-function families as generic-shaped (explicit stretch, overclaiming guardrail) | §3.1 (extension) |
| REQ-LANG-4 | MUST | Print `volatile` as a keyword (currently tracked but never emitted) | [printer-output-quality.md](../02-flaws/printer-output-quality.md) Flaw 4 |
| REQ-LANG-5 | MUST | Composable `const`/`volatile`/`restrict` qualifiers on `Datatype` | §3.2 |
| REQ-LANG-6 | MUST | No regression of existing `ProtoModel` calling-convention fidelity during refactor | [signature-calling-conventions.md](../01-review/signature-calling-conventions.md) |
| REQ-LANG-7 | SHOULD | Standalone calling-convention type attribute (unconfirmed need — verify before building) | §3.5 |
| REQ-LANG-8 | SHOULD | Member-function-pointer-with-context type | §3.3 |
| REQ-LANG-9 | MUST | Recursive/forward-declared composite type support preserved/improved | type-system review |
| REQ-LANG-10 | SHOULD | Anonymous struct/union/enum parity with modern C | type-system review |
| REQ-LANG-11 | MUST/SHOULD | Namespace/scoping fidelity for C++ symbols (MUST at symbol level, SHOULD extended to type names) | type-system review |
| REQ-LANG-12 | MAY | Union-level bitfields | §3.4 |
| REQ-LANG-13 | MAY | Packed/explicit-alignment-override representation | §3.6 |

**Key grounding finding**: Ghidra's own `GnuDemangler` already parses C++ template arguments into a
structured model (`DemangledTemplate`), but `DemangledTemplate.getDataType()` explicitly throws
`UnsupportedOperationException` today — REQ-LANG-1/2 are about *plumbing already-parsed data through*,
not re-deriving templates from raw bytes (which is infeasible from a single binary and is stated as
out of scope throughout the detail doc).

## REQ-OUT — Output Fidelity & UX ([detail](output-fidelity-ux.md), ticket 0018)

| ID | Priority | Statement | Traces to |
|---|---|---|---|
| REQ-OUT-1 | MUST | Cast safety floor: no correctness-required cast may be omitted (operationally defined) | [printer-output-quality.md](../02-flaws/printer-output-quality.md) Flaw 1 |
| REQ-OUT-2 | SHOULD | Cast readability ceiling: no cast beyond the safety floor or a closed idiom-exception list | Flaw 1 |
| REQ-OUT-3 | SHOULD | Goto-minimization: no regression vs. an instrumented `datatests` baseline rate | [rule-engine-block-structuring.md](../02-flaws/rule-engine-block-structuring.md) Flaw 4 |
| REQ-OUT-4 | MUST | Variable-merge safety floor: no silent over-merging of distinct source variables | [varnode-ssa-type-system.md](../02-flaws/varnode-ssa-type-system.md) §4.1 |
| REQ-OUT-5 | SHOULD | Merge readability ceiling: don't under-merge/fragment (via precision work, not relaxing REQ-OUT-4) | §4.2 |
| REQ-OUT-6 | MUST | Determinism: same P-code in → byte-identical C text out, always | [java-integration-testing.md](../01-review/java-integration-testing.md) §3.1 |
| REQ-OUT-7 | MUST | Diagnosability: every fallback (goto/overflow-loop/raw-cast/unknown-type) traceable to the rule/heuristic decision behind it | [rule-engine-block-structuring.md](../02-flaws/rule-engine-block-structuring.md) Flaws 1, 5 |
| REQ-OUT-8 | SHOULD | Qualifier fidelity in printed output (overlaps REQ-LANG-4/5, filed from the output-quality angle) | Flaw 4 |

**Determinism verification method proposed**: extend `datatests`/`FunctionTestCollection` with a new
full-text golden-snapshot assertion type, plus a repeated-run (N≥3, fresh process) byte-diff check.
This is the same mechanism REQ-ARCH-6/7 (below) formalize as the characterization-test bar.

## REQ-ARCH — Extensibility, API & Integration/Testability ([detail](extensibility-integration-api.md), ticket 0019)

| ID | Priority | Statement | Traces to |
|---|---|---|---|
| REQ-ARCH-1 | MUST | Explicit wire-protocol compatibility bar (internals refactor MUST NOT force simultaneous Java changes, with 4 stated exceptions for when a version bump is acceptable) | [architecture-extensibility.md](../02-flaws/architecture-extensibility.md) Flaw 3 |
| REQ-ARCH-2 | MUST | Real `registerProgram` protocol version handshake (today only the signature-module version is checked) | Flaw 3 |
| REQ-ARCH-3 | SHOULD | Single-sourced or CI-cross-checked `ElementId`/`AttributeId` tables across Java/C++ | Flaw 3 |
| REQ-ARCH-4 | MUST | Independent unit-testability for Rank 1-4 subsystems per the Refactor Sequencing Risk table | Flaw 2 |
| REQ-ARCH-5 | MUST | A single `Rule` testable via synthetic P-code + minimal `Funcdata` | [rule-engine-block-structuring.md](../02-flaws/rule-engine-block-structuring.md) Flaw 2 |
| REQ-ARCH-6 | MUST | Characterization-test coverage bar: 100% of the 89-file `datatests` corpus byte-identical before/after any Rank 1-3 change | [architecture-extensibility.md](../02-flaws/architecture-extensibility.md) Flaw 2 |
| REQ-ARCH-7 | MUST | The golden-snapshot harness implementing REQ-ARCH-6 (shared mechanism with REQ-OUT-6) | Flaw 2 |
| REQ-ARCH-8 / 8a | MUST / SHOULD | Unify Rule/Action registration on `CapabilityPoint` (floor); a `ModelRule`-style declarative layer is a later, explicitly deferred SHOULD | Flaw 1 |
| REQ-ARCH-9–11 | MUST/SHOULD | Preserve architecture/ProtoModel and output-language extensibility; route `ArchitectureGhidra` through capability discovery | Flaw 1 |
| REQ-ARCH-12 | SHOULD | Retire the dead `rulecompile.cc` DSL rather than dual-track it | [rule-engine-block-structuring.md](../02-flaws/rule-engine-block-structuring.md) Flaw 3 |
| REQ-ARCH-13 | MUST | Preserve **and document** the console-driver standalone build/run loop | [architecture-extensibility.md](../02-flaws/architecture-extensibility.md) Flaw 4 |
| REQ-ARCH-14 | SHOULD | Wire `make test` into CI | Flaw 4 |

## Cross-domain notes for epic 0020

- REQ-OUT-6 and REQ-ARCH-6/7 both point at the **same missing piece of infrastructure**: a
  golden/snapshot output-comparison harness. Any selected solution in ticket 0022 should treat building
  this harness as a near-zero-regret first step regardless of which architecture option is chosen —
  it's required by both the output-quality and the testability requirement sets independently.
- REQ-ARCH-8/8a (unify extension registration) and REQ-LANG-1/2 (template representation) are the two
  requirements most likely to force a real design choice rather than an incremental fix — flag these
  specifically when comparing options in ticket 0022.
- REQ-LANG-6 (no calling-convention regression) and REQ-ARCH-1 (wire-protocol compatibility) are the
  two explicit **non-regression** guardrails threaded through every other requirement; any solution
  option that can't satisfy both should be disqualified early in 0022 rather than carried through the
  full comparison matrix.

## Status

Epic 0016 is **complete** — all 3 child tasks (0017–0019) are `done`. See
[`.tasks/0016-epic-requirements-definition.md`](../../.tasks/0016-epic-requirements-definition.md).
