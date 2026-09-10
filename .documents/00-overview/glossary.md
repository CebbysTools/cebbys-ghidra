# Glossary

Shared terminology for the decompiler refactor documentation set. Add terms here as review/flaw docs
introduce them rather than redefining them locally; link back to this file instead. Each entry should
note the primary defining source file.

Contributors: append new terms alphabetically, keep entries to 1-3 sentences, and prefer linking to
the review document that treats the term in depth over re-explaining it here.

## Core terms (seed list — expand during review)

- **P-code**: Ghidra's processor-independent intermediate representation, emitted by SLEIGH from raw
  machine code. The decompiler's sole input. (Out of scope for this refactor; see root ticket
  non-goals.)
- **Varnode**: A P-code operand — a fixed-size storage location (address-space offset + size) at one
  point in the raw op-code stream. See `varnode.hh`. Detailed in
  [`../01-review/ssa-varnode-heritage.md`](../01-review/ssa-varnode-heritage.md).
- **PcodeOp**: A single P-code operation (opcode + input/output varnodes). See `op.hh`.
- **Heritage**: The SSA-construction pass that decides which storage locations need phi-node
  (`MULTIEQUAL`) insertion. See `heritage.hh`.
- **HighVariable**: The source-level variable a group of related SSA `Varnode`s merges into for
  display purposes. See `variable.hh`.
- **Funcdata**: The per-function object owning the P-code graph, symbol scope, and block structure —
  effectively the working set for decompiling one function. See `funcdata.hh`.
- **Datatype**: The decompiler's internal type representation (distinct from, but synchronized with,
  Ghidra's `DataTypeManager`-backed types). See `type.hh`.
- **Action / Rule**: The simplification-engine building blocks. An `Action` is a scheduled pass; a
  `Rule` is a single local pattern-driven transformation applied to a fixed point. See `action.hh`,
  `ruleaction.hh`.
- **FlowBlock / BlockGraph**: The structured control-flow representation built from the raw CFG by the
  structuring algorithm (`blockaction.cc`). See `block.hh`.
- **ProtoModel / FuncProto**: `ProtoModel` is a static calling-convention description (from `.cspec`);
  `FuncProto` is a specific function's recovered prototype instantiated against a model. See
  `fspec.hh`.
- **PrintLanguage / PrintC**: The abstract token-emission base and its concrete C-language
  implementation that turn the structured, typed representation into source text. See
  `printlanguage.hh`, `printc.hh`.
- **CastStrategy**: The policy object deciding when an explicit cast must appear in printed output.
  See `cast.hh`.
- **Capability**: The self-registration mechanism new architectures/rules/languages use to plug into
  the decompiler at startup. See `capability.hh`.
- **ConstantPool / CPoolRecord**: The container (`ConstantPool`) and per-entry record (`CPoolRecord`)
  used to represent constant-pool references for deferred-compilation languages (e.g. Java byte-code) —
  an instruction-level reference resolves to a primitive value, string, method, field, or data-type
  descriptor. See `cpool.hh`. Detailed in
  [`../01-review/type-system.md`](../01-review/type-system.md).
- **ResolvedUnion**: A cached record of which field of a `TypeUnion` (or effectively-union struct/
  pointer) a specific data-flow access resolves to, produced by the `ScoreUnionFields` heuristic search
  and cached per `(PcodeOp, slot)` edge. See `unionresolve.hh`, detailed in
  [`../01-review/type-system.md`](../01-review/type-system.md).
- **TypeFactory**: The per-`Architecture` container that owns, interns (by name+id), and canonicalizes
  every `Datatype` object; `TypeFactoryGhidra` is the subclass that additionally queries Ghidra's
  `DataTypeManager` over the wire on a cache miss. See `type.hh`, `typegrp_ghidra.hh`, detailed in
  [`../01-review/type-system.md`](../01-review/type-system.md).
- **TypeOp**: The per-P-code-opcode policy object pairing operator emulation/display with data-type
  propagation rules (`propagateType()`); one instance per `OpCode`, referenced by every `PcodeOp` of
  that opcode. See `typeop.hh`, detailed in [`../01-review/type-system.md`](../01-review/type-system.md).
- **typelock**: A `Varnode` flag marking its `Datatype` as locked (usually inherited from a type-locked
  `Symbol`) so the type-propagation engine may refine an unlocked Varnode's type but can never overwrite
  a locked one. See `varnode.hh`, detailed in
  [`../01-review/type-system.md`](../01-review/type-system.md).
