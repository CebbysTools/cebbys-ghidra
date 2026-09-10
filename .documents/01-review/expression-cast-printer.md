# Expression / Cast Logic & the C Pretty-Printer

> Produced by [`.tasks/0009`](../../.tasks/0009-task-review-expression-cast-printer.md), child of
> [`.tasks/0002`](../../.tasks/0002-epic-review-current-implementation.md). See
> [`00-index.md`](00-index.md) for how this fits the end-to-end pipeline narrative, and
> [`java-integration-testing.md`](java-integration-testing.md) (task 0010) for how the Java-side GUI
> consumes the token stream described here in detail.

## Scope and file map

| File | Role |
|---|---|
| `printlanguage.hh/.cc` | Abstract base: RPN expression-stacking engine, `Emit` markup dispatch, per-p-code-op virtual hooks |
| `printc.hh/.cc` (~3500 lines) | The C emitter — precedence table, declaration printing, statement/expression formatting |
| `printjava.hh/.cc` | The Java emitter — a thin `PrintC` subclass, not a `PrintLanguage` sibling |
| `cast.hh/.cc` | `CastStrategy` — the *policy* for whether a conversion needs a visible cast |
| `prettyprint.hh/.cc` | `Emit`/`TokenSplit`/`EmitPrettyPrint` — token buffering and the Oppen line-wrap algorithm |
| `expression.hh/.cc` | Mixed bag; only its bitfield-expression classes are printer-relevant (see [below](#expressionhhcc-mostly-out-of-scope)) |
| `grammar.hh/.cc/.y` | **Not** SLEIGH-related, and not printer-related either — see [below](#grammarhhccy-out-of-scope-but-not-for-the-reason-the-ticket-guessed) |

All line numbers below are relative to
`Ghidra/Features/Decompiler/src/decompile/cpp/`.

---

## 1. The `Emit`/token-stream abstraction

### 1.1 Two decoupled layers

`PrintLanguage` (`printlanguage.hh:138`) does not write characters to a stream itself. It decides *what*
needs to be said — which tokens, in what order, with what parenthesization — and hands each decision to
an `Emit` object (`prettyprint.hh:95`) that decides *how* to lay it out (line breaks, indentation,
markup). This is the central design split the ticket asked about:

- **`PrintLanguage` (and `PrintC`/`PrintJava`)** walk the `Funcdata`'s p-code graph and block tree and
  call `emit->tagVariable(...)`, `emit->tagOp(...)`, `emit->openParen(...)`, `emit->tagLine()`, etc.
  These are pure semantic events — "here is a variable token", "start a new statement" — with no line
  budget or indentation math embedded in them.
- **`Emit`** (and its concrete descendants) turn that stream of semantic events into an actual character
  stream, inserting spaces, line breaks and indent levels, and optionally wrapping each token in
  structured markup.

Three concrete `Emit` implementations exist:
- `EmitNoMarkup` (`prettyprint.hh:559`) — dumps tokens straight to an `ostream`, no pretty-printing, no
  markup. Used as the low-level sink when no line-wrapping is wanted.
- `EmitMarkup` (`prettyprint.hh:512`) — wraps every token in an XML element (`ELEM_VARIABLE`,
  `ELEM_OP`, `ELEM_FUNCNAME`, …) carrying a `varref`/`opref` attribute that is a numeric ID
  (`vn->getCreateIndex()` / `op->getTime()`) back into the underlying data-flow graph
  (`prettyprint.cc:183-207`). This is the wire format consumed by the GUI (§1.3).
- `EmitPrettyPrint` (`prettyprint.hh:1068`) — the one actually used in practice. It is a **decorator**,
  not a replacement: it holds a `lowlevel` `Emit*` (`prettyprint.cc:570`, defaults to `EmitNoMarkup`, but
  is swapped for an `EmitMarkup` via `setMarkup()`/`setPackedOutput()`) and implements the line-wrapping
  algorithm on top, forwarding fully-decided tokens down to `lowlevel`. `PrintLanguage`'s constructor
  always allocates an `EmitPrettyPrint` (`printlanguage.cc:69`), so pretty-printing is not optional —
  only whether markup rides along underneath it is.

### 1.2 The RPN expression engine

Expression printing does not walk the p-code tree top-down. `PrintLanguage` maintains a manual
Reverse-Polish-Notation stack, `revpol` (`printlanguage.hh:265`, `vector<ReversePolish>`), and a
`nodepend` queue of not-yet-visited operand Varnodes (`printlanguage.hh:266`). The core recursive walk
happens through mutual recursion between:

- `pushOp`/`pushAtom` (`printlanguage.cc:129`, `:162`) — push an operator token or a leaf token; as soon
  as an operator has seen all its operands (`visited == tok->stage`), it is emitted and popped
  (`printlanguage.cc:173-185`).
- `recurse()` (`printlanguage.cc:518`) — drains `nodepend`, and for each *implied* Varnode (one with no
  explicit name — an inline sub-expression) dispatches to `defOp->getOpcode()->push(this, defOp, op)`,
  which is a `TypeOp` virtual method that in turn calls back into one of `PrintLanguage`'s ~70 `op*`
  virtuals (`opIntAdd`, `opPtradd`, `opCast`, …).
- `opBinary`/`opUnary` (`printlanguage.cc:550`, `:570`) — the common-case helpers most `op*` overrides
  delegate to; they push the operator then push both operand sub-expressions (in reverse order, for
  stack efficiency — explicitly commented on at `printlanguage.cc:560`).

Explicit Varnodes (ones with a real symbol name) are pushed directly as leaf `Atom`s via
`pushVnExplicit`/`pushSymbolDetail` (`printlanguage.cc:218`, `:239`) rather than recursed into, which is
what stops the RPN walk at variable boundaries instead of re-deriving the whole function as one giant
expression.

### 1.3 GUI consumption (high level; see 0010 for the deep dive)

When markup is enabled, each token the RPN engine emits becomes an `ELEM_VARIABLE`/`ELEM_OP`/`ELEM_FUNCNAME`/…
XML element with a `varref`/`opref` id (`prettyprint.cc:21-38`, `:183-215`). On the Java side,
`ghidra/app/decompiler/ClangToken.java` (`ClangToken.buildToken`, `decode`) reconstructs one `ClangToken`
object per element and resolves those ids back to `Varnode`/`PcodeOp`/`HighVariable` objects via a
`PcodeFactory`, which is what lets the Decompiler window highlight all occurrences of a variable on
click and drive the GUI's rename/retype actions. The full inventory of `ClangToken` subclasses,
`DecompInterface`'s XML round-trip, and the associated test infrastructure is task 0010's territory —
see **`java-integration-testing.md`** rather than this document for details.

---

## 2. `PrintC` specifics

### 2.1 Precedence and parenthesization

`PrintC` defines ~50 static `OpToken` objects (`printc.cc:23-77`), each an operator's complete printing
description: literal text, RPN "stage" count, a numeric **precedence**, associativity, a `tokentype`
(`binary`, `unary_prefix`, `postsurround` — e.g. `a[b]`, `presurround` — e.g. `(cast)x`, `space`, or
`hiddenfunction`), and spacing/indent-bump hints. Precedence is an arbitrary integer scale roughly
mirroring the C standard's precedence table, e.g.:

```
comma        = 2      assignment    = 14     boolean_or = 18   ...   bitwise_or = 26
equal        = 38     less_than     = 42     shift_left = 46   binary_plus = 50
multiply     = 54     typecast/unary = 62    scope/member/subscript/call = 66-70
```//(printc.cc:23-77)

Parenthesization is *not* decided per-operator with hard-coded rules; it is computed once, generically,
in `PrintLanguage::parentheses()` (`printlanguage.cc:270-324`) purely from the `OpToken` metadata of the
operator being emitted and the operator underneath it on the RPN stack. The logic is a `switch` on the
*outer* token's `tokentype`:

- `binary`/`space`: parenthesize the inner expression if the outer precedence is strictly higher, never
  if strictly lower, and for equal precedence only if the tokens aren't identical-and-associative
  (`printlanguage.cc:278-287`) — this is where `a + b + c` avoids parenthesizing `a + b` (associative
  `+`) but `a - (b - c)` keeps its parens (subtraction isn't marked associative, `printc.cc:40`).
  There's also a special case at `:286` for a `postsurround` (`a[i]`, `f(x)`) operand appearing as the
  *first* stage of the outer binary — since `a[i]` is fully bracket-delimited already, no extra parens
  are needed even at equal precedence.
- `unary_prefix`, `presurround` (a cast), `postsurround` (subscript/call): each has its own small
  variant of the same precedence comparison, plus a check for "inside the surround" (`stage==1`/`stage==0`
  respectively) where no parens are ever needed because the surrounding bracket/paren already delimits
  the sub-expression (`printlanguage.cc:294-309`).
- `hiddenfunction`: a synthetic operator (`printc.hh:68`, `hidden = {"", "", ...}`) used by
  `opHiddenFunc` (`printc.cc:493`) to force correct grouping for pcode ops that print *nothing at all*
  (e.g. a suppressed extension, §3) but whose implicit precedence still needs representing so a
  neighboring associative operator doesn't silently merge with it (`printlanguage.cc:310-320`, and the
  doc-comment at `printc.cc:485-492`).

This generic algorithm is one of the two biggest pieces of *language-agnostic* machinery in the whole
subsystem (the RPN engine in §1.2 is the other) — `PrintJava` inherits it unmodified and gets correct
Java precedence purely by giving its overridden tokens (e.g. `instanceof`, `printjava.cc:21`) the right
numeric `precedence`.

### 2.2 Declaration printing (function pointers, arrays)

C's declarator syntax is famously non-linear (`int (*f)(char*, int[10])`), and `PrintC` handles it with
a two-phase push, symmetric with the RPN model above:

1. **`buildTypeStack`** (`printc.cc:143-164`) walks from the outer type inward — pointer → pointee,
   array → element, function → return type — pushing each layer onto a `vector` until it reaches a type
   with an actual name (the base type). This produces the layers **outside-in**.
2. **`pushTypeStart`** (`printc.cc:264-303`) pushes the base type name first, then walks the stack
   **back outward**, pushing `ptr_expr` (`*`) for a pointer layer, `array_expr` (`[`…`]`,
   `postsurround`) for an array layer, or `function_call` (`(`…`)`, `postsurround`) for a function-type
   layer — i.e. the declarator is built from the identifier outward, exactly as C requires (`int (*f)`
   needs the parens around `*f` precisely because `postsurround` binds `[]`/`()` tighter than the unary
   `*`, so the generic `parentheses()` logic from §2.1 inserts them automatically once `ptr_expr` and
   `function_call`/`array_expr` are pushed with their real precedences — `printc.cc:75-76`: both
   `ptr_expr` and `array_expr` reuse the same numeric precedences as the *expression* `*`/`[]`
   operators).
3. **`pushTypeEnd`** (`printc.cc:313-346`) pushes the trailing pieces — array bracket contents (the
   element count, `:327`) and function parameter lists (`pushPrototypeInputs`, `:334`) — walking the
   same layer chain a second time now that the identifier has already been placed.

Because parenthesization for these synthetic `ptr_expr`/`array_expr`/`function_call` tokens is decided
by the *same* generic `parentheses()` routine used for expressions, a pointer-to-array-of-int and an
array-of-pointer-to-int naturally come out with different, correct parenthesization
(`int (*p)[10]` vs `int *p[10]`) without any declaration-specific parenthesization code existing at all
— the declarator syntax literally reuses the expression-printing engine.

`PrintJava::pushTypeStart`/`pushTypeEnd` (`printjava.cc:76-119`) completely replace this logic rather
than reusing it, because Java's `int[] x` array syntax doesn't nest the way C's does — see §4.

### 2.3 A few other notable PrintC behaviors

- `checkArrayDeref` (`printc.cc:353-369`) decides, for a `LOAD`/`STORE` off a computed address, whether
  to print `a[i]`/`s.field` member syntax instead of `*(...)` pointer-dereference syntax — it looks for
  an implied `PTRSUB`/`PTRADD` feeding the load/store address.
- `checkAddressOfCast` (`printc.cc:395-439`) recognizes a `CAST` from `T (*)[N]` to `T*` where the
  input is a symbol of array type, and rewrites it as `&arr` instead of `(int*)&arr[0]`-style cast
  syntax — a readability special case layered on top of the generic cast-printing path (§3).

---

## 3. `CastStrategy`: when does a cast get printed?

`cast.hh:42` documents `CastStrategy`'s job as four kinds of decisions: whether an assignment needs a
cast, whether a conversion op needs to *look like* a cast, whether an extension/comparison matches
expected integer promotion, and what type an arithmetic op produces. It is important to separate two
distinct places this policy gets consulted, because they answer different questions:

### 3.1 Producer side — inserting `CPUI_CAST` ops (Action/Rule engine, `coreaction.cc`)

Before printing ever begins, `ActionSetCasts` (`coreaction.hh:321`, an ordinary `Action` in the
simplification engine — see `action-rule-engine.md`) walks every p-code op in dominance order
(`ActionSetCasts::apply`, `coreaction.cc:2863-2923`) and, for each input and output, calls
`CastStrategy::castStandard(reqtype, curtype, care_uint_int, care_ptr_uint)`
(`cast.cc:300-392`). A **non-null return means "insert an explicit `CPUI_CAST` op here"**:

- **Rule 1 — output cast** (`ActionSetCasts::castOutput`, `coreaction.cc:2675-2755`): if the type an
  operator's `TypeOp` says it naturally produces (`getOutputToken`) differs from what the consuming
  context expects, and `castStrategy->castStandard(outct, tokenct, false, true)` returns non-null
  (`coreaction.cc:2727`), a new `CPUI_CAST` op is spliced in right after the producing op
  (`coreaction.cc:2734-2748`).
- **Rule 2 — input cast** (`ActionSetCasts::castInput`, `coreaction.cc:2794` onward, called from
  `apply` at `:2909`): symmetric — if an operand's current type doesn't match what the consuming
  operator's `TypeOp` requires for that slot, a `CPUI_CAST` is spliced in before the op.
- **Rule 3 — pointer-vs-integer signedness is never silently ignored**: `castStandard` itself
  (`cast.cc:300`) is explicit about this — coming from a `TYPE_VOID` pointer always forces a cast
  (`:305-306`), a change in pointed-to size, address space, or word size always forces a cast
  (`:313-317`), and a `TYPE_UINT`/`TYPE_INT` mismatch forces a cast whenever `care_uint_int` is true
  (`:344-361`, `:362-377`) — every one of the >15 call sites in `typeop.cc` picks `care_uint_int` per
  operator (e.g. comparisons pass `true`, generic arithmetic often passes `false`), which is how the
  same policy object produces different cast decisions for e.g. `a == b` vs `a + b` with the same
  operand types.

`castOutput` additionally special-cases pointer-to-struct/array decay: `testStructOffset0`
(`coreaction.cc:2493-2522`) checks whether the required cast is really just "take the address of field
0" and, if so, emits a `CPUI_PTRSUB` (println's `->`/`.` syntax) instead of a `CPUI_CAST`
(`coreaction.cc:2723-2725`) — a cast-avoidance rule that trades a visible cast for implicit
member-access syntax when the two are semantically equivalent.

### 3.2 Consumer side — deciding how an *existing* op prints (`printc.cc`)

Separately, three p-code ops that are conceptually conversions but are not `CPUI_CAST` — `INT_ZEXT`,
`INT_SEXT`, and a size-truncating `SUBPIECE` — ask `CastStrategy` whether *they themselves* should be
rendered using cast syntax `(T)x` rather than as a function-like op. This is where "implicit vs.
explicit" is actually decided at print time:

- **Rule 4 — zero-extension as cast**: `PrintC::opIntZext` (`printc.cc:805-816`) calls
  `castStrategy->isZextCast(outType, inType)` (`cast.cc:457-469`: true only when the output is
  int/uint-metatype *and* the input is unsigned/bool-metatype — signed input printed via `zext()` "as a
  function" instead, since sign-extension can't be represented by C's implicit unsigned-widening rule).
  If it *is* a valid cast **and** `option_hide_exts` is set **and**
  `castStrategy->isExtensionCastImplied(op, readOp)` (`cast.cc:249-298`) says the surrounding
  expression's own integer promotion would already perform this exact extension, the cast is dropped
  entirely (`opHiddenFunc`, printing nothing but preserving precedence via the `hidden` token from
  §2.1) — this is the single biggest source of "the decompiler silently promoted my variable, is that
  cast really needed" readability judgment calls the ticket flagged.
- **Rule 5 — sign-extension as cast**: `PrintC::opIntSext` (`printc.cc:818-829`) is the mirror image
  using `isSextCast` (`cast.cc:443-455`, requires *signed* input) and the same `isExtensionCastImplied`
  hiding check.
- **Rule 6 — truncation as cast**: `PrintC::opSubpiece` (`printc.cc:862-897`) first tries to print a
  `SUBPIECE` as a structure-field extraction (`.field`, `:881-887`); only if that fails does it ask
  `castStrategy->isSubpieceCast(outType, inType, offset)` (`cast.cc:411-432`, offset must be 0, and both
  types must be scalar-ish) to decide between cast syntax (`opTypeCast`) and functional `SUB4(x, 0)`
  syntax (`opFunc`) (`printc.cc:891-896`).

`markExplicitUnsigned`/`markExplicitLongSize` (`cast.cc:38-105`, base-class, non-virtual, shared by both
strategies) are a related but distinct decision: not whether to cast, but whether an integer *constant
literal* needs a `u`/`L` suffix so that it independently parses back to the right type without relying
on cast/promotion context at all.

### 3.3 `CastStrategyC` vs `CastStrategyJava`

`CastStrategyJava` (`cast.hh:198`, `cast.cc:471-544`) is a small subclass of `CastStrategyC`, overriding
only `castStandard` and `isZextCast`. Its `castStandard` (`cast.cc:471`) is deliberately more
conservative about pointers: any pointer-involving cast returns "no cast needed" (`:478-479`, "there
must be an explicit cast op between objects" — object casts in Java bytecode are already materialized as
their own p-code ops upstream, so the printer doesn't need to synthesize one), and it drops the
address-space/pointer-size comparisons `CastStrategyC` does (irrelevant to the JVM's uniform reference
model). `isZextCast` (`cast.cc:533-544`) also encodes JVM-specific promotion sizes (byte/short/int/long)
rather than C's `int`-promotion rule.

---

## 4. `PrintC` vs `PrintJava`: what's actually shared

The ticket asks whether multi-language output is "well-factored" today. The concrete answer: **`PrintJava`
does not implement the `PrintLanguage` interface independently — it is a `PrintC` subclass**
(`printjava.hh:57`, `class PrintJava : public PrintC`), not a sibling of `PrintC` under `PrintLanguage`.
`printjava.cc` is 365 lines total against `printc.cc`'s ~3500; it overrides 9 methods.

**Shared (inherited from `PrintC` unchanged):** every arithmetic/logical/comparison `op*` method (`opIntAdd`,
`opBoolAnd`, all ~40 inline one-liners in `printc.hh:286-348`), the entire RPN/precedence engine
(inherited from `PrintLanguage`, §1.2/§2.1), all statement and control-flow emission (`emitStatement`,
`emitBlockIf`, `emitBlockWhileDo`, `emitForLoop`, switch/case handling), all comment handling
(`emitLineComment` from `PrintLanguage`, `emitCommentGroup`/`emitCommentBlockTree` from `PrintC`),
variable declaration statement structure (`emitVarDeclStatement`, `emitLocalVarDecls`), function
prototype emission except parameter-list punctuation, constant printing (`push_integer`, `push_float`,
`pushCharConstant`, `pushEnumConstant`), and bitfield printing (§ below). This is a large majority of
`PrintC`'s surface area.

**Overridden by `PrintJava` (`printjava.hh:57-75`):**

| Method | Why it differs |
|---|---|
| `pushTypeStart`/`pushTypeEnd` (`:76-119`) | Java's `int[] x` doesn't nest pointer/array declarators the way C's `int (*x)[N]` does — counts wrapping array layers (`isArrayType`, `:137-157`) and prints them all after the identifier, no function-pointer declarator support needed |
| `adjustTypeOperators` (`:121-127`) | Java uses `.` for scope, `>>>` for the extra unsigned-right-shift operator C doesn't have |
| `opLoad`/`opStore` (`:228-257`) | Java array-object dereference needs synthetic `[0]` syntax (`needZeroArray`, `:171-182`) because the decompiler models a scalar array reference as a pointer, unlike C where the pointer *is* the value |
| `opCallind` (`:259-292`) | Java indirect calls route through a different hidden-`this`-slot convention |
| `opCpoolRefOp` (`:294-363`) | Wholly Java-specific: resolves JVM constant-pool references (string literals, class refs, `instanceof`, field/method refs) — has no C equivalent at all |
| `printUnicode` (`:184-226`) | Java-specific escape set (`\b`, `\ux....` form) |
| `docFunction` (`:57-69`) | Pushes an implicit enclosing-class scope before delegating to `PrintC::docFunction` |
| constructor (`:39-48`) | Swaps in `CastStrategyJava` (§3.3) and Java's lower-case `null` token |

**What this means for "is multi-language output well-factored":** the abstract `PrintLanguage` base
class is a real, working language-agnostic core (the RPN stack, the generic `parentheses()` algorithm,
the `Emit`/`TokenSplit` pretty-printing engine are all used as-is by both languages, and were plainly
designed to be language-independent). But in practice there is exactly **one** other language
back-end, and it was built by inheriting the ~3500-line C-specific class wholesale and patching ~9
methods, rather than by building an independent implementation against the `PrintLanguage` contract.
`PrintLanguageCapability`/`PrintLanguage`'s pure-virtual interface (`printlanguage.hh:138-609`, over 70
abstract `op*` methods alone) *looks* like it supports arbitrary sibling languages, but the only
existing evidence for that claim is `PrintJava`, and `PrintJava` sidesteps almost all of that interface
by inheriting `PrintC`'s implementations instead of providing its own. A genuinely different target
language (e.g. one without C-style declarator syntax or without an `if`/`while`/`switch`-shaped
structured-control-flow model) would have much less to inherit and would put real pressure on how much
of `PrintLanguage` vs. `PrintC` is actually reusable — this is a concrete data point worth carrying into
the flaw-analysis phase (`02-flaws/printer-output-quality.md`, task 0014).

---

## 5. `prettyprint.hh/.cc`: the line-wrap engine

`EmitPrettyPrint` implements a variant of the classic Derek C. Oppen pretty-printing algorithm (doc
comment at `prettyprint.hh:1055-1067`), adapted to also carry markup through to an inner `Emit`.

- **`TokenSplit`** (`prettyprint.hh:623-962`) is the buffered unit: either real content
  (`tokenstring`) or a structural marker (`begin`/`end` for a printing group, `begin_indent`/`end_indent`
  for a nesting level, `tokenbreak` for a place a line break is *allowed*). Every `PrintLanguage`
  `emit->tagOp(...)`/`openGroup()`/etc. call becomes one `TokenSplit` pushed onto `tokqueue`, a
  `circularqueue<TokenSplit>` (`prettyprint.hh:1082`).
- Groups are **not** immediately known to fit on a line. An opening group token is buffered with a
  negative placeholder size; only once its matching close token is scanned (or a forced overflow occurs)
  does the group's total size become known and get "committed" — at which point `advanceleft()`
  (`prettyprint.cc:735`) flushes the now-fully-sized run of tokens down to the `lowlevel` emitter. This
  is what lets the algorithm decide, with full knowledge of an entire parenthesized sub-expression's
  final width, whether it fits on the current line before it has actually started emitting characters
  for it.
- **`scan()`** (`prettyprint.cc:766`) is the dispatch loop that processes each new token and decides
  whether pending groups can be committed yet; `checkstart`/`checkstring`/`checkend`/`checkbreak`
  (`:831-891`) are its per-token-class helpers.
- **`overflow()`** (`prettyprint.cc:609-634`) is the escape hatch when a single token or group simply
  cannot fit even after breaking — it forcibly redistributes indent levels to guarantee at least half
  the line width, rather than looping forever trying to fit an oversized token.
- `EmitPrettyPrint::print(const TokenSplit&)` (`prettyprint.cc:639-726`) is the actual space-accounting
  state machine: it tracks `spaceremain` against `maxlinesize` (default 100,
  `resetDefaultsPrettyPrint`, `prettyprint.hh:1092`) and decides, per `tokenbreak`, whether the pending
  whitespace becomes a plain space or an actual `tagLine()` + reindent.

This whole engine is unaware of C or Java syntax — it operates purely on the abstract `begin`/`end`/
`tokenbreak` shape of the stream `PrintLanguage` handed it, which is the concrete mechanism by which
"what to say" (§1.1) stays decoupled from "how to lay it out".

---

## 6. `expression.hh/.cc`: mostly out of scope

`expression.hh` (238 lines) is **not** primarily a printer-support file — despite living next to
`printc.cc`/`cast.cc` and despite the ticket's working title guess ("`PartialCastPropagation`?"), most of
its content (`BooleanExpressionMatch`, `TermOrder`, `AddExpression`, `functionalEquality`,
`pointerEquality`) is expression-*matching* infrastructure consumed by the **Action/Rule simplification
engine** — confirmed by its actual call sites: `condexe.cc`, `ruleaction.cc`, `blockaction.cc`,
`coreaction.cc`, `funcdata_op.cc` (grep across `decompile/cpp/*.cc`). That material belongs with
`action-rule-engine.md` (task 0006), not here.

The one genuine exception: `BitFieldExpression` and its two subclasses `InsertExpression`
(`expression.hh:198`) and `PullExpression` (`expression.hh:222`), plus `InsertStoreExpression`
(`expression.hh:211`), *are* printer-consumed — `PrintC::emitBitFieldExpression`
(`printc.cc:2610-2613`), `PrintC::emitBitFieldStore` (`printc.cc:2586`), and the `checkBitFieldMember`
helper (`printc.cc:1292`, `:1318`) construct one of these to recover the parent structure/field
description encoded across a `ZPULL`/`SPULL`/`INSERT` op sequence before deciding whether to print
`.` or `->` member syntax for a bitfield access. So `expression.hh/.cc` is correctly in this ticket's
file list, but only for this one narrow purpose — the bulk of the file is a false positive that belongs
to the Action/Rule review instead.

## `grammar.hh/.cc/.y`: out of scope, but not for the reason the ticket guessed

`grammar.y`/`grammar.cc`/`grammar.hh` is a Bison grammar (`%define api.prefix {grammar}`,
`grammar.y:15-24`) implementing a **C declaration parser** ("Grammar taken from ISO/IEC 9899",
`grammar.y:39`) — it parses textual C type/function declarations *into* `Datatype`/`TypeDeclarator`
objects. Its only call site is `parse_C()` from `ifacedecomp.cc:366,396`, part of the interactive
console interface's `parse C_code` command, used to let a user type a C declaration to override a
function signature or define a type. It is genuinely unrelated to SLEIGH (the ticket's own guess) — but
it is *also* unrelated to this ticket's subject, because it runs in the opposite direction from
everything else described in this document: `PrintC` turns internal structures **into** C text, while
`grammar.cc` turns C text **into** internal structures. It is correctly out of scope here; if it belongs
anywhere in the review set it would be alongside the console/interface tooling, not the printer.

---

## Summary for the flaw-analysis phase

Concrete data points this review surfaces for `02-flaws/printer-output-quality.md` (task 0014):

1. Cast insertion is split across two layers that don't share a single decision point: the Action-engine
   pass `ActionSetCasts` (§3.1, in `coreaction.cc`, conceptually part of the Rule engine reviewed in
   0006) inserts real `CPUI_CAST`/`PTRSUB` ops into the graph, while `PrintC`'s `op*` methods (§3.2)
   separately decide whether *already-implicit* conversion ops (`ZEXT`/`SEXT`/`SUBPIECE`) should *look
   like* a cast at print time. Both paths funnel through the same `CastStrategy` object but via
   different entry points (`castStandard` vs. `isZextCast`/`isSextCast`/`isSubpieceCast`), which is a
   plausible source of the "why is there a cast here but not there" inconsistency reports the ticket
   anticipated.
2. `option_hide_exts` + `isExtensionCastImplied` (§3.2, Rule 4/5) is a single boolean option gating
   whether a genuinely-happening integer promotion is shown to the user at all — worth flagging as a
   correctness-vs-readability tradeoff point explicitly, since hiding a real extension can make an
   expression's actual computed width non-obvious from the printed text.
3. The "multi-language" story (§4) is effectively "one language, plus one 90%-inherited variant" — any
   requirements work around genuinely new target languages (§03-requirements) should budget for building
   much more than 9 methods' worth of new code, since so much of what looks like `PrintLanguage`'s
   generic contract is actually only exercised by `PrintC` today.

## Related documents

- [`00-index.md`](00-index.md) — pipeline narrative this document is one leg of
- [`java-integration-testing.md`](java-integration-testing.md) — `ClangToken`/`DecompInterface`/GUI consumption detail (task 0010)
- [`action-rule-engine.md`](action-rule-engine.md) — where `ActionSetCasts` and most of `expression.cc` actually live (task 0006)
- [`type-system.md`](type-system.md) — `Datatype`/`TypeOp::getInputCast`/`getOutputToken`, the type-level machinery `CastStrategy` consults (task 0005)
- [`../00-overview/glossary.md`](../00-overview/glossary.md) — shared terminology
