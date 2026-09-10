# 0010 — [TASK] Java-Side Integration & Test Infrastructure Review

## Type
Task

## Status
`done`

## Parent
0002 (epic) / 0001 (root)

## Objective
Document how the C++ decompiler process is driven from Ghidra's Java front end, and what testing
infrastructure exists for the P-code-to-C pipeline (this scopes what a refactor can lean on for
regression safety).

## Scope / Primary Files
- Java: `ghidra/app/decompiler/DecompInterface.java`, `DecompileProcess*.java`,
  `DecompileCallback.java`, `DecompileResults.java`, `DecompiledFunction.java`,
  `ghidra/app/decompiler/component/**` (token/GUI rendering consumers of the C++ output — high level
  only, this is UI not core logic), `ghidra/program/model/pcode/ClangToken*.java` (if present under
  this feature or under SoftwareModeling — note actual location).
- C++ test infra: `src/decompile/datatests/**` (89 files — sample structure/format, don't read all),
  `src/decompile/unittests/**` (7 files), `test.cc/hh`, `testfunction.cc/hh`, `consolemain.cc`.
- Also check `Ghidra/Features/Decompiler/src/test.slow/java/**` for higher-level Java-driven
  regression tests.

## Deliverables
`.documents/01-review/java-integration-testing.md` covering:
1. The Java↔C++ boundary: process lifecycle, request/response marshaling (link to 0003's wire-protocol
   section rather than re-deriving it), error handling/timeouts, versioning (how Java and C++ agree on
   protocol compatibility).
2. What the Java layer does with decompiler output beyond display (e.g. signature commit-back,
   PDB/DWARF interplay) at a level sufficient to know what a refactor must keep compatible.
3. Test infrastructure inventory: what `datatests` actually validates (format of one sample test file,
   explained), what `unittests` covers, and — critically — **what is NOT covered** (e.g. is there
   any test that pins exact C output text for known binaries? Any performance/regression benchmark?).
4. A concrete assessment: "if we refactor X, how would we know we didn't regress it?" for each major
   subsystem named in 0004–0009, based on actual test coverage found.

## Acceptance Criteria
- [x] Document exists, linked from index, file:line cited throughout.
- [x] Explicit list of test coverage gaps relevant to a refactor (this directly feeds 0015/0019).
- [x] At least one concrete `datatests` example walked through end-to-end (input → assertion).

## Dependencies
None — parallel with 0003–0009.
