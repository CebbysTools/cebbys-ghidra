# 0003 — [TASK] Architecture & Pipeline Overview Review

## Type
Task

## Status
`todo`

## Parent
0002 (epic) / 0001 (root)

## Objective
Document the top-level architecture of the decompiler: process model, how it's invoked from Ghidra,
and the end-to-end pipeline stages a function goes through.

## Scope / Primary Files
- `architecture.cc/hh`, `capability.cc/hh`, `libdecomp.cc/hh`, `options.cc/hh`
- `database_ghidra.cc/hh`, `ghidra_arch.cc/hh`, `ghidra_process.cc/hh`, `ghidra_translate.cc/hh`
- `interface.cc/hh`, `ifacedecomp.cc/hh`, `ifaceterm.cc/hh` (console/interactive driver, useful for
  understanding the command surface used by tests/tools)
- `consolemain.cc`, `docmain.hh`, `doccore.hh`
- The Java side of the process boundary: `ghidra/app/decompiler/DecompInterface.java`,
  `DecompileProcess*.java`, `DecompileCallback.java` (search `src/main/java/ghidra/app/decompiler`)
- `Ghidra/Features/Decompiler/src/main/doc/decompileplugin.xml` for existing user-facing docs

## Deliverables
`.documents/01-review/architecture-pipeline-overview.md` covering:
1. Process/IPC model — decompiler runs as a separate process per-Ghidra-session; wire format
   (XML today — cite `marshal.cc/hh` since coreaction depends on it), request/response cycle for
   decompiling one function.
2. The `Architecture` object: what it owns (type factory, symbol database, PrintLanguage, ProtoModel
   registry, extra-pool of `Rule`/`Action` objects registered via `Capability`).
3. The `Action` pipeline stages at a glance (forward reference to 0006 for depth) — just enough here
   to show where SSA construction, typing, rule simplification, and structuring/printing sit in
   sequence.
4. Extension points: `Capability` self-registration mechanism, how new `Rule`s/`Action`s/architectures
   plug in today, and what that implies for refactor-ability.
5. A simple sequence diagram (Mermaid) of one `decompile <func>` request end to end.

## Acceptance Criteria
- [ ] Document exists at the path above and is linked from `.documents/01-review/00-index.md` and
      `0002`.
- [ ] Includes a Mermaid sequence or flow diagram.
- [ ] Cites concrete file:line for each major claim.
- [ ] Notes anything surprising/undocumented (e.g. global state, singleton-like patterns).

## Dependencies
None — can start immediately in parallel with 0004–0010.
