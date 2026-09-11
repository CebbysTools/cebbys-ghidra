# Flaws: Action/Rule Engine & Control-Flow Structuring

Ticket: [`.tasks/0013-task-flaws-rule-engine-block-structuring.md`](../../.tasks/0013-task-flaws-rule-engine-block-structuring.md)
(parent epic [`.tasks/0011`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md), root
[`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md)). Built on top of the review
documents [`action-rule-engine.md`](../01-review/action-rule-engine.md) (0006) and
[`control-flow-structuring.md`](../01-review/control-flow-structuring.md) (0007); shares terminology
with [`glossary.md`](../00-overview/glossary.md). Consolidated, prioritized across all four flaw docs
in [`00-index.md`](00-index.md) (owned by the epic ticket, not edited here).

Each flaw below follows the epic's required structure: **Observation → Why it's a problem (concretely)
→ Severity/impact → Improvement options (2-4, pros/cons) → Open questions.**

## Flaw 1 — Rule and block-structuring ordering/interaction is real, undeclared, and only empirically
patched

### Observation

Both subsystems this ticket covers converge on the same design: a catalog of small, independently
authored transformation rules tried against a graph in a **fixed, positional order**, re-run to a
fixed point, with **no machine-checked record of which rules depend on which**. The
`ruleaction.hh` file header states this is deliberate — rules "are generally applied simultaneously
from a pool ... and can interact with each other to produce an emergent transformation"
(`ruleaction.hh:19-23`, quoted in [action-rule-engine.md §1](../01-review/action-rule-engine.md)).
The block-structuring catalog (`CollapseStructure::ruleBlockX`, `blockaction.cc:1052-1901`) makes the
same bet at the level of CFG shapes instead of P-code ops. In both places, the *only* record of an
ordering constraint is a `//` comment at the call site — nothing a compiler, linker, or test enforces.
At least eight concrete instances of this fragility being visible in the source, spanning both
subsystems:

1. **A block-structuring rule permanently disabled by exactly one line, because block enumeration
   order broke it.** `ruleBlockOr` (the `&&`/`||` merge rule, `blockaction.cc:1321-1378`) has one
   candidate-clause check left permanently commented out, with the surviving comment reading: *"This
   line was always commented out. I assume minor block order variations were screwing up this rule"*
   (`blockaction.cc:1343-1344`). No one currently working on the code knows why removing that check was
   necessary — only that it was, empirically. (This is the case 0007's review flagged as a candidate for
   this document; see its "Notes for the flaws pass" at
   [control-flow-structuring.md §5 end](../01-review/control-flow-structuring.md).)
2. **The same rule is also excluded from the main collapse loop entirely and re-run as an isolated
   up-front pass.** `ruleBlockOr` is commented out of `collapseInternal`'s try-list
   (`blockaction.cc:1837-1840`) and instead driven separately to a fixed point by
   `collapseConditions()` *before* the main loop starts (`blockaction.cc:1132-1133`, called from
   `collapseAll`). Two independent mitigations (item 1 and item 2) exist for the same rule's ordering
   sensitivity, with no shared explanation connecting them.
3. **Two structuring rules are deliberately held back until every other rule in the catalog has stopped
   firing.** `ruleBlockIfNoExit` and `ruleCaseFallthru` are tried only after `collapseInternal`'s main
   loop reaches a fixed point (`blockaction.cc:1843-1856`), with the comment *"Applying IfNoExit rule
   too early can cause other (preferable) rules to miss. Only apply the rule if nothing else can
   apply."* This is the clearest documented admission that the catalog is **not confluent**: firing
   order changes which (legal) structuring is chosen, so a hand-tuned priority order compensates.
4. **A rule exists purely to detect and reverse an earlier rule's output once more information is
   available.** `RulePtraddUndo` and `RulePtrsubUndo` (`ruleaction.hh:1105-1129`, wired into the late
   `oppool2`/"typerecovery" pool, `coreaction.cc:6039`) exist because `RulePtrArith`
   (`ruleaction.hh:1070`, wired earlier in `oppool1`) speculatively converts integer-arithmetic trees
   into `PTRADD`/`PTRSUB` based on whatever type information is available *at the moment it fires* —
   which can be wrong. The doc comments are explicit: *"It is possible for Varnodes to be assigned
   incorrect types in the middle of simplification. This leads to incorrect PTRADD conversions. Once
   the correct type is found, the PTRADD must be converted back to an INT_ADD"* (`ruleaction.cc:6839-6844`)
   and the parallel comment for `RulePtrsubUndo` (`ruleaction.cc:6873-6878`). This is a rule whose entire
   job is compensating for another rule having fired too early relative to type-inference convergence —
   an ordering dependency solved by adding a second rule rather than by sequencing the first one later.
5. **A rule's special-case branch exists solely to avoid a bad interaction with a named sibling rule.**
   Inside the zero-extension-collapse logic (`RuleZextEliminate`/neighbouring code, `ruleaction.cc:3770-3799`),
   a branch handling a CDQ/`IDIV`-shaped pattern carries the comment *"This would be handled by the base
   case, but it interacts with RuleSubZext sometimes"* (`ruleaction.cc:3787`) — i.e. the general-case
   logic is known to misbehave when a specific other rule has already touched the same subgraph, so a
   parallel special case was added rather than the interaction being fixed structurally.
6. **Cross-`Action` ordering is pinned by prose alone.** `ActionDynamicMapping` and `ActionSpacebase` are
   inserted into `mainloop` with the comments *"Must come before restructurevarnode and infertypes"* and
   *"Must come before infertypes and nonzeromask"* respectively (`coreaction.cc:5877-5878`). Nothing
   about `ActionGroup::apply()`'s strict list-order execution (`action.cc:506-527`) checks these
   constraints; swapping the two `addAction()` lines compiles, links, and silently changes behavior.
7. **The same pattern repeats for merge-phase actions.** `ActionMarkImplied` carries *"This must come
   BEFORE general merging"* (`coreaction.cc:6100`) and `ActionMarkIndirectOnly` carries *"Must come after
   required merges but before speculative"* (`coreaction.cc:6105`) — two more purely-positional
   constraints with no enforcement.
8. **A fully-authored rule was silently excised from the pipeline with zero recorded rationale.**
   `RuleIndirectConcat` is a complete `Rule` subclass — declaration (`ruleaction.hh:859-866`,
   implementation `ruleaction.cc:4716-4760`, roughly) and its `addRule()` call site
   (`coreaction.cc:6040`) — but every one of those three locations is commented out in the source, with
   no comment anywhere explaining why. This is a stronger case than the documented ones above: at least
   `ruleBlockOr` and `ruleBlockIfNoExit` left a trail explaining the ordering problem; here the response
   to (presumably) an interaction problem was to delete the evidence along with the rule.

(Related, not counted above: `RuleEquality`, `ruleaction.hh:242-251`, is declared and implemented but
never `addRule()`-ed anywhere — flagged as dead code by 0006's review, §2/§4 — a milder version of the
same "nothing enforces that the declared catalog matches the wired catalog" gap.)

### Why it's a problem (concretely)

- Every one of the eight cases above is evidence that the two engines' correctness has, at some point,
  depended on the *exact* current traversal/registration order — and that order lives in `//` comments
  a refactor (or an innocuous cleanup PR) can invalidate without any build or test failure. The
  `ActionPool` restart-on-opcode-change behavior compounds this: `ActionPool::processOp`
  (`action.cc:822-875`) resets `rule_index` to `0` and re-scans a `PcodeOp`'s current-opcode rule bucket
  from the start whenever a rule changes that op's opcode (`action.cc:858-861`), meaning even *within* a
  single pool, which rule "sees" an op next is a function of registration order interacting with how
  many other rules already touched it in this pass — not something a reader can determine by inspecting
  one rule in isolation.
- A refactor that reorganizes `ActionDatabase::universalAction()` (`coreaction.cc:5835-6123`, the single
  function that wires the entire default pipeline — see 0006 §1) — for example to modularize it into
  smaller files, a stated goal of a decompiler refactor — risks silently reintroducing bugs the original
  authors already found and patched around, because the patches (items 1-8) are not attached to any
  machine-checkable precondition.
- This is squarely a **root-cause candidate**, not a symptom: it explains *why* two independent review
  documents (0006 and 0007), looking at two different files, each independently found at least one
  "the code doesn't know why this order matters, but it does" comment.

### Severity / impact

**High.** This does not currently manifest as a known bug (the shipped pipeline is presumably stable at
its current, carefully-tuned ordering), but it is a **standing risk multiplier** for any refactor work:
every reordering, split, or parallelization of the `Action`/`Rule` pipeline or the `CollapseStructure`
catalog must be manually re-validated against the full `datatests` suite (0006 §4 / the "What's missing"
list) rather than against any local, checkable contract. The risk is proportional to how much the
refactor intends to touch this code — which, per the epic's stated goals, is exactly this subsystem.

### Improvement options

1. **Explicit rule-dependency/ordering declarations.** Give `Rule`/`Action` (and the block-structuring
   `ruleBlockX` catalog) a declared partial order — e.g. `mustRunBefore()`/`mustRunAfter()` metadata, or
   named "phases" checked at pipeline-construction time (`ActionDatabase::universalAction()` could
   assert the declared order matches construction order).
   - *Pro*: turns items 1, 2, 3, 6, 7 above from silent invariants into build- or startup-time-checked
     ones; directly documents intent for future maintainers.
   - *Con*: requires manually reverse-engineering the *actual* dependency for all ~162 wired rules and
     ~10 `ruleBlockX` rules from their current behavior, since (per this section) that knowledge is
     currently implicit; a wrong declaration is a plausible new failure mode of its own.
2. **A confluence/idempotence self-check mode.** Add an optional debug mode that re-runs the same
   `ActionPool`/`CollapseStructure` pass against a randomized (or reversed) rule-registration order on
   the same input and diffs the result, to *detect* order-sensitivity like items 1-5 automatically
   instead of relying on someone noticing a bad decompile.
   - *Pro*: catches order-sensitivity the current comments show was previously found only by trial and
     error; usable incrementally without redesigning the engine.
   - *Con*: does not fix anything by itself — it is a detection tool, and for genuinely-intended
     "emergent" interactions (`ruleaction.hh:19-23`) it would flag expected non-confluence as noise
     unless paired with an allowlist.
3. **Reduce the "undo" pattern (item 4) by staging type-dependent rules later, not adding reversers.**
   Where a rule's correctness genuinely depends on type inference having stabilized (as with
   `RulePtrArith`/`RulePtraddUndo`), prefer moving the speculative rule to run only after the relevant
   fixed point, over shipping a second rule whose entire purpose is repairing the first.
   - *Pro*: shrinks the rule count and the ordering-dependency surface simultaneously; addresses the
     specific case in item 4 at the root.
   - *Con*: may not be achievable without breaking the "emergent transformation" design goal — some
     rules are pointer-arithmetic recognition specifically *because* they run early and get corrected
     later, per that doc comment; this is a case-by-case redesign, not a blanket policy.

### Open questions

- Is the "emergent transformation" design (`ruleaction.hh:19-23`) something the refactor should
  preserve as a deliberate architectural choice, or is it — as items 1, 3, 4, 5, 8 above suggest in
  practice — mostly accidental complexity that a declared-dependency model (option 1) could mostly
  eliminate without losing real expressiveness? This needs a decision, not just tooling.
- For `CollapseStructure` specifically: is the fixed try-order in `collapseInternal`
  (`blockaction.cc:1789-1841`) actually necessary because some `ruleBlockX` patterns are genuinely
  more specific/greedy, or could a scored/priority-queue matching strategy (each rule reports a
  "confidence"/specificity score, highest wins) replace positional ordering entirely and remove items
  1-3 at once? This is an algorithm-design question for 0021, but the flaw analysis should flag it as
  motivated by concrete evidence rather than speculative.

## Flaw 2 — Individual `Rule`s cannot be unit-tested in isolation today

### Observation

`Rule::applyOp(PcodeOp *op, Funcdata &data)` (`action.hh:246`) is the entry point every one of the
~162 wired `Rule`s implements, and its signature is the barrier: it needs a live `PcodeOp*` embedded in
a live `Funcdata`. `Funcdata`'s constructor takes a `Scope*` (`funcdata.cc:34`), and `Scope` is itself
part of the `Architecture`/database graph — there is no lightweight, dependency-free way to build "one
`PcodeOp` plus its operand `Varnode`s" and hand it to a single rule's `applyOp()` without first standing
up a substantial fraction of the surrounding decompiler state (address spaces, a symbol scope, a type
factory). `Rule` also documents that it "is allowed to keep state that is specific to a given function
... `reset()` is invoked to purge this state for each new function" (`action.hh:192-193`) — so a rule is
not even guaranteed to be a pure function of its immediate arguments across a whole pass, which a
fixture-based unit test would need to account for.

Confirming there is no existing fine-grained harness: the codebase's real C++ unit-test framework
(`test.hh`/`test.cc`, a `TEST(name){...}` macro) is used by `testtypes.cc`, `testfuncproto.cc`, and
`testparamstore.cc` (per 0006's related-review coverage table in
[00-index.md](../01-review/00-index.md), sourced from ticket 0010's review) — none of those exercise
`ruleaction.cc`/`blockaction.cc`. The one testing surface that *does* touch this subsystem,
`testfunction.hh`'s `FunctionTestProperty` (`testfunction.hh:29-47`), is the machinery behind the 89
`datatests` files: it decompiles an entire function from a loaded binary image and regex-matches lines
of the final printed C output (`FunctionTestProperty::processLine`, doc comment `testfunction.hh:31-33`)
— i.e. it is whole-pipeline, black-box, and textual, not a way to assert "`RuleXxx::applyOp` fires on
this specific `PcodeOp` shape."

### Why it's a problem (concretely)

- A change to one rule's matching logic can only be validated today by running whole functions through
  the entire ~162-rule fixed-point pipeline (plus, for block-structuring rules, the entire
  `CollapseStructure` pass) and diffing the final decompiled text — exactly the workflow 0006's review
  already identified as the *only* coverage this subsystem has (§4, "No rule-level unit isolation").
  This means a correct, narrowly-scoped rule fix and an accidental regression in an unrelated rule can
  both show up as "some `datatests` file's regex no longer matches," with no way to localize *which*
  rule changed behavior without manual bisection.
- It directly blocks Flaw 1's mitigations: a declared-dependency model (Flaw 1, option 1) is much easier
  to validate incrementally if each rule's own behavior can be pinned down by a fast, isolated test
  first; without that, every dependency-model change still has to be validated end-to-end.
- It also blocks safely doing Flaw 4's structuring-algorithm changes and any future "correctness vs.
  readability" split (Flaw 5) — both need a way to assert what a single rule does on a minimal input,
  independent of everything else in the pipeline.

### Severity / impact

**High.** This is a testability gap across the *entire* rule catalog (both the ~162 `Rule` subclasses
and the ~10 `ruleBlockX` structuring rules), not one or two edge cases, and it sits directly on the
critical path of any refactor that touches this code — which, again, is the explicit target of this
epic.

### Improvement options

1. **A minimal synthetic-`Funcdata` test fixture.** Build a small helper (new code, or an extension of
   `testfunction.hh`) that constructs the smallest possible `Architecture`/`Scope`/`Funcdata` needed to
   host a handful of manually-created `PcodeOp`s/`Varnode`s, then calls one `Rule::applyOp()` directly
   and asserts on the resulting graph shape.
   - *Pro*: directly closes the gap; reuses the existing `test.hh` `TEST()` framework already proven for
     `testtypes.cc` et al.; can be added incrementally, rule by rule, starting with the highest-risk
     ones identified in Flaw 1.
   - *Con*: real engineering effort — building even a minimal `Funcdata` today requires understanding
     and stubbing a fair amount of `Architecture` setup (address spaces, at minimum); some rules'
     `getOpList()`/multi-op matching (e.g. rules that walk several ops back from the trigger op, common
     throughout `ruleaction.cc`) will need a correspondingly non-trivial synthetic graph to exercise.
2. **A P-code-fragment micro-DSL for test authoring.** Rather than building `PcodeOp`s via the C++ API
   op-by-op, add a small textual notation (à la SLEIGH's own p-code, or reusing ideas from the — dead —
   `rulecompile.cc` pattern grammar's *op-reference* syntax, see Flaw 3) purely for *constructing* test
   fixtures, not for matching.
   - *Pro*: makes individual rule tests much more readable/writable than raw builder-API C++; a natural
     complement to option 1.
   - *Con*: another small language to maintain; scope creep risk if it grows matching semantics of its
     own (exactly the trap `rulecompile.cc` fell into, per Flaw 3) — needs a firm "fixture construction
     only" boundary.
3. **Do nothing beyond documenting the gap.** Accept that whole-pipeline `datatests` diffing remains the
   only validation method.
   - *Pro*: zero cost now.
   - *Con*: does not address the epic's stated goal of assessing *improvement options*, and leaves every
     future change to this subsystem exposed to the same slow, hard-to-localize validation loop; actively
     works against Flaw 1's dependency-declaration option, which benefits from fast per-rule tests to
     validate incrementally.

### Open questions

- Should a rule-level test harness (option 1) be scoped to new/changed rules going forward, or is a
  retroactive test-writing pass across all 162 rules in scope for this refactor? The latter is a
  substantial, separately-schedulable effort.
- Do the `rule_debug`/statistics affordances already in `Rule` (`count_tests`/`count_apply`,
  `action.hh:209-210`, surfaced via `print actionstats`, per 0006 §4) provide enough signal to at least
  detect *which* rules fired differently between two pipeline runs, as a cheaper interim step before a
  full fixture-based harness exists?

## Flaw 3 — Two ways to author a rule, one of them dead: `rulecompile.cc`'s pattern DSL

### Observation

`rulecompile.hh`/`rulecompile.cc` implement a full second language for expressing a `Rule`'s match
pattern — a lexer (`RuleLexer`, `rulecompile.hh:23-61`), a grammar, and a constraint-graph builder
(`RuleCompile`, `rulecompile.hh:96-128`) that compiles pattern text into a `ConstraintGroup` driven at
runtime by a general graph-unification engine (`unify.hh`), wrapped as a genuine `Rule` subclass
(`RuleGeneric`, `rulecompile.hh:145-158`). Per 0006's review (§3, and restated in
[00-index.md](../01-review/00-index.md)'s "Known-surprising findings"), **this entire mechanism is dead
in shipped builds**:

- `RuleGeneric::build()` — the only way a DSL-described rule becomes runnable — is reachable from
  exactly one call site, `Architecture::decodeDynamicRule()` (`architecture.cc:706-738`), and that call
  site references an undeclared variable (`el->getContent()` against a `Decoder& decoder`, not an
  `Element*`, at `architecture.cc:732`) — it cannot currently compile.
- The whole file is wrapped in `#ifdef CPUI_RULECOMPILE ... #endif` (`rulecompile.cc:16`/`:889`), and
  `CPUI_RULECOMPILE` is a commented-out, opt-in `Makefile` flag (`Makefile:114`) not enabled even by the
  debug build targets (`Makefile:125,133`).
- All 162 wired rules that actually ship are conventional hand-written `Rule` subclasses; zero are
  `RuleGeneric` instances (0006 §2/§3).
- The DSL's one demonstrably-live use is `parse rule <file>` (`IfcParseRule`, `ifacedecomp.cc:3221-3256`,
  same `#ifdef`), which compiles pattern text and pretty-prints the *equivalent hand-written C++* via
  `UnifyCPrinter` — i.e. even when this code path is reachable (a developer manually enabling
  `CPUI_RULECOMPILE`), its practical role is generating scaffolding to hand-copy into `ruleaction.cc`,
  not shipping DSL-interpreted rules (0006 §3's conclusion, which this document adopts).

### Why it's a problem (concretely)

- The codebase presents *two* conceptual ways to write a rule (hand-written C++ traversal vs. a
  declarative pattern grammar), but only one is real. A newcomer reading `rulecompile.hh`'s documented
  grammar (`rulecompile.hh:160-202`) or discovering the `<rule>` XML schema and `parse rule`/
  `experimental rules` console commands has no signal from the source alone that this path is
  unreachable in any release build — they would have to independently notice the `#ifdef`, the
  commented-out `Makefile` flag, and the broken call site, exactly as 0006's review did.
- It is unmaintained-but-present dead weight: `rulecompile.cc` is a non-trivial file (a hand-written
  lexer plus a grammar-driven builder) that must still be read, and could still silently bit-rot further
  (as its one call site already has) without anyone noticing, because nothing exercises it.
- It directly overlaps with Flaw 1's core finding: if resurrected as a live, dual-tracked rule-authoring
  path, it would add a *second* execution/ordering model (`unify.hh`'s constraint-graph evaluation)
  alongside `ActionPool`'s opcode-bucket ordering — compounding, not reducing, the undeclared-ordering
  risk surface Flaw 1 describes, since the pattern grammar has no concept of cross-rule ordering or
  dependency either; it only describes one rule's own match shape.

### Severity / impact

**Medium.** It is not causing incorrect decompiler output today (it cannot run in a shipped build at
all), so it is not an active-bug-severity issue. But it is a real **decision-forcing** item for the
refactor: leaving it in its current ambiguous state (present in source, absent in practice) is itself a
choice with cost — every future contributor who encounters it has to re-derive its dead status.

### Explicit recommendation: replace/retire — do not dual-track

The refactor should **not** keep `rulecompile.cc`'s pattern DSL as a live, dual-tracked way to author
runtime rules, for three concrete reasons grounded in the above:

1. **There is no real userbase to preserve.** Because the DSL has been unreachable in shipped builds
   (broken call site + disabled build flag, both confirmed above), no `.cspec`/`.pspec` author is
   plausibly depending on `<rule>` XML customization today — fixing and re-enabling it would be adding a
   *new* feature under an old name, not restoring one.
2. **Reviving it would add a second rule-execution semantics, which is the wrong direction given Flaw
   1.** The refactor's biggest evidenced risk in this subsystem is *undeclared* ordering/interaction
   between rules (Flaw 1). A second, syntactically different way to describe rule matching — with its
   own unification-engine evaluation order — adds surface area to that exact problem rather than
   reducing it, and the DSL grammar has no answer for cross-rule ordering at all (it only expresses one
   rule's match pattern).
3. **If a declarative rule layer is wanted later, it should be designed as part of Flaw 1's
   dependency-declaration work, not resurrected from `rulecompile.cc`.** A declarative layer whose
   purpose is *also* expressing ordering/dependency metadata (Flaw 1, option 1) is a different, more
   useful shape than a pattern-matching grammar that only replaces hand-written C++ traversal code
   one-for-one.

A narrower, optional carve-out: the `parse rule`/`UnifyCPrinter` **codegen convenience** (pattern text →
generated C++ scaffolding for a human to hand-copy into `ruleaction.cc`) has some standalone developer
value and costs nothing at runtime, since it never ships. If kept at all, it should be decoupled from the
broken `decodeDynamicRule` runtime-loading path (which should simply be deleted, not fixed) so that the
codegen tool's continued existence doesn't imply the dynamic-rule-loading feature is supported — bundling
both under one `#ifdef`, as today, is precisely what let the runtime path's bug go unnoticed.

### Improvement options

(Presented for completeness per the epic's required structure, though the recommendation above already
resolves the primary "keep/replace/dual-track" question the ticket asks for.)

1. **Delete `rulecompile.cc`/`.hh`, `unify.hh`'s consumers specific to it, `decodeDynamicRule`, and the
   `parse rule`/`experimental rules` console commands wholesale.**
   - *Pro*: removes genuinely dead, `#ifdef`-gated code and its bit-rotted call site in one pass; no
     ambiguity left for future readers.
   - *Con*: loses the (currently broken but conceptually salvageable) codegen-scaffolding workflow
     entirely unless someone explicitly wants to reimplement it later.
2. **Fix the broken call site and re-enable `CPUI_RULECOMPILE` by default, keeping it as a supported
   dual-tracked authoring mechanism.**
   - *Pro*: preserves optionality for `.cspec` authors who might want dynamic rules without a C++
     rebuild.
   - *Con*: directly contradicts the recommendation above; commits maintenance effort (a second grammar,
     lexer, and constraint-evaluation engine to keep working and tested) to a feature with no evidenced
     current demand, and widens Flaw 1's ordering-risk surface.
3. **Keep only the codegen/prototyping workflow, explicitly decoupled from runtime loading (the
   narrower carve-out described above).**
   - *Pro*: retains the one plausibly-useful part (pattern-to-C++ scaffolding for developers writing new
     rules) without resurrecting a shipped dynamic-rule feature; removes the broken/misleading
     `decodeDynamicRule` runtime path.
   - *Con*: still requires maintaining the lexer/grammar/`ConstraintGroup` machinery for a
     developer-convenience feature whose actual usage (how often anyone actually runs `parse rule`
     today) is unknown and unmeasured.

### Open questions

- Is there any external/downstream consumer (a fork, a plugin, a documented workflow outside this
  repository) actually relying on `<rule>` XML dynamic-rule loading that this review would not be able
  to see? If so, that changes the recommendation's premise (no real userbase) materially.
- If the narrower codegen carve-out (option 3) is chosen, should its output format change to target
  whatever new declarative/dependency-aware rule-authoring shape Flaw 1's option 1 eventually produces,
  rather than today's hand-written-`Rule`-subclass C++?

## Flaw 4 — The structuring algorithm has known-weak input classes, handled only by correctness-preserving
fallback, never by better structuring

### Observation

`CollapseStructure`'s local pattern-matching catalog (`blockaction.cc:1052-1901`, cataloged in
[control-flow-structuring.md §1.2](../01-review/control-flow-structuring.md)) is not a general
structuring algorithm in the classic interval-analysis sense — it is a fixed set of shape-matching
rules plus a goto/`while(true)`-fallback safety net for whatever those rules cannot recognize. Several
concrete classes of input where this is known-weak are documented directly in the source (all cataloged
in more depth in [control-flow-structuring.md §5](../01-review/control-flow-structuring.md), repeated
here with the flaws-relevant framing):

- **Irreducible CFGs are never structured, only demoted to goto.** Any edge flagged `f_irreducible` by
  Tarjan's reducibility test (`BlockGraph::findIrreducible`, `block.cc:1160`) is masked together with
  back-edges and existing gotos by `isDecisionOut`/`isGotoOut` (`block.hh:348-359`), so **no**
  `ruleBlockX` will ever attempt to absorb it into a structured shape — it is unconditionally emitted as
  a `goto`/`BlockMultiGoto`. There is no structuring strategy here beyond "don't try."
- **Structurally-complex loop conditions can never print as an idiomatic `while(cond)`.**
  `ruleBlockWhileDo` (`blockaction.cc:1546-1553`) detects when a condition block is too complex
  (`FlowBlock::isComplex`, `block.hh:259`) and permanently sets `f_whiledo_overflow`, forcing
  `while(true) { ...; if (!cond) break; }`-shaped output for the life of that block
  (`BlockWhileDo::hasOverflowSyntax`, `block.hh:761-762`); `finalTransform` (`block.c:3482`) explicitly
  refuses for-loop recovery once this flag is set.
- **Duplicated ("unswitched"/jump-threaded) conditional expressions have no matching `ruleBlockX`
  shape at all** and require a dedicated pre-pass, `ConditionalJoin` (`blockaction.hh:234-265`,
  `blockaction.cc:2073-2116`), to rejoin the duplicate blocks *before* `CollapseStructure` runs, because
  — per the doc comment — the un-rejoined shape ("two splitting blocks and no direct merge",
  `blockaction.hh:230-233`) is not one the catalog recognizes.
- **Unrolled switch guards duplicated across predecessor blocks** are a documented special case
  `JumpBasic::checkUnrolledGuard` (`jumptable.cc:1372-1414`) exists specifically to detect, because the
  generic guard-folding logic (`markFoldableGuards`, `jumptable.cc:1282-1294`) does not otherwise expect
  a switch's range check to appear more than once.
- **Multistage (indirect-of-indirect) jump tables cannot be resolved in one pass** and require a full
  analysis restart (`JumpTable::checkForMultistage`/`recoverMultistage`, `jumptable.cc:2896-2909,2697-2720`)
  — a correctness/completeness gap in the *first* pass that is patched by re-running more analysis, not
  by a smarter single-pass model.
- **The authors' own comments flag the irreducibility-detection fixed point and its failsafe as
  not fully trusted.** `structureLoops` (`block.cc:2212-2228`) must loop `findSpanningTree`/
  `findIrreducible` until stable because marking one tree edge irreducible invalidates the very tree used
  to find it (`block.cc:1192-1193`); and the separate DFS-based `calcLoop` failsafe carries the comment
  *"Technically we should never reach here if the reducibility algorithms work"* (`block.cc:2152`) —
  i.e., a defensive code path against a suspected-but-unconfirmed bug in the primary algorithm, still
  present in shipped code.

### Why it's a problem (concretely)

- Every one of these is a case where the *hardest* real-world inputs (hand-written state machines,
  heavily-optimized loops, jump-threaded or loop-unswitched code, multi-stage computed jumps) are
  precisely where the algorithm is weakest — the goto-fallback and `while(true)`-overflow paths preserve
  **correctness** (the emitted C is still semantically faithful) but sacrifice exactly the **readability**
  a decompiler exists to provide, on exactly the inputs where a reverse engineer most needs help.
- The goto-edge-selection heuristic itself (`selectGoto`/`TraceDAG::selectBadEdge`,
  `blockaction.cc:1260-1277`/`:730`, scored via `BadEdgeScore`, `blockaction.hh:143-154`) is a
  hand-tuned scoring function, not a proof of minimality — there is no stated guarantee it selects the
  fewest or least-disruptive gotos possible for a given irreducible region, only that it prefers
  "structurally less damaging" edges by heuristic.
- Because none of this is exercised by any per-shape unit test (Flaw 2), a change intended to improve
  one of these known-weak cases has the same validation problem as any other rule change: whole-pipeline
  `datatests` diffing is the only feedback loop.

### Severity / impact

**Medium-High.** Correctness is preserved in all documented cases (this is not a bug-severity finding),
but it is a direct, source-confirmed limit on output *quality* for a meaningful, real-world class of
inputs — and it is squarely in scope for "alternative structuring algorithms" being one of the refactor's
named goals.

### Improvement options

1. **Adopt a node-splitting or condition-synthesis structuring strategy for the irreducible/goto-fallback
   case specifically**, rather than accepting goto as the only outcome. Named (not compared in depth —
   that comparison is 0021's job): Sharir's structural-analysis/interval approach and its "no more gotos"
   descendants; Schwartz et al.'s semantics-preserving iterative structural-analysis approach (handles
   irreducibility via node splitting instead of falling back to goto); the "relooper"/stackifier
   algorithm used for WebAssembly control-flow recovery (a different systematic DAG-to-structured-code
   transform); DREAM/DREAM++'s (Yakdan et al.) goto-free Boolean-condition-synthesis approach, which
   targets exactly the readability-over-goto tradeoff this flaw describes.
   - *Pro*: directly targets the flaw's core complaint (readability loss on hard inputs) rather than
     just improving diagnostics around it.
   - *Con*: substantial algorithmic work and correctness risk of its own; several of these named
     alternatives trade one class of readability problem for another (e.g. node-splitting can duplicate
     code), so the choice needs the deeper comparison explicitly deferred to 0021.
2. **Instrument and report goto-fallback frequency as a quality metric**, independent of picking a new
   algorithm: log/count how often `selectGoto` fires (and on what CFG shapes) across a representative
   corpus, to turn "structuring is known-weak here" from a source-comment-level claim into a measured
   one that can track regressions/improvements over time.
   - *Pro*: low-cost, immediately actionable, and gives any future algorithm change (option 1) a
     baseline to improve against.
   - *Con*: does not itself fix anything; only useful if paired with an actual improvement effort or
     used to prioritize one.
3. **Leave the goto/overflow-syntax fallback as the deliberate, documented boundary of "correctness over
   idiomatic structure," and invest instead in making that fallback's output as readable as possible**
   (e.g. better goto-target naming, or tightening `selectGoto`'s heuristic with more of `TraceDAG`'s
   scoring information) rather than replacing the algorithm.
   - *Pro*: much lower risk than option 1; several of the hard cases above (multistage jump tables,
     unrolled guards) already have dedicated, working recovery code — the remaining gap is specifically
     the irreducible/overly-complex-condition tail, which may be a small enough fraction of real binaries
     that marginal improvement is the right cost/benefit tradeoff.
   - *Con*: does not close the gap with newer structuring approaches literature has developed since
     `CollapseStructure`'s design; the decompiler would remain behind the state of the art rather than
     catching up.

### Open questions

- How common are irreducible CFGs / goto-fallback triggers in practice, across a representative modern
  binary corpus (post-LTO, post-PGO optimized code, common obfuscation)? This directly determines whether
  option 1's investment is worthwhile — 0007's review did not measure this (it is a code-structure
  review, not an empirical one), so it is an open question for whichever epic scopes 0021's algorithm
  comparison.
- Should `CollapseStructure`'s local-pattern-plus-fallback design and a from-literature alternative (per
  option 1) be pursued as a *replacement*, or should the existing fast-path catalog remain for the common
  reducible case with a fundamentally different algorithm invoked only once irreducibility/goto-fallback
  is detected? The latter would preserve most of the current, presumably-well-tested behavior while only
  changing behavior on the known-weak tail.

## Flaw 5 — The rule catalog's only built-in taxonomy (`group`) cannot separate correctness-required
rules from readability rules

### Observation

Every `Action`/`Rule` is constructed with a `group` string consumed only by `ActionGroupList::contains()`
(`action.hh:31-40`) for whole-group enable/disable toggling (`option currentaction <group> on|off`,
`options.cc:648-680`, per 0006 §4). Per 0006's review methodology (§2), of the 162 wired rules, **112
(69%) share the single literal tag `"analysis"`** — the source's own categorization is essentially "most
things" plus about a dozen named special cases (`subvar`, `typerecovery`, `bitfields`, `cleanup`, etc.).
Nothing in this tag, or anywhere else in `Rule`/`Action`'s interface (`action.hh:194-254`), distinguishes
"if this rule misfires, the emitted C's apparent semantics change" from "if this rule misfires or is
simply absent, the output is just less idiomatic C."

### Why it's a problem (concretely)

- A refactor cannot use anything in the source today to decide, for example, which rules most need the
  rule-level test harness proposed in Flaw 2 first, or which rules would be safe to expose through a
  future third-party extension point (0006's review, "Extension model" section of
  [00-index.md](../01-review/00-index.md), separately notes rules don't use the codebase's
  `CapabilityPoint` extension mechanism at all) without individually re-reading all 162 doc comments, as
  0006's review had to do to build its own reviewer-imposed taxonomy (§2's table).
  the `group` tag being 69% one value means it cannot be used as even a rough proxy today.
- This compounds Flaw 1 and Flaw 2: without a correctness/readability split, there is no way to
  prioritize which of the many undeclared ordering interactions (Flaw 1) are worth guarding against
  first, or which rules most urgently need isolated tests (Flaw 2) versus which could reasonably ship
  with only whole-pipeline coverage.
- It is also a concrete blocker to a specific improvement the ticket names directly: "separating
  correctness-required vs. readability rules so they can be reasoned about differently" is not
  achievable as a small change — it requires an audit against a source that currently offers no
  signal to start from.

### Severity / impact

**Medium.** This is not causing any bug today — it is a missing piece of infrastructure/metadata that
blocks reasoning about, and prioritizing work on, Flaws 1 and 2, rather than a defect with its own
independent failure mode.

### Improvement options

1. **Add an explicit rule-classification field** (e.g. a `RuleClass{ correctness, readability, both }`
   enum or similar, alongside — not replacing — the existing `group` string) and retrofit all 162 wired
   rules via a one-time audit.
   - *Pro*: directly enables staged testing/rollout, prioritized dependency-declaration work (Flaw 1),
     and a principled future extension boundary.
   - *Con*: a real one-time audit cost across 162 rules; the correctness/readability line is genuinely
     fuzzy for some (e.g. `RuleSub2Add`'s `V - W ⇒ V + W*-1`, `ruleaction.hh:753`, is semantics-preserving
     by construction but changes surface form — is that "correctness" infrastructure or "readability"
     normalization?), so the taxonomy needs careful, rule-by-rule judgment rather than a mechanical pass.
2. **Derive the split empirically instead of by manual audit**, once Flaw 1's dependency/instrumentation
   work or Flaw 4's goto-fallback instrumentation exists — e.g. flag any rule that can affect a `CALL`'s
   parameters, a return value, or a pointer's effective type as "correctness-adjacent" by construction,
   and treat local arithmetic/idiom rules as "readability" by default.
   - *Pro*: less manual/subjective than option 1; reuses infrastructure Flaws 1/2 already motivate.
   - *Con*: a second-order fix — it depends on other infrastructure existing first, so it cannot be the
     immediate answer if the classification is needed soon.
3. **Do not add a new taxonomy; instead document the existing 0006-review taxonomy (§2's table) as the
   de facto categorization** and keep it current by convention as new rules are added.
   - *Pro*: essentially free — the categorization work has already been done once, by 0006's review.
   - *Con*: not machine-checkable (the exact gap this flaw is about), and "keep it current by
     convention" is the same failure mode already observed elsewhere in this document (e.g. Flaw 1 item
     8's silent, unexplained rule removal) — conventions with no enforcement have already been shown not
     to hold reliably in this codebase.

### Open questions

- Who is the right owner to adjudicate ambiguous correctness/readability classifications (option 1) —
  is this a decompiler-team call, or does it need input from downstream consumers who might rely on
  either category's stability guarantees differing?
- Does a "both" category (a rule that is readability-motivated but happens to also be relied upon for
  correctness elsewhere, per Flaw 1 item 4's `RulePtrArith`/`RulePtraddUndo` pattern) need to exist, or
  should such cases be considered a design smell to eliminate rather than a category to classify into?

## Related documents

- [`action-rule-engine.md`](../01-review/action-rule-engine.md) (0006) — primary source for Flaws 1
  (rule-side), 2, 3, 5.
- [`control-flow-structuring.md`](../01-review/control-flow-structuring.md) (0007) — primary source for
  Flaws 1 (structuring-side) and 4; also the origin of the `blockaction.cc:1343-1344` case this document
  was asked to expand on.
- [`glossary.md`](../00-overview/glossary.md) — shared terminology.
- Consolidated across all four flaw docs: [`00-index.md`](00-index.md) (owned by ticket 0011, not this
  document).