- **ActionDatabase**: `Architecture`'s registry of named *root* `Action`s (`decompile`, `jumptable`,
  `normalize`, `paramid`, `register`, `firstpass`) sliced out of one hand-built "universal" `Action`
  tree. See `action.hh`, detailed in
  [`../01-review/architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md).
- **ArchitectureCapability**: The `Capability` extension point for bootstrapping an `Architecture` from
  a file (e.g. Bfd, raw binary, XML save-file). Notably *not* how `ArchitectureGhidra` is created — see
  `architecture.hh` and
  [`../01-review/architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md).
- **ArchitectureGhidra**: The `Architecture` subclass used whenever the decompiler is driven by the
  Ghidra Java client; its `build*` overrides fetch symbols, p-code, types, etc. from that client over
  the wire protocol instead of from a local file. See `ghidra_arch.hh`.
- **GhidraCommand / GhidraCapability**: The `Capability`-registered dispatch mechanism for commands the
  Ghidra client can issue to the `decompile` subprocess (`registerProgram`, `decompileAt`, `setAction`,
  …). See `ghidra_process.hh`.
- **PackedEncode / PackedDecode**: The compact binary TLV encoding actually used on the wire between
  Ghidra and the `decompile` subprocess for commands and query responses (distinct from the XML
  `Encoder`/`Decoder` pair used for spec files and console save/restore). See `marshal.hh`.
- **CollapseStructure**: The control-flow structuring algorithm — repeatedly matches and collapses
  sub-graphs of the CFG into `FlowBlock` structure nodes until one node remains, marking edges as
  unstructured (`goto`) when no further collapse rule applies. See `blockaction.hh`. Detailed in
  [`../01-review/control-flow-structuring.md`](../01-review/control-flow-structuring.md).
- **Irreducible edge**: A control-flow edge that must be removed for the CFG to become reducible (a
  loop entered only through its header). Detected via a Tarjan-style spanning-tree/reachunder analysis
  and thereafter treated like a `goto` edge by the structuring algorithm. See `block.cc`
  (`BlockGraph::findIrreducible`). Detailed in
  [`../01-review/control-flow-structuring.md`](../01-review/control-flow-structuring.md).
- **JumpTable**: Recovers a `switch` statement's case-value-to-block mapping from a `CPUI_BRANCHIND`
  op, independent of (and prior to) block structuring. See `jumptable.hh`. Detailed in
  [`../01-review/control-flow-structuring.md`](../01-review/control-flow-structuring.md).
- **Symbol**: The formal, named, typed entity in the scope/symbol-table hierarchy that a `HighVariable`
  may be tied to via one or more `SymbolEntry` storage mappings. See `database.hh`. Detailed in
  [`../01-review/ssa-varnode-heritage.md`](../01-review/ssa-varnode-heritage.md).
- **Cover**: The topological liveness range (per basic block, def-point to last use) of a single
  `Varnode`'s SSA value; the primitive `Merge` intersection-tests before grouping `Varnode`s into a
  `HighVariable`. See `cover.hh`.
- **Merge**: The post-heritage pass that groups `Varnode`s into `HighVariable`s, split into *forced*
  merges (must happen) and *speculative* merges (attempted, abandoned on any `Cover` intersection). See
  `merge.hh`.
- **VarnodeData**: The minimal wire-level `{address space, offset, size}` triple with no dataflow
  links; what raw P-code speaks before any SSA construction, as opposed to the richer `Varnode`. See
  `pcoderaw.hh`.
- **ActionPool**: An `Action` that pools many `Rule` objects and dispatches each `PcodeOp` to only the
  rules registered for its opcode (via `Rule::getOpList()`), applying them repeatedly to a fixed point.
  The two largest instances (`oppool1`/`oppool2` in `coreaction.cc`) hold nearly the entire `Rule`
  catalog. See `action.hh`. Detailed in
  [`../01-review/action-rule-engine.md`](../01-review/action-rule-engine.md).
- **Root Action / grouplist**: A named, complete transformation pipeline (e.g. `"decompile"`,
  `"jumptable"`, `"normalize"`) derived by cloning one `universal` Action tree against a named list of
  `group` tags (a `grouplist`); each `Action`/`Rule` is tagged with a group string at construction and
  is included in a derived root Action only if its group is in that root's grouplist. See
  `ActionDatabase` in `action.hh`. Detailed in
  [`../01-review/action-rule-engine.md`](../01-review/action-rule-engine.md).
- **RuleGeneric / rule DSL**: A small, separate pattern-matching mini-language (`rulecompile.hh/cc`)
  for describing a `Rule`'s match pattern declaratively instead of as hand-written C++; compiles to a
  `RuleGeneric` instance driven by a `ConstraintGroup`/`UnifyState` graph unifier. Gated behind the
  `CPUI_RULECOMPILE` build flag, which is off by default, so no shipped `Rule` is a `RuleGeneric`
  today. Detailed in [`../01-review/action-rule-engine.md`](../01-review/action-rule-engine.md).
