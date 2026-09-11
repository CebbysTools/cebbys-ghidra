# Architecture Options for the Decompiler Refactor

Epic: [`.tasks/0020`](../../.tasks/0020-epic-solution-research-and-selection.md) · Ticket:
[`.tasks/0022`](../../.tasks/0022-task-research-architecture-options-internal.md) · Root:
[`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md)

## Purpose and method

This document develops concrete architectural options for *this specific codebase* and compares them
against the flaw analysis (epic 0011), the requirements set (epic 0016), and the prior-art synthesis
(ticket 0021). It does not pick a winner — that is ticket 0023's job — but it is written so a winner is
apparent from the evidence, per the ticket's own instruction.

Inputs consulted in full: [`../02-flaws/00-index.md`](../02-flaws/00-index.md) and its four detail docs
(`varnode-ssa-type-system.md`, `rule-engine-block-structuring.md`, `printer-output-quality.md`,
`architecture-extensibility.md`); [`../03-requirements/00-index.md`](../03-requirements/00-index.md) (the
master REQ table); [`prior-art-external-decompilers.md`](prior-art-external-decompilers.md); and
[`../01-review/00-index.md`](../01-review/00-index.md) for the pipeline narrative and file-level detail.
Terminology follows [`../00-overview/glossary.md`](../00-overview/glossary.md).

**Scope note on the "four incompatible pipeline stages" question.** Ticket 0021's closing synthesis
recommends treating "introduce named, ordered, independently-testable pipeline stages" as the
load-bearing decision. This document takes that recommendation seriously (see "Foundation Layer" below)
but evaluates it rather than adopting it uncritically: named-stage-plus-characterization-tests is a
**testing/ordering discipline**, not a description of what changes about `Funcdata`, the type system, or
the printer's actual data structures. It answers "how do we know a change didn't break anything," not
"what does the code look like after the change." Every one of the three ticket-mandated options needs
this discipline to be safe to execute at all — none of them can responsibly proceed without it, per
REQ-ARCH-6/7 being MUST-priority and the flaw index's own sequencing-risk guidance. That is precisely why
it is **not** listed as a fourth, competing option: it doesn't specify an alternative target shape for the
codebase the way Options 1-3 do, so it cannot be scored against them as an alternative — scoring it as a
peer would understate that all three real options already depend on it. It is presented below as a
**Foundation Layer**, applied identically (and required) under every option, and it is *why* the
comparison matrix's REQ-ARCH-6/7 columns come out identical across Options 1-3: that identity is the
direct, visible consequence of factoring it out as shared infrastructure rather than a differentiator.

A genuinely distinct fourth option **is** developed, but it is a different idea from named pipeline
stages: ticket 0022's own option list names "adopting an interval-based structuring algorithm as a
drop-in replacement for `blockaction.cc`'s approach, scoped independently of the broader IR question" as
an example 4th option. That is a real, separate axis — a narrow, single-subsystem algorithm swap
decoupled from any IR/`Funcdata`/pipeline decision — and prior art (DREAM, SAILR; sources 7 and 9 in
ticket 0021) gives it enough independent substance to develop on its own. It is Option 4 below.

## Foundation Layer: named pipeline stages + characterization tests (applies under every option)

**What it is.** Give the existing (or new) pipeline phases — SSA construction/heritage, type propagation,
the `Action`/`Rule` fixed-point pool(s), signature/parameter recovery, block structuring, printing —
explicit names, an explicit declared order (ideally checked at startup against actual registration order,
per flaw #1's improvement option 1), and a full-text golden-snapshot output harness run over the 89-file
`datatests` corpus (REQ-ARCH-6/7, REQ-OUT-6) before and after every change to a named stage. This is
infrastructure, not a rewrite: it does not require deciding whether `Funcdata` gets decomposed or whether
a new IR layer is introduced.

**Why every option needs it regardless of shape.** The flaw index's Refactor Sequencing Risk table
(`architecture-extensibility.md`, reproduced in `00-index.md`) ranks SSA/heritage, the Action/Rule engine,
and `Architecture`/pipeline as the three highest-risk subsystems to touch **precisely because they have
no characterization coverage today** — and all three of Options 1-3 touch at least one of them. Prior art
converges independently on the same point (RetDec's LLVM-pass-level tests, Hex-Rays' named maturity
levels, angr's AIL pass ordering, Reko's front/core/back split — ticket 0021's closing synthesis). Without
this layer, *any* of Options 1-3 is refactoring blind against the codebase's own documented risk map.

**Blast radius of the Foundation Layer itself.** New code only, no behavior change: a golden-snapshot
harness extending `testfunction.hh`/`FunctionTestProperty` (full-function diff, not the existing regex
`<stringmatch>` assertions) and, for flaw #1, a declared-order-vs-registration-order assertion added to
`ActionDatabase::universalAction()` (`coreaction.cc:5835-6123`). Wiring `make test` into CI (REQ-ARCH-14,
flaw #4/0015-Flaw-4) is a natural companion but not strictly required for the harness to exist.

**Requirement/flaw mapping.** Directly satisfies REQ-ARCH-6, REQ-ARCH-7, and materially de-risks
REQ-OUT-6 (determinism) for all three canonical options equally — which is why those rows are identical
across Options 1-3 in the comparison matrix. It also directly targets flaw #4 (test-coverage gaps) and
gives flaw #1 (undeclared rule ordering) a mechanism, though not yet a full fix (the ordering-declaration
work itself is additional, option-specific effort — see each option's flaw-#1 row). It does **not** by
itself address flaw #2 (no per-`Rule` unit isolation), flaw #3 (extension model split), flaw #5 (type
system split-brain), flaw #6 (`Funcdata` god object), or flaw #7 (union-field resolution heuristic) — a
full-function golden diff catches *that* a change altered output, not *why*, and does nothing for
sub-function unit testability or the god-object coupling those flaws are about.

---

## Option 1 — In-place incremental refactor

Keep the `Funcdata`/`Action`/`Rule` architecture as the permanent shape of the codebase. Address flaws
subsystem-by-subsystem behind the Foundation Layer's characterization tests; evolve the type system
in place (add qualifier bits, a template/generic `Datatype` kind, etc., directly inside `type.hh`/
`type.cc`); narrow `Funcdata`'s interface incrementally rather than replacing it.

### What changes

- `type.hh`/`type.cc`: new `Datatype` kinds/fields for qualifiers (REQ-LANG-5), templates (REQ-LANG-1/2),
  in place — extending the existing `type_metatype` enum and interning scheme rather than replacing them.
- `printc.cc`/`printc.hh`: the `buildTypeStack`/`pushTypeStart`/`pushTypeEnd` three-way switch
  (`printc.cc:143-164,264-303,313-346`) gains new branches for qualifiers/templates; `volatile` printing
  added directly to `pushSymbol`'s declaration path.
- `ruleaction.hh`/`ruleaction.cc`/`coreaction.cc`: flaw #1's dependency-declaration metadata is added to
  the existing `Rule`/`Action` classes in place; flaw #5's rule-classification tag is added to the
  existing `group` mechanism rather than replacing it.
- `funcdata.hh` + its four `.cc` files: narrow interfaces (flaw #6 improvement option A) are extracted
  incrementally, `Funcdata` still exists as the one concrete implementation underneath.
- `capability.hh`/`capability.cc`, `coreaction.cc`: `Rule`/`Action` registration is unified onto
  `CapabilityPoint` (REQ-ARCH-8) as a `RuleCapability` addition to the four existing hierarchies.
- `type.cc`'s `findAdd`/`typegrp_ghidra.cc`: flaw #5's split-brain gets a smaller in-place mitigation
  (recoverable conflict instead of hard abort, or per-type invalidation) rather than an architectural
  change to how the two type systems relate.

### What stays

Everything else: the SSA/heritage construction algorithm (`Heritage`, phi-placement via dominator tree),
`CollapseStructure`'s pattern-catalog structuring approach, the `PrintLanguage`/`Emit` split, the wire
protocol, the `ProtoModel`/`ModelRule` signature-recovery machinery (already the codebase's best-designed
declarative layer per the review), and the overall six-root-`Action` pipeline shape in
`ActionDatabase::universalAction()`.

### Blast radius

Touches nearly every subsystem, but each touch is additive/local rather than structural: `type.hh`/
`type.cc` (~4,000+ lines, `TypeFactory`), `printc.cc`/`printc.hh` (~3,500 lines) and `printjava.cc`/
`.hh` (a *second*, independent edit for templates per flaw #5's own finding that `PrintJava` doesn't
inherit `PrintC`'s declarator code), `ruleaction.hh`/`.cc` (~500 classes, 162 wired), `blockaction.cc`/
`.hh`, `coreaction.cc` (the 700-line `universalAction()`), `funcdata.hh` + 4 `.cc` files (~6,300 lines,
incrementally), `architecture.hh`/`.cc`, `capability.hh`/`.cc`. No file is deleted or replaced wholesale;
most files gain code rather than being restructured. This is the option with the largest *number* of
touched files (because every flaw is fixed roughly where it lives) but the smallest structural diff per
file.

### MUST-requirement satisfaction

| REQ | Verdict | Rationale |
|---|---|---|
| REQ-LANG-1 (template `<...>` printing) | ✅ | Achievable via the printer-side declarator work (flaw #5 option 1) directly inside `printc.cc`, once `Datatype` gains a template kind. |
| REQ-LANG-4 (`volatile` keyword) | ✅ | Flaw #4's own option 1 — this is explicitly a small, isolated `printc.cc`-only fix, cheapest under any option. |
| REQ-LANG-5 (composable qualifiers) | 🟡 | Achievable (flaw #3.2 option A), but the review's own open question — whether any `Rule` relies on pointer-equality-as-type-equality — is a direct risk *because* qualifiers are added to the same live `Datatype`/`TypeFactory` interning tree every existing `Rule` already queries; regression risk is real and hard to bound without auditing all ~500 rule classes. |
| REQ-LANG-6 (no `ProtoModel` regression) | ✅ | Signature recovery is untouched by this option's changes — lowest risk of any option for this guardrail, since nothing about `ProtoModel`/`ModelRule`/`FuncProto` is being restructured. |
| REQ-LANG-9 (recursive/forward-declared composites preserved) | ✅ | Not touched; existing `TypeFactory` recursive-definition handling is unchanged. |
| REQ-LANG-11 (namespace/scoping fidelity, MUST at symbol level) | ✅ | `Symbol`/`Scope` untouched structurally; unaffected by in-place type-system additions. |
| REQ-OUT-1 (cast safety floor) | ✅ | Achievable via flaw #1's cast-decision unification (option 1: one `CastStrategy` method with a role parameter) done in place in `cast.cc`. |
| REQ-OUT-4 (variable-merge safety floor) | 🟡 | Flaw's own "Merge heuristics abandoned not corrected" finding (§4.1) is addressable in place, but `Merge`/`Cover` code has zero unit coverage today (rank 1 in the sequencing table) — fixing it safely *requires* the Foundation Layer first, same as every option, but this option has no structural change that makes `Merge` easier to isolate for testing, unlike Option 2's cleaner IR boundary. |
| REQ-OUT-6 (determinism) | ✅ | Satisfied by the Foundation Layer's golden-snapshot harness, applied identically under this option. |
| REQ-OUT-7 (diagnosability) | 🟡 | Achievable via flaw #1's dependency-declaration metadata plus flaw #5's rule-classification tag, both addable to the existing `Rule`/`Action` classes — but diagnosability is only as good as how completely 500 rule classes get annotated, a large incremental audit under this option specifically. |
| REQ-ARCH-1 (wire-protocol compat, no forced Java change) | ✅ | This option never touches `marshal.hh`/`ghidra_process.cc`'s wire format at all — the cleanest possible answer to this guardrail. |
| REQ-ARCH-2 (real `registerProgram` version handshake) | ✅ | Addable in place (flaw #3 option 3) without any structural change elsewhere. |
| REQ-ARCH-4 (independent unit-testability, Rank 1-4) | 🟡 | The Foundation Layer's golden-diff harness gives *detection*, but real sub-`Funcdata` unit isolation (flaw #6's option A) is the largest, riskiest piece of this option — extracting narrow interfaces from a 6,300-line, `friend`-coupled class in place, without a clean architectural seam to lean on, is acknowledged by the flaw doc itself as "Large" and contentious (virtual-dispatch overhead on a hot per-opcode loop). |
| REQ-ARCH-5 (single `Rule` testable via synthetic P-code) | 🟡 | Same constraint as REQ-ARCH-4 — flaw #2's own option 1 (a minimal synthetic-`Funcdata` fixture) is real engineering effort with no new architectural boundary to make it easier; achievable, but the hardest-won requirement under this option. |
| REQ-ARCH-6 (100% `datatests` byte-identical) | ✅ | Foundation Layer, identical under every option. |
| REQ-ARCH-7 (golden-snapshot harness) | ✅ | Foundation Layer, identical under every option. |
| REQ-ARCH-8 (unify Rule/Action registration on `CapabilityPoint`) | ✅ | Directly addressable via flaw #1's option 3 (extend `CapabilityPoint` to a `RuleCapability` hierarchy) in place — smallest, most mechanical fix, explicitly named in the flaw doc as a low-risk floor. |
| REQ-ARCH-9/10/11 (preserve architecture/output-language extensibility, route `ArchitectureGhidra` through capability discovery) | ✅ | Flaw #1's option 4 (`ArchitectureGhidra` through `findCapability`) is a small, isolated fix compatible with this option's incremental philosophy. |
| REQ-ARCH-13 (preserve/document console-driver loop) | ✅ | Zero conflict — this option doesn't touch the build system; documenting the Makefile loop (flaw #4 option 1) is orthogonal, cheap busywork under any option. |

**Summary**: 12 of 19 MUST rows ✅, 7 🟡, 0 ❌. The 🟡 rows cluster precisely where the flaw docs
themselves flagged the highest-risk, least-testable subsystems (`Funcdata`, `Merge`, rule-level testing)
— this option inherits those flaws' full difficulty because it does not change the structural boundary
that makes them hard to test in the first place.

### High-impact flaw handling

| # | Flaw | Verdict | Rationale |
|---|---|---|---|
| 1 | Rule/structuring ordering undeclared | 🟡 | Addressable via explicit dependency metadata added to the existing `Rule`/`Action` classes (flaw's own option 1) — real progress, but the underlying "emergent transformation" design (`ruleaction.hh:19-23`) is explicitly preserved as-is, so ordering risk is *documented and checked*, not eliminated. |
| 2 | Rules not unit-testable | 🟡 | Achievable (flaw #2 option 1) but hardest under this option specifically — no new seam, `Funcdata` construction remains as heavyweight as today. |
| 3 | Extension model split (`CapabilityPoint` vs. hand-wired `Rule`) | ✅ | Directly fixed via `RuleCapability` (flaw #1's option 3), a small mechanical change compatible with in-place philosophy. |
| 4 | Test-coverage gaps | ✅ | Directly addressed by the Foundation Layer, present under every option. |
| 5 | `TypeFactory`/`DataTypeManager` split-brain | 🟡 | In-place mitigation only (flaw's own option B/C: recoverable conflict or documented-flush discipline) — option A (real per-type invalidation) requires Java-side instrumentation regardless of C++-side architecture, so this is capped at "smaller mitigation" under every option, but Option 1 has no structural change that makes the cross-language bridge itself cleaner. |
| 6 | `Funcdata` god object | 🟡 | Flaw's own framing: incremental interface-narrowing (option A) is explicitly sized "Large" with real hot-path performance risk (virtual dispatch in `ActionPool`'s tight loop) — the flaw doc's own open question asks whether this needs a dedicated design pass, which in-place incrementalism defers rather than forces. |
| 7 | Union-field resolution heuristic (`ScoreUnionFields`) | 🟡 | Addressable via cross-path consistency tests (flaw §1.3 option C) or extracting a shared resolver (option A/B) in place — real improvement possible, but no structural change reduces the fundamental "three independent entry points into one heuristic" duplication risk faster than any other option would. |

### Risk

**Medium-low technical risk, medium-high schedule risk.** No single change is architecturally dangerous
in isolation (this is exactly incrementalism's strength), but the two hardest, highest-value fixes
(`Funcdata` decomposition for testability, and the rule-dependency-declaration audit across ~500 classes)
are large, slow-burn efforts with no forcing function to prioritize them — the codebase can plausibly
"incrementally refactor" for years while never actually reaching the state that makes SSA/heritage and
the rule engine safely testable, since nothing about this option's shape requires reaching that state
before shipping other improvements. This is a real risk given the root epic's own framing of `Funcdata`
decomposition as "the central architectural fact a refactor has to reckon with."

### First "most basic decompiler features" slice (guidance for epic 0024)

Under this option, a first slice would be: stand up the Foundation Layer harness against the current
pipeline (near-zero-regret, needed regardless); ship the two cheapest, highest-value REQ-LANG/REQ-OUT
wins directly in place (`volatile` printing, REQ-LANG-4; cast-decision unification, REQ-OUT-1); unify
`Rule`/`Action` registration onto `CapabilityPoint` (REQ-ARCH-8, flaw #3); and begin — but not finish — a
synthetic-`Funcdata` test fixture for the highest-risk rule classes identified by flaw #1's eight
documented ordering cases, since that fixture is a prerequisite for every deeper fix that follows. No new
files/modules are introduced; all work lands inside existing translation units.

---

## Option 2 — New intermediate typed IR/AST layer

Introduce a distinct, richer typed IR/AST between the SSA P-code graph and the printer, directly
addressing flaw #14 in printer-output-quality.md's Flaw 2 ("introduce a proper visitor/AST intermediate
between structured blocks and text instead of direct emission" — the ticket's own Analysis Focus item 5)
and flaw #5's templates/generics blocker. SSA construction, heritage, and the `Action`/`Rule`
simplification catalog stay largely as-is; a new stage consumes their output and produces the new IR,
which the printer consumes instead of talking to `Funcdata`/`Varnode`/`PcodeOp`/`BlockGraph` almost
directly the way `PrintC` does today.

### What changes

- A new IR/AST layer (new files, e.g. a `hirast.hh`/`hirast.cc`-shaped module, name TBD by epic 0024):
  richer than `Datatype`+`ClangToken`, with first-class nodes for qualifiers, template instantiations,
  and statement-level constructs (`if`/`while`/`switch`/`goto`/`return`) — directly closing flaw #2's
  four documented FIXME-marked direct-emission bypasses (`printc.cc:558,604,779,2965`) by construction,
  since the new IR *is* the structured intermediate that Analysis Focus item 5 asks for.
  This is the layer where REQ-LANG-1/2/5 (templates, qualifiers) naturally attach: a `Datatype`-level
  template concept still has to exist (this option does not avoid flaw #5's representation-side half),
  but the printer-side declarator work now happens once, against the new IR's own node types, instead of
  inside `PrintC`'s closed three-way `type_metatype` switch — and, critically, the same IR can in
  principle serve `PrintJava` too, reducing (not eliminating) flaw #3's "9-method-subclass" duplication
  risk for a third case.
- `PrintC`/`PrintJava`: retargeted to walk the new IR rather than `Funcdata`'s block/op graph directly;
  the four FIXME'd direct-emission methods (`opCbranch`/`opBranchind`/`opReturn`/`emitBlockCondition`)
  are replaced by IR-node printing, not patched in place.
- A new "lowering" stage: `Funcdata`'s post-`CollapseStructure` structured-block graph + `Varnode`/
  `HighVariable`/`Symbol` state is translated into the new IR. This is genuinely new code, not a
  refactor of existing code — it is where most of this option's real engineering cost lives.
- `cast.cc`: cast-necessity decisions (flaw #1 in printer-output-quality.md) can be unified at the
  lowering boundary — the new IR's cast nodes are produced once, from one decision, rather than being a
  producer-side `CPUI_CAST` op plus a separate consumer-side display heuristic.

### What stays

SSA construction (`Heritage`), the `Action`/`Rule` fixed-point engine (`ruleaction.cc`, `coreaction.cc`),
`CollapseStructure`'s structuring algorithm, the type system's core representation (`type.hh`/`type.cc`,
extended for qualifiers/templates but not replaced), the signature/calling-convention subsystem, and the
wire protocol. `Funcdata` itself is not decomposed under this option — it remains the pre-lowering
representation; the new IR is an addition downstream of it, not a replacement for it.

### Blast radius

Concentrated but deep: a wholly new module (the IR/AST itself — genuinely new code, unbounded a priori,
likely comparable in size to `printc.cc`'s ~3,500 lines given it has to represent everything `PrintC`
currently derives ad hoc), a new lowering stage bridging `Funcdata`/`BlockGraph`/`Varnode` to it, and a
full rewrite (not incremental patch) of `printc.cc`/`printc.hh`/`printjava.cc`/`printjava.hh` to consume
it instead of `Funcdata` directly. `printlanguage.hh`/`printlanguage.cc` (the RPN `pushOp`/`parentheses()`
engine) either gets subsumed by the new IR's own printing pass or kept as a compatibility layer under the
new IR — a design decision epic 0024 would need to make explicitly. `Emit`/`EmitPrettyPrint`
(`emit.hh`/`.cc`) can plausibly be reused as-is (it's the layout engine, orthogonal to what feeds it).
`cast.cc`/`cast.hh` restructured, not merely extended. Everything upstream of structuring (`Heritage`,
`ruleaction.cc`, `blockaction.cc`, `type.hh`/`type.cc`) is touched only for the qualifier/template
representation additions shared with Option 1 — the SSA/rule/structuring engines themselves are not
restructured.

### MUST-requirement satisfaction

| REQ | Verdict | Rationale |
|---|---|---|
| REQ-LANG-1 (template printing) | ✅ | This is the option's strongest fit — a genuine declarator-layer concept for templates is a natural IR node type, not a fourth branch bolted onto a closed three-way switch; also the only option that structurally reduces (not just tolerates) flaw #3's `PrintJava` duplication risk for this specific feature. |
| REQ-LANG-4 (`volatile` keyword) | ✅ | Trivial under the new IR (a qualifier node prints itself); also trivially achievable under every other option — no differentiation here. |
| REQ-LANG-5 (composable qualifiers) | ✅ | The new IR's node types can represent qualifier composition natively at the point of printing; still requires the same `Datatype`-level addition as Option 1 for qualifiers to exist upstream (this option doesn't remove that prerequisite), but the printer-side risk (auditing whether existing `Rule`s assume pointer-equality-as-type-equality) is *lower* here because qualifier composition is resolved at the IR/print boundary, one step removed from the same live interning tree every `Rule` queries mid-pipeline. |
| REQ-LANG-6 (no `ProtoModel` regression) | ✅ | Signature recovery is upstream of the new IR and untouched structurally. |
| REQ-LANG-9 (recursive/forward-declared composites) | ✅ | Unaffected; `TypeFactory`'s existing handling is unchanged, only how the printer *consumes* it changes. |
| REQ-LANG-11 (namespace/scoping, MUST at symbol level) | ✅ | `Symbol`/`Scope` untouched; the new IR consumes already-resolved symbol/scope data during lowering. |
| REQ-OUT-1 (cast safety floor) | ✅ | Best fit of any option: a single cast-decision point at the lowering boundary directly eliminates flaw #1's two-entry-point split rather than unifying it after the fact. |
| REQ-OUT-4 (variable-merge safety floor) | 🟡 | `Merge`/`Cover` heuristics live upstream of the new IR (in `Funcdata`/`Heritage`) and are untouched by this option — same risk profile as Option 1 for this specific requirement, since the new IR doesn't change how merges are decided, only how the result is printed. |
| REQ-OUT-6 (determinism) | ✅ | Foundation Layer, identical under every option — and arguably *easier* to keep once achieved, since the new IR is a single well-defined snapshot point to diff against, rather than the printer's current direct, stateful walk of `Funcdata`. |
| REQ-OUT-7 (diagnosability) | 🟡 | The new IR can carry provenance metadata (which rule/stage produced each cast/goto/fallback) by design if epic 0024 builds that in from the start — a real opportunity this option creates — but it is not automatic; if the lowering stage doesn't thread provenance through, this requirement is no better served than under Option 1. Scored 🟡 rather than ✅ because it depends on a design choice not yet made. |
| REQ-ARCH-1 (wire-protocol compat) | ✅ | The new IR is entirely internal to the C++ engine, downstream of everything the wire protocol carries — no Java-visible change required. |
| REQ-ARCH-2 (version handshake) | ✅ | Orthogonal; addressable identically to Option 1. |
| REQ-ARCH-4 (independent unit-testability, Rank 1-4) | 🟡 | The new IR itself becomes trivially unit-testable in isolation (construct an IR fragment, print it, assert text) — a genuine win for the printer (rank 6) — but does nothing for the actual rank-1/2/3 subsystems (SSA/heritage, Action/Rule engine, Architecture/pipeline), which remain exactly as hard to test as under Option 1, since this option doesn't touch them. |
| REQ-ARCH-5 (single `Rule` testable) | ❌ | Not addressed at all by this option — `Rule`s operate on `PcodeOp`/`Funcdata`, entirely upstream of the new IR; this requirement needs the same synthetic-`Funcdata`-fixture work as Option 1, with zero help from the new IR layer. |
| REQ-ARCH-6 (100% `datatests` byte-identical) | ✅ | Foundation Layer, identical under every option — though notably *harder* to satisfy mechanically during the transition, since a full printer rewrite is far more likely to introduce incidental formatting drift than Option 1's localized patches; the harness is equally necessary but the bar is harder to clear mid-migration. |
| REQ-ARCH-7 (golden-snapshot harness) | ✅ | Foundation Layer, identical. |
| REQ-ARCH-8 (unify Rule/Action registration) | 🟡 | Not addressed by this option's core change at all (the IR layer is downstream of `Rule`/`Action`); still requires the same `RuleCapability` work as Option 1, done as a parallel, unrelated effort rather than something the IR introduction helps or hinders. |
| REQ-ARCH-9/10/11 (preserve/extend architecture & output-language extensibility) | ✅ | `ArchitectureGhidra` capability-registration fix is orthogonal and equally achievable; output-language extensibility (REQ-ARCH-10, implicitly) is arguably *improved* under this option since a new target language would work against the new IR's node types instead of subclassing 3,500 lines of `PrintC` the way `PrintJava` does today — directly responsive to flaw #3's finding that `PrintLanguage`'s reusability is unproven beyond one thin data point. |
| REQ-ARCH-13 (preserve console-driver loop) | ✅ | Unaffected; orthogonal to this option's scope. |

**Summary**: 13 of 19 ✅, 5 🟡, 1 ❌. This option's ❌ (REQ-ARCH-5) and flat 🟡s are honest: it is the
strongest option for output-fidelity/printer-side MUSTs (REQ-LANG-1/5, REQ-OUT-1, REQ-ARCH-9/10/11) but
provides **no help at all** for the SSA/heritage- and rule-testability MUSTs (REQ-ARCH-4/5) that the flaw
index ranks as the highest refactor risk in the codebase — those remain exactly as hard as under Option 1.

### High-impact flaw handling

| # | Flaw | Verdict | Rationale |
|---|---|---|---|
| 1 | Rule/structuring ordering undeclared | ❌ | Entirely upstream of this option's change — the `Action`/`Rule`/`CollapseStructure` ordering problem is untouched; this option does not even indirectly help, since the new IR only consumes structuring's *output*. |
| 2 | Rules not unit-testable | ❌ | Same reasoning as REQ-ARCH-5 above — no structural help from a downstream IR layer. |
| 3 | Extension model split | ❌ | Untouched; requires the same separate `RuleCapability` fix as every other option. |
| 4 | Test-coverage gaps | 🟡 | The Foundation Layer applies, plus this option adds genuinely new, easily-unit-tested surface (the IR itself, and the printer once retargeted) — a real, option-specific improvement to *printer* coverage specifically (closing "no golden/exact-text coverage anywhere" more thoroughly than Option 1 could, since the new IR gives print-testing a clean, stable input format) — but does nothing for the SSA/heritage or Action/Rule coverage gaps that dominate the flaw's overall severity. |
| 5 | `TypeFactory`/`DataTypeManager` split-brain | ❌ | Untouched — this is a `TypeFactory`↔Java bridge problem, entirely orthogonal to a print-side IR. |
| 6 | `Funcdata` god object | ❌ | Untouched by design — this option deliberately does not decompose `Funcdata`; if anything it adds one more (well-scoped) consumer of `Funcdata`'s current interface (the lowering stage), which is a small *increase* in `Funcdata`'s fan-out, not a reduction. |
| 7 | Union-field resolution heuristic | ❌ | Untouched; `ScoreUnionFields`/`unionresolve.cc` sit entirely upstream of the new IR. |

**Summary**: this option is the strongest available for flaw #14/Flaw-2-in-0014 (the direct-emission
FIXMEs) and for flaw #5/Flaw-5-in-0014 (templates), which are the two findings it was scoped to address
directly — but it scores worse than Option 1 on the High-impact-flaw table overall, because six of the
seven High-severity flaws live upstream of where this option changes anything.

### Risk

**Medium-high technical risk, concentrated in one place.** The core engineering risk is real but bounded
in scope: a full printer rewrite against a brand-new IR is a large, all-or-nothing-feeling piece of work
for one subsystem (rank 6 in the sequencing table — one of the *safer*, more downstream subsystems to
rewrite, since nothing later in the pipeline depends on printer output), which somewhat mitigates the
"large rewrite" risk profile — a printer bug cannot cascade into wrong program semantics the way an
SSA/rule-engine bug could. The larger risk is scope discipline: it is easy for "introduce an IR layer"
to quietly grow into "also decompose `Funcdata`" or "also fix the rule engine" once the lowering stage's
authors discover how much of `Funcdata`'s current surface they need to consume — a risk explicitly
foreshadowed by flaw #6's own open question about whether `Funcdata`/`Architecture` decomposition should
be a joint design exercise with whatever epic 0022 chooses.

### First "most basic decompiler features" slice (guidance for epic 0024)

Under this option, a first slice is naturally narrower than Option 1's: build the Foundation Layer harness
first (unconditional); then design and build the smallest possible IR (a handful of node kinds — variable
reference, binary op, call, basic statement forms) sufficient to reproduce a small, real subset of
`datatests` scenarios byte-for-byte through the new path, proving the lowering+print round-trip works
*before* adding template/qualifier nodes. This is a "walking skeleton" slice — get one simple function
family (say, straight-line integer arithmetic, no control flow) through IR-based printing and verified
against the golden harness, then grow the node vocabulary. SSA/rule/structuring work in this slice is
zero; all first-slice effort is in the new IR + lowering + printer retarget.

---

## Option 3 — Staged replacement with a compatibility shim / feature-flag cutover

Build new subsystem(s) alongside the old ones behind the existing wire protocol. A feature flag (per
architecture, or per named pipeline stage) selects old-vs-new at `registerProgram`/decompile-request time.
Cut over incrementally once the new subsystem proves byte-identical (or intentionally-improved, explicitly
diffed) parity against the `datatests` corpus; delete the old implementation only after cutover is
complete and stable.

### What changes

- Whichever subsystem(s) epic 0024 targets first get a **second, parallel implementation** living beside
  the first — e.g. a new `Rule` engine, a new structuring pass, or the new IR/printer from Option 2 — each
  gated by a flag threaded through `Architecture`'s option database (`OptionDatabase`,
  `architecture.hh`) or a `.cspec`-level toggle, not a compile-time `#ifdef` (which would defeat the
  point: both must be runtime-selectable and both must ship in the same binary during the transition).
