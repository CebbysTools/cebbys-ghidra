# 0005 — [TASK] Type System Review

## Type
Task

## Status
`todo`

## Parent
0002 (epic) / 0001 (root)

## Objective
Document the decompiler's C-flavored type system: the `Datatype` class hierarchy, how types are
recovered/propagated, and how composite/union/function types and the constant pool are represented.

## Scope / Primary Files
- `type.cc/hh` (the core `Datatype` hierarchy — `TypeBase`, `TypePointer`, `TypeArray`, `TypeStruct`,
  `TypeUnion`, `TypeEnum`, `TypeCode`, etc.)
- `typeop.cc/hh` (per-opcode type-propagation/behavior rules — `TypeOp` hierarchy)
- `unionresolve.cc/hh` (union field resolution during propagation)
- `typegrp_ghidra.cc/hh` (bridge to Ghidra's DB-backed `DataTypeManager`)
- `signature.cc/hh`, `signature_ghidra.cc/hh` (function signature/prototype typing)
- `cpool.cc/hh`, `cpool_ghidra.cc/hh` (constant pool, relevant for Java bytecode-origin types)
- `bitfield.cc/hh`

## Deliverables
`.documents/01-review/type-system.md` covering:
1. `Datatype` class hierarchy diagram (Mermaid classDiagram), including which types are "core" vs
   dynamically registered, and how equality/hashing works (types are interned — explain the id/name
   scheme and why that matters for correctness).
2. Type propagation: how a `Varnode`'s type is inferred/refined across `PcodeOp`s via `TypeOp`
   subclasses, including confidence/locking (`typelock`) semantics.
3. The Ghidra ↔ decompiler type bridge (`TypeFactory` vs Ghidra's `DataTypeManager`) — two type
   systems that must stay in sync; document the sync mechanism and its failure modes.
4. Union/struct field resolution, and known limitations (e.g. no real template/generic support, no
   first-class function-pointer-with-context type, bitfield handling).
5. Explicitly answer: which of these datatypes look like they overlap/duplicate responsibility, as
   raw input for epic 0011 (flaws). Just list candidates here — deep analysis belongs to 0012.

## Acceptance Criteria
- [ ] Document exists, linked from index, file:line cited throughout.
- [ ] Class diagram present and accurate (verified against the header, not guessed).
- [ ] Candidate list of overlapping/duplicated type concepts included as a dedicated section.

## Dependencies
None — parallel with 0003, 0004, 0006–0010.
