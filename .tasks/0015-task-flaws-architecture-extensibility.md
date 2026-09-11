# 0015 — [TASK] Flaws: Overall Architecture, Extensibility & API/Testability

## Type
Task

## Status
`done`

## Parent
0011 (epic) / 0001 (root)

## Objective
Step back from individual subsystems and assess the decompiler as a whole: process architecture,
extension model, API surface, build/test practices, and how well the codebase supports safe,
incremental change (which is exactly what this refactor needs to be able to do).

## Inputs (read first)
- `.documents/01-review/architecture-pipeline-overview.md` (0003)
- `.documents/01-review/java-integration-testing.md` (0010)
- `.documents/01-review/signature-calling-conventions.md` (0008) — for the config-vs-hardcoded
  findings
- All other 0002 review docs at least skimmed for cross-cutting patterns

## Analysis Focus
1. **Extension model**: `Capability`-based self-registration — does it actually enable third-party
   extension today (new architectures, new output languages, new rules), or is it mostly used
   internally? What's missing for a plugin author?
2. **Test coverage gaps** (pull forward from 0010): which subsystems have essentially no regression
   protection, and what does that imply about *sequencing* risk for the refactor (i.e. don't refactor
   an untested subsystem first without adding characterization tests).
3. **API/ABI stability**: what's the practical contract between the C++ decompiler and its Java caller
   today (wire format via `marshal.cc`)? How much does a refactor's internal-only changes risk breaking
   vs. changes that touch the wire format?
4. **Build/tooling**: anything notable about how this C++ code is built/tested (gradle integration,
   `consolemain`/interactive test driver) that constrains how a refactor could be validated
   incrementally (e.g. can you build/run just the decompiler binary standalone for fast iteration?).
5. **Improvement options**: e.g. characterization-test generation strategy before touching a subsystem,
   a stable internal-module-boundary proposal (which subsystems should become independently
   buildable/testable units), API surface a refactor must not break.

## Deliverable
`.documents/02-flaws/architecture-extensibility.md`, structured per the parent epic's format, plus a
dedicated "Refactor Sequencing Risk" table ranking subsystems by (test coverage) x (coupling) so later
epics can use it directly.

## Acceptance Criteria
- [x] Document exists, follows required structure, cites file:line/review-doc throughout.
- [x] Refactor Sequencing Risk table present, covering at minimum the 6 subsystems reviewed in 0002.
- [x] Explicit answer to "can the decompiler be built/run standalone for fast local iteration" with
      evidence.

## Dependencies
0003, 0008, 0010 (and awareness of all 0002 docs).
