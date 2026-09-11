# 0022 — [TASK] Research: Internal Architecture Options

## Type
Task

## Status
`done`

## Parent
0020 (epic) / 0001 (root)

## Objective
Generate and rigorously compare concrete architectural options for *this specific codebase*, informed
by 0011 (flaws), 0016 (requirements), and 0021 (prior art). This is where the plan gets opinionated
about mechanism, without yet making the final call (that's 0023).

## Inputs (read first)
- `.documents/02-flaws/00-index.md` and all 4 flaw docs (0011)
- `.documents/03-requirements/00-index.md` and all 3 requirement docs (0016)
- `.documents/04-solution-research/prior-art-external-decompilers.md` (0021)

## Options to Develop (minimum set — flesh each out, add more if warranted)
1. **In-place incremental refactor**: keep `Funcdata`/`Action`/`Rule` architecture, address flaws
   subsystem-by-subsystem behind characterization tests, evolve the type system in place.
2. **New intermediate layer**: introduce a distinct, richer typed IR/AST between the SSA P-code graph
   and the printer (addressing the "printer talks almost directly to low-level structures" pattern
   found in 0014), while keeping SSA construction/heritage/rules mostly as-is.
3. **Staged replacement with a compatibility shim**: build new subsystem(s) alongside old ones behind
   the existing wire protocol, cut over per-architecture or per-feature-flag, deprecate old code once
   parity is proven on the datatests corpus.
4. (Optional 4th) Any option suggested distinctly by 0021's prior art (e.g. adopting an
   interval-based structuring algorithm as a drop-in replacement for `blockaction.cc`'s approach,
   scoped independently of the broader IR question).

For each option, document: what changes, what stays, estimated blast radius (files/subsystems
touched), how it satisfies/fails each MUST requirement from 0016, how it addresses/doesn't address each
High-impact flaw from 0011, risk, and rough sequencing feasibility for "iterative implementation of
most basic decompiler features" (epic 0024) as a first slice.

## Deliverable
`.documents/04-solution-research/architecture-options.md`, structured as one subsection per option plus
a comparison matrix (options × MUST-requirements × High-impact flaws) at the end.

## Acceptance Criteria
- [x] Document exists, minimum 3 options fully fleshed out per the required fields.
- [x] Comparison matrix present and complete (no blank cells — explicit "N/A" with reason if genuinely
      not applicable).
- [x] Each option's write-up explicitly says what a first "most basic decompiler features" slice
      (epic 0024) would look like under that option.

## Dependencies
0011, 0016, 0021.
