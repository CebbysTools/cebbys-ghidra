# 0002 — [EPIC] Review of Current Decompiler Implementation

## Type
Epic (sub-epic of 0001)

## Status
`done`

## Parent
0001

## Summary
Corresponds to main goal #1: "Perform full review of the current implementation, all components,
services, datatypes, architecture designs." Produces the authoritative map of how P-code becomes
pseudo-C today, split by subsystem so it can be reviewed in parallel by multiple agents without
file-ownership conflicts. This is read-only, exploratory work — no source changes.

## Scope
`Ghidra/Features/Decompiler/src/decompile/cpp/**` (the `libdecomp` C++ engine) and, where it is the
direct Java-side counterpart, `Ghidra/Features/Decompiler/src/main/java/**`. Each child task below
owns a disjoint file set to avoid overlapping edits to `.documents/`.

- [x] 0003 — Architecture & pipeline overview (Architecture, Capability, libdecomp, wire protocol)
- [x] 0004 — SSA / varnode / heritage construction (Funcdata, Varnode, Heritage, database, cover, merge)
- [x] 0005 — Type system (Datatype hierarchy, TypeOp, union resolution, composite/function types)
- [x] 0006 — Action/Rule simplification engine (Action, Rule, RuleAction, coreaction, condexe, subflow)
- [x] 0007 — Control-flow structuring (BlockGraph, BlockAction, JumpTable, goto elimination)
- [x] 0008 — Function signature & calling-convention recovery (FuncProto, ProtoModel, ParamID, override)
- [x] 0009 — Expression/cast logic & the C pretty-printer (PrintLanguage, PrintC, Cast, PrettyPrint)
- [x] 0010 — Java-side decompiler integration & test infrastructure (DecompInterface, ClangToken,
      datatests/unittests)

## Deliverables
Each child task produces one document under `.documents/01-review/`, and this epic's own
`.documents/01-review/00-index.md` ties them together with a single end-to-end narrative of the
pipeline: raw P-code in → `Funcdata` construction → SSA/heritage → type propagation → Action/Rule
simplification passes → block structuring → PrintC token emission → final C text.

## Acceptance Criteria
- [x] Every child task's document exists, is linked from `.documents/01-review/00-index.md`, and cites
      concrete `file:line` locations for the structures/functions it describes.
- [x] The set of documents together lets a new contributor trace one function's entire journey from
      P-code to printed C without reading the source first.
- [x] Every significant datatype (class) touched by the pipeline is named at least once across the
      review docs, with its owning file.

## Related Documents
- `.documents/01-review/00-index.md`
