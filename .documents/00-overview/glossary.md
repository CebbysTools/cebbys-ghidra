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
