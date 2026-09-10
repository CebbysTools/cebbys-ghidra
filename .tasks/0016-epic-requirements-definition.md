# 0016 — [EPIC] Requirements Definition

## Type
Epic (sub-epic of 0001)

## Status
`blocked` (benefits from 0011's flaw docs but is primarily forward-looking; may start once at least
0012's "type system expressiveness gaps" section exists)

## Parent
0001

## Summary
Corresponds to main goal #3: "Define requirements - Language rich features like templates/generics."
Turns the flaw analysis into forward-looking, testable requirements for the refactored decompiler.
This is the first genuinely *prescriptive* epic (0002/0011 are descriptive/critical; this one says
what the new system must do), so it is written as requirement statements with rationale and, where
possible, acceptance-test sketches — not prose essays.

## Scope
Three requirement domains, corresponding to child tasks below. All requirements must be traceable
either to a flaw in 0011 or to an explicit goal from the user's original request (rich language
features, extensibility).

## Child Tasks
- [ ] 0017 — Requirements: rich language feature support (templates/generics, modern type fidelity)
- [ ] 0018 — Requirements: output fidelity & readability (UX of the generated C)
- [ ] 0019 — Requirements: extensibility, API & integration/testability

## Requirement Format
Each requirement gets a stable ID (`REQ-<domain>-<n>`), a MUST/SHOULD/MAY priority (RFC-2119 style),
a one-line statement, rationale with links back to 0011 flaw entries, and — where feasible — a sketch
of how you'd know it's satisfied (a test or example input/output).

## Deliverables
Child documents under `.documents/03-requirements/`, plus `.documents/03-requirements/00-index.md`
consolidating the full requirements table (ID, priority, one-liner, source doc) for easy reference by
epic 0020.

## Acceptance Criteria
- [ ] All 3 child documents exist with requirements in the specified format.
- [ ] `00-index.md` has one master table of every requirement ID.
- [ ] Every MUST-priority requirement traces to at least one 0011 flaw entry or an explicit statement
      in this epic's own summary (rich language features).
- [ ] At least one concrete worked example (real-ish decompiled function shape) showing current output
      vs. desired output for a templates/generics-adjacent case (e.g. a C++ template-instantiated
      function, or a family of type-punned functions that should collapse to one generic).

## Related Documents
- `.documents/03-requirements/00-index.md`
- Depends on: `.documents/02-flaws/*` (epic 0011)
