# 0019 — [TASK] Requirements: Extensibility, API & Integration/Testability

## Type
Task

## Status
`todo`

## Parent
0016 (epic) / 0001 (root)

## Objective
Define requirements for how the refactored decompiler must be structured so it can evolve safely —
module boundaries, testability, and compatibility with the existing Java front end and wire protocol.

## Inputs (read first)
- `.documents/02-flaws/architecture-extensibility.md` (0015)
- `.documents/01-review/architecture-pipeline-overview.md` (0003)
- `.documents/01-review/java-integration-testing.md` (0010)

## Requirements to Define (non-exhaustive starting list — expand/justify each)
- Wire-protocol compatibility: refactored internals MUST NOT require simultaneous changes to the Java
  front end unless a requirement explicitly calls for a protocol version bump (state the bar for when
  that's acceptable).
- Subsystem boundaries MUST be independently unit-testable (tie directly to 0015's Refactor Sequencing
  Risk table — this requirement exists specifically to fix what that table calls out).
- A single `Rule` MUST be testable in isolation with a synthetic P-code snippet, without invoking the
  full pipeline (directly answers 0013's testability finding).
- Characterization-test coverage MUST reach an explicit bar (e.g. "N% of `datatests` corpus produces
  byte-identical output before/after any internal refactor step") before a subsystem identified as
  high-risk in 0015 may be touched.
- Extension model MUST remain (or become) usable by out-of-tree consumers for at least: new
  architectures/ProtoModels, new output languages (PrintLanguage subclasses), and new Rules —
  matching or improving on today's `Capability` mechanism per 0015's findings.
- Standalone/local build-and-run loop for the decompiler binary MUST be preserved or improved (ties to
  0015's build/tooling findings) to keep refactor iteration fast.

## Deliverable
`.documents/03-requirements/extensibility-integration-api.md`, REQ-ARCH-<n> format per parent epic.

## Acceptance Criteria
- [ ] Document exists, REQ-ARCH-<n> IDs with priorities, each traced to a 0011/0015 flaw.
- [ ] Explicit wire-protocol compatibility bar stated.
- [ ] Explicit characterization-test coverage bar stated (a number or concrete method, not just
      "good coverage").

## Dependencies
0015, 0003, 0010.
