# Flaw Analysis Index — Consolidated, Prioritized Findings

Epic: [`.tasks/0011`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md) · Root:
[`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md)

This synthesizes the four flaw-analysis documents into one prioritized list. Each detail document
follows an Observation → Impact/Why it's a problem → Severity → Improvement options → Open questions
structure (or, for [`varnode-ssa-type-system.md`](varnode-ssa-type-system.md), a numbered-table
variant covering more, narrower findings). Read the linked section for full evidence, citations, and
improvement-option trade-offs — this index only carries the headline and severity.

## The four documents

| Document | Ticket | Subsystem |
|---|---|---|
| [`varnode-ssa-type-system.md`](varnode-ssa-type-system.md) | 0012 | Datatype merge candidates, god objects, type-system expressiveness gaps, SSA/merge correctness risk |
| [`rule-engine-block-structuring.md`](rule-engine-block-structuring.md) | 0013 | Rule/structuring ordering risk, rule testability, the dead `rulecompile.cc` DSL, structuring weak spots |
| [`printer-output-quality.md`](printer-output-quality.md) | 0014 | Cast-layer duplication, `Emit`-abstraction bypass, `PrintLanguage` reusability, qualifier printing bug, templates/generics blocker |
| [`architecture-extensibility.md`](architecture-extensibility.md) | 0015 | Extension-model inconsistency, test-coverage/sequencing risk, wire-protocol version safety, build tooling |

## Consolidated priority table

Ordered High → Medium → Low. "High-priority as requirements input" is called out separately where a
finding isn't itself a High-severity defect but is the single most important input to epic 0016.

| # | Flaw | Severity | Source |
|---|---|---|---|
| 1 | Rule/block-structuring ordering & interaction is real, undeclared, only empirically stable | High | [rule-engine-block-structuring.md](rule-engine-block-structuring.md) Flaw 1 (0013) |
| 2 | Individual `Rule`s cannot be unit-tested in isolation today (~500-class catalog, zero unit coverage) | High | [rule-engine-block-structuring.md](rule-engine-block-structuring.md) Flaw 2 (0013) |
| 3 | Extension model is two incompatible mechanisms wearing one name (`CapabilityPoint` vs. hand-wired `Rule`/`Action` registration) | High | [architecture-extensibility.md](architecture-extensibility.md) Flaw 1 (0015) |
| 4 | Test-coverage gaps directly gate safe refactor ordering (SSA/heritage and the printer have ~zero direct coverage) | High | [architecture-extensibility.md](architecture-extensibility.md) Flaw 2 (0015); see also [`01-review/00-index.md`](../01-review/00-index.md) coverage table |
| 5 | `TypeFactory` vs. Ghidra `DataTypeManager` is a split-brain type system with only coarse (full-flush) invalidation | High | [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §1.8 (item 8) (0012) |
| 6 | `Funcdata` god object: ~6,300 lines across 4 files, no narrow interfaces, `friend`-based coupling to `Varnode`/`PcodeOp` | High (refactor risk) / Medium (correctness) | [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §2.1 (item 9) (0012) |
| 7 | Union-field resolution (`ScoreUnionFields`) is a bounded heuristic search, not exhaustive — can silently pick the wrong field | High | [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §4.3 (item 20) (0012) |
| 8 | **Templates/generics output blocker is a two-layer gap**: `Datatype` has no type-parameter-list concept (0012 §3.1) *and* `PrintC`'s declarator switch (`buildTypeStack`/`pushTypeStart`/`pushTypeEnd`) is a hard-closed `TYPE_PTR`/`TYPE_ARRAY`/`TYPE_CODE` chain that would silently ignore a template type even if one existed — and the gap repeats a third time in `PrintJava`, which doesn't share `PrintC`'s declarator code | Medium (no bug today) / **High as direct input to ticket 0017** | [printer-output-quality.md](printer-output-quality.md) Flaw 5 (0014); [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §3.1 (0012) |
| 9 | No wire-protocol version handshake — Java/C++ agreement rests on same-build discipline only, with a **silent-corruption** failure mode, not a build error | Medium-High | [architecture-extensibility.md](architecture-extensibility.md) Flaw 3 (0015) |
| 10 | Structuring algorithm has known-weak input classes (documented in source comments), correctness-preserving fallback only via goto, no readability recovery | Medium-High | [rule-engine-block-structuring.md](rule-engine-block-structuring.md) Flaw 4 (0013) |
| 11 | `Varnode`/`HighVariable`/`Symbol` field & flag duplication (type, lock, volatility mirrored across the trio) | Medium-High | [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §1.7 (item 7) (0012) |
| 12 | Speculative `Merge` heuristics are abandoned, not corrected, on conflict — plausible-but-wrong variable grouping is possible | Medium-High | [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §4.1 (item 18) (0012) |
| 13 | `Architecture` god object; capability-registration asymmetry (see #3) rooted here | Medium-High | [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §2.2 (item 10) (0012) |
| 14 | `rulecompile.cc` pattern DSL is dead in shipped builds (`Makefile`-gated off, one call site references an undeclared variable) — explicit recommendation: **retire, do not dual-track** | Medium (decision-forcing, not an active bug) | [rule-engine-block-structuring.md](rule-engine-block-structuring.md) Flaw 3 (0013) |
| 15 | Cast-necessity decisions split across `ActionSetCasts` (producer) and `PrintC`'s own `opIntZext`/`opIntSext`/`opSubpiece` (consumer) with no shared check | Medium | [printer-output-quality.md](printer-output-quality.md) Flaw 1 (0014) |
| 16 | `PrintLanguage` reusability beyond `PrintC`+`PrintJava` is unproven; the parts a third language would stress most (statement emission, declarators) have the least multi-language evidence | Medium | [printer-output-quality.md](printer-output-quality.md) Flaw 3 (0014) |
| 17 | `volatile` is tracked (`Symbol::isVolatile()`) but never printed as a keyword anywhere in `printc.cc`; `const` is unrepresentable in `Datatype` at all | Medium (demonstrable output bug, cheap standalone fix) | [printer-output-quality.md](printer-output-quality.md) Flaw 4 (0014); cross-ref [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §3.2 (0012) |
| 18 | Rule catalog's only built-in taxonomy (`group` tag) can't separate correctness-required rules from readability-only rules | Medium | [rule-engine-block-structuring.md](rule-engine-block-structuring.md) Flaw 5 (0013) |
| 19 | Two independent, undocumented C++ build/run paths; a fast standalone iteration loop exists (console driver) but isn't discoverable | Medium | [architecture-extensibility.md](architecture-extensibility.md) Flaw 4 (0015) — **standalone build/run: yes, via the interactive console driver** |
| 20 | Statement-level printing already bypasses the `Emit`/RPN abstraction in 4 places (original-author `FIXME` comments) | Low-Medium | [printer-output-quality.md](printer-output-quality.md) Flaw 2 (0014) |
| 21 | Type-system expressiveness gaps beyond templates: no composable const/volatile, no first-class function-pointer-with-context type, no union bitfields, no packed/alignment override, no graded/confidence-scored inference | Low–Medium each | [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §3.2–§3.7 (items 12–17) (0012) |
| 22 | `TypePartial*` triad, duplicate `compare()`/`compareDependency()`, duplicated union-resolution logic, `TypeSpacebase` pseudo-struct, overlapping "what kind of type" enums | Low–Medium each | [varnode-ssa-type-system.md](varnode-ssa-type-system.md) §1.1–§1.6 (items 1–6) (0012) |

## "Datatypes to merge or simplify" — direct answer

Per the root epic's explicit call-out, the concrete merge/simplification candidates (all sourced from
[`varnode-ssa-type-system.md`](varnode-ssa-type-system.md) §1) are:

- The **`TypePartial*` triad** — three parallel "byte-range slice" types with overlapping purpose.
- **Union/struct-field resolution logic**, currently duplicated across `TypeUnion`, `TypePointer`, and
  `TypePartialUnion`.
- **Two independently-coded "ephemeral type, here's the real one" mechanisms.**
- **Three overlapping "what kind of type is this" enums** (`type_class`/`type_metatype`/
  `sub_metatype`).
- **`Varnode`/`HighVariable`/`Symbol`**, which mirror `type` and lock/volatility flags instead of
  having one owning source of truth.
- `TypeSpacebase` standing in for address-space-relative symbol lookup as a pseudo-struct rather than
  a distinct concept.

## Refactor sequencing risk (pulled forward)

[`architecture-extensibility.md`](architecture-extensibility.md) contains the full **Refactor
Sequencing Risk** table (test coverage × coupling) required by ticket 0015 — consult it directly
before ordering any implementation work in the future epic 0024. Headline: SSA/heritage and the
printer are the least-tested, most-coupled subsystems and should not be the first to be touched
without first adding characterization tests.

## Status

Epic 0011 is **complete** — all 4 child tasks (0012–0015) are `done`. See
[`.tasks/0011-epic-flaw-analysis-and-improvements.md`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md).
