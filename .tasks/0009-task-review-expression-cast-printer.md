# 0009 — [TASK] Expression / Cast Logic & C Pretty-Printer Review

## Type
Task

## Status
`done`

## Parent
0002 (epic) / 0001 (root)

## Objective
Document the final leg of the pipeline: turning the typed, structured P-code/block tree into actual
C source text, including cast insertion and language-specific formatting.

## Scope / Primary Files
- `printlanguage.cc/hh` (abstract base — the token/markup emission model shared across languages)
- `printc.cc/hh` (the concrete C emitter — operator precedence, parenthesization, statement/expression
  formatting, declaration syntax including function pointers and arrays)
- `printjava.cc/hh` (the Java-flavored emitter — review briefly, primarily as a comparison point for
  "how much of PrintLanguage is really language-agnostic vs. C-biased")
- `cast.cc/hh` (`CastStrategy` — decides when an explicit cast must appear in output for correctness
  or clarity)
- `prettyprint.cc/hh` (token stream buffering, line-wrapping/indentation engine, `Emit` abstraction
  feeding both text and "structured" clients like the Ghidra GUI's token stream)
- `expression.cc/hh` (`PartialCastPropagation`? confirm actual role — expression-tree-level helpers
  used prior to printing)
- `grammar.cc/hh` if it plays a role here (confirm; otherwise note it's SLEIGH-grammar-related and out
  of scope, don't force a connection)

## Deliverables
`.documents/01-review/expression-cast-printer.md` covering:
1. The `Emit`/token-stream abstraction: how PrintLanguage decouples "what to say" from "how to lay it
   out" (markup begin/end, line breaks), and how the Ghidra GUI consumes this stream to get clickable
   tokens (link to the relevant Java `ClangToken`/decompiler component classes at a high level, cross-
   referencing 0010 rather than duplicating).
2. `PrintC` specifics: precedence table handling, how it decides parenthesization, declaration
   printing for complex types (pointers-to-arrays, function pointers).
3. `CastStrategy`: the rules for when a cast is inserted, and where "implicit" vs. "explicit" casts
   are decided — this is a common source of both correctness bugs and readability complaints.
4. Compare `PrintC` and `PrintJava` structurally: what's genuinely shared via `PrintLanguage` vs.
   what's duplicated per-language, as a concrete data point on "is multi-language output actually
   well-factored today."

## Acceptance Criteria
- [x] Document exists, linked from index, file:line cited throughout.
- [x] Explicit shared-vs-duplicated breakdown between PrintC and PrintJava.
- [x] At least 3 concrete cast-insertion rules explained with the source condition that triggers them.

## Dependencies
None — parallel with 0003–0008, 0010.
