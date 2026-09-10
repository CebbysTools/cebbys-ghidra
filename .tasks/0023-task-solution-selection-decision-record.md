# 0023 — [TASK] Decision: Architecture Decision Record for the Refactor Approach

## Type
Task

## Status
`todo`

## Parent
0020 (epic) / 0001 (root)

## Objective
Make and record the final call: which architecture option from 0022 (or a hybrid) is selected, with
explicit rationale, consequences, and what it means for the next two (currently out-of-scope) epics.

## Inputs (read first)
- `.documents/04-solution-research/architecture-options.md` (0022) — required
- `.documents/03-requirements/00-index.md` (0016) — required
- `.documents/02-flaws/00-index.md` (0011) — required

## Deliverable
`.documents/04-solution-research/decision-record.md`, in standard ADR format:
- **Status**: Proposed/Accepted
- **Context**: 2-3 paragraph summary of the problem (link back rather than repeat 0011/0016 in full)
- **Decision**: the selected option (or explicit hybrid), stated in one clear paragraph
- **Rationale**: why this beat the alternatives, referencing the 0022 comparison matrix directly
- **Consequences**: what becomes easier, what becomes harder, what requirements from 0016 are
  explicitly deferred (not dropped) and why
- **Scope for Epic 0024** ("iterative implementation of most basic decompiler features"): a concrete,
  ordered list of the first 3-6 implementation slices this decision implies, each small enough to be
  its own future ticket — but DO NOT create those tickets yet; that happens when 0024 is actually
  started, per this epic's own non-goals.
- **Scope note for Epic 0025** ("next milestone prep"): one paragraph on what "done" for the basic-
  feature milestone should mean, to set up that epic later.

## Acceptance Criteria
- [ ] Document exists in ADR format with all sections above.
- [ ] Decision explicitly references the 0022 comparison matrix (not a fresh, unlinked judgment call).
- [ ] Deferred requirements are listed explicitly with reasons, not silently dropped.
- [ ] Epic 0024 scope list exists as guidance text inside this doc, but no new ticket files are created
      by this task — ticket creation for 0024 is future work.
- [ ] Update `.tasks/0001-epic-decompiler-full-refactor.md` and `.tasks/0020-epic-solution-research-
      and-selection.md` statuses to reflect completion once this lands.

## Dependencies
0022, and transitively everything in 0011/0016.
