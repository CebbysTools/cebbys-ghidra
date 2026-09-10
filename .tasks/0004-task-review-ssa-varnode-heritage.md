# 0004 — [TASK] SSA / Varnode / Heritage Construction Review

## Type
Task

## Status
`done`

## Parent
0002 (epic) / 0001 (root)

## Objective
Document how raw P-code operations and address-space storage locations become the SSA-form varnode
graph that the rest of the decompiler operates on.

## Scope / Primary Files
- `funcdata.cc/hh`, `funcdata_op.cc`, `funcdata_varnode.cc`, `funcdata_block.cc`
- `varnode.cc/hh`, `variable.cc/hh`, `op.cc/hh`, `pcoderaw.cc/hh`
- `heritage.cc/hh` (the SSA "heritage" / phi-node insertion pass)
- `database.cc/hh` (symbol/scope database), `varmap.cc/hh`
- `cover.cc/hh` (live-range/coverage tracking), `merge.cc/hh` (variable merging for output)
- `dynamic.cc/hh`, `prefersplit.cc/hh`, `constseq.cc/hh`

## Deliverables
`.documents/01-review/ssa-varnode-heritage.md` covering:
1. Core object model: `Varnode` vs `VarnodeData` vs `Symbol`/`HighVariable` — what each represents and
   how they relate (this trio is a common source of confusion; document it precisely with a diagram).
2. The heritage algorithm: how `Heritage` decides which storage locations need SSA form, phi-node
   (`MULTIEQUAL`) placement, and how this interacts with address-space aliasing/`LOAD`/`STORE`.
3. `Funcdata`'s role as the per-function "god object" — what it owns, why (historically), and what
   that centralization costs in coupling.
4. Coverage (`Cover`) and merge: how SSA varnodes get grouped back into source-level variables
   (`HighVariable`) for printing, and where that can go wrong (merge conflicts, "OK to merge" rules).
5. A short glossary of terms (varnode, PcodeOp, HighVariable, Symbol, heritage, cover, merge) for
   reuse by `.documents/00-overview/glossary.md`.

## Acceptance Criteria
- [ ] Document exists, is linked from the index, cites file:line for each structure. (Document exists
      at `.documents/01-review/ssa-varnode-heritage.md` and cites concrete `file:line` locations
      throughout; linking from `.documents/01-review/00-index.md` is owned by the orchestrator, not
      this task, and is left unchecked here pending that.)
- [x] Includes at least one diagram (class relationship or data-flow) in Mermaid.
- [x] Explicitly flags any place object lifetime/ownership is unclear or documented only by
      convention (e.g. raw pointers into arena-allocated pools).

## Dependencies
None — parallel with 0003, 0005–0010.
