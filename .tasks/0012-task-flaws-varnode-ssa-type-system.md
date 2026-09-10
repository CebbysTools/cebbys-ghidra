# 0012 — [TASK] Flaws: Varnode/SSA Representation & Type System

## Type
Task

## Status
`todo`

## Parent
0011 (epic) / 0001 (root)

## Objective
Critically assess the `Varnode`/`HighVariable`/`Symbol` trio and the `Datatype` hierarchy for design
flaws, redundancy, and maintainability cost.

## Inputs (read first)
- `.documents/01-review/ssa-varnode-heritage.md` (0004)
- `.documents/01-review/type-system.md` (0005)
- Source itself as needed to verify specific claims: `varnode.hh`, `variable.hh`, `database.hh`,
  `type.hh`, `typeop.hh`.

## Analysis Focus
1. **Datatype overlap/merge candidates**: is there real duplication between e.g. `Varnode` metadata
   and `HighVariable`/`Symbol` metadata? Between `TypeFactory`'s interned types and Ghidra's
   `DataTypeManager`? Name specific classes/fields, not generalities.
2. **God-object risk**: `Funcdata` and `Architecture` — what breaks if you try to split responsibility
   out of them? Is the coupling essential (performance, correctness) or historical?
3. **Type-system expressiveness gaps**: enumerate concretely what today's `Datatype` hierarchy cannot
   represent well (templates/generics, parametric/opaque types, function types with calling-convention
   attributes attached, recursive/self-referential types, const/volatile qualifier fidelity) — this
   feeds requirements task 0017 directly, so be exhaustive here.
3. **Correctness-adjacent risk areas**: merge/cover heuristics that are "best effort" — where does the
   design admit it can silently produce wrong (not just ugly) output?
4. **Improvement options** for each finding: e.g. "introduce a distinct ParametricType with explicit
   instantiation-argument list" vs. "keep struct+opaque-pointer convention", with pros/cons and a
   rough size-of-change estimate (small/medium/large).

## Deliverable
`.documents/02-flaws/varnode-ssa-type-system.md`, structured per the parent epic's required format.

## Acceptance Criteria
- [ ] Document exists, follows Observation/Impact/Options structure, every claim cites file:line or a
      review-doc section.
- [ ] Explicit "datatypes to merge or simplify" subsection with named candidates.
- [ ] Explicit "type system expressiveness gaps" subsection usable as direct input to 0017.

## Dependencies
0004, 0005 (their documents must exist).
