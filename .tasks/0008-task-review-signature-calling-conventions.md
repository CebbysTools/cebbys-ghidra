# 0008 — [TASK] Function Signature & Calling-Convention Recovery Review

## Type
Task

## Status
`todo`

## Parent
0002 (epic) / 0001 (root)

## Objective
Document how the decompiler recovers function prototypes (parameter count/types/storage, return
value, varargs) and models calling conventions across architectures.

## Scope / Primary Files
- `fspec.cc/hh` (`FuncProto`, `ProtoParameter`, `ProtoModel` — the prototype/ABI model)
- `modelrules.cc/hh` (declarative rules for parameter-storage assignment per ABI)
- `paramid.cc/hh` (parameter-ID analysis, used for signature commit-back to Ghidra)
- `override.cc/hh` (user/analysis overrides of recovered signatures, e.g. call-fixup, prototype
  override)
- `userop.cc/hh` (CALLOTHER/user-defined-op handling, relevant to injected calling-convention
  behavior)
- Related `.pspec`/`.cspec` XML concepts (just enough to explain where `ProtoModel` data originates —
  see `Ghidra/Features/Decompiler/src/main/doc/cspec.xml` for the spec format reference)

## Deliverables
`.documents/01-review/signature-calling-conventions.md` covering:
1. `ProtoModel`/`FuncProto` relationship: static ABI description (from `.cspec`) vs. per-function
   recovered prototype, and how they interact with `ModelRule`s for storage assignment.
2. The parameter-recovery pipeline: from raw input `Varnode`s touched at a call site, to committed
   `ProtoParameter`s, including heuristics for "trimming" unused/spurious parameters.
3. Varargs detection, and multi-return/structure-return (hidden pointer) handling.
4. How overrides (`override.cc`) let analysis or the user short-circuit recovery, and what that implies
   about trust boundaries between recovered vs. asserted signatures.
5. Cross-architecture concerns: what's generic vs. what's necessarily ABI-specific, as input for
   "what must remain pluggable" in the requirements epic.

## Acceptance Criteria
- [ ] Document exists, linked from index, file:line cited throughout.
- [ ] Clear statement of the data flow from `.cspec` ProtoModel definition to a concrete recovered
      `FuncProto` for one function.
- [ ] Section listing what's config-driven (XML) vs. hardcoded in C++, since that's a key refactor
      lever.

## Dependencies
None — parallel with 0003–0007, 0009–0010.
