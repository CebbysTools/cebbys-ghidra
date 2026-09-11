# Requirements: Output Fidelity & Readability

Produced by [`.tasks/0018`](../../.tasks/0018-task-requirements-output-fidelity-ux.md), part of the
requirements-definition epic [`.tasks/0016`](../../.tasks/0016-epic-requirements-definition.md), under
root [`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md). Format follows 0016's shared
contract: each requirement gets a stable `REQ-OUT-<n>` ID, an RFC-2119 priority (MUST/SHOULD/MAY),
a one-line statement, rationale traced to a specific [`epic 0011`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md)
flaw entry, and — where feasible — an acceptance-test sketch. Shared terms:
[`../00-overview/glossary.md`](../00-overview/glossary.md).

Scope, per the parent ticket: the quality of the generated C itself — correctness fidelity (never
misrepresent the binary's behavior) and readability (matches what a skilled reverse engineer would
write) — independent of the templates/generics requirements covered by
[`rich-language-features.md`](rich-language-features.md) (ticket 0017) and the extensibility/API
requirements covered by [`extensibility-api-testability.md`](extensibility-api-testability.md)
(ticket 0019).

## Why "correctness" and "readability" are treated as one property, not two goals in tension

Every requirement below is a version of the same shape: the decompiler must not say *less* than the
truth (a correctness floor — omitting information that changes what a reader believes the binary
does) and should not say *more* than the truth needs (a readability ceiling — noise that a skilled
human wouldn't write). Both directions are testable, but only the floor is a MUST: getting the floor
wrong is a bug, getting the ceiling wrong is a quality regression. This distinction is used
consistently to assign priority below, and it is itself motivated by
[`printer-output-quality.md`](../02-flaws/printer-output-quality.md) Flaw 1 (0014), which shows the
codebase does not currently have one shared definition of "necessary" to check either direction
against.

## Requirements

### REQ-OUT-1 — Cast safety floor: a semantically-required cast MUST NOT be omitted

**Priority:** MUST

**Statement:** For every point in the printed C where an implicit C conversion would compute a
different value/width/signedness than the P-code operation actually being represented, the printer
MUST emit an explicit cast (or otherwise disambiguate, e.g. via a suffix) — regardless of which of
the two independent cast-decision entry points (producer-side `ActionSetCasts::castStandard`,
consumer-side `PrintC::isZextCast`/`isSextCast`/`isSubpieceCast`) would be responsible for it today.

**Operational definition of "necessary for correctness":** a cast at a given expression position is
*correctness-necessary* iff parsing and evaluating the printed C text under the output language's own
standard conversion rules, starting only from the printed literal/operand types, would compute a
different runtime value (bit pattern, sign, or width) than the P-code graph actually produces at that
point. This is a black-box, re-derivable test — it does not require trusting whichever internal
component made the decision, only the printed text and the language's own semantics — and it is
distinct from *readability-necessary* (REQ-OUT-2), which is a human-judgment property, not a
re-derivable one.

**Rationale:** [`printer-output-quality.md`](../02-flaws/printer-output-quality.md) Flaw 1 (0014)
establishes that cast decisions are split across two policy entry points — `ActionSetCasts` (inserts
real `CPUI_CAST`/`CPUI_PTRSUB` ops for correctness during simplification) and `PrintC`'s own
`isZextCast`/`isSextCast`/`isSubpieceCast` (decides whether an *existing* conversion op is *shown* as
a cast) — that "never compare notes." The flaw's own Impact §1 names the concrete failure mode this
requirement exists to close: when `option_hide_exts` triggers `isExtensionCastImplied` on an opcode
inside its closed eleven-opcode whitelist, "the expression's *actual* computed width is no longer
visible from the printed text, even though the extension genuinely happened in the p-code graph" —
described there as "the single biggest source of 'the decompiler silently promoted my variable'
confusion." That is precisely a correctness-floor violation under this requirement's definition: the
printed text, read at face value, computes a different value than the graph does.

**Acceptance-test sketch:** (1) Give `CastStrategy` (or its refactored successor) one queryable
decision method usable from both roles — `castForInsertion` and `castForDisplay` — per Flaw 1's own
improvement option 1, so the two current entry points become two callers of one shared answer instead
of two independent answers. (2) Add a debug/audit mode (Flaw 1's option 3) that, for every printed
function in the `datatests` corpus, re-runs the correctness-floor check above against every
`ZEXT`/`SEXT`/`SUBPIECE`/arithmetic-conversion op the consumer side chose to hide, and asserts zero
divergences. A regression is any corpus function where the audit mode finds a hidden or missing cast
at a position the operational definition above marks correctness-necessary.

---

### REQ-OUT-2 — Cast readability ceiling: no cast beyond what correctness or an idiomatic human write-up requires

**Priority:** SHOULD

**Statement:** The printer SHOULD NOT emit a cast at a position where REQ-OUT-1's correctness test
would pass without it, unless the cast matches a documented, closed set of "idiomatic human style"
exceptions (e.g. a cast a skilled reverse engineer would add anyway for clarity at a pointer
reinterpretation, even when the underlying conversion is a no-op).

**Rationale:** Same source finding as REQ-OUT-1 — [`printer-output-quality.md`](../02-flaws/printer-output-quality.md)
Flaw 1 (0014) — read from the opposite direction: because `isExtensionCastImplied`'s whitelist is a
hand-enumerated `switch` over eleven opcodes with `default: return false` (`cast.cc:290`), "any future
`Rule` or `TypeOp` that legitimately relies on integer promotion through some other opcode gets no
hiding at all (a false negative — an extra cast appears that a human wouldn't write)." That is a
concrete, source-identified readability-ceiling violation distinct from REQ-OUT-1's floor violation,
which is why this is tracked as its own requirement at SHOULD rather than folded into REQ-OUT-1's MUST
— an extraneous cast is a quality defect, not a misrepresentation of behavior.

**Acceptance-test sketch:** Extend the audit mode from REQ-OUT-1 to also flag the reverse divergence:
any cast printed at a position REQ-OUT-1's definition marks *not* correctness-necessary, cross-checked
against the closed exception list. Track the count of such "extraneous cast" flags per `datatests`
corpus run as a trend metric (see REQ-OUT-3 for the same trend-metric pattern applied to gotos) rather
than a hard gate, since the "idiomatic exception" set is a judgment call that will need tuning.

---

### REQ-OUT-3 — Goto-minimization target with a measurable baseline

**Priority:** SHOULD

**Statement:** Structuring changes MUST NOT increase the goto/fallback rate measured against the
existing `datatests` corpus baseline (this half is a MUST-not-regress gate), and the refactor SHOULD
work toward measurably reducing that rate over time on the same corpus (this half is the improvement
target, hence overall SHOULD).

**Rationale:** [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md)
Flaw 4 (0013) documents that `CollapseStructure` is "not a general structuring algorithm ... it is a
fixed set of shape-matching rules plus a goto/`while(true)`-fallback safety net," with irreducible CFGs,
overly-complex loop conditions, and unswitched/duplicated conditionals as documented weak classes —
and that "none of this is exercised by any per-shape unit test... whole-pipeline `datatests` diffing is
the only feedback loop." The flaw's own improvement option 2 names exactly this gap: "Instrument and
report goto-fallback frequency as a quality metric... to turn 'structuring is known-weak here' from a
source-comment-level claim into a measured one that can track regressions/improvements over time." This
requirement adopts that option directly as the measurement method, because — per
[`java-integration-testing.md`](../01-review/java-integration-testing.md) (0010) §4 — "there's no
coverage tracking how many of the 89 `datatests` programs currently require goto-fallback vs. clean
structuring, so a structuring regression that still produces *technically valid but uglier* C (more
gotos, worse loop shape) can pass every existing test" today.

**Acceptance-test sketch:** Add instrumentation at `selectGoto`/`TraceDAG::selectBadEdge`
(`blockaction.cc`) and at `BlockWhileDo::hasOverflowSyntax`'s `f_whiledo_overflow` flag to count, per
`datatests` file, (a) the number of edges demoted to `goto`, and (b) the number of loops forced into
`while(true){...; if(!cond) break;}` overflow syntax. Record this as a baseline table (one row per
corpus file) at the start of the refactor; any PR touching `CollapseStructure` or its `ruleBlockX`
catalog runs this instrumentation and fails CI if any file's goto/overflow count increases without an
explicit, reviewed waiver. A dashboard tracking the corpus-wide total over time operationalizes the
SHOULD-improve half.

---

### REQ-OUT-4 — Variable-merge safety floor: no silent over-merging of distinct source variables

**Priority:** MUST

**Statement:** The decompiler MUST NOT merge two `Varnode`s into one displayed `HighVariable` when
doing so changes what a reader would believe about the underlying program's data flow — i.e. when the
merge conflates values that the source program actually kept distinct in a way relevant to reading the
function's behavior. This applies specifically to *speculative* merges (opportunistic grouping,
e.g. `mergeByDatatype`); *forced* merges are out of scope here since they are already required for
correctness by construction (they exist to prevent register/stack-slot aliasing bugs, not to be
readability decisions).

**Rationale:** [`varnode-ssa-type-system.md`](../02-flaws/varnode-ssa-type-system.md) §4.1 (0012,
consolidated as flaw #12 in [`00-index.md`](../02-flaws/00-index.md)) is explicit that speculative
merges are "simply *abandoned*, with no dataflow change, if a `Cover` intersection is found" — the
*safe* half of the design — but that the real risk sits in `Symbol::isolate`, "an explicit user/
heuristic override to stop speculative auto-merging for a `Symbol` altogether — an acknowledgment that
the speculative heuristics sometimes produce a *plausible but wrong* grouping that a human has to
veto." The flaw document's own framing is decisive for this requirement's priority: "A *plausible but
wrong* grouping is worse than an obviously-failed one: it doesn't crash or warn, it presents as a
normal-looking single variable that actually elides a real distinction in the underlying SSA values" —
this is squarely the "silently produce wrong (not just ugly) output" category the flaw ticket asked
about, and severity was assessed there as Medium-High. It is elevated to MUST here because it is a
correctness-of-belief failure (the reader is given false information about the program with no visual
signal anything is uncertain), not a style issue.

**Acceptance-test sketch:** Ship [`varnode-ssa-type-system.md`](../02-flaws/varnode-ssa-type-system.md)
§4.1's own improvement option B first (instrument/log every speculative-merge acceptance and every
`isolate` veto across the `datatests` corpus and, ideally, a larger representative binary corpus) to
convert "sometimes produces a plausible-but-wrong grouping" from a design-comment-level claim into a
measured rate — mirroring the same instrument-before-redesign pattern used for REQ-OUT-3. Once a
baseline exists, a regression test asserts that a change to `Merge`'s speculative heuristics does not
increase the corpus-wide rate of `isolate`-worthy (i.e., manually vetoed or heuristically flagged)
merges. A hand-built minimal fixture (two SSA values with disjoint `Cover`s but a plausible shared
datatype, constructed the way [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md)
Flaw 2's proposed synthetic-`Funcdata` harness would) gives a fast, targeted regression check
independent of the full corpus.

---

### REQ-OUT-5 — Variable-merge readability ceiling: no silent under-merging that fragments one source variable

**Priority:** SHOULD

**Statement:** The decompiler SHOULD merge `Varnode`s that a skilled reverse engineer would recognize
as "the same variable" (e.g. a value copied through a register shuffle, or re-derived identically
across a loop back-edge with no semantic difference) rather than leaving them as separately-named
`HighVariable`s, provided doing so does not violate REQ-OUT-4's safety floor.

**Rationale:** Same source as REQ-OUT-4 —
[`varnode-ssa-type-system.md`](../02-flaws/varnode-ssa-type-system.md) §4.2 — describes the deliberate
counterweight: aliasing-driven `INDIRECT` guards are conservative by design, "trad[ing] wrong-output
risk for over-conservative output," and the review explicitly frames this as "not every 'best effort'
heuristic in this subsystem is a correctness risk; some are conservative-by-design precision costs
instead." This requirement exists so that the refactor does not treat REQ-OUT-4's safety fix as license
to relax merge precision arbitrarily (§4.2's own severity note: "any refactor 'improving' merge
precision by relaxing this conservatism without also strengthening `LoadGuard` refinement would
reintroduce real silent-wrong-output risk, not just restore lost precision") — i.e. under-merging is a
real readability cost, but the *sanctioned* way to reduce it is investing in `LoadGuard`/value-set-
analysis precision (§4.2 option A), not loosening the `Cover`-intersection safety check that REQ-OUT-4
depends on.

**Acceptance-test sketch:** Track, per `datatests` corpus function, the count of `HighVariable`s
displayed versus a hand-annotated "expected distinct source variables" count for a curated subset of
files (this is inherently a judgment call requiring human annotation, unlike REQ-OUT-3/REQ-OUT-4's
count-based metrics — flag this explicitly as harder to automate). A `LoadGuard` precision improvement
should reduce over-fragmentation on the annotated subset without increasing REQ-OUT-4's
speculative-merge-risk metric.

---

### REQ-OUT-6 — Deterministic output: same P-code input always produces byte-identical C text

**Priority:** MUST

**Statement:** For a fixed P-code graph, fixed type/symbol database state, and fixed decompiler
version/configuration, decompiling the same function MUST produce byte-identical printed C text on
every run — across repeated invocations within one process, across fresh process instances, and across
machines/platforms running the same decompiler build.

**Rationale:** This requirement is foundational rather than derived from a single flaw entry, but it is
directly motivated by the gap [`java-integration-testing.md`](../01-review/java-integration-testing.md)
(0010) §3.1/§4 identifies: "there is no test anywhere in this tree — `datatests`, `unittests`, or
`test.slow` — that pins the full, exact printed-C text of any function. Every `datatests` assertion is
a small regex over one or a few lines with an occurrence-count bound... nothing diffs a complete
function's output against a golden/expected file." Any future snapshot/golden-text testing strategy —
including the one this document's own acceptance-test sketches above lean on (REQ-OUT-3's per-file
counts, REQ-OUT-4's corpus-wide rates) — silently assumes determinism already holds; if it does not,
those tests would be measuring run-to-run noise rather than genuine regressions. It is also the
precondition the refactor needs before it can trust *any* characterization-test strategy built by
capturing today's output as a baseline and diffing the refactored pipeline against it — exactly the
sequencing risk [`00-index.md`](../02-flaws/00-index.md) (0011's consolidated index) names directly:
"SSA/heritage and the printer are the least-tested, most-coupled subsystems and should not be the first
to be touched without first adding characterization tests" (from the Refactor Sequencing Risk summary,
detailed in [`architecture-extensibility.md`](../02-flaws/architecture-extensibility.md), flaw #4).
Concrete non-determinism risks this requirement rules out, grounded in what the review/flaw docs
already surfaced: `TypeFactory` interning order or pointer-identity-derived iteration (any code path
touching raw pointer comparison rather than `compare()`/`compareDependency()`, per
[`varnode-ssa-type-system.md`](../02-flaws/varnode-ssa-type-system.md) §1.1-§1.6's note on duplicated
comparison logic), `ActionPool`'s opcode-bucket rule dispatch order interacting with unordered
containers, and any future parallelization of the `Action`/`Rule` pipeline (a plausible refactor goal)
that iterates rule results from concurrent workers without a deterministic merge order.

**Proposed verification method:** Extend the existing `datatests`/`FunctionTestCollection` machinery
(`testfunction.hh`, per 0010 §3.1) rather than inventing a parallel framework, since it already builds
synthetic binary images and drives `decompile`/`print C` through the same console-command interpreter a
human uses interactively:

1. **Golden/snapshot corpus.** Add a `<goldentext>` (or equivalent) assertion type alongside the
   existing `<stringmatch>` regex assertions that captures and compares the **full, exact** printed C
   text of a function against a checked-in golden file, rather than a partial regex. Seed this corpus by
   snapshotting current output for the existing 89 `datatests` files (giving broad coverage for free)
   plus new files targeting the non-determinism risks named above (heavy union/pointer-type mixing,
   large rule-pool fixed points, deeply-nested merge scenarios).
2. **Repeated-run diffing as the actual determinism check** (distinct from #1, which is a regression
   check against a fixed golden file and would not by itself catch two *wrong-but-consistent* runs
   agreeing with each other). Run each golden-corpus function N times (N ≥ 3, in fresh `ghidra_test`
   process instances to rule out any in-process cache warming effects) and assert byte-for-byte equality
   across all N outputs, independent of comparing against the golden file. This is the direct test of
   the requirement's actual claim ("same input always produces byte-identical output"); #1 alone only
   tests "output matches what it was last time," which is necessary but not sufficient.
3. **Cross-environment check**, lower priority / longer-horizon: run the same golden corpus across at
   least two platforms/build configurations (e.g. Linux and Windows CI runners, or single- vs.
   multi-threaded build flags if the refactor introduces parallelism) and assert the same byte-for-byte
   equality, since platform-dependent iteration order (e.g. hash-map iteration differing across STL
   implementations) is a classic, easy-to-miss determinism leak that a same-machine repeated-run test
   (#2) cannot catch by itself.

**Acceptance-test sketch:** A CI job that runs `ghidra_test datatests <goldentext-enabled files>` three
times per commit and fails on any byte-level diff between runs, plus a separate, slower nightly/
pre-merge job that diffs run output against the checked-in golden corpus from step 1 and reports a
line-level diff (not just a pass/fail count) on mismatch — closing the specific gap 0010 §3.1 names
("a failing `<stringmatch>` shows up only in the aggregate pass count, not as a diff").

---

### REQ-OUT-7 — Diagnosability: every fallback/give-up representation must be traceable to a specific decision

**Priority:** MUST

**Statement:** Whenever the decompiler emits a "give up" or degraded representation — a `goto`/
`BlockMultiGoto` fallback edge, `while(true){...; if(!cond) break;}` overflow-syntax loop, a raw
unsimplified cast the printer could not resolve to an idiomatic form, or an unresolved/unknown-type
variable — it MUST be possible for a developer (not necessarily an end user) to determine, after the
fact, *which* rule, heuristic, or structuring decision produced that fallback and, ideally, why the
preferred path was unavailable for this specific input. This is a developer-experience requirement as
much as an end-user one, per the parent ticket's own framing.

**Rationale:** [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md)
Flaw 1 (0013) is the direct root cause this requirement targets: rule and block-structuring ordering
"is real, undeclared, and only empirically patched" — "the *only* record of an ordering constraint is a
`//` comment at the call site — nothing a compiler, linker, or test enforces." Concretely, that same
document's item 3 shows `ruleBlockIfNoExit`/`ruleCaseFallthru` are "tried only after `collapseInternal`'s
main loop reaches a fixed point," with the comment "Applying IfNoExit rule too early can cause other
(preferable) rules to miss" — i.e. *today*, whether a given CFG shape gets clean structuring or a goto
fallback can depend on exactly this kind of undeclared firing-order interaction, and nothing records,
at the point a fallback is chosen, which of the ~162 `Rule`s or ~10 `ruleBlockX` structuring rules
almost-but-didn't-fire, or why. Undeclared ordering and non-diagnosable output are the same underlying
gap viewed from two directions: if the pipeline's own decision order isn't recorded anywhere
machine-checkable (Flaw 1's core finding), then by construction there is nothing to point to when
asked "why did this function get a goto here instead of a clean `if`?" This requirement is also
reinforced by [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md)
Flaw 5 (0013): the rule catalog's only taxonomy (`group`) cannot even separate "this rule is required
for correctness" from "this rule is readability-only," so today there is no way to know, from a
fallback alone, whether the *cause* was a correctness-driven refusal to structure (safe but goto-heavy)
or a missed readability opportunity (a bug worth fixing) — diagnosability requires both a decision trace
*and* enough classification metadata to interpret it.

**Acceptance-test sketch:** Two complementary mechanisms, both grounded directly in improvement options
already proposed in the source flaw documents rather than invented fresh:

1. **A decision-trace / provenance log**, extending
   [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md) Flaw 1's
   improvement option 2 (a confluence/idempotence self-check mode) and reusing the existing
   `count_tests`/`count_apply` rule-statistics affordance the same document's Open Questions note already
   exists (`action.hh:209-210`, surfaced via `print actionstats`): when `CollapseStructure` falls back to
   `goto`/overflow syntax for a specific block, or `ActionSetCasts` leaves a value un-cast where none of
   the standard rules applied, record which candidate rules were *tried and rejected* at that specific
   graph location (not just which fired), with enough context (op/block identity, rejecting rule name) to
   reconstruct the decision after the fact. Expose this via the existing console-debug machinery
   (`DecompileDebug`, per [`java-integration-testing.md`](../01-review/java-integration-testing.md) §1.3's
   description of the "decompiler debug" feature) rather than a new subsystem, so it plugs into
   infrastructure that already exists for diagnosing a specific bad decompilation.
2. **A regression test that asserts the trace exists and is non-empty for every fallback instance** in
   the goto/overflow-syntax instrumentation already required by REQ-OUT-3: for every edge REQ-OUT-3's
   instrumentation counts as a goto-fallback, assert the decision-trace log for that edge names at least
   one specific rejected structuring rule (not merely "no rule applied," which is not actionable) —
   turning "diagnosability exists" into a checkable property of the test corpus rather than a manual
   debugging affordance nobody is required to keep working.

---

### REQ-OUT-8 — Qualifier fidelity: known `volatile` (and, once representable, `const`) semantics must be visible in printed output

**Priority:** SHOULD

**Statement:** When the decompiler's internal model already knows a symbol is volatile
(`Symbol::isVolatile()`), the printed declaration SHOULD include the `volatile` keyword, not merely a
GUI-only syntax-highlight color. Once `const`-equivalent information becomes representable in
`Datatype` (out of scope for this document; see [`rich-language-features.md`](rich-language-features.md),
0017), the same requirement extends to `const`.

**Rationale:** [`printer-output-quality.md`](../02-flaws/printer-output-quality.md) Flaw 4 (0014)
verifies directly against source that "`volatile` is tracked, but the printer only ever uses it to pick
a syntax-highlight color, never to emit the keyword" — a direct search of `printc.cc`/`printc.hh` found
zero occurrences of the literal string `"volatile"`. The flaw document's own severity assessment calls
this "a real, demonstrable information-loss bug with a small, well-localized fix... unlike most of this
document's findings, this one is fixable in `printc.cc` alone without any type-system change" and
frames it as correctness-adjacent: "a reader of the printed C... has no way to know from the text that a
given access must not be reordered/cached/optimized away, even though the decompiler's own internal
model... already knows it." This is included here at SHOULD (not MUST) because, per that same document,
it is a readability/information-completeness gap rather than a case where the printed text computes a
provably wrong value (REQ-OUT-1's bar) — but it is flagged as a cheap, immediately actionable item
distinct from every other requirement in this document, which mostly require refactor-scale work.

**Acceptance-test sketch:** A `datatests` case with a symbol mapped onto a memory-mapped/volatile-flagged
address, asserting the printed declaration contains the literal `volatile` keyword token adjacent to the
declared type — directly analogous to the existing `<stringmatch>` pattern already used elsewhere in the
corpus (per [`java-integration-testing.md`](../01-review/java-integration-testing.md) §3.1's
`condconst.xml` walkthrough).

---

## Traceability summary

| ID | Priority | One-liner | Primary 0011 source |
|---|---|---|---|
| REQ-OUT-1 | MUST | Never omit a correctness-required cast | [printer-output-quality.md](../02-flaws/printer-output-quality.md) Flaw 1 (0014) |
| REQ-OUT-2 | SHOULD | Never print a cast beyond correctness/idiom | [printer-output-quality.md](../02-flaws/printer-output-quality.md) Flaw 1 (0014) |
| REQ-OUT-3 | SHOULD | No goto-rate regression vs. measured `datatests` baseline | [rule-engine-block-structuring.md](../02-flaws/rule-engine-block-structuring.md) Flaw 4 (0013) |
| REQ-OUT-4 | MUST | Never silently over-merge distinct source variables | [varnode-ssa-type-system.md](../02-flaws/varnode-ssa-type-system.md) §4.1 (0012) |
| REQ-OUT-5 | SHOULD | Don't leave one source variable fragmented across displayed names | [varnode-ssa-type-system.md](../02-flaws/varnode-ssa-type-system.md) §4.2 (0012) |
| REQ-OUT-6 | MUST | Same P-code in → byte-identical C text out, always | [00-index.md](../02-flaws/00-index.md) flaw #4 / [java-integration-testing.md](../01-review/java-integration-testing.md) (0010) §3.1 |
| REQ-OUT-7 | MUST | Every fallback representation traceable to a specific rejected decision | [rule-engine-block-structuring.md](../02-flaws/rule-engine-block-structuring.md) Flaw 1 & Flaw 5 (0013) |
| REQ-OUT-8 | SHOULD | Print `volatile` (and later `const`) from known qualifier data | [printer-output-quality.md](../02-flaws/printer-output-quality.md) Flaw 4 (0014) |

## Related documents

- [`.tasks/0018`](../../.tasks/0018-task-requirements-output-fidelity-ux.md) — the ticket this document
  satisfies.
- [`.tasks/0016`](../../.tasks/0016-epic-requirements-definition.md) — parent epic, requirement-format
  contract.
- [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md) (0013) — primary
  source for REQ-OUT-3, REQ-OUT-7.
- [`printer-output-quality.md`](../02-flaws/printer-output-quality.md) (0014) — primary source for
  REQ-OUT-1, REQ-OUT-2, REQ-OUT-8.
- [`varnode-ssa-type-system.md`](../02-flaws/varnode-ssa-type-system.md) (0012) — primary source for
  REQ-OUT-4, REQ-OUT-5.
- [`java-integration-testing.md`](../01-review/java-integration-testing.md) (0010) — primary source for
  REQ-OUT-6's motivation and verification method.
- [`../00-overview/glossary.md`](../00-overview/glossary.md) — shared terminology.
- Sibling requirement documents (owned by tickets 0017/0019, not this one):
  `rich-language-features.md`, `extensibility-api-testability.md`.
- [`00-index.md`](00-index.md) — consolidated requirements table across all three domains (owned by the
  orchestrator, not edited here).
