# 0018 — [TASK] Requirements: Output Fidelity & Readability

## Type
Task

## Status
`done`

## Parent
0016 (epic) / 0001 (root)

## Objective
Define requirements for the quality of generated C itself — correctness fidelity (never produce C that
misrepresents the binary's behavior) and readability (matches what a skilled human reverse engineer
would write), independent of the templates/generics push.

## Inputs (read first)
- `.documents/02-flaws/printer-output-quality.md` (0014)
- `.documents/02-flaws/rule-engine-block-structuring.md` (0013)
- `.documents/01-review/java-integration-testing.md` (0010), for what "correctness" can even be
  measured against today.

## Requirements to Define (non-exhaustive starting list — expand/justify each)
- Cast-minimality: no more casts than necessary for correctness (readability), but never fewer than
  necessary for correctness (safety) — define this as a testable property, not just prose.
- Goto-minimization guarantees/limits: a stated target (e.g. "no regression vs. current goto rate on
  the existing datatests corpus," with a path to formal measurement).
- Variable naming/merge quality: does not silently over-merge distinct source variables into one
  (correctness) or under-merge (readability) — tie back to 0012's cover/merge findings.
- Deterministic output: same input P-code always produces byte-identical C text (needed for any
  snapshot-testing strategy the refactor adopts).
- Diagnostics: when the decompiler falls back to a "give up" representation (goto soup, raw casts,
  unknown types), it should be possible to know *why* (traceable to a rule/heuristic decision) —
  this is as much a developer-experience requirement as an end-user one.

## Deliverable
`.documents/03-requirements/output-fidelity-ux.md`, REQ-OUT-<n> format per parent epic.

## Acceptance Criteria
- [x] Document exists, REQ-OUT-<n> IDs with priorities, each traced to a 0011 flaw or explicit
      rationale.
- [x] Determinism requirement explicitly stated with a proposed verification method.
- [x] At least one requirement directly addresses diagnosability of "why did the output look like
      this."

## Dependencies
0013, 0014, 0010.
