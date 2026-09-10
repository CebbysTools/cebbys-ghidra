# 0011 — [EPIC] Flaw Analysis & Improvement Options

## Type
Epic (sub-epic of 0001)

## Status
`blocked` (waiting on 0002 review docs to land before deep analysis; may start once at least the
directly-relevant review document(s) exist rather than waiting for all of 0002)

## Parent
0001

## Summary
Corresponds to main goal #2: "Analyze flaws with the current design, options to improve, datatypes to
merge or anything else." Takes the descriptive output of epic 0002 and turns it into a critical,
opinionated assessment: what's actually wrong, what's merely old-but-fine, and what concrete
improvement options exist — without yet committing to one (that commitment happens in 0020/0023).

## Scope
Same subsystem split as 0002, regrouped into 4 analysis tasks (coarser than review, since flaw
analysis benefits from cross-subsystem comparison rather than strict per-file isolation).

## Child Tasks
- [ ] 0012 — Flaws: varnode/SSA representation & type system
- [ ] 0013 — Flaws: Action/Rule engine & control-flow structuring
- [ ] 0014 — Flaws: expression/cast logic & printer output quality
- [ ] 0015 — Flaws: overall architecture, extensibility & API/testability

## Deliverables
Each child task produces a document under `.documents/02-flaws/`, each following the same structure:
**Observation → Why it's a problem (concretely) → Severity/impact → Improvement options (2-4,
pros/cons) → Open questions for the requirements/solution epics.** This epic's own
`.documents/02-flaws/00-index.md` synthesizes a single prioritized flaw list across all four.

## Acceptance Criteria
- [ ] All 4 child documents exist and follow the required structure.
- [ ] `.documents/02-flaws/00-index.md` contains one consolidated, prioritized (High/Med/Low impact)
      table of flaws with links into the detail docs.
- [ ] Every flaw entry cites the specific review-doc section and file:line it stems from — no
      unsourced claims.
- [ ] At least one entry explicitly addresses "datatypes to merge" per the user's stated goal.

## Related Documents
- `.documents/02-flaws/00-index.md`
- Depends on: `.documents/01-review/*` (epic 0002)
