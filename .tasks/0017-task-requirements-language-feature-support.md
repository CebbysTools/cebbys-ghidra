# 0017 — [TASK] Requirements: Rich Language Feature Support

## Type
Task

## Status
`todo`

## Parent
0016 (epic) / 0001 (root)

## Objective
Define explicit requirements for what "rich language features" the refactored type system and printer
must support, directly answering the user's call-out of "templates/generics" and extending it to the
full set of gaps found in 0012/0014.

## Inputs (read first)
- `.documents/02-flaws/varnode-ssa-type-system.md` (0012), specifically its expressiveness-gaps section
- `.documents/02-flaws/printer-output-quality.md` (0014)
- External research (use WebSearch/WebFetch): how other decompilers/tools represent generics-like
  constructs recovered from binaries — note that C binaries fundamentally erase template/generic
  information at compile time, so "support" here means: (a) recognizing and re-collapsing
  monomorphized instantiations back into one generic-looking definition where a pattern strongly
  suggests it, and (b) at minimum, representing C++ template instantiations found via existing
  demangled-name/RTTI metadata (Ghidra already demangles names — check `GnuDemangler`/
  `MicrosoftDemangler` features) with proper template-argument syntax instead of flattened mangled-ish
  names. Cite what's realistic vs. speculative.

## Requirements to Define (non-exhaustive starting list — expand/justify each)
- Representing C++ template instantiations with correct template-argument syntax in signatures/casts
  (using demangler-provided info, not template re-derivation from scratch — that's out of scope/
  infeasible from a single binary).
- Detecting families of near-identical monomorphized functions and optionally presenting them as
  "generic-shaped" with a note, vs. leaving them distinct (this MAY be a stretch goal, not a MUST —
  justify the priority).
- Qualifier fidelity: const/volatile/restrict propagation through the type system and printer.
- Function-pointer/calling-convention-attributed types printed correctly (e.g. `__fastcall`, calling
  convention decorations already modeled by `ProtoModel` per 0008 — requirement is to preserve that
  fidelity through any refactor, not regress it).
- Recursive/self-referential and forward-declared composite types.
- Anonymous struct/union/enum handling parity with modern C.
- Namespacing/scoping fidelity for C++-originated symbols in output.

## Deliverable
`.documents/03-requirements/language-feature-support.md`, in the REQ-LANG-<n> format defined by the
parent epic, each requirement with rationale + traceability + example sketch.

## Acceptance Criteria
- [ ] Document exists, uses REQ-LANG-<n> IDs with MUST/SHOULD/MAY priorities.
- [ ] Explicitly distinguishes "feasible from single-binary decompilation" vs. "would require external
      info (PDB/DWARF with template metadata) to ever satisfy" for every generics/templates-related
      requirement — do not overpromise.
- [ ] At least 2 external references (demangler docs, a paper/blog on recovering C++ semantics in
      decompilation) cited with links.
- [ ] At least one before/after worked example.

## Dependencies
0012, 0014.
