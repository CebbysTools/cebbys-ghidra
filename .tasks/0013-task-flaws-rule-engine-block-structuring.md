# 0013 — [TASK] Flaws: Action/Rule Engine & Control-Flow Structuring

## Type
Task

## Status
`done`

## Parent
0011 (epic) / 0001 (root)

## Objective
Critically assess the simplification-rule engine and the CFG-structuring algorithm for design flaws.

## Inputs (read first)
- `.documents/01-review/action-rule-engine.md` (0006)
- `.documents/01-review/control-flow-structuring.md` (0007)
- Source as needed: `action.hh`, sampled sections of `ruleaction.cc`, `block.hh`, `blockaction.hh`.

## Analysis Focus
1. **Rule interaction/ordering risk**: with ~500 independently-authored `Rule`s converging via a
   fixed-point loop, what evidence exists (comments, ordering hacks, "don't apply if X already ran")
   that rule interaction is fragile? Concrete examples.
2. **Testability of individual rules**: can a single `Rule` be unit-tested in isolation today? If not,
   why not, and what would it take?
3. **The `rulecompile.cc` pattern DSL**: is having two ways to express rules (hand-written C++ visitor
   vs. pattern DSL) a maintenance liability? Should the refactor pick one?
4. **Structuring algorithm limits**: concrete classes of input CFG (irreducible loops, certain
   optimizer-mangled switch patterns) where structuring is known-weak, per source comments or observed
   goto-fallback triggers from 0007.
5. **Improvement options**: e.g. explicit rule-dependency declarations, a rule-level test harness,
   separating "correctness-required" rules from "readability" rules so they can be reasoned about
   differently, alternative structuring algorithms from literature (name them; deep comparison belongs
   in 0021).

## Deliverable
`.documents/02-flaws/rule-engine-block-structuring.md`, structured per the parent epic's format.

## Acceptance Criteria
- [x] Document exists, follows required structure, cites file:line/review-doc throughout.
- [x] At least 5 concrete examples of fragile rule ordering/interaction, each with a source citation.
- [x] Explicit recommendation (not yet a decision) on whether to keep, replace, or dual-track the
      `rulecompile.cc` DSL.

## Dependencies
0006, 0007.
