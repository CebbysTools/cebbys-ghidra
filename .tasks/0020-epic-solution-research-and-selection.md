# 0020 — [EPIC] Solution Research & Selection

## Type
Epic (sub-epic of 0001)

## Status
`done`

## Parent
0001

## Summary
Corresponds to main goal #4: "Research different solutions how we could implement refactor. And
finally select the best solution." Surveys prior art (other decompilers, academic literature on
structuring/type recovery) and internally-generated architecture options, evaluates each against the
requirements from 0016, and produces a single, explicit, justified decision.

## Child Tasks
- [x] 0021 — Research: prior art in other decompilers & academic literature
- [x] 0022 — Research: internal architecture options for this codebase specifically
- [x] 0023 — Decision: Architecture Decision Record selecting the refactor approach

## Deliverables
- `.documents/04-solution-research/prior-art-external-decompilers.md`
- `.documents/04-solution-research/architecture-options.md`
- `.documents/04-solution-research/decision-record.md` (the ADR — this is the single most important
  artifact of tasks 1–4; everything upstream exists to make this decision well-informed)

## Acceptance Criteria
- [x] All 3 child documents exist and are linked from `.documents/00-overview/README.md`.
- [x] The ADR explicitly evaluates at least 3 real options against the 0016 requirements table
      (a comparison matrix, not just prose preference).
- [x] The ADR states what's in scope for epic 0024 (iterative implementation of "most basic
      decompiler features") as a direct consequence of the chosen approach, without actually starting
      that implementation.

## Related Documents
- Depends on: `.documents/02-flaws/*`, `.documents/03-requirements/*`
