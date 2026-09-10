# 0007 — [TASK] Control-Flow Structuring Review

## Type
Task

## Status
`todo`

## Parent
0002 (epic) / 0001 (root)

## Objective
Document how the decompiler turns an arbitrary (possibly irreducible) control-flow graph of basic
blocks into structured C control flow (if/else, while/for/do, switch, and minimal goto fallback).

## Scope / Primary Files
- `block.cc/hh` (the `BlockGraph`/`FlowBlock` hierarchy of structured-block types —
  `BlockIf`, `BlockWhileDo`, `BlockDoWhile`, `BlockSwitch`, `BlockList`, `BlockCondition`, etc.)
- `blockaction.cc/hh` (the structuring algorithm — collapse rules that build the block hierarchy from
  the raw CFG, loop/DAG detection, goto placement for irreducible/unstructurable regions)
- `jumptable.cc/hh` (switch/jump-table recovery — a substantial, mostly-independent subsystem)
- `graph.cc/hh` (generic graph utilities used by the above, if applicable)

## Deliverables
`.documents/01-review/control-flow-structuring.md` covering:
1. The structuring algorithm at a conceptual level (this is a known hard problem in decompilation —
   summarize the approach used, e.g. interval-based/condition-based reduction) with references to any
   in-source comments describing the theory.
2. `FlowBlock` hierarchy diagram (Mermaid) and what each structured-block subtype represents in
   generated C.
3. How irreducible control flow is handled — when the algorithm gives up structuring and falls back to
   `goto`, and how that's decided.
4. Jump-table/switch recovery: how case values and fallthrough/default are determined from P-code,
   and its relationship to the type system (enum recovery) and Action/Rule engine.
5. Known hard cases noted in source comments (loop unswitching artifacts, compiler-specific patterns
   like Duff's device, etc.).

## Acceptance Criteria
- [ ] Document exists, linked from index, file:line cited throughout.
- [ ] `FlowBlock` hierarchy diagram present.
- [ ] Explicit description of the goto-fallback trigger condition, quoting/paraphrasing source
      comments where they exist.

## Dependencies
None — parallel with 0003–0006, 0008–0010. Benefits from (but does not require) 0006's rule-engine
findings, since structuring is itself implemented as a family of actions.
