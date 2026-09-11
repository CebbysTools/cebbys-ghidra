# Prior Art: External Decompilers & Academic Literature

Epic: [`.tasks/0020`](../../.tasks/0020-epic-solution-research-and-selection.md) · Ticket:
[`.tasks/0021`](../../.tasks/0021-task-research-prior-art-external-decompilers.md) · Root:
[`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md)

## Purpose and method

This document surveys how comparable problems — P-code/IR-to-C structuring, type recovery,
templates/generics handling in decompiled output, and pipeline/extension architecture — are solved
by decompilers and academic work outside Ghidra, as grounding for the architecture options in ticket
0022 and the decision record in 0023. Sources were located and read via live web search/fetch (not
recalled from memory); every link below was verified reachable during this research pass (September
2026). Per the ticket's scope constraint: **no code was copied from any source** — only
architectural/algorithmic ideas are referenced, and the license of each referenced project/paper is
noted so 0022/0023 can flag anything that would require attribution or would be unsafe to lift code
from even informally.

Terminology follows [`../00-overview/glossary.md`](../00-overview/glossary.md). Relevance notes cite
`REQ-<DOMAIN>-<n>` IDs from [`../03-requirements/00-index.md`](../03-requirements/00-index.md) and/or
numbered flaw entries from [`../02-flaws/00-index.md`](../02-flaws/00-index.md).

---

## 1. RetDec (Avast)

**Link:** https://github.com/avast/retdec

RetDec is a retargetable decompiler that lifts machine code to **LLVM IR** — each Capstone-disassembled
instruction is expanded into a template LLVM IR sequence, and the resulting module is then run through
LLVM's own (independently-tested, off-the-shelf) optimization passes before a final C/Python-like emitter
prints it. This is architecturally the opposite of Ghidra: instead of a bespoke ~500-class `Rule`/`Action`
catalog hand-built into one C++ codebase, RetDec reuses a mature, separately-maintained optimizer whose
individual passes are independently unit-tested via LLVM's own `opt`/FileCheck tooling. RetDec also treats
C++ RTTI/vtable class-hierarchy reconstruction and multi-vendor (GCC/MSVC/Borland) symbol demangling as a
distinct, named decompilation stage feeding the type/print layers, rather than folding it into general type
propagation. License: MIT (with a zlib/libpng-licensed vendored `PeLib` and other third-party components
listed separately).

**Relevance:** Flaw #2 (no per-`Rule` unit testability) and REQ-ARCH-5 (a single `Rule` testable via
synthetic P-code) — RetDec's reliance on LLVM's pass-level test infrastructure is the concrete industry
counter-example to "individual transformation rules cannot be tested in isolation." Also relevant to
flaw #8 / REQ-LANG-1 (template/RTTI-derived output): RetDec's separate, named "recover C++ class hierarchy"
stage is a useful architectural comparison for how Ghidra's `GnuDemangler`-parsed template data could be
plumbed through as its own stage rather than blocked on `DemangledTemplate.getDataType()`.

---

## 2. angr / pypcode

**Links:** https://github.com/angr/pypcode · https://github.com/angr/angr

`pypcode` is a Python binding to Ghidra's own SLEIGH library, letting any Python tool disassemble and lift
to P-code without going through Ghidra's Java client or wire protocol at all — it was built primarily to
feed angr, which lifts to its own IR (VEX or, via pypcode, P-code) and then converts basic blocks into
angr's "AIL" SSA-style IR for its independent decompilation pipeline (structuring, simplification, and the
SAILR passes — see source 9 below — all run as ordered analyses on AIL, not on raw P-code). This is a
concrete existence proof that P-code can be consumed as a standalone library decoupled from a specific
client process. License: `pypcode`'s own code is 2-Clause BSD; the wrapped SLEIGH library and processor
definitions are Apache 2.0 (inherited from Ghidra itself, so already compatible). `angr` itself is
2-Clause BSD.

**Relevance:** REQ-ARCH-1/2/3 (wire-protocol compatibility bar, real `registerProgram` version handshake,
single-sourced `ElementId` tables) — `pypcode` demonstrates that decoupling a P-code frontend from its
original host process is tractable and already done once against the very same SLEIGH library Ghidra ships,
which is a useful existence proof (not a direct solution, since Ghidra's coupling problem is Java↔C++ over
a stateful synchronous callback protocol, not just "lift P-code standalone"). Also touches flaw #3
(extension model split) since angr's AIL analyses register through one uniform pass-manager-style API
rather than two incompatible mechanisms.

---

## 3. Reko decompiler

**Link:** https://github.com/uxmal/reko

Reko is an open-source, C#/.NET decompiler with an explicit three-tier split: **front ends** (per-format,
per-architecture lifters — x86, ARM, MIPS, PowerPC, RISC-V, SPARC, 68k, and more, each a separately
pluggable module), a shared **core decompiler engine**, and **back ends** (CLI, Windows Forms GUI, an
in-progress cross-platform Avalonia GUI, plus an ASP.NET front end). Public project documentation is thin
on the specific structuring algorithm and type-inference internals (this research did not clone or read
Reko's source, consistent with the "ideas not code" scope), so the transferable idea here is architectural
rather than algorithmic: front-end/core/back-end modularity with per-architecture plugin isolation, as a
comparison point for how thin such a split can be made in practice. License: GPL-2.0 — notably copyleft,
which matters if 0022/0023 ever considers vendoring or closely mirroring any Reko-derived design in a way
that could be construed as a derivative work; referencing the architectural idea alone carries no such risk.

**Relevance:** REQ-ARCH-9/10/11 (preserve/extend architecture and output-language extensibility) and
REQ-ARCH-4 (independent unit-testability for Rank 1-4 subsystems) — Reko's front-end/core/back-end
separation is a smaller, more modular restatement of what Ghidra's `Architecture`/`PrintLanguage`
abstractions already attempt, useful as a "how thin can this split practically be" reference point rather
than a source of new algorithmic ideas.

---

## 4. Hex-Rays public design talks — microcode/ctree pipeline

**Links:**
- https://i.blackhat.com/us-18/Thu-August-9/us-18-Guilfanov-Decompiler-Internals-Microcode-wp.pdf
  (Ilfak Guilfanov, "Decompiler Internals: Microcode", Black Hat USA 2018 whitepaper)
- https://recon.cx/2018/brussels/resources/slides/RECON-BRX-2018-Decompiler-internals-microcode.pdf
  (companion RECON 2018 slides)
- https://hex-rays.com/blog/spying-on-decompiler-internals-the-hex-rays-microcode-viewer (Hex-Rays
  blog, 2025, on the IDA 9.2 Microcode Viewer)
- https://www.elastic.co/security-labs/introduction-to-hexrays-decompilation-internals (third-party
  technical explainer citing the Hex-Rays SDK docs, useful for stage names)

Hex-Rays (IDA) has no public source, but its published design talks describe microcode as passing through
**nine explicitly named, ordered maturity levels** — `MMAT_GENERATED` (direct assembly translation),
`MMAT_PREOPTIMIZED`, `MMAT_LOCOPT` (local optimization), `MMAT_CALLS`, `MMAT_GLBOPT1`–`MMAT_GLBOPT3`
(global optimization), and `MMAT_LVARS` (local-variable allocation) — before a structurally distinct
**ctree** (a heterogeneous statement/expression AST, `cinsn_t`/`cexpr_t` under a common `citem_t`) is
built for final output. Every optimization pass is pinned to a maturity level, and the IDA SDK/plugin API
lets third-party code query and hook at a specific level. This is the sharpest available point of contrast
with Ghidra's current model, where `Action`/`Rule` ordering is a property of call-site sequence inside one
large C++ function rather than a named, queryable stage. No source code is available — everything here is
design/architecture description from public talks and blog posts, copyright Hex-Rays; referenced as
architectural idea only, closed-source commercial product.

**Relevance:** This is the single most directly actionable comparison for **flaw #1** (rule/block-structuring
ordering is real, undeclared, only empirically stable) and REQ-ARCH-8/8a (unify `Rule`/`Action` registration
on `CapabilityPoint`, with a later declarative `ModelRule`-style layer) — Hex-Rays' named-maturity-level
model is direct precedent for making Ghidra's implicit ordering an explicit, checkable, queryable property
of each `Rule`/`Action` rather than emergent behavior of `oppool1`/`oppool2` construction order in
`coreaction.cc`.

---

## 5. Snowman (formerly SmartDec)

**Link:** https://github.com/x64dbg/snowman (canonical upstream now largely dormant; originally
https://github.com/yegord/snowman)

Snowman targets x86/AMD64/ARM(experimental MIPS), and ships as a standalone GUI, CLI, and plugins for IDA,
radare2, and x64dbg. It is one of the few fully open-source, non-LLVM-based native decompilers with its
own independent structuring implementation (inherited from the earlier SmartDec project), making it useful
purely for structuring-algorithm variety as the ticket requested — though public documentation of its
specific algorithm is thin, and the repository has been in a "LOOKING FOR MAINTAINER" state for several
years with open issues accumulating unaddressed. License: GPLv3-or-later for Snowman's own code, aggregating
several GPL/LGPL-licensed third-party components (Capstone, LLVM's disassembler bits, libudis86, etc.) —
copyleft, referenced for architectural ideas only.

**Relevance:** REQ-OUT-3 (goto-minimization, no regression vs. `datatests` baseline) and flaw #10
(structuring known-weak input classes, correctness-preserving fallback only via goto, no readability
recovery) — Snowman's stalled maintenance state is itself a useful negative data point: a structuring
engine without ongoing characterization-test investment (the same investment REQ-ARCH-6/7 calls for)
stagnates and accumulates unfixed structuring gaps over time, which is an argument for treating the
golden-snapshot harness as protective infrastructure, not just a nice-to-have.

---

## 6. Cifuentes, "Reverse Compilation Techniques" (PhD thesis, QUT, 1994)

**Link:** https://www.phatcode.net/res/228/files/decompilation_thesis.pdf (public mirror); canonical
record at https://eprints.qut.edu.au/36820/

Cristina Cifuentes' foundational thesis established the three-phase decompiler architecture — a
machine-specific **front end** that lifts to an intermediate representation, an architecture-independent
**analysis** phase (control-flow, data-flow, and type analysis), and a **back end** that emits high-level
code — that essentially every decompiler surveyed here, including Ghidra, still follows structurally.
It introduces interval-based control-flow structuring for reducible graphs (a direct conceptual ancestor
of Ghidra's own `CollapseStructure`) and an early type-recovery-from-usage approach, and is explicit that
irreducible/unstructured control flow is an inherent limitation of interval-based structuring, not a
solvable edge case. Academic thesis, standard copyright, publicly mirrored PDF; referenced for ideas only.

**Relevance:** Foundational context for nearly the whole requirements set, but most concretely for
REQ-OUT-3 and flaw #10 — the thesis's own acknowledgment that goto fallback is inherent to interval-style
structuring frames why REQ-OUT-3 is deliberately scoped as "no regression vs. baseline" rather than
"eliminate gotos": the historical literature treats zero-goto output as a genuinely hard, separately-solved
problem (see sources 7 and 9), not an oversight in Ghidra's design.

---

## 7. "No More Gotos" — DREAM (Yakdan, Eschweiler, Gerhards-Padilla, Smith — NDSS 2015)

**Links:** https://www.ndss-symposium.org/ndss2015/ndss-2015-programme/no-more-gotos-decompilation-using-pattern-independent-control-flow-structuring-and-semantics/
(NDSS program page) · PDF: https://net.cs.uni-bonn.de/fileadmin/ag/martini/Staff/yakdan/dream_ndss2015.pdf

DREAM introduces a **pattern-independent** structuring algorithm that, unlike pattern-matching structurers
(including Ghidra's `CollapseStructure`, which falls back to `goto` whenever no known collapse pattern
matches an edge), uses controlled node-splitting/duplication plus a semantics-preserving condition-based
refinement step to recover structured control flow for essentially any reducible — and most irreducible —
CFGs, at the cost of some code size growth from duplication. The paper reports zero goto statements across
its evaluation corpus versus thousands for then-current Hex-Rays and Phoenix. Published at NDSS (Internet
Society copyright), publicly available PDF; referenced for the algorithmic idea only, no code.

**Relevance:** Directly REQ-OUT-3 (goto-minimization) and flaw #10 (structuring known-weak input classes,
correctness-preserving fallback only via goto, no readability recovery) — the concrete idea worth carrying
into ticket 0022 is controlled node-splitting as a *second* fallback tier ahead of Ghidra's current binary
choice of "structure via a matched pattern, or fall back straight to goto," which could reduce Ghidra's
goto rate without a full structuring-engine rewrite.

---

## 8. DREAM++ — "Helping Johnny to Analyze Malware" (Yakdan, Dechand, Gerhards-Padilla, Smith — IEEE S&P 2016)

**Links:** Slides: https://www.ieee-security.org/TC/SP2016/slides/yakdan.pdf · PDF:
https://net.cs.uni-bonn.de/fileadmin/ag/martini/Staff/yakdan/dream_oakland2016.pdf

DREAM++ extends DREAM (source 7) with three explicit categories of **readability-oriented,
semantics-preserving transformations** layered strictly after goto-free structuring: expression
simplification (boolean/condition simplification), further control-flow simplification, and
semantics-aware variable naming — and validates the result with a human user study (malware analysts
solved roughly 2-3x more tasks with DREAM++ than with Hex-Rays or plain DREAM), not just an automated
metric. The paper's contribution most relevant here is architectural: it treats "make the structurally
correct output more readable" as its own separately-categorized transform pass, distinct from the
correctness-establishing structuring pass that precedes it. Published at IEEE S&P (IEEE copyright),
publicly hosted PDF; referenced for the taxonomy/methodology idea only.

**Relevance:** Flaw #18 (Rule catalog's only taxonomy, the `group` tag, can't separate correctness-required
rules from readability-only rules) and REQ-OUT-2/REQ-OUT-3/REQ-OUT-7 (cast readability ceiling,
goto-minimization, diagnosability) — DREAM++'s three-category taxonomy of post-structuring transforms is a
directly transferable classification scheme for what REQ-ARCH-8a's proposed declarative rule layer could
tag rules by (correctness vs. readability), and its human-study methodology is a precedent worth flagging
for 0023: REQ-OUT-3's "no regression" bar could eventually be validated with a small human study, not only
an automated goto-count diff.

---

## 9. SAILR — "Ahoy SAILR! There is No Need to DREAM of C" (Basque et al. — USENIX Security 2024)

**Links:** https://www.usenix.org/conference/usenixsecurity24/presentation/basque · PDF:
https://www.usenix.org/system/files/usenixsecurity24-basque.pdf

SAILR is a 2024 **compiler-aware** structuring algorithm, implemented as a set of optimization passes in
angr's open-source decompiler, that takes a fundamentally different strategy from DREAM's node-splitting:
it identifies the specific goto-inducing CFG transformations that GCC (and, the paper shows, other
compilers similarly) apply during code generation — such as loop-condition duplication and jump
threading — and precisely *inverts* them, producing structured output whose shape maps much more closely
back to the original source than either pattern-matching structurers or DREAM's approach. Unusually for
this survey, the paper directly benchmarks against **Ghidra's own decompiler by name**, alongside Hex-Rays
and angr itself, across 26 real Debian C packages, and reports Ghidra's measured goto-rate and
structural-similarity numbers as part of its comparison. SAILR's passes and its evaluation harness
(`sailr-eval`) are open source under angr's standard 2-Clause BSD license.

**Relevance:** The single most directly on-point external source for REQ-OUT-3 (goto-minimization: no
regression vs. an instrumented `datatests` baseline) precisely because it already measured Ghidra's current
goto-rate/structural-fidelity as an external baseline — ticket 0022/0023 should pull SAILR's published
Ghidra numbers as an independent sanity check on whatever internal baseline the `datatests`-based harness
produces under REQ-ARCH-6/7. Also bears on flaw #10: "compiler-aware inversion" is a third distinct
structuring strategy (alongside Ghidra's pattern-collapse and DREAM's node-splitting) that 0022's
architecture comparison should list explicitly rather than treating structuring improvement as a single
axis.

---

## 10. DIRTY — "Augmenting Decompiler Output with Learned Variable Names and Types" (Chen, Lacomis, Schwartz, Le Goues, Neubig, Vasilescu — USENIX Security 2022)

**Links:** https://www.usenix.org/conference/usenixsecurity22/presentation/chen-qibin · PDF:
https://www.usenix.org/system/files/sec22-chen-qibin.pdf · Code:
https://github.com/CMUSTRUDEL/DIRTY (MIT license)

DIRTY is a transformer-based model that reads *already-produced* decompiler output (with the original
variable names/types erased, as Ghidra or Hex-Rays would emit it) and predicts human-like variable names
and refined types, trained on a large GitHub-mined dataset of human-written C paired with its own
decompiled form (DIRT, 75K+ programs). It recovers the original name 66.4% and the original type 75.8% of
the time in the paper's evaluation. Architecturally, the important point is that DIRTY operates as an
**external, decompiler-agnostic post-processing stage** consuming a decompiler's finished output, rather
than modifying the decompiler's internal type-propagation/print pipeline — proving that meaningful
type/name-quality improvement doesn't strictly require surgery inside a `Datatype`/`typelock`-style
internal type system. License: MIT (both code and released dataset).

**Relevance:** REQ-LANG-2/REQ-LANG-3 (first-class `Datatype`-level template/generic representation; detect
monomorphized-function families) and REQ-OUT-5 (merge readability ceiling without relaxing REQ-OUT-4) —
DIRTY is evidence that a statistically-driven, post-hoc type/name-refinement layer is a viable,
already-proven pattern that 0022 should weigh as a lower-risk complement to (not replacement for) direct
`Datatype`-model changes for some REQ-LANG items. Important caveat for 0022/0023: DIRTY explicitly does
not attempt template/generic recovery, so it cannot substitute for the demangler-plumbing work that
REQ-LANG-1/REQ-LANG-2 actually require — it is relevant to the type/name-quality problem generally, not to
the templates-specific blocker (flaw #8).

---

## 11. Ghidra's own upstream — decompiler modernization discussion (or lack thereof)

**Links:** https://github.com/NationalSecurityAgency/ghidra/issues/8788 ("Feature Request: Expand
decompiler interface") · https://github.com/NationalSecurityAgency/ghidra/discussions/5621
("Improving the Decompiler output")

Issue #8788 (open at time of writing, `Status: Triage`, assigned to `caheckman` — the decompiler's primary
upstream maintainer) requests almost exactly the kind of staged decoupling this refactor is investigating:
splitting disassembly/P-code generation, P-code optimization/control-flow recovery, and P-code-to-C
translation into independently addressable stages, so that extensions can intercept/modify P-code before C
generation, or re-run only the print stage without a full pipeline re-run. As of this research there is no
visible maintainer commitment to a redesign, no linked design doc, and no separate roadmap discussion —
discussion #5621 is a narrower, incremental thread (compound-assignment printing, dead-code display in
output, jump-table recovery limits) with community-submitted patches referenced but no architectural
statement attached. No evidence of a formal upstream "decompiler modernization" RFC or design doc was
found anywhere in the `NationalSecurityAgency/ghidra` repository's issues or discussions as of September
2026. This is GitHub discussion content, not code; no license concern beyond standard GitHub ToS, and
Ghidra itself is Apache 2.0.

**Relevance:** Directly informs REQ-ARCH-4/REQ-ARCH-5 (independent unit-testability, single-`Rule`
testability) and the root ticket's explicit goal of not duplicating or conflicting with upstream direction —
issue #8788 is functionally the same request as REQ-ARCH-4/5, filed independently by an upstream user
against the real `NationalSecurityAgency/ghidra` repository, which is a positive signal that this
refactor's staged-decoupling direction has organic upstream demand rather than fighting a known contrary
plan. **Caveat for 0022/0023:** "no roadmap found" is a time-bound observation, not a permanent one — this
should be re-checked immediately before implementation work begins in epic 0024, since upstream could
publish a conflicting design between now and then.

---

## Closing synthesis

**The common thread across prior art that's most applicable here is: make the pipeline's stage boundaries
explicit, named, and independently testable before touching any individual algorithm inside a stage.**
Every source that offers a genuine architectural contrast to Ghidra converges on this same point from a
different angle — Hex-Rays' nine named microcode maturity levels (source 4), RetDec's reuse of LLVM's
independently-testable pass pipeline (source 1), Reko's front-end/core/back-end module split (source 3),
and even angr's separation of P-code lifting (`pypcode`) from its own AIL analysis passes (source 2) — and
Ghidra's own upstream maintainers are independently being asked for the identical thing in issue #8788
(source 11), unprompted by this refactor effort. This is not a coincidence: it is the same conclusion
flaw #1 (undeclared rule/structuring ordering), flaw #2 (no per-`Rule` unit testing), flaw #3 (two
incompatible extension mechanisms), and REQ-ARCH-4/5/6/7/8/8a already independently arrived at from
Ghidra's own code. The specific *algorithmic* choices surveyed here — DREAM's node-splitting vs. SAILR's
compiler-aware transformation-inversion for structuring (sources 7, 9); DIRTY's external learned-refinement
layer vs. deeper `Datatype`-model surgery for type/name quality (source 10) — are real, useful, and should
stay on the table for 0022's comparison matrix, but they are all **secondary** decisions: each becomes
dramatically safer to evaluate, swap, or run in parallel once Ghidra's own Action/Rule/structuring/print
pipeline has explicit, addressable stage boundaries with characterization tests pinned at each boundary
(the REQ-ARCH-6/7 golden-snapshot harness, already flagged in the requirements index as a near-zero-regret
first step). Recommendation for 0022: treat "introduce named, ordered, independently-testable pipeline
stages" as the load-bearing architectural decision to make first, and frame the structuring-algorithm and
type-recovery-strategy choices as pluggable strategies evaluated *within* that staged architecture rather
than as competing all-or-nothing rewrites of the whole decompiler.
