# 0001 — [EPIC] Fully Refactored C-Code Decompiler

## Type
Epic (root / master)

## Status
`in-progress`

## Parent
None (root epic)

## Summary
Ghidra's decompiler back end (`Ghidra/Features/Decompiler/src/decompile/cpp`) turns a function's
raw P-code into structured, typed pseudo-C. It is a mature, ~230-file, ~150k-line C++ codebase
(`ruleaction.cc` alone is 11k lines) that has accreted for over a decade. This epic covers a full,
disciplined refactor initiative for the **P-code → pseudo-C** pipeline specifically: SSA/varnode
construction, the type system, the `Action`/`Rule` simplification engine, control-flow structuring,
function-signature recovery, expression/cast logic, and the C-language pretty printer. It explicitly
excludes SLEIGH (the disassembler/P-code-generation language and its compiler), the loader/analysis
front end in Java, and processor modules — those are inputs to, or independent consumers of, the
decompiler and are only touched where they are direct dependencies of it.

## Goals
- Produce a complete, accurate, human-readable map of the current decompiler architecture.
- Identify concrete design flaws, redundant/overlapping datatypes, and maintainability hot spots.
- Define explicit requirements for the target design, including rich-language features
  (templates/generics, richer type folding, modern C/C++ idioms) that the current design cannot
  express well.
- Research and select, with an explicit rationale, the best architectural approach for the refactor
  (incremental in-place refactor vs. staged rewrite vs. new IR layer, etc.).
- Only once 1–4 are complete: iteratively implement the most basic decompiler features against the
  new design, then plan the next milestone.

## Non-Goals
- Rewriting SLEIGH, the P-code emulator, the loader/importer subsystem, or processor modules.
- Changing the Ghidra-to-decompiler wire protocol (XML/marshal) unless a selected solution requires it
  and that requirement is captured explicitly in 0020/0023.
- Shipping a production-ready replacement decompiler within this epic — see 0024/0025 for the
  implementation and next-milestone epics, which are intentionally scoped as stubs for now.

## Sub-Epics
- [x] 0002 — Review of current implementation (architecture, components, datatypes)
- [ ] 0011 — Flaw analysis & improvement options
- [ ] 0016 — Requirements definition (incl. rich language features)
- [ ] 0020 — Solution research & selection
- [ ] 0024 — Iterative implementation of basic decompiler features (stub — out of scope this pass)
- [ ] 0025 — Preparation for next milestone (stub — out of scope this pass)

## Working Agreement
- One incrementing ID sequence across all tickets, filename pattern `NNNN-{epic|task}-short-desc.md`.
- Every task ticket must produce at least one Markdown document under `.documents/` and must link back
  to its ticket ID; every document must link the ticket(s) that produced it plus any sibling documents
  it depends on or informs.
- Documents are written for a human reader unfamiliar with the specific source files: explain *what*
  the subsystem does, *why* it's shaped that way historically, *where* the code lives (file:line
  references), and *what's notable/surprising*.
- External prior art must be linked (papers, other decompilers' docs, blog posts) wherever it informs
  a conclusion.

## Acceptance Criteria (for this root epic, current pass)
- [ ] All tickets for sub-epics 0002, 0011, 0016, 0020 exist with clear scope and are resolved.
- [ ] `.documents/` contains a coherent, cross-linked set of documents covering review, flaws,
      requirements, and solution research/selection.
- [ ] `.documents/00-overview/README.md` is a working index into everything produced.
- [ ] A single Architecture Decision Record (0023) states the selected refactor approach.

## Related Documents
- `.documents/00-overview/README.md`
- `.documents/00-overview/glossary.md`
