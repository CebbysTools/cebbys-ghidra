# Action/Rule Simplification Engine

Ticket: [`.tasks/0006-task-review-action-rule-engine.md`](../../.tasks/0006-task-review-action-rule-engine.md)
(parent epic [`.tasks/0002`](../../.tasks/0002-epic-review-current-implementation.md), root
[`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md)). Related documents: this doc is
referenced from [`00-index.md`](00-index.md) (owned by the epic ticket, not edited here) and shares
terminology with [`glossary.md`](../00-overview/glossary.md).

## What this subsystem does

Once a function's raw P-code is loaded and put into SSA form (see the SSA/heritage review), the
decompiler does not go straight to control-flow structuring. Instead it runs the P-code graph through
several hundred small, local, pattern-driven rewrites — collapsing constant arithmetic, undoing
compiler idioms for division/modulo, converting integer arithmetic on pointers into typed
`PTRADD`/`PTRSUB` operations, narrowing sub-register logical values, folding bitfield insert/extract
sequences, recognizing memcpy-shaped STORE sequences, and hundreds of other micro-transformations —
until no further local rewrite applies. This is the **Action/Rule engine**: a small scheduling
framework (`action.hh`/`action.cc`) that runs a fixed, hand-assembled pipeline of passes
(`coreaction.cc`) built mostly out of a large catalog of `Rule` subclasses
(`ruleaction.hh`/`ruleaction.cc`, plus a handful of satellite files: `condexe.cc/hh`, `subflow.cc/hh`,
`bitfield.cc/hh`, `double.cc/hh`, `constseq.cc/hh`). It is what turns an SSA P-code graph into
something whose expression trees, pointer arithmetic, and boolean logic already look close to
source-level C, *before* control-flow structuring and printing ever run.

## 1. The scheduling model

### `Action`, `ActionGroup`, `ActionPool` — three ways to schedule work

Everything that transforms a `Funcdata` is an `Action` (`action.hh:52-135`). `Action::apply()` is the
extension point; `Action::perform()` (`action.cc:298-362`) is the driver every caller actually calls —
it wraps `apply()` with breakpoint/warning bookkeeping and, if the `rule_repeatapply` flag is set,
re-invokes `apply()` in a loop until a call makes no further changes (`count` stops increasing). Three
concrete shapes exist:

- **`ActionGroup`** (`action.hh:143-165`, `action.cc:364-527`): an ordered `vector<Action*>`, applied
  in list order. `ActionGroup::apply()` (`action.cc:506-527`) walks the list once per outer call; if
  the group itself has `rule_repeatapply`, `perform()` re-runs the whole ordered list to a fixed point.
  This is how the engine expresses "do A, then B, then C" and "repeat this whole block of passes until
  stable" — both are ordinary `ActionGroup`s, just with different flags.
- **`ActionRestartGroup`** (`action.hh:173-182`, `action.cc:529-582`): an `ActionGroup` variant used
  exactly once, as the outermost wrapper (`coreaction.cc:5847`). If any `Action`/`Rule` calls
  `Funcdata::setRestartPending()` (e.g. a discovered `Datatype` union resolution forces re-analysis),
  this clears the function's analysis and restarts the whole pipeline, up to `maxrestarts` (`1`, for
  the built-in pipeline) times (`action.cc:558-581`).
- **`ActionPool`** (`action.hh:262-285`, `action.cc:728-975`): a flat set of `Rule*` objects, indexed
  by opcode. `Rule::getOpList()` (`action.hh:237`, default impl `action.cc:706-713`) lets a rule
  declare which `OpCode`s it cares about; `ActionPool::addRule()` (`action.cc:740-751`) files the rule
  into a `vector<Rule*> perop[CPUI_MAX]` bucket per opcode it registered for. A rule that does not
  override `getOpList()` is bucketed under *every* opcode — e.g. `RuleEarlyRemoval`
  (`ruleaction.hh:85-94`) and `RuleCollapseConstants` (`ruleaction.hh:703-712`) both run on literally
  every `PcodeOp` in the function on every pass, by explicit doc-comment admission ("This rule applies
  to all ops"). `ActionPool::apply()` (`action.cc:877-888`) walks every live `PcodeOp` in the function
  (`data.beginOpAll()`/`endOpAll()`) and, for each, `processOp()` (`action.cc:822-875`) tries every
  rule bucketed for that op's current opcode, *in the order they were `addRule()`-ed*, restarting the
  per-op rule scan from index 0 whenever a rule changes the op's opcode (so a later rule can react to
  an earlier one's rewrite within the same visit). `ActionPool` is virtually always given
  `rule_repeatapply`, so the whole op-by-op sweep repeats until one full pass makes zero changes.

`Rule::applyOp(PcodeOp*, Funcdata&)` (`action.hh:246`) is the actual per-rule entry point — return `0`
for "did not apply", non-zero for "applied" (and increment the pool's aggregate `count`).

### Groups, root Actions, and the `ActionDatabase`

Every `Action`/`Rule` is constructed with a **group name** string (e.g. `"analysis"`, `"typerecovery"`,
`"bitfields"`, `"subvar"` — see the constructor calls throughout `coreaction.cc:5835-6123`). This has
nothing to do with `ActionGroup`; it is a tag consumed by `ActionGroupList::contains()`
(`action.hh:31-40`) inside every `clone(const ActionGroupList&)` override (present on every single
`Action`/`Rule` subclass, always following the same boilerplate: return `nullptr` if the current
grouplist does not contain this object's group, otherwise clone). `ActionDatabase` (`action.hh:298-324`,
`action.cc:974-1160`) builds one **universal** `Action` tree containing every `Action`/`Rule` the
codebase knows about (`ActionDatabase::universalAction()`, `coreaction.cc:5835-6123` — this is the
single function that assembles the entire pipeline; see below) and then derives named **root Actions**
by cloning the universal tree against a named grouplist (`ActionDatabase::deriveAction()`,
`action.cc:1145-1160`). `ActionDatabase::buildDefaultGroups()` (`coreaction.cc:5792-5831`) defines six
built-in grouplists — `"decompile"` (the normal, full pipeline), `"jumptable"` (a cut-down pipeline
used mid-analysis for jump-table recovery, omitting e.g. `"typerecovery"`/`"merge"`/`"casts"`),
`"normalize"`, `"paramid"` (parameter-ID recovery), `"register"`, and `"firstpass"` — each just a list
of group-name strings. This is the mechanism by which the same catalog of `Action`/`Rule` objects (74
`Action` subclasses plus 163 `Rule` subclasses, ~237 total — see §2's methodology for how the `Rule`
count was obtained; the ticket that spawned this review estimated "~500+", which this review found to
be an overestimate) is reused to build several *different* pipelines for different purposes, and it is
also the supported
way to add/remove whole feature groups at runtime: `option currentaction <group> <on|off>`
(`OptionCurrentAction`, `options.hh:251-253`, doc comment `options.cc:648-655`) calls
`ActionDatabase::toggleAction()` (`action.cc:1036-1053`), and `option setaction <name> [newname]`
(`OptionSetAction`, `options.cc:626-646`) can clone a whole named root Action to make a customized copy.

### The pipeline itself: `ActionDatabase::universalAction()`

`coreaction.cc:5835-6123` is the one function that wires up the entire default pipeline, as nested
`ActionGroup`/`ActionPool` objects (not code — direct `new`/`addAction`/`addRule` calls executed once
at startup). Structurally, from the outside in:

```
universal (ActionRestartGroup, max 1 restart)      coreaction.cc:5847
 ├─ ActionStart, ActionConstbase, ActionNormalizeSetup, ActionDefaultParams,
 │    ActionExtraPopSetup, ActionPrototypeTypes, ActionFuncLink, ActionFuncLinkOutOnly
 ├─ fullloop (ActionGroup, repeat)                  coreaction.cc:5860
 │   ├─ mainloop (ActionGroup, repeat)              coreaction.cc:5862
 │   │   ├─ ActionUnreachable, ActionVarnodeProps, ActionHeritage, ActionParamDouble, ...
 │   │   ├─ stackstall (ActionGroup, repeat)        coreaction.cc:5882
 │   │   │   ├─ oppool1 (ActionPool, repeat, 134 Rules)   coreaction.cc:5884
 │   │   │   ├─ ActionLaneDivide, ActionMultiCse, ActionShadowVar,
 │   │   │   │    ActionDeindirect, ActionStackPtrFlow
 │   │   ├─ ActionRedundBranch, ActionBlockStructure, ActionConstantPtr
 │   │   ├─ oppool2 (ActionPool, repeat, 6 Rules: pointer/struct/LOAD-STORE)  coreaction.cc:6034
 │   │   ├─ ActionDeterminedBranch, ActionUnreachable, ActionNodeJoin,
 │   │   │    ActionConditionalExe, ActionConditionalConst
 │   ├─ ActionLikelyTrash, ActionDirectWrite(x2), ActionDeadCode, ActionDoNothing,
 │        ActionSwitchNorm, ActionReturnSplit, ActionUnjustifiedParams, ActionStartTypes,
 │        ActionActiveReturn
 ├─ ActionMappedLocalSync, ActionStartCleanUp
 ├─ cleanup (ActionPool, repeat, 22 Rules: late/bitfield/string/split rewrites) coreaction.cc:6067
 ├─ ActionPreferComplement, ActionStructureTransform, ActionNormalizeBranches,
 │    ActionAssignHigh, ActionMergeRequired, ActionMarkExplicit, ActionMarkImplied,
 │    ActionMergeMultiEntry, ActionMergeCopy, ActionDominantCopy, ActionDynamicSymbols,
 │    ActionMarkIndirectOnly, ActionMergeAdjacent, ActionMergeType, ActionHideShadow,
 │    ActionCopyMarker, ActionLateDoNothing, ActionBlockStructure (again),
 │    ActionPreferComplement (again), ActionStructureTransform (again),
 │    ActionOutputPrototype, ActionInputPrototype, ActionMapGlobals, ActionDynamicSymbols,
 │    ActionNameVars, ActionSetCasts, ActionFinalStructure, ActionPrototypeWarnings, ActionStop
```

(88 top-level `addAction()` calls total inside `universalAction()`; counted by grepping the function
body.) The two big `ActionPool`s (`oppool1`/`oppool2`) are where almost the entire `ruleaction.cc`
catalog runs; `oppool1` alone holds 134 of the engine's 162 wired-in `Rule` objects and sits *inside*
two nested repeat-to-fixed-point loops (`stackstall` inside `mainloop` inside `fullloop`), so in
practice most rules get many opportunities to fire and re-fire across a single decompilation as SSA
heritage, type inference, and block structuring each feed new information back into the graph. Note
`ActionBlockStructure`/`ActionStructureTransform`/`ActionFinalStructure`/`ActionPreferComplement`
(control-flow structuring proper, `blockaction.hh`) and `ActionSetCasts`/`ActionInferTypes` (cast/type
propagation) are interleaved *into* this same pipeline rather than being separate downstream stages —
those subsystems are covered by tickets 0007 and 0005/0009 respectively; they are named here only to
show where rule-based simplification hands off to them mid-pipeline.

### Ordering and rule interaction: explicit vs. emergent

Ordering is captured at exactly two levels, both of them **positional, not declared as a dependency**:

1. **Between `Action`s / `ActionGroup`s**: strict list order (`ActionGroup::apply()`,
   `action.cc:506-527`) — an `Action` earlier in an `addAction()` sequence always runs, and (if
   repeating) stabilizes, before a later one starts its own repeat loop. The *only* record of *why*
   one Action must precede another is inline comments at the call site, e.g.
   `coreaction.cc:5877-5879` ("`Must come before restructurevarnode and infertypes`" /
   "`Must come before infertypes and nonzeromask`") and `coreaction.cc:6105` ("`Must come after
   required merges but before speculative`"). These are prose, not anything the compiler or a test
   checks — reordering two `addAction()` lines is legal C++ and silently changes decompiler behavior.
2. **Between `Rule`s sharing an opcode inside one `ActionPool`**: strict `addRule()` call order per
   opcode bucket (`ActionPool::processOp()`, `action.cc:822-875`) — for a given `PcodeOp`, the rules
   registered for its current opcode are tried in registration order, and the scan restarts from the
   first rule whenever one of them changes the op's opcode. There is no priority/precedence field, no
   static check that two rules can't both claim to apply and produce different results depending on
   which fires first, and no declared "rule X must run before rule Y" relationship anywhere in the
   `Rule` interface.

The `ruleaction.hh` file header is explicit that this is by design, not oversight: rules "are generally
applied simultaneously from a pool ... and **can interact with each other to produce an emergent
transformation**" (`ruleaction.hh:19-23`). In other words, the intended way the 162-wired-rule catalog
achieves complex rewrites (e.g. recognizing signed-division-by-constant compiler idioms, which takes
`RuleDivOpt` + `RuleSignNearMult` + `RuleSignDiv2` + `RuleDivTermAdd`/`RuleDivTermAdd2` +
`RuleModOpt`/`RuleSignMod2nOpt` cooperating across several passes, `ruleaction.hh:1249-1399`) is
**emergent composition of many small, individually-simple, individually-tested-by-inspection rules**,
with global convergence relying on `rule_repeatapply` fixed-pointing rather than any single rule
"knowing" about the others. This is efficient and has evidently converged in practice for two decades
of processor modules, but it means there is no artifact in the codebase — no dependency graph, no
partial order, no assertion — that a reader (or a refactor) can consult to know that two rules are
safe to reorder, safe to run in parallel, or mutually exclusive; see §4 and the companion flaws
document (ticket 0013) for the consequences.

## 2. Taxonomy of the `Rule` catalog

**Methodology**: 163 concrete `Rule` subclasses are declared across seven headers under
`Ghidra/Features/Decompiler/src/decompile/cpp` (`ruleaction.hh` 137, `subflow.hh` 12, `bitfield.hh` 6,
`double.hh` 4, `condexe.hh` 1, `constseq.hh` 2, `rulecompile.hh` 1 generic). 162 of those 163 are wired
into the default `"decompile"` pipeline via `addRule( new RuleXxx(...) )` calls in
`ActionDatabase::universalAction()` (`coreaction.cc:5835-6123`) — confirmed by counting occurrences of
that exact call pattern (134 in `oppool1` + 6 in `oppool2` + 22 in the `cleanup` pool = 162, with no
class instantiated twice). The one declared class with **no** `addRule()` call anywhere,
`RuleGeneric` (`rulecompile.hh:145`), is expected to be unwired — it is the pattern-DSL wrapper class
discussed in §3, only ever constructed dynamically by `RuleGeneric::build()`, never statically; the
other declared-but-unused class, `RuleEquality` (`ruleaction.hh:242-251`), has no such excuse and is
simply dead code (see §4). Categories below are a **reviewer-imposed** grouping built by reading every
class's Doxygen `\brief`/`\class` comment (`ruleaction.cc` alone carries 137 `\class` doc blocks) plus
the class declarations themselves; they do **not** correspond to anything enforced in the source. In
fact the codebase's *own* only categorization is the flat `group` string each rule is constructed with,
and that categorization is heavily skewed: of the 162 wired rules, **112 (69%) all share the single
group tag `"analysis"`** — i.e. the source's built-in taxonomy is essentially "most things" vs. a dozen
named special cases (`subvar`, `typerecovery`, `bitfields`, `floatprecision`, `doubleprecis`,
`cleanup`, `conditionalexe`, `segment`, `splitpointer`/`splitcopy`, `constsequence`, `stackvars`,
`protorecovery`, `nodejoin`, `deadcode`). Counts below are therefore approximate and assigned by
subject matter, not by group tag.

| Category | ~Rules | Representative examples (file:line) |
|---|---|---|
| **Constant folding & pure algebraic simplification** | ~25 | `RuleCollapseConstants` — collapses any collapsible constant expression (`ruleaction.hh:703`, applyOp `ruleaction.cc:3832`); `RuleSub2Add` — `V - W ⇒ V + W*-1` (`ruleaction.hh:753`, `ruleaction.cc:3997-4005`); `RuleCarryElim` — `carry(V,c) ⇒ -c <= V` (`ruleaction.hh:743`, `ruleaction.cc:3962-3973`); `RuleAddMultCollapse`, `RuleXorCollapse` (`ruleaction.hh:773`, `763`) |
| **Bit-twiddling / shift-mask idiom reversal** | ~30 | `RuleShiftBitops`, `RuleAndMask`/`RuleOrMask` (`ruleaction.hh:212,162,152`); `RuleShift2Mult` — power-of-2 shift → multiply (`ruleaction.hh:682`); `RuleHumptyDumpty`/`RuleDumptyHump`/`RuleHumptyOr` (SUBPIECE/PIECE round-trip cancellation, `ruleaction.hh:959-988`) |
| **Boolean/comparison normalization** | ~20 | `RuleBooleanNegate`, `RuleLessEqual`, `RuleEqual2Zero`, `RuleSLess2Zero` (`ruleaction.hh:571,469,1050,1039`); `RuleThreeWayCompare` — recognizes the `zext(V<W)+zext(V<=W)-1` three-way-compare idiom (`ruleaction.hh:1513`, applyOp `ruleaction.cc:10132`) |
| **Division/modulo compiler-idiom reversal** | ~15 | `RuleDivOpt` — `INT_MULT`+shift ⇒ `INT_DIV`/`INT_SDIV` (`ruleaction.hh:1283`, `ruleaction.cc:8267-8281`); `RuleModOpt` (`ruleaction.hh:1353`, `ruleaction.cc:8598-8607`); `RuleSignNearMult` (`ruleaction.hh:1342`, `ruleaction.cc:8537-8545`); `RuleSignDiv2`, `RuleDivChain`, `RuleDivTermAdd`/`RuleDivTermAdd2` (`ruleaction.hh:1249-1320`) |
| **Pointer arithmetic & struct/array recovery** | ~10 | `AddTreeState` helper class that classifies an `INT_ADD` tree into pointer-multiple/offset/remainder terms (`ruleaction.hh:44-83`); `RulePtrArith` — converts integer-arithmetic trees into `PTRADD`/`PTRSUB` (`ruleaction.hh:1070`, `ruleaction.cc:6553-6578`); `RuleStructOffset0` — `LOAD`/`STORE` of a struct's first field ⇒ `PTRSUB(,0)` (`ruleaction.hh:1082`, `ruleaction.cc:6602-6617`); `RulePushPtr`, `RuleLoadVarnode`/`RuleStoreVarnode` (`ruleaction.hh:1092,793,807`) |
| **Sub-variable / lane narrowing (type-width reduction)** | ~12 | `SubvariableFlow` — traces a logical narrow value through a wider containing `Varnode` and rewrites the graph around the narrow value (`subflow.hh:43-131`); `RuleSubvarAnd`/`RuleSubvarSubpiece`/`RuleSubvarShift`/`RuleSubvarZext`/`RuleSubvarSext` triggers (`subflow.hh:134-214`, e.g. applyOp `subflow.cc:1553`); `SplitFlow`/`RuleSplitFlow` — splits artificially-joined Varnodes (`subflow.hh:222-249`); `LaneDivide` — vector-lane splitting, driven by `ActionLaneDivide` (`subflow.hh:429`, `coreaction.hh:113-123`) |
| **Double-precision / wide-value stitching** | 4 | `RuleDoubleIn`/`RuleDoubleOut` — join/split double-precision operand pairs (`double.hh:320,334`); `RuleDoubleLoad`/`RuleDoubleStore` — collapse adjacent `LOAD`/`STORE`+`CONCAT`/`SUBPIECE` into one wide access (`double.hh:347,360`, applyOp `double.cc:3427`) |
| **Bitfield insert/extract folding** | 6 | `RuleBitFieldStore`/`RuleBitFieldOut`/`RuleBitFieldLoad`/`RuleBitFieldIn` (`bitfield.hh:181-229`, e.g. applyOp `bitfield.cc:1701`); `RulePullAbsorb`/`RuleInsertAbsorb` — fold surrounding ops into synthetic `ZPULL`/`SPULL`/`INSERT` bitfield ops (`bitfield.hh:229,251`) |
| **Memory-sequence / string-idiom recognition** | 2 | `HeapSequence` — collects a maximal run of character `STORE`s through one pointer (`constseq.hh:83-96`); `RuleStringCopy`/`RuleStringStore` — rewrite the run into a `strncpy`/`memcpy`-shaped user-op call (`constseq.hh:122,133`, applyOp `constseq.cc:981`) |
| **Control-flow-adjacent cleanup** | ~5 (+4 Actions) | `ConditionalExecution` — detects redundant re-evaluation of the same branch condition across a diamond and removes the join block (`condexe.hh:91-127`); `ActionConditionalExe` (`condexe.hh:133`, apply `condexe.cc:477`); `RuleOrPredicate` — `tmp1|tmp2` predication idiom ⇒ conditional move (`condexe.hh:172`, applyOp `condexe.cc:653`); plus Action-level `ActionUnreachable`, `ActionDoNothing`, `ActionRedundBranch`, `ActionDeterminedBranch` (`coreaction.hh:493-546`) |
| **Floating-point precision/conversion cleanup** | ~8 | `RuleFloatCast` — cancel back-to-back float widen/narrow casts (`ruleaction.hh:1455`, `ruleaction.cc:9537-9546`); `RuleIgnoreNan` (`ruleaction.hh:1466`, `ruleaction.cc:9726`); `RuleUnsigned2Float` — recognizes the software unsigned-to-float idiom (`ruleaction.hh:1480`, `ruleaction.cc:9770-9781`); `RuleFloatSign`/`RuleFloatSignCleanup` (`ruleaction.hh:1573,1584`) |
| **ABI / architecture-specific quirks** | ~4 (+extension point) | `RuleFuncPtrEncoding` — strips ARM/THUMB low-bit function-pointer encoding (`ruleaction.hh:1502`, `ruleaction.cc:9900-9912`); `RuleSegment` — propagates constants through segmented-memory `SEGMENTOP` (`ruleaction.hh:1400`, `ruleaction.cc:8991-8999`); `RulePtrFlow` — marks pointer truncation points for architectures with sub-register pointers (`ruleaction.hh:1411`, `ruleaction.cc:9036-9163`); `ActionStackPtrFlow` — models extra-pop stack-pointer adjustment across calls (`coreaction.hh:89-105`); processor-specific rules can also be injected wholesale via `Architecture::extra_pool_rules` (`architecture.hh:189`, consumed at `coreaction.cc:6020-6022`) |
| **Late "cleanup"-pool rewrites** | 22 | Runs once, after the main fixed-point loop, over the already-structured graph: `Rule2Comp2Sub`, `RuleAddUnsigned`, `RuleMultNegOne` (`ruleaction.hh:1154,1143,1132`); `RulePtrsubCharConstant` (`ruleaction.hh:1176`, `ruleaction.cc:7299`); `RuleExtensionPush` (`ruleaction.hh:1188`, `ruleaction.cc:7362`); `RuleSplitCopy`/`RuleSplitLoad`/`RuleSplitStore` — break composite-typed COPY/LOAD/STORE into per-field ops (`subflow.hh:317-356`, applyOp `subflow.cc:2964`) |
| **Copy propagation / dataflow bookkeeping** | ~6 | `RulePropagateCopy` (`ruleaction.hh:723`, `ruleaction.cc:3902-3910`); `RuleTransformCpool` — resolves Java/DEX constant-pool references (`ruleaction.hh:713`, `ruleaction.cc:3862-3873`); `RuleEarlyRemoval` — removes dead ops early, runs on every opcode (`ruleaction.hh:85-94`) |
| **Predication / conditional-move recognition** | 1 | `RuleConditionalMove` — collapses `if(cond) x=1; else x=0;`-shaped diamonds into a single assignment (`ruleaction.hh:1439`, `ruleaction.cc:9347-9376`) |

At least 26 distinct `Rule`/helper classes are cited above with file:line (exceeding the ticket's
15-rule minimum), spread across all listed categories.

One dead-code note surfaced by this survey: `RuleEquality` is fully declared and implemented
(`ruleaction.hh:242-251`) but is **never** `addRule()`-ed anywhere in `coreaction.cc` — it is
unreachable. Nothing in the build enforces that every declared `Rule` subclass is actually wired into
a pipeline, so such drift is silent (see §4).

## 3. The `rulecompile.cc` pattern DSL

`rulecompile.hh`/`rulecompile.cc` implement a small, separate, purpose-built language for describing a
`Rule`'s match pattern declaratively instead of as hand-written C++ traversal code. The grammar is
documented directly in the header (`rulecompile.hh:160-202`): statements build up named references to
ops/varnodes/constants (`o -> v` "v is the output of o", `o <-(1) #c` "input 1 of o is a constant now
named c", `v ->! o` "o is the *only* op reading v", etc.) and attach constraints (`o(+)` "opcode must
be INT_ADD", `v1(==v2)` "same varnode"), with `[ ... | ... ]` for alternation and parenthesized
grouping for sub-statement lists. A hand-written lexer (`RuleLexer`, `rulecompile.hh:23-61`) feeds a
`ruleparse.y`-generated grammar (`RuleCompile::run()`) that builds a tree of `ConstraintGroup` objects
(from `unify.hh`, a general graph-unification engine — not itself part of this review's scope) via
`RuleCompile`'s `newOp`/`newVarnode`/`opInput`/`varCompareConstraint`/... builder methods
(`rulecompile.hh:96-128`). The compiled `ConstraintGroup` is wrapped in a `RuleGeneric`
(`rulecompile.hh:145-158`), a genuine `Rule` subclass whose `applyOp()` just drives the constraint
graph against one `PcodeOp` via `UnifyState` (`rulecompile.cc:864-871`) instead of running bespoke C++.

**How much of the catalog uses it: none, in production.** `RuleGeneric::build()` is reachable from only
one call site, `Architecture::decodeDynamicRule()` (`architecture.cc:706-738`), which parses a
`<rule name=... group=...>` XML element (intended for per-processor `.cspec`/`.pspec` customization)
and is itself gated behind `#ifdef CPUI_RULECOMPILE` (`architecture.cc:731`) — and that whole call site
already contains a stale reference to an undeclared `el` variable (`architecture.cc:732`,
`el->getContent()` where the surrounding code uses a `Decoder& decoder`, not an `Element* el`),
strongly suggesting this path has not compiled, let alone run, in some time. `rulecompile.cc`'s entire
body — not just this one function — is wrapped in `#ifdef CPUI_RULECOMPILE ... #endif`
(`rulecompile.cc:16` / `:889`), and `CPUI_RULECOMPILE` is listed in the build's `Makefile` only as a
commented-out, opt-in flag ("`# CPUI_RULECOMPILE # Allow user defined dynamic rules`",
`Makefile:114`) — off by default, including in the `decomp_dbg` debug build target
(`Makefile:125,133` only add `-DCPUI_DEBUG`). All 162 rules that actually ship
(§2) are conventional hand-written `Rule` subclasses in `ruleaction.cc` and its satellite files; none
are `RuleGeneric` instances.

**Why it exists anyway**: the interface layer's `parse rule <file> [debug]` console command
(`IfcParseRule`, guarded by the same `#ifdef`, `ifacedecomp.cc:3221-3256`) compiles a `.rule`-DSL file
and then runs it through `UnifyCPrinter` to **pretty-print the equivalent generated C++** to the
console. Read together with the always-disabled dynamic-loading path, this points to the DSL's actual
intended workflow: a developer prototypes a candidate rule's match pattern tersely in the mini-language,
uses `parse rule` to see (and presumably hand-copy into `ruleaction.cc`) the C++ a real `Rule::applyOp`
override would need, rather than shipping the DSL-interpreted form. It is best understood as **a
second, informal, mostly-vestigial "language" for rule authoring that exists to generate scaffolding
for the first (hand-written C++) rule catalog**, not as an alternate runtime rule format — it is
currently unreachable in any release build.

## 4. Extension and debugging affordances vs. known gaps

### What exists

- **Grouplist toggling** — `option currentaction <group> on|off` (`OptionCurrentAction`,
  `options.cc:648-680`) and `option setaction <name> [newcopy]` (`OptionSetAction`, `options.cc:626-646`)
  let a `.cspec`/console user turn whole named rule/action groups on or off, or clone a root Action to
  customize, without touching C++ — this is the one genuinely-live, always-compiled extension point.
- **Per-processor rule injection** — `Architecture::extra_pool_rules` (`architecture.hh:189`, drained
  into `oppool1` at `coreaction.cc:6020-6022`) lets processor-specific setup code hand the engine extra
  `Rule*` objects (in practice, only ever populated by the `CPUI_RULECOMPILE`-gated dynamic-rule loader
  described in §3, so this extension point is currently dormant too).
- **Per-Action/per-Rule enable/disable and warnings** — `Action::disableRule()`/`enableRule()`
  (`action.cc:226-251`) and `Rule::setDisable()`/`turnOnWarnings()` (`action.hh:219-227`), addressed by
  a `:`-separated name path (`Action::getSubRule()`, `action.cc:285-289` and overrides), let any single
  named rule anywhere in the tree be switched off or made to print a one-time warning when it fires —
  always compiled in, not gated.
- **Statistics** — `print actionstats` / `reset actionstats` console commands
  (`IfcPrintActionstats`/`IfcResetActionstats`, `ifacedecomp.cc:3273-3303`) dump/reset per-Action and
  per-Rule `count_tests`/`count_apply` counters (`Action::printStatistics`, `action.cc:93-97`;
  `Rule::printStatistics`, `action.cc:697-701`) — cheap, always-available insight into which rules are
  actually firing on a given binary.
- **Breakpoint/tracing facility** — `Action`/`Rule` both support `break_start`/`break_action`
  breakpoints (`action.hh:73-78`, checked in `Action::perform()` at `action.cc:307-341`) and, when
  built with `OPACTION_DEBUG`, per-op debug tracing keyed by address range and sequence-number range
  (`Funcdata::debugSetRange`/`debugSetBreak`, driven by console commands `debug action`,
  `trace break`, `trace address`, `trace enable/disable/clear/list`, `break jumptable` —
  `ifacedecomp.cc:3457-3529`, registered at `ifacedecomp.cc:150-157`). This is the most powerful
  debugging affordance in the engine (single-step which rule fired on which op at which point in the
  pipeline) but **`OPACTION_DEBUG` is commented out by default in the `Makefile` (`Makefile:117`) and
  is not turned on even by the `decomp_dbg`/debug build targets** (`Makefile:125,133` set only
  `-DCPUI_DEBUG`) — a developer must manually edit the Makefile to get it, so in practice almost nobody
  exercises this path.

### What's missing

- **No rule-level unit isolation.** There is no harness that constructs a minimal P-code fragment and
  asserts a single named `Rule::applyOp()` fires (or doesn't) on it in isolation; correctness is
  established only by running whole functions through the full pipeline and diffing decompiler output
  (see ticket 0010's Java-side `datatests` inventory for how testing actually happens today). A change
  to one rule can only be validated by its effect on the emergent, whole-pipeline result.
- **No declared ordering/dependency model.** As described in §1, all cross-Action and cross-Rule
  ordering constraints live only in `//` comments at the `addAction()`/`addRule()` call sites
  (`coreaction.cc:5877-5879,6105`, etc.). There is no type, annotation, or assertion that would catch a
  reordering that violates one of these documented (or undocumented) dependencies — a merge conflict or
  a well-intentioned cleanup pass reordering two `addAction()` lines compiles and links exactly as
  before, and would only surface as a behavior regression on some future decompile.
- **No confluence/termination guarantee beyond "make progress or stop."** `rule_repeatapply` fixed-points
  purely on "did `count` increase"; nothing checks that the ~134-rule `oppool1` pool is confluent
  (produces the same normal form regardless of firing order) — the file-header comment
  (`ruleaction.hh:19-23`) explicitly frames rule interaction as *emergent*, i.e. by design, not proven.
- **Silent dead code.** `RuleEquality` (§2) is fully implemented but wired into no pipeline; nothing in
  the build flags an unreferenced `Rule` subclass.
- **The pattern DSL is effectively unshippable.** As detailed in §3, its one production call site
  (`Architecture::decodeDynamicRule`, `architecture.cc:706-738`) references an undeclared variable and
  its enabling macro is off by default everywhere — a `.cspec` author cannot actually use `<rule>`
  elements against a stock build today, despite the XML schema and console tooling (`parse rule`,
  `experimental rules`) all still existing.
- **Category-level classification exists only at the granularity of the flat `group` string**, and 112
  of 162 rules (69%) share the single tag `"analysis"` (§2) — there is no source-level equivalent of
  the "arithmetic simplification / pointer recovery / cast cleanup / ABI quirk" taxonomy in this
  document; a reader has to read each rule's doc comment individually (as this review did) to recover
  it.

## Related documents

- Control-flow structuring (`ActionBlockStructure`/`ActionFinalStructure`/`blockaction.hh`, only
  referenced here for pipeline-ordering context) — ticket 0007,
  [`control-flow-structuring.md`](control-flow-structuring.md).
- Type inference/cast insertion (`ActionInferTypes`/`ActionSetCasts`, interleaved into the same
  pipeline) — tickets 0005/0009, [`type-system.md`](type-system.md),
  [`expression-cast-printer.md`](expression-cast-printer.md).
- SSA/heritage construction that feeds the first pass of this engine — ticket 0004,
  [`ssa-varnode-heritage.md`](ssa-varnode-heritage.md).
- Rule-engine flaw analysis (ordering/interaction risk in depth) — ticket 0013,
  `02-flaws/rule-engine-block-structuring.md` (not yet written at the time of this document).