- **Emit / EmitPrettyPrint**: `Emit` is the abstract low-level token-layout interface (line breaks,
  indenting, optional XML markup) that `PrintLanguage` targets instead of writing characters directly;
  `EmitPrettyPrint` is the concrete Oppen-style line-wrapping implementation used in practice, wrapping
  an inner `Emit` (`EmitMarkup` or `EmitNoMarkup`). See `prettyprint.hh`. Detailed in
  [`../01-review/expression-cast-printer.md`](../01-review/expression-cast-printer.md).
- **OpToken**: A static per-operator printing descriptor (text, precedence, associativity, spacing) used
  by `PrintC`/`PrintJava`; the language-agnostic `PrintLanguage::parentheses()` algorithm decides
  parenthesization purely from these fields, for both expressions and complex-type declarators. See
  `printlanguage.hh`. Detailed in
  [`../01-review/expression-cast-printer.md`](../01-review/expression-cast-printer.md).
- **ParamTrial / ParamActive**: `ParamTrial` is one putative parameter (address, size, flags) observed
  at a call site or function entry; `ParamActive` is the working set of trials for one call/function,
  iteratively narrowed to a formal parameter list. See `fspec.hh`. Detailed in
  [`../01-review/signature-calling-conventions.md`](../01-review/signature-calling-conventions.md).
- **ModelRule**: A declarative, `.cspec`-configured rule (`DatatypeFilter` + optional `QualifierFilter`
  → `AssignAction`) that assigns storage for a parameter/return value class, layered on top of (and
  falling back to) the older hardcoded ordered-resource-list assignment algorithm. See
  `modelrules.hh`. Detailed in
  [`../01-review/signature-calling-conventions.md`](../01-review/signature-calling-conventions.md).
- **Override**: A per-function container of commands (prototype overrides, indirect-call retargeting,
  flow-type conversion, forced gotos) that let asserted (user- or analysis-supplied) information
  short-circuit the decompiler's own recovery for one function/call site. See `override.hh`. Detailed
  in [`../01-review/signature-calling-conventions.md`](../01-review/signature-calling-conventions.md).
- **ParamID**: A secondary, independent analysis (`ParamIDAnalysis`) that scores each already-recovered
  (or raw) parameter's data-flow usage for confidence, feeding Ghidra's bulk signature-commit tooling;
  it does not itself assign storage. See `paramid.hh`. Detailed in
  [`../01-review/signature-calling-conventions.md`](../01-review/signature-calling-conventions.md).
- **DecompInterface**: The Java-side façade for a single decompiler subprocess — owns its lifecycle
  (spawn, auto-respawn on crash, dispose), caches configuration, and exposes
  `decompileFunction`/`structureGraph`/`generateSignatures`. See
  `Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/DecompInterface.java`. Detailed in
  [`../01-review/java-integration-testing.md`](../01-review/java-integration-testing.md).
- **DecompileCallback**: The Java handler for queries the native `decompile` process makes back into
  Ghidra mid-decompilation (bytes, comments, symbols, p-code injection, datatypes, strings) — the "pull"
  side of `GhidraCommand`/`ArchitectureGhidra`'s wire protocol. See
  `Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/DecompileCallback.java`. Detailed in
  [`../01-review/java-integration-testing.md`](../01-review/java-integration-testing.md).
- **HighFunction**: The Java-side decoded mirror of one decompiled function — `HighVariable`s, `FuncProto`,
  and symbol mappings reconstructed from the wire response — that both the UI and commit-back actions
  (`HighFunctionDBUtil`) operate on. See
  `Ghidra/Framework/SoftwareModeling/src/main/java/ghidra/program/model/pcode/HighFunction.java`.
  Detailed in [`../01-review/java-integration-testing.md`](../01-review/java-integration-testing.md).
- **ClangTokenGroup**: The root of the parsed C-markup token tree (`DecompileResults.getCCodeMarkup()`)
  that the `ghidra.app.decompiler.component` UI package renders, navigates, and highlights — a consumer
  of, not a participant in, the P-code→C pipeline. See
  `Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/ClangTokenGroup.java`. Detailed in
  [`../01-review/java-integration-testing.md`](../01-review/java-integration-testing.md).
- **datatests / FunctionTestCollection**: The decompiler's data-driven regression format — one XML file
  per scenario, embedding a synthetic binary image, a console script, and `<stringmatch>` regex
  assertions (with min/max occurrence bounds) checked against printed C output; loaded and run by
  `FunctionTestCollection` in the separate `ghidra_test` executable, distinct from the production
  `decompile` binary. See `Ghidra/Features/Decompiler/src/decompile/cpp/testfunction.hh`. Detailed in
  [`../01-review/java-integration-testing.md`](../01-review/java-integration-testing.md).
