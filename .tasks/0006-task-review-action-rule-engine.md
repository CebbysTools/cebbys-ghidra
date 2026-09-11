# 0006 — [TASK] Action/Rule Simplification Engine Review

## Type
Task

## Status
`done`

## Parent
0002 (epic) / 0001 (root)

## Objective
Document the transformation engine that simplifies the raw SSA P-code graph into something
structurally close to source-level C: the `Action`/`Rule` framework and the ~11,000-line catalog of
individual rules.

## Scope / Primary Files
- `action.cc/hh` (the `Action`/`ActionGroup`/`ActionPool` framework — how rules are scheduled/applied
  to fixed points)
- `coreaction.cc/hh` (the standard action pipeline definition — the ordered list of passes)
- `ruleaction.cc/hh` (the large catalog of individual `Rule` subclasses — sample broadly, don't read
  all 11k lines; identify categories/patterns by grepping class names and doc comments)
- `condexe.cc/hh` (conditional-execution/branch simplification)
- `subflow.cc/hh` (subvariable flow — narrowing operations to smaller types)
- `rulecompile.cc/hh` (the mini pattern-matching DSL used to express some rules — note this is a
  second, informal "language" inside the codebase)

## Deliverables
`.documents/01-review/action-rule-engine.md` covering:
1. The scheduling model: `Action` fixed-point iteration, `ActionPool`, how rules opt into which
   opcodes they match (`getOpList`), and how ordering/dependency between rules is (or isn't) made
   explicit today.
2. A taxonomy of what the ~500+ `Rule` subclasses in `ruleaction.cc` actually do — group into
   categories (arithmetic simplification, pointer arithmetic recovery, cast insertion/removal,
   control-flow-adjacent cleanup, ABI-specific quirks, etc.) with representative examples and
   file:line per category, not an exhaustive listing.
3. The `rulecompile.cc` pattern DSL: what it is, why it exists, how much of the ruleset uses it vs.
   hand-written C++ visitors.
4. Known extension/debugging affordances (e.g. `-Action` tracing/breakpoints in the interface layer)
   and what's missing (e.g. no rule-level unit isolation, unclear rule interaction/ordering
   guarantees).

## Acceptance Criteria
- [x] Document exists, file:line cited (including at least 15 distinct representative `Rule`
      subclasses with their purpose — 26 cited). Linking from `00-index.md` is owned by the
      orchestrator (ticket 0002), not this task.
- [x] Taxonomy table of rule categories with counts (approximate is fine, state methodology).
- [x] Explicit note on how rule ordering/interaction is currently reasoned about (or isn't).

## Dependencies
None — parallel with 0003–0005, 0007–0010.