- `ArchitectureGhidra`/`ghidra_process.cc`: the `registerProgram` handshake gains a feature-selection
  parameter (naturally combinable with REQ-ARCH-2's version-handshake work — both touch the same
  `registerProgram` call site).
- The Foundation Layer's golden-snapshot harness becomes load-bearing infrastructure, not just a safety
  net: it is the literal cutover gate — a subsystem does not flip its default until its new
  implementation is byte-identical (or documented-different) against 100% of `datatests` under both
  flag settings.
- Old code is deleted only after cutover — this is explicitly the *opposite* of dual-tracking flaw #3's
  dead `rulecompile.cc` DSL (which the flaw doc explicitly recommends against keeping dual-tracked
  indefinitely); this option's shim is a deliberately time-boxed migration aid, not a permanent second
  code path, and epic 0024/0023 should set an explicit deprecation trigger (e.g. "N releases after
  default flip") to avoid recreating exactly the kind of undocumented dead-but-present code flaw #3 found.

### What stays

Whatever hasn't been cut over yet, unmodified, running behind the flag's "old" branch. The wire protocol
itself is unchanged (both implementations speak the same `PackedEncode`/`PackedDecode` format on the same
existing `ElementId` table) — only the *selection* of which implementation runs is new, and that selection
is itself a small protocol addition (a feature-flag parameter), not a protocol rewrite.

### Blast radius

Highly variable by design — this option is a *rollout strategy*, not a fixed target shape, so its blast
radius equals whichever subsystem(s) are chosen for staged replacement, **plus** a fixed overhead of
flag-plumbing work: `architecture.hh`/`.cc` (`OptionDatabase`), `ghidra_process.cc`/`RegisterProgram`
(`registerProgram` handshake), and — critically — **maintaining two working implementations
simultaneously** for every in-flight subsystem, which is a standing tax on every unrelated change to that
subsystem until cutover completes. If Option 2's IR/printer work is chosen as the first staged subsystem,
this option's blast radius is a strict superset of Option 2's (same files) plus the flag/shim
infrastructure; if Option 1's incremental fixes are staged instead, the superset relationship holds against
Option 1 instead. This is the key structural fact about Option 3: **it is an orthogonal rollout mechanism
that can wrap either Option 1 or Option 2's actual subsystem work — it does not, by itself, specify what
changes inside `Funcdata`, the type system, or the printer.**

### MUST-requirement satisfaction

Because Option 3 wraps whichever underlying subsystem work is staged, its requirement satisfaction for
REQ-LANG/REQ-OUT rows is identical *in the limit* (post-cutover) to whichever of Option 1/2 was chosen for
that subsystem. The rows below focus on where Option 3 differs materially — the REQ-ARCH rows, which are
about rollout/compatibility, and REQ-OUT-6/7, where a dual-implementation period genuinely changes the
risk profile mid-transition.

| REQ | Verdict | Rationale |
|---|---|---|
| REQ-LANG-1/4/5/9/11, REQ-LANG-6 | 🟡 (inherits) | Identical to whichever of Option 1/2 supplies the actual subsystem change — Option 3 adds a rollout mechanism around that change, not a different answer to these requirements. Marked 🟡 uniformly here (rather than per-row ✅) because the answer is genuinely conditional on a choice this document does not make. |
| REQ-OUT-1/4 | 🟡 (inherits) | Same reasoning — cast-safety and merge-safety floors are properties of the underlying implementation being staged, not of the staging mechanism. |
| REQ-OUT-6 (determinism) | ✅ (post-cutover) / 🟡 (mid-transition) | Post-cutover, identical to the Foundation Layer's guarantee under any option. **Mid-transition**, this option uniquely *strengthens* REQ-OUT-6's verification: the golden-snapshot harness runs both flag states against the same corpus, catching determinism regressions the moment they're introduced rather than only at a single before/after diff point — a genuine advantage specific to running old and new side-by-side. |
| REQ-OUT-7 (diagnosability) | 🟡 | Inherits the underlying implementation's answer; the flag state itself is one more piece of diagnosable metadata "which implementation produced this output" that a debugging session benefits from, a small option-specific plus. |
| REQ-ARCH-1 (wire-protocol compat, no forced Java change) | 🟡 | This is the requirement Option 3 puts the most direct pressure on: adding a feature-flag parameter to `registerProgram` **is** a Java-visible wire-protocol change, even if a small, additive one — REQ-ARCH-1's own text allows exceptions for exactly this kind of controlled, explicit change, but it is the one option where satisfying this MUST requires deliberately invoking one of REQ-ARCH-1's stated exceptions rather than avoiding the question entirely (as Options 1/2/4 do). This needs explicit sign-off in ticket 0023, not an assumption. |
| REQ-ARCH-2 (version handshake) | ✅ | Naturally combinable — the same `registerProgram` touch-point extension needed for the flag can carry the version handshake in the same change, making this option's REQ-ARCH-1 cost partially self-funding. |
| REQ-ARCH-4 (independent unit-testability) | 🟡 (inherits) | Whatever the staged subsystem's own testability profile is (Option 1's or Option 2's), plus a genuine bonus: a flag-gated new implementation is *inherently* easier to test in isolation than a live-only replacement, since the old implementation remains available as a running comparison oracle for every test case during development — a real, option-specific testability improvement not available to Options 1/2 alone. |
| REQ-ARCH-5 (single `Rule` testable) | 🟡 (inherits) | Same reasoning; no different from whichever underlying option is staged. |
| REQ-ARCH-6/7 (golden-snapshot, 100% corpus) | ✅ | Foundation Layer — and, as REQ-OUT-6 notes above, *load-bearing* rather than just protective under this option, since it is literally the cutover gate. |
| REQ-ARCH-8 (unify Rule/Action registration) | 🟡 (inherits) | No different from Option 1's answer if that's the subsystem staged. |
| REQ-ARCH-9/10/11 (preserve/extend extensibility) | 🟡 (inherits) | Same. |
| REQ-ARCH-13 (preserve console-driver loop) | ✅ | Unaffected by the staging mechanism itself. |

**Summary**: Option 3's distinguishing MUST-requirement fact is **REQ-ARCH-1**, the one row where it is
weaker than every other option (a small, explicit, controlled wire change vs. zero wire change), traded
against a genuine strengthening of REQ-ARCH-4 and REQ-OUT-6's *verification* story during the transition
period specifically. Every other row is inherited from whichever subsystem-level option is wrapped.

### High-impact flaw handling

| # | Flaw | Verdict | Rationale |
|---|---|---|---|
| 1 | Rule/structuring ordering undeclared | 🟡 (inherits) | Same as whichever of Option 1/2 is staged — no independent effect from the shim mechanism itself. |
| 2 | Rules not unit-testable | 🟡→improved | Inherits the underlying option's answer, but with the same "old implementation as a live comparison oracle" bonus noted under REQ-ARCH-4 — genuinely easier to validate a new `Rule` implementation incrementally when the old one is a running, flag-selectable baseline rather than something that has already been deleted. |
| 3 | Extension model split | 🟡 (inherits) | No independent effect. |
| 4 | Test-coverage gaps | ✅ | Strongest of any option specifically because the golden-snapshot harness's cutover-gating role *forces* 100% `datatests` parity to be checked continuously during development, not just once at the end — this option cannot proceed at all without exercising the Foundation Layer harness on every subsystem it stages, which is a stronger guarantee than "should be checked" under Options 1/2. |
| 5 | `TypeFactory`/`DataTypeManager` split-brain | 🟡 (inherits) | No independent effect unless the type-system bridge itself is the staged subsystem, in which case a flag-gated dual-cache period would need its own careful design (not detailed here — out of scope unless epic 0024 chooses this subsystem first). |
| 6 | `Funcdata` god object | 🟡 (inherits) | No independent effect; if `Funcdata` decomposition is the staged subsystem, the shim mechanism gives a genuine advantage (old and new `Funcdata` shapes can be compared per-function) but this document does not assume that specific staging choice. |
| 7 | Union-field resolution heuristic | 🟡 (inherits) | No independent effect. |

**Summary**: Option 3's flaw-table advantage is concentrated in flaw #4 (test-coverage gaps), where its
rollout mechanism structurally *requires* the exact discipline flaw #4's improvement options recommend.
Every other row inherits its answer from the underlying subsystem-level choice.

### Risk

**Lowest per-change technical risk; highest process/coordination risk.** The core safety property — never
lose a working implementation, always have a fallback flag — is exactly what a legacy engine with the
correctness stakes described in REQ-OUT-1/REQ-OUT-4 (silent wrong output is worse than ugly output) wants.
But it carries real, distinct costs the other options don't: (1) **double maintenance burden** for every
in-flight subsystem — a bug found in production during the transition period may need fixing in *both*
implementations if it also exists in the old one; (2) **flag proliferation risk** if multiple subsystems
are staged concurrently without discipline, producing a combinatorial matrix of old/new states that is
itself hard to test exhaustively (this risk is exactly what "100% `datatests` under both flag settings"
is meant to bound, but it bounds correctness, not combinatorial *test time*); (3) **the REQ-ARCH-1 tension**
already flagged above — this is the one option that cannot honestly claim zero wire-protocol touch;
(4) **deprecation discipline** — per flaw #3's explicit warning against dual-tracking `rulecompile.cc`
indefinitely, this option only pays off if old implementations are actually deleted on a defined schedule,
which is a process commitment, not a technical property the architecture itself enforces.

### First "most basic decompiler features" slice (guidance for epic 0024)

Under this option, a first slice is: build the Foundation Layer harness (unconditional, and here
doubly justified since it's the cutover gate); add the minimal `registerProgram` flag-plumbing plus
REQ-ARCH-2's version handshake in the same change (self-funding, per the MUST table above); then pick
**one** narrow, low-blast-radius subsystem to stage first as a proof of the mechanism — the printer
(rank 6, lowest downstream coupling per the sequencing table) is the safest candidate, effectively staging
Option 2's IR/printer work behind a flag rather than cutting it over directly. This makes Option 3's first
slice a strict superset of Option 2's first slice, plus the flag/version-handshake plumbing — a deliberate
choice to prove the shim mechanism on the option that already carries the least downstream-correctness
risk if something goes wrong mid-transition.

---

## Option 4 — Targeted structuring-algorithm replacement (scoped, orthogonal)

A narrow, single-subsystem option distinct from Options 1-3: replace `CollapseStructure`'s pattern-catalog
structuring approach (`blockaction.cc:1052-1901`, flaw #4/#10 — known-weak on irreducible CFGs, complex
loop conditions, jump-threaded duplication) with one of the prior-art alternatives ticket 0021 surveyed —
DREAM's node-splitting (source 7), SAILR's compiler-aware transformation-inversion (source 9), or a
classic interval-based structuring pass — **without** committing to any answer on the `Funcdata`/IR/
staging question. This is explicitly the ticket's own named example of a possible 4th option, and it is
kept separate from Options 1-3 (rather than folded into one of them as a sub-bullet) because it is
orthogonal on a different axis entirely: it is a single-algorithm swap that could, in principle, be
combined with *any* of Options 1, 2, or 3 as a determination of "what specifically happens inside the
structuring stage" once that stage exists as an addressable unit.

### What changes

`blockaction.cc`/`blockaction.hh` specifically: the `CollapseStructure`/`ruleBlockX` pattern-matching
catalog and its `TraceDAG`/`selectGoto` goto-fallback heuristic are replaced (or, more conservatively,
supplemented — see Risk below) by a chosen alternative algorithm. `block.cc`/`block.hh`'s reducibility
detection (`findIrreducible`, `structureLoops`) either feeds the new algorithm directly (if adopting
DREAM/SAILR's own reducibility handling) or stays as the trigger condition for when the new algorithm's
fallback path kicks in (the more conservative choice — "keep the fast pattern-matching path for the common
reducible case, invoke the new algorithm only once irreducibility/goto-fallback is detected," exactly the
open question flaw #4's own analysis raises). `JumpTable`'s multistage/unrolled-guard special cases
(`jumptable.cc`) are unaffected — they are upstream of structuring and orthogonal to which structuring
algorithm runs.

### What stays

Everything else: SSA/heritage, the `Action`/`Rule` engine, the type system, the printer, `Funcdata`'s
shape, the wire protocol. This is the narrowest-scoped option of the four by a wide margin.

### Blast radius

The smallest of any option: `blockaction.cc`/`blockaction.hh` (the structuring rule catalog and
`TraceDAG`), `block.cc`/`block.hh` only for the reducibility-detection integration points, and
`jumptable.cc` not at all (confirmed independent per the review — jump-table recovery is "an independent,
earlier pass" in the pipeline narrative). No printer, type-system, `Funcdata`, or wire-protocol changes
are required by this option in isolation.

### MUST-requirement satisfaction

Because this option's scope is narrow, most MUST rows are simply **N/A (out of this option's scope)** —
it neither helps nor hurts them, since it doesn't touch those subsystems at all. This is itself useful
information for the comparison matrix: Option 4 cannot be evaluated as a *complete* answer to the
requirements set the way Options 1-3 can, only as a component.

| REQ | Verdict | Rationale |
|---|---|---|
| REQ-LANG-1/2/4/5/6/9/11 | N/A (out of scope) | This option touches only block structuring; none of the language-feature requirements are structuring concerns. |
| REQ-OUT-1 (cast safety) | N/A (out of scope) | Cast logic (`cast.cc`) is untouched by a structuring-algorithm swap. |
| REQ-OUT-3 (goto-minimization, SHOULD not MUST, included for context since it's this option's namesake requirement) | ✅ | This is precisely the requirement this option targets — DREAM reports zero gotos across its evaluation corpus (ticket 0021 source 7); SAILR directly benchmarks against Ghidra's own current goto rate (source 9) as an existing baseline this option could measurably improve against. Not a MUST, but flagged here because it is the one requirement this option answers better than any of Options 1-3 could without also adopting this option. |
| REQ-OUT-4 (merge safety floor) | N/A (out of scope) | `Merge`/`Cover` heuristics are upstream of structuring, in `Funcdata`/`Heritage`. |
| REQ-OUT-6 (determinism) | ✅ | Satisfied via the Foundation Layer, applied identically — this option, like the others, needs the golden-snapshot harness before any structuring-algorithm change is safe to make, since `control-flow-structuring.md`'s own coverage note says structuring has "good 'by shape name' `datatests` coverage... but no goto-fallback/structuring-quality regression tracking." |
| REQ-OUT-7 (diagnosability) | 🟡 | A new structuring algorithm *could* be built with better fallback-decision provenance than today's hand-tuned `BadEdgeScore` heuristic, but this is not automatic — depends on how epic 0024 scopes the replacement, same caveat as Option 2's REQ-OUT-7 row. |
| REQ-ARCH-1/2 (wire protocol, version handshake) | N/A (out of scope) | No Java-visible surface in `blockaction.cc`. |
| REQ-ARCH-4/5 (unit-testability) | 🟡 | `CollapseStructure` is rank 5 in the sequencing table (already meaningfully better-tested than ranks 1-3) — a new algorithm needs its *own* new unit tests (a structuring algorithm is exactly the kind of thing that benefits from synthetic-CFG-shape test fixtures, mirroring flaw #2's `Rule`-fixture idea but for block shapes instead of `PcodeOp`s), which this option would need to build fresh rather than inherit from anywhere. |
| REQ-ARCH-6/7 (golden-snapshot, 100% corpus) | ✅ | Foundation Layer, identical. |
| REQ-ARCH-8/9/10/11/13 | N/A (out of scope) | Extension-model and build-tooling requirements are entirely orthogonal to which structuring algorithm runs. |

**Summary**: 4 of 19 MUST rows apply at all (the rest are N/A by genuine scope exclusion); of those, 3 ✅
and 1 🟡. Option 4 is not a competitor to Options 1-3 on the requirements table as a whole — it is a
complement that any of them could adopt for the structuring stage specifically.

### High-impact flaw handling

| # | Flaw | Verdict | Rationale |
|---|---|---|---|
| 1 | Rule/structuring ordering undeclared | 🟡 | Directly relevant to the *structuring* half of flaw #1 (items 1-3, the `ruleBlockOr`/`ruleBlockIfNoExit` ordering cases) — a genuinely different algorithm (DREAM/SAILR) sidesteps the specific ordering bugs documented against `CollapseStructure`'s pattern catalog, but does nothing for the *rule-engine* half of flaw #1 (items 4-8, `ActionPool`/`ruleaction.cc` ordering), which is a separate subsystem this option doesn't touch. |
| 2 | Rules not unit-testable | N/A (out of scope) | This flaw is about `Rule::applyOp`/`ActionPool`, not `CollapseStructure`'s `ruleBlockX` catalog — different classes entirely, not addressed by a structuring-algorithm swap. |
| 3 | Extension model split | N/A (out of scope) | Not a structuring concern. |
| 4 | Test-coverage gaps | 🟡 | Improves structuring's own coverage (a new algorithm needs new tests, a forcing function this option creates) but does nothing for the SSA/heritage or Action/Rule gaps that dominate this flaw's severity. |
| 5 | `TypeFactory`/`DataTypeManager` split-brain | N/A (out of scope) | Not a structuring concern. |
| 6 | `Funcdata` god object | N/A (out of scope) | `CollapseStructure` already operates on a disposable copied graph (`BlockCopy`, per the sequencing table's rank-5 rationale) rather than mutating `Funcdata` in place — this option doesn't touch `Funcdata`'s interface at all. |
| 7 | Union-field resolution heuristic | N/A (out of scope) | Not a structuring concern. |

**Summary**: this option **directly and specifically** addresses the flaw the ticket itself most closely
associates with it (flaw #4/#10, structuring's known-weak input classes) and partially helps flaw #1's
structuring half — every other High-impact flaw is out of its declared scope, genuinely, not by omission.

### Risk

**Low technical risk, real algorithmic-choice risk.** Blast radius is small and contained (rank 5
subsystem, already reasonably tested, already isolated via `BlockCopy`), which makes this one of the
safer options to attempt in isolation. The real risk is algorithmic: DREAM's node-splitting trades gotos
for code duplication (ticket 0021 source 7's own tradeoff, echoed in the flaw doc's option 1 con); SAILR's
compiler-aware inversion is GCC-pattern-specific and its generality across other compilers/optimization
levels is unproven outside its own paper's Debian-package corpus; and Cifuentes' own foundational framing
(source 6) is explicit that irreducible control flow is not a fully solvable problem, only a better- or
worse-approximated one. Whichever algorithm is chosen, REQ-OUT-3's "no regression vs. baseline" framing
(rather than "eliminate gotos entirely") is the correctly-scoped bar — this document does not recommend a
specific algorithm, consistent with ticket 0021's own explicit deferral of that choice.

### First "most basic decompiler features" slice (guidance for epic 0024)

Because this option is narrow, its "first slice" is close to its whole scope: instrument current
goto-fallback frequency across the `datatests` corpus as a baseline (flaw #4's own improvement option 2,
and directly reusable as the "before" side of the Foundation Layer's golden-diff for this specific
subsystem), then prototype the chosen alternative algorithm against a handful of the corpus's
already-known-weak scenarios (irreducible/loop-unswitched cases) before attempting a full cutover. This
slice is naturally **combinable** with whichever of Options 1/2/3 is chosen for the rest of the codebase —
it does not require choosing among them first.

---

## Comparison Matrix

### Options × MUST-requirements

Legend: ✅ satisfies · 🟡 partially/conditionally satisfies (real risk or dependent on execution/later
design choice) · ❌ does not satisfy / not addressed by this option alone · N/A (reason) genuinely out of
scope for that option.

| REQ (MUST) | Opt 1: In-place | Opt 2: New IR/AST | Opt 3: Staged shim | Opt 4: Structuring swap |
|---|---|---|---|---|
| REQ-LANG-1 (template printing) | ✅ | ✅ (strongest fit) | 🟡 (inherits 1 or 2) | N/A (out of scope) |
| REQ-LANG-4 (`volatile` keyword) | ✅ | ✅ | 🟡 (inherits) | N/A (out of scope) |
| REQ-LANG-5 (composable qualifiers) | 🟡 | ✅ (lower audit risk) | 🟡 (inherits) | N/A (out of scope) |
| REQ-LANG-6 (no `ProtoModel` regression) | ✅ | ✅ | 🟡 (inherits; also the one option with any wire touch at all) | ✅ (untouched) |
| REQ-LANG-9 (recursive/forward-declared composites) | ✅ | ✅ | 🟡 (inherits) | N/A (out of scope) |
| REQ-LANG-11 (namespace/scoping, symbol-level MUST) | ✅ | ✅ | 🟡 (inherits) | N/A (out of scope) |
| REQ-OUT-1 (cast safety floor) | ✅ | ✅ (best fit — single decision point) | 🟡 (inherits) | N/A (out of scope) |
| REQ-OUT-4 (merge safety floor) | 🟡 | 🟡 (upstream, untouched) | 🟡 (inherits) | N/A (out of scope) |
| REQ-OUT-6 (determinism) | ✅ (Foundation) | ✅ (Foundation; easier to hold post-transition) | ✅ post-cutover / 🟡 mid-transition (uniquely *stronger* verification) | ✅ (Foundation) |
| REQ-OUT-7 (diagnosability) | 🟡 | 🟡 (opportunity, not automatic) | 🟡 (inherits, small plus from flag metadata) | 🟡 |
| REQ-ARCH-1 (wire-protocol compat, no forced Java change) | ✅ (cleanest) | ✅ (cleanest) | 🟡 (**only option with any wire-visible change**) | ✅ (cleanest) |
| REQ-ARCH-2 (version handshake) | ✅ | ✅ | ✅ (self-funding, same touch-point as the flag) | N/A (out of scope) |
| REQ-ARCH-4 (independent unit-testability, Rank 1-4) | 🟡 (largest, riskiest lift) | 🟡 (helps printer only, ranks 1-3 untouched) | 🟡 (inherits + comparison-oracle bonus) | 🟡 (helps rank-5 structuring only) |
| REQ-ARCH-5 (single `Rule` testable) | 🟡 | ❌ (no help at all) | 🟡 (inherits + comparison-oracle bonus) | N/A (out of scope — different class hierarchy) |
| REQ-ARCH-6 (100% `datatests` byte-identical) | ✅ (Foundation) | ✅ (Foundation; harder to hold mid-rewrite) | ✅ (Foundation; **load-bearing as cutover gate**) | ✅ (Foundation) |
| REQ-ARCH-7 (golden-snapshot harness) | ✅ (Foundation) | ✅ (Foundation) | ✅ (Foundation) | ✅ (Foundation) |
| REQ-ARCH-8 (unify Rule/Action registration) | ✅ (smallest, most mechanical fix) | 🟡 (unrelated, parallel effort) | 🟡 (inherits) | N/A (out of scope) |
| REQ-ARCH-9/10/11 (preserve/extend extensibility) | ✅ | ✅ (arguably improved for output-language extensibility) | 🟡 (inherits) | N/A (out of scope) |
| REQ-ARCH-13 (preserve console-driver loop) | ✅ | ✅ | ✅ | ✅ |

**Row totals** (✅ / 🟡 / ❌ / N/A, out of 19): Opt 1 = 12/7/0/0 · Opt 2 = 13/5/1/0 · Opt 3 = 3/15/0/1\*
(\*Option 3's row-by-row count is dominated by "🟡 inherits," since most rows are properties of whichever
subsystem-level option it wraps — see narrative above; its two genuinely independent, option-specific
rows are REQ-ARCH-1, where it is uniquely weaker, and REQ-ARCH-2/REQ-ARCH-4/REQ-OUT-6-mid-transition, where
it is uniquely stronger) · Opt 4 = 3/2/0/14 (correctly reflecting its narrow, complementary scope).

### Options × High-impact flaws

| Flaw | Opt 1: In-place | Opt 2: New IR/AST | Opt 3: Staged shim | Opt 4: Structuring swap |
|---|---|---|---|---|
| #1 Rule/structuring ordering undeclared | 🟡 (declared metadata, design preserved) | ❌ (upstream, untouched) | 🟡 (inherits) | 🟡 (fixes structuring half only) |
| #2 Rules not unit-testable | 🟡 (hardest lift, no new seam) | ❌ (no help) | 🟡 (inherits + oracle bonus) | N/A (out of scope) |
| #3 Extension model split | ✅ (small mechanical fix) | ❌ (untouched, parallel effort needed) | 🟡 (inherits) | N/A (out of scope) |
| #4 Test-coverage gaps | ✅ (Foundation) | 🟡 (Foundation + new printer coverage; SSA/rule gaps remain) | ✅ (Foundation; **forced** by cutover gating — strongest) | 🟡 (Foundation + new structuring coverage only) |
| #5 `TypeFactory` split-brain | 🟡 (smaller in-place mitigation only) | ❌ (untouched) | 🟡 (inherits) | N/A (out of scope) |
| #6 `Funcdata` god object | 🟡 (large, hot-path-risky lift) | ❌ (unchanged; one more consumer added) | 🟡 (inherits; oracle bonus *if* staged) | N/A (out of scope) |
| #7 Union-field resolution heuristic | 🟡 (in-place consistency checks) | ❌ (untouched) | 🟡 (inherits) | N/A (out of scope) |

**Row totals** (✅ / 🟡 / ❌ / N/A, out of 7): Opt 1 = 2/5/0/0 · Opt 2 = 0/2/5/0 · Opt 3 = 1/6/0/0 ·
Opt 4 = 0/2/0/5.

### Reading the matrices together

No option is a strict superset of another, which is itself informative:

- **Option 1** is the broadest, most evenly-distributed answer — it is the only option with a non-zero ✅
  on every single High-impact flaw row that's in scope for *any* option (flaws #3 and #4), and it never
  scores ❌ anywhere, because by construction it touches (at least partially) every subsystem a flaw lives
  in. Its cost is that nearly everything is 🟡 rather than ✅ — real progress everywhere, full resolution
  nowhere, on exactly the hardest problems (flaws #1, #2, #6 — rule ordering, rule testability, and the
  `Funcdata` god object, which are also the three highest-ranked risks in the sequencing table).
- **Option 2** has the strongest MUST-requirement profile for output-facing work (REQ-LANG-1/5, REQ-OUT-1,
  REQ-ARCH-9/10/11) but the weakest flaw-table profile of the four real options (five ❌s) — it is a
  genuine, well-targeted fix for the printer-quality flaws (#14 in 0014, the templates blocker) but
  provides **zero** help for the flaws the flaw index itself ranks highest-severity and highest-refactor-
  risk (rule ordering, rule testability, the type-system split-brain, the `Funcdata` god object). Choosing
  Option 2 as the *only* architectural move would leave the codebase's most-flagged risks completely
  unaddressed.
- **Option 3** is not a competitor to Options 1/2 on either table in the way its row totals might suggest
  at a glance — its huge 🟡 count is an artifact of "inherits from whichever subsystem it wraps," not a
  genuine finding about the option itself. Its real, independent signal is: uniquely weaker on REQ-ARCH-1
  (the only option with any wire-protocol touch), uniquely stronger on flaw #4/REQ-ARCH-6 (test-coverage
  gaps — the cutover mechanism *forces* the discipline every other option only recommends), and uniquely
  stronger on REQ-ARCH-4/5's testability story via the "old implementation as comparison oracle" property.
  This makes Option 3 read less like "option 3" and more like **a rollout discipline that materially
  de-risks whichever of Option 1 or Option 2 is chosen for the underlying work** — which is the same shape
  of judgment call this document made about the Foundation Layer, but for a genuinely separable, optional
  layer rather than a mandatory prerequisite. Unlike the Foundation Layer, Option 3 is not required by
  Options 1/2/4 — it is an available *strengthening* of any of them, at the real cost (REQ-ARCH-1 pressure,
  double-maintenance burden) documented above.
- **Option 4** is correctly read as a complement, not a competitor — its N/A-heavy rows are a feature of
  its honestly narrow scope, not a weakness relative to the other three. It is the only option that
  directly, specifically answers REQ-OUT-3 (goto-minimization) better than any combination of the other
  three could without also adopting it, and it is combinable with whichever of Options 1/2/3 is chosen for
  everything else.

**What this implies for ticket 0023, stated plainly since this document's matrix is meant to make a
direction apparent even though the decision itself belongs to 0023:** the data does not point to any
single option as sufficient alone. Option 1 is the only option that touches every High-impact flaw at all,
which matters given the flaw index's own severity ranking; Option 2 is the strongest available answer to
the requirements set's output-fidelity/template MUSTs specifically, which Option 1 alone struggles with
(REQ-LANG-5's audit risk, the closed printer switch); and Option 3's cutover discipline is the strongest
available answer to REQ-ARCH-6/flaw #4 specifically, which is the requirement/flaw pair the requirements
index itself flags as "a near-zero-regret first step regardless of which architecture option is chosen."
A composite reading — Option 1's subsystem-by-subsystem fixes for the flaws it uniquely reaches (#1, #2,
#3, #5, #6, #7), Option 2's printer/IR work for the templates/qualifier/cast-unification MUSTs it uniquely
reaches best, optionally wrapped in Option 3's staged-cutover discipline for whichever of those carries
the highest regression risk, plus Option 4 as an independent, combinable structuring-algorithm track — is
the shape the evidence in this document points toward, without this document making that call for 0023.

## Status

This document is complete. Ticket 0023 (decision record) is the next step and is explicitly out of scope
here — this document lays out options and comparison data only.
