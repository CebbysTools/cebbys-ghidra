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
