# Flaws: Expression/Cast Logic & Printer Output Quality

Produced by [`.tasks/0014`](../../.tasks/0014-task-flaws-printer-output-quality.md), part of the
flaw-analysis epic [`.tasks/0011`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md), under root
[`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md). Builds on
[`expression-cast-printer.md`](../01-review/expression-cast-printer.md) (review ticket 0009) and
cross-references [`varnode-ssa-type-system.md`](varnode-ssa-type-system.md) §3.1 (flaw ticket 0012) for
the type-system half of the templates/generics question. Shared terms:
[`../00-overview/glossary.md`](../00-overview/glossary.md).

Each flaw follows the structure required by ticket 0014: **Observation → Impact → Severity → Improvement
options (2-4, pros/cons) → Open questions.** All line numbers are relative to
`Ghidra/Features/Decompiler/src/decompile/cpp/` and were re-verified against source in this session
(exact `grep`/`sed` excerpts, not carried over unchecked from the review doc).

---

## Flaw 1 — Cast display is decided by two policy entry points that never compare notes

### Observation

`CastStrategy` is consulted from two structurally separate places that answer different questions with
no shared decision record between them (`expression-cast-printer.md` §3, verified):

- **Producer side**, before printing starts: `ActionSetCasts::apply` (`coreaction.cc:2863-2923`, an
  ordinary `Action` in the simplification pipeline, not part of the printer) calls
  `CastStrategy::castStandard(reqtype, curtype, care_uint_int, care_ptr_uint)` (`cast.cc:300-392`) for
  every op input/output and splices a real `CPUI_CAST` op into the graph whenever it returns non-null.
- **Consumer side**, at print time: `PrintC::opIntZext`/`opIntSext`/`opSubpiece`
  (`printc.cc:805-816`, `:818-829`, `:862-897`) separately ask `isZextCast`/`isSextCast`/`isSubpieceCast`
  (`cast.cc:457-469`, `:443-455`, `:411-432`) whether an *already-existing*, non-`CAST` conversion op
  (`INT_ZEXT`/`INT_SEXT`/`SUBPIECE`) should be *rendered* using cast syntax `(T)x` instead of function
  syntax like `zext(x)`.

Layered on top of the consumer side, `option_hide_exts` gates `isExtensionCastImplied`
(`cast.cc:249-298`), which decides whether a real, semantically-happening extension is shown at all.
That function is not a general analysis of C's integer-promotion rules; it is a hand-enumerated `switch`
over eleven specific opcodes (`CPUI_PTRADD`, the `INT_*` arithmetic/comparison family,
`cast.cc:262-278`) plus a `default: return false` for anything else (`cast.cc:290`). Both entry points
funnel through one `CastStrategy` object, but neither one calls the other, and nothing checks that a
value cast on the producer side would also be judged cast-worthy by the consumer side's narrower
opcode whitelist, or vice versa.

### Impact

This is the concrete mechanism behind the "why is there a cast here but not there" class of readability
complaint the parent review flagged as plausible (`expression-cast-printer.md` §3.1 numbered summary,
item 1). Two consequences follow directly from the source:

1. **A real correctness-adjacent readability gap, not just cosmetics.** When `option_hide_exts` is set
   and `isExtensionCastImplied` returns true, the extension is printed as nothing at all
   (`PrintC::opHiddenFunc`, `printc.cc:485-492` doc comment) — the expression's *actual* computed width
   is no longer visible from the printed text, even though the extension genuinely happened in the
   p-code graph. `expression-cast-printer.md` §3.2 already names this the single biggest source of "the
   decompiler silently promoted my variable" confusion.
2. **The whitelist is closed, not general.** Because `isExtensionCastImplied`'s `switch` only recognizes
   eleven specific consuming opcodes, any future `Rule` or `TypeOp` that legitimately relies on integer
   promotion through some other opcode gets no hiding at all (a false negative — an extra cast appears
   that a human wouldn't write) while the reverse (a false positive — a cast is hidden where promotion
   doesn't actually apply) cannot be ruled out without auditing all eleven cases against every possible
   operand-type combination, which the source gives no evidence was done exhaustively.

### Severity

**Medium.** No demonstrated case of the two entry points disagreeing on the same op was found — this is
a structural risk (matching the recurring "convention rather than enforcement" pattern
`varnode-ssa-type-system.md` §1.7 identifies elsewhere in the codebase), not a proven bug. But it sits
squarely on the "correctness vs. readability" fault line the ticket asks about, and the closed opcode
whitelist is a concrete, checkable limitation today, not a hypothetical one.

### Improvement options

1. **Give `CastStrategy` one unified decision method with an explicit "role" parameter**
   (`castForInsertion` vs. `castForDisplay`) that both `ActionSetCasts` and `PrintC`'s `op*` methods call,
   instead of `castStandard` and `isZextCast`/`isSextCast`/`isSubpieceCast` being independent virtuals.
   - *Pro*: makes the two-question split ticket 0009 documented an explicit, single-class contract
     instead of an implicit one two call sites happen to both respect; a future `CastStrategy` subclass
     (a new target language) is forced to reason about both roles together.
   - *Con*: `castStandard` operates on `Datatype*` pairs pre-insertion while `isZextCast`/`isSextCast`
     operate on an existing `PcodeOp`'s already-fixed opcode — unifying the signature risks papering over
     a genuine difference in what information is available at each call site rather than actually
     merging the logic. *Size*: Medium.
2. **Replace `isExtensionCastImplied`'s opcode whitelist with a query against `TypeOp`'s own
   integer-promotion metadata**, if `TypeOp` (`typeop.hh`, reviewed in `type-system.md`) can express "this
   opcode performs standard promotion on this operand" generically, rather than re-encoding the same
   eleven-opcode judgment call independently in `cast.cc`.
   - *Pro*: removes a second, hand-maintained copy of "which opcodes promote" if `TypeOp` already has (or
     could cheaply gain) an equivalent notion; a closed switch becomes an open, per-opcode query.
   - *Con*: unverified whether `TypeOp` already carries this information in a form reusable here — this
     ticket did not audit `typeop.cc` for it, so the option's actual cost is unknown until that audit
     happens. *Size*: Medium (contingent on an unresolved open question, below).
3. **Add a debug-mode consistency check**: for every printed function, re-run `castStandard` against the
   producer-side decision for every `ZEXT`/`SEXT`/`SUBPIECE` op the consumer side chose to hide or show as
   a cast, and assert the two entry points agree on "does a real conversion exist here."
   - *Pro*: small, matches the cheap-mitigation pattern already recommended elsewhere in this document set
     (`varnode-ssa-type-system.md` §1.2/§1.3/§1.7 option B/C); catches drift without redesigning
     `CastStrategy`. *Con*: does not reduce the underlying duplication, only detects divergence after the
     fact. *Size*: Small.

### Open questions

- Does `TypeOp` (per `type-system.md`) already carry a reusable "this opcode performs standard integer
  promotion" fact that `isExtensionCastImplied`'s eleven-opcode `switch` could delegate to instead of
  re-deriving independently? This determines option 2's real cost.
- Is `option_hide_exts` ever set differently per output consumer (e.g. GUI display vs. a diffable text
  export), and if so, does hiding a real extension only in one of those contexts make the
  correctness-vs-readability tension asymmetric across consumers in a way that matters for tooling built
  on top of the printed text (test fixtures, diffing)?

---

## Flaw 2 — `PrintC` documents, in its own source, that statement printing bypasses the abstraction expression printing relies on

### Observation

`expression-cast-printer.md` §1.1 establishes that `PrintLanguage`'s design is a deliberate two-layer
split: `PrintLanguage` decides *what* to say via the RPN `pushOp`/`pushAtom` engine (§1.2), and hands
fully-decided tokens to an `Emit` object that decides *how* to lay them out. Four control-flow-statement
methods in `printc.cc` carry `FIXME` comments admitting they violate exactly this split by calling
`emit->` methods directly instead of going through the token-stacking engine:

- `PrintC::opCbranch` (`printc.cc:558`): `// FIXME:  This routine shouldn't emit directly`
- `PrintC::opBranchind` (`printc.cc:604`): the identical comment
- `PrintC::opReturn` (`printc.cc:779`): the identical comment
- `PrintC::emitBlockCondition` (`printc.cc:2965`): `// FIXME: get rid of parens and properly emit && and
  ||` — this one is more specific: it names the exact consequence (boolean-condition parenthesization
  and operator emission for `&&`/`||` block conditions is handled by ad hoc paren-tracking in this method
  rather than by the generic `parentheses()` algorithm `expression-cast-printer.md` §2.1 describes as one
  of "the two biggest pieces of language-agnostic machinery in the whole subsystem").

All four are original-author comments (not something this review is inferring) — the codebase itself
records that these are known deviations from its own design, not accepted final behavior.

### Impact

Because `opCbranch`/`opBranchind`/`opReturn`/`emitBlockCondition` write tokens straight to `emit` instead
of pushing `OpToken`s through `pushOp`, the statement-level constructs they print do not get the benefit
of the generic `parentheses()` precedence logic (`printlanguage.cc:270-324`) the same way ordinary
expression operators do — `emitBlockCondition`'s own comment says as much for `&&`/`||` block conditions
specifically. Concretely:

1. **A second, informal parenthesization mechanism exists alongside the generic one.** Anyone extending
   or retargeting `PrintLanguage` (the exact scenario ticket 0014 was asked to assess for a new target
   language) has to special-case these four methods rather than trusting that "override the `OpToken`
   table and precedence numbers" (the pattern `PrintJava` otherwise follows almost everywhere else, per
   `expression-cast-printer.md` §4) is sufficient — because these four statement forms don't go through
   the `OpToken`-driven path at all.
2. **This is precisely the failure mode the ticket's own Analysis Focus item 5 anticipates as an
   improvement direction** ("introduce a proper visitor/AST intermediate between structured blocks and
   text instead of direct emission") — the codebase already has four documented instances of exactly the
   problem that improvement direction would fix, which is strong evidence the direction is well-targeted
   rather than speculative.

### Severity

**Low-Medium.** No incorrect output was found to result from this today (the shipped C output for
`if`/`goto`/`switch`/`return`/`&&`/`||` is presumably correct and well-tested via `datatests`'s regex
assertions) — but per `00-index.md`'s test-coverage table, "no golden/exact-text coverage anywhere" exists
for the printer, so a regression in exactly these four hand-written paths would be one of the least
likely categories of printer bug to be caught by existing tests. The severity is chiefly
**architectural**: these are the four places most likely to need rewriting, not merely subclassing, for
any language whose statement-level syntax differs from C's (e.g. no `goto`, or short-circuit boolean
operators with different associativity/precedence than C's).

### Improvement options

1. **Route these four methods through `pushOp`/the RPN engine like everything else**, introducing
   whatever new `OpToken`s (a `goto_stmt`, an `if_cond` wrapper, etc.) are needed so the generic
   `parentheses()` logic — already proven to handle synthetic declarator tokens like `ptr_expr`/
   `array_expr` correctly (`expression-cast-printer.md` §2.2) — also owns `&&`/`||` block-condition
   grouping.
   - *Pro*: closes the exact gap `emitBlockCondition`'s own FIXME names; makes the four statement forms
     subject to the same "override the token table" extension story as the rest of `PrintC`.
   - *Con*: these four are statement-level constructs interacting with `Emit`'s block/indent structure
     (`tagLine`, `openParen`/`closeParen` at statement scope), not pure operand expressions — folding them
     into the operand-stacking RPN engine may not be a clean fit without also extending what the RPN
     engine itself models; nontrivial, so the risk of subtly changing today's correct output is real.
     *Size*: Medium.
2. **Introduce the ticket-suggested visitor/AST intermediate**: instead of `PrintLanguage` walking the
   `Funcdata` block/op graph and calling `emit->`/`pushOp` inline, first build a small structured
   intermediate tree (one node per statement/expression) and print that tree in a second pass. This is
   the largest of the options and is what Analysis Focus item 5 names directly.
   - *Pro*: would eliminate this entire class of "direct emission" escape hatch by construction, not just
     the four known instances — any future statement form gets the same guarantee.
   - *Con*: this is a genuine architectural rewrite of the printer's core traversal strategy, not a
     localized fix; touches the RPN engine (§1.2), the `Emit`/`EmitPrettyPrint` line-wrap engine (§5), and
     every `op*` override in both `PrintC` and `PrintJava`. *Size*: Large.
3. **Leave the four call sites as-is, but promote the `FIXME`s to a tracked, named limitation** (e.g. a
   short doc comment cross-referencing this document) so a future retargeting effort budgets for them
   explicitly instead of discovering them by trial and error.
   - *Pro*: zero behavior risk, cheapest possible action, and makes the existing self-documentation
     (already present as `FIXME`s) discoverable by name rather than only by `grep`.
   - *Con*: fixes nothing; a genuinely new target language still has to do the rewrite in option 1 or 2
     eventually. *Size*: Small (documentation only).

### Open questions

- Do these four `FIXME`s predate `PrintJava`'s introduction, and if so, did `PrintJava` inherit the same
  direct-emission behavior silently (since it does not override `opCbranch`/`opBranchind`/`opReturn`/
  `emitBlockCondition` per the override table in `expression-cast-printer.md` §4) — i.e. is Java's
  `if`/`goto`/`return`/`&&`/`||` output also subject to whatever informal parenthesization gap exists
  here, or does C's happen to be close enough to Java's that it doesn't matter in practice?
- Is there a reason (performance, historical) these four specifically were left as direct-emission rather
  than an oversight — the comments record *that* they're a problem but not *why* the fix was deferred.

---

## Flaw 3 — `PrintLanguage`'s "genuinely reusable abstraction" claim rests on exactly one data point, and that data point mostly declines to test it

### Observation

`expression-cast-printer.md` §4 (echoed in `00-index.md`'s "Known-surprising findings") establishes the
central fact directly: `PrintJava` is declared `class PrintJava : public PrintC` (`printjava.hh:57`), not
an independent `PrintLanguage` implementation, and overrides only 9 methods against `printc.cc`'s ~3500
lines. Re-checked against the review's own override table (§4): the 9 overrides are `pushTypeStart`/
`pushTypeEnd`, `adjustTypeOperators`, `opLoad`/`opStore`, `opCallind`, `opCpoolRefOp`, `printUnicode`,
`docFunction`, plus the constructor. Every one of `PrintC`'s ~40 arithmetic/logical/comparison `op*`
one-liners, its entire statement/control-flow emission (`emitBlockIf`, `emitBlockWhileDo`,
`emitForLoop`, switch/case handling — including the very direct-emission methods Flaw 2 above discusses),
all comment handling, variable-declaration-statement structure, and constant printing are inherited by
`PrintJava` completely unchanged.

`PrintLanguage`'s own interface (`printlanguage.hh:138-609`) is large — over 70 pure-virtual `op*`
methods alone — and was, per the review, "plainly designed to be language-independent" (§4). But the only
concrete evidence that design goal is *achieved*, rather than merely intended, is `PrintJava`, and
`PrintJava` exercises almost none of that 70-method surface independently: it satisfies the interface by
inheriting `PrintC`'s answers, not by providing its own.

### Impact

This matters directly for the ticket's own framing: "the printer is where language-specific syntax
ultimately has to land" for any "rich language features" requirement. Two concrete consequences:

1. **The abstraction's reusability is unproven for the cases that would actually stress it.** Java is
   close enough to C (C-family statement syntax, similar operator set, no fundamentally different
   declarator model except arrays) that inheriting 3500 lines of `PrintC` and patching 9 methods was
   sufficient. A genuinely different target — a language without `if`/`while`/`switch`-shaped structured
   control flow, without C-style operator precedence, or with a declarator syntax that doesn't compose the
   way pointer/array/function nesting does — has no existing evidence for how much of `PrintC` (not just
   `PrintLanguage`) it could actually reuse. The review's own words: "would have much less to inherit and
   would put real pressure on how much of `PrintLanguage` vs. `PrintC` is actually reusable" (§4).
2. **The two genuinely language-agnostic pieces are real, but they are not the hard part of adding a new
   language.** The RPN stack (§1.2) and the generic `parentheses()` algorithm (§2.1) are used as-is by
   both languages and are legitimately reusable machinery. But per Flaw 2 above, even *within* the current
   two-language set, four statement-printing methods already escape that generic machinery via direct
   `emit->` calls — so the parts of the abstraction most likely to need new work for a third language
   (statement-level control flow, declarator syntax for non-C-shaped type systems) are exactly the parts
   with the least existing multi-language evidence behind them.

### Severity

**Medium.** Not a bug — `PrintC`/`PrintJava` both work correctly for their existing consumers — but a
direct, concrete risk to planning: if a refactor's "rich language features" requirement (ticket 0017)
assumes `PrintLanguage` is a proven, cheap extension point because the class hierarchy has two
implementations, that assumption is not supported by what those two implementations actually share.

### Improvement options

1. **Treat a third, deliberately-dissimilar target language as a validation exercise before committing to
   any "add language N" requirement.** Rather than assuming `PrintLanguage`'s 70-method contract is
   sufficient, prototype (even minimally, without full correctness) a language that violates one of C's
   assumptions on purpose (e.g. no implicit pointer arithmetic, or expression-oriented control flow) and
   measure how much of `PrintC` is actually reusable versus how much has to be replaced.
   - *Pro*: converts an untested architectural claim into measured evidence before the refactor commits
     resources to "just subclass `PrintLanguage`" as its extensibility story.
   - *Con*: real, non-trivial exploratory effort with no guaranteed reusable output — this is investigation
     work, not a shippable improvement, and may not fit neatly into a staged refactor plan's deliverables.
     *Size*: Medium (spike/investigation).
2. **Explicitly split `PrintLanguage`'s interface into a genuinely-generic core (RPN engine,
   `parentheses()`, `Emit` dispatch) and a C-family-specific layer (currently folded into `PrintC` but
   conceptually closer to the abstract contract than `PrintC`'s truly C-only parts, e.g. structured
   `if`/`while`/`switch` emission) that a non-C-family language could opt out of independently.**
   - *Pro*: makes the "what's actually generic" boundary explicit in the type hierarchy instead of only in
     a review document; directly addresses the finding that `PrintJava` reuses C-family control-flow
     machinery by inheritance rather than by genuine interface satisfaction.
   - *Con*: a real hierarchy change to a ~3500-line class with no second consumer yet to validate the new
     boundary against — risks guessing the wrong split without option 1's evidence first. *Size*: Large.
3. **Leave the hierarchy as-is and document the actual reuse boundary precisely** (which of
   `PrintLanguage`'s 70 abstract methods are, in practice, only ever implemented by `PrintC`, versus which
   are genuinely exercised independently) so future language work starts from an accurate map instead of
   the hierarchy's optimistic appearance.
   - *Pro*: cheap, immediately useful, and this document's Observation section is most of the raw
     material such a map would need. *Con*: doesn't reduce the risk itself, only documents it. *Size*:
     Small.

### Open questions

- Is a third output language (beyond C and Java) actually in scope for this refactor's requirements
  phase, or is "rich language features" more about C-family syntax richness (templates, better qualifier
  fidelity — Flaw 5 below) than about genuinely new target languages? The answer changes whether option 1
  above is worth its cost.
- `PrintJava`'s `opCpoolRefOp` override (`expression-cast-printer.md` §4 table) is described as "wholly
  Java-specific... has no C equivalent at all" — is there a broader pattern here (a hook for
  target-specific constant/reference resolution with no generic default) that a new language would need
  its own analogous override for, and if so, is that hook itself part of `PrintLanguage`'s abstract
  contract or bolted onto `PrintC` the same way the declarator methods are?

---

## Flaw 4 — Declaration printing silently drops real qualifier information: `const` cannot be represented at all, and `volatile` is tracked but never printed

### Observation

Two independent, verifiable facts, checked directly against source in this session:

1. **`const` has no representation anywhere in the type system the printer draws from.**
   `varnode-ssa-type-system.md` §3.2 (ticket 0012) already establishes that `Datatype`'s full boolean flag
   set (`type.hh:180-198`) has no `const` bit at any granularity. This review confirms the consequence on
   the printer side: `printc.cc`/`printc.hh` contain **zero** occurrences of the string `"const"` as
   printable output (verified by direct search across both files). There is no code path by which `const`
   could ever appear in emitted C, because the printer has no signal to check even if it wanted to.
2. **`volatile` is tracked, but the printer only ever uses it to pick a syntax-highlight color, never to
   emit the keyword.** The one and only use of `Symbol::isVolatile()` in the printer is
   `PrintC::pushSymbol` (`printc.cc:1971`): `if (sym->isVolatile()) tokenColor = EmitMarkup::special_color;`
   — a color selection for markup/highlighting, not a text decision. A direct search for the literal
   string `"volatile"` across `printc.cc`/`printc.hh` returns **zero** matches. A `Symbol` the decompiler
   itself knows is volatile (accessed via a hardware register, a memory-mapped I/O location, etc.) is
   printed with the exact same declaration text as a non-volatile one of the same type — the only visible
   difference is a GUI-only syntax-highlight color that a plain-text/`datatests`-style consumer of the
   printed output cannot observe at all.

By contrast, the parts of declaration printing ticket 0014 also asked about — function-pointer and
array-of-pointer declarator nesting (`buildTypeStack`/`pushTypeStart`/`pushTypeEnd`,
`expression-cast-printer.md` §2.2) — were already verified correct by the parent review via the generic
`parentheses()` reuse argument (§2.2); this review found no contradicting evidence and no open `TODO`/
`FIXME` in that code path. The concrete, demonstrable declaration-printing gap is qualifiers, not
pointer/array/function nesting.

### Impact

1. **`volatile` is a correctness-relevant, silently-dropped signal, not merely a style gap.** A reader of
   the printed C (or any tool consuming it, e.g. a downstream static analyzer fed the decompiler's text
   output) has no way to know from the text that a given access must not be reordered/cached/optimized
   away, even though the decompiler's own internal model (`Symbol::isVolatile()`) already knows it. This
   is a stronger finding than a typical "nice to have" printer gap — the information exists and is
   discarded specifically at the print boundary, not lost earlier in the pipeline.
2. **`const` is a pure representational ceiling, not a printer bug** — consistent with
   `varnode-ssa-type-system.md` §3.2's framing: since `Datatype` carries no `const` bit, this is not
   something `printc.cc` could fix in isolation even if it wanted to; the fix has to start upstream in the
   type system (§3.2's own improvement options A/B/C apply here unchanged — this document does not repeat
   them, see the cross-reference below).

### Severity

**Medium** for `volatile` (a real, demonstrable information-loss bug with a small, well-localized fix —
unlike most of this document's findings, this one is fixable in `printc.cc` alone without any type-system
change). **Low-Medium** for `const`, matching `varnode-ssa-type-system.md` §3.2's own severity judgment,
since it is a display/readability ceiling rather than an incorrect-vs-correct output difference (missing
`const` makes output less informative, not wrong).

### Improvement options

1. **Print `volatile` as a keyword wherever a volatile symbol's declaration is emitted**, mirroring how
   `const` would eventually be printed once available (option 2 of §3.2 in `varnode-ssa-type-system.md`,
   for symbol-level qualifiers) — the simplest form is emitting the keyword token in
   `emitVarDeclStatement`/wherever a `Symbol`'s formal declaration is built, guarded by the same
   `isVolatile()` check `pushSymbol` already performs for coloring.
   - *Pro*: small, isolated to `printc.cc`, requires no type-system change since the underlying
     `Symbol::isVolatile()` flag already exists and is already consulted — this is close to the smallest
     possible fix in this entire document. *Con*: `volatile` genuinely belongs on the *type* in C
     semantics (`const int * volatile p` vs. `volatile int *p` are different types, not different
     symbols) — printing it purely from the `Symbol` flag, as this option does, reproduces the same
     type-vs-storage conflation `varnode-ssa-type-system.md` §3.2 already flags as a modeling gap, just
     now made visible in output instead of silently absent. *Size*: Small.
2. **Adopt `varnode-ssa-type-system.md` §3.2's option A (add a `const`/`volatile` qualifier bitmask to
   `Datatype` itself) and have the printer's declarator layers (`pushTypeStart`/`pushTypeEnd`) print the
   qualifier at the correct declarator position** — this is the only option that gets pointer-vs-pointee
   qualifier placement right (`const int *` vs. `int *const`), which option 1 above cannot express.
   - *Pro*: closes both the `const` and the `volatile`-placement gap correctly and at the root, matching
     real C semantics exactly. *Con*: this is entirely a type-system change (§3.2's own "Large" sizing and
     interning-explosion caveat apply unchanged) — nothing in `printc.cc` can deliver this on its own; the
     printer-side work (new declarator-stack cases, symmetric with `ptr_expr`/`array_expr`) is small once
     the type-system data exists, but that precondition is not. *Size*: Large (dominated by the
     type-system half).
3. **Ship option 1 (symbol-level `volatile` printing) now as an independent, low-risk fix, and treat full
   `const`/qualifier-placement fidelity (option 2) as downstream of whichever `varnode-ssa-type-system.md`
   §3.2 option the type-system work adopts**, rather than blocking the cheap, real fix on the large one.
   - *Pro*: captures the clearly-worthwhile small win immediately without waiting on a type-system
     decision that is out of this ticket's scope. *Con*: means the codebase temporarily has `volatile`
     printed from `Symbol` while (eventually) `const` is printed from `Datatype` — two different qualifier
     sources for adjacent concepts, a smaller-scale version of the "two independently-coded stripped-form
     mechanisms" pattern `varnode-ssa-type-system.md` §1.4 already flags elsewhere in the type system.
     *Size*: Small now, with a known follow-up cost later.

### Open questions

- Given `varnode-ssa-type-system.md` §3.2's own open question (whether any `Rule` relies on type-pointer
  equality as type-equality in a way qualifier-bearing `Datatype`s would break), does fixing `volatile`
  printing first (option 1/3) create user-visible expectation ("the printer already shows volatile, why
  not const?") that puts schedule pressure on the harder type-system-side fix? Worth flagging to whoever
  sequences ticket 0017's requirements against ticket 0020's roadmap.
- Is there a reason `pushSymbol`'s `isVolatile()` check was added only for coloring and never extended to
  text — e.g. was it deliberately deferred pending a `const` companion so both would ship together, or is
  this simply an oversight? No comment in `printc.cc` records a reason either way.

---

## Flaw 5 — No templates or generics: a precise, two-layer architectural blocker, not a single missing feature

### Observation

This is the direct cross-reference the ticket asks for by name. `varnode-ssa-type-system.md` §3.1 ("No
templates, generics, or parametric types," ticket 0012) already establishes the representation-side half
of this gap: every `Datatype` is fully concrete (`type.hh:580,633`), there is no type-parameter concept,
and two monomorphizations of the same generic become two unrelated `TypeStruct` objects with no recorded
relationship. This review verifies the second, printer-side half of the blocker directly, and the two
combine into a complete answer to "what would templates/generics output even look like given the current
design."

**The type-level blocker, re-verified here:** `type_metatype` (`type.hh:80-100`) is a closed 18-value
enum — `TYPE_VOID` through `TYPE_PARTIALUNION` — with no `TYPE_TEMPLATE`/`TYPE_GENERIC` kind and no
field anywhere in `Datatype`'s flag set (`type.hh:180-198`, the same set Flaw 4 checked for `const`) that
could hold a type-parameter list or an instantiation-argument list.

**The printer-level blocker, newly verified here:** `PrintC`'s declaration printer — the exact code path
`expression-cast-printer.md` §2.2 showed correctly handles C's non-linear declarator syntax — is not
merely *missing* template support; it is **structurally closed over exactly three composable layer
kinds** and cannot be handed a fourth without new code, because it dispatches on `Datatype::getMetatype()`
via a hard-coded three-way branch in two places:

- `PrintC::buildTypeStack` (`printc.cc:143-164`): walks outer-to-inner, and its `if`/`else if` chain
  recognizes exactly `TYPE_PTR` → `getPtrTo()`, `TYPE_ARRAY` → `getBase()`, `TYPE_CODE` → the function's
  return type via `FuncProto::getOutputType()`; anything else falls through to `break` ("Some other
  anonymous type," `printc.cc:161`) and the walk simply stops.
- `PrintC::pushTypeStart` (`printc.cc:264-303`): walks the stack back outward and its loop
  (`printc.cc:288-297`) recognizes exactly the same three cases — `TYPE_PTR` → push `ptr_expr`,
  `TYPE_ARRAY` → push `array_expr`, `TYPE_CODE` → push `function_call` — with a fourth, unreachable-today
  branch that `throw LowlevelError("Bad type expression")` (`printc.cc:299-301`) rather than printing
  anything. `PrintC::pushTypeEnd` (`printc.cc:313-346`) mirrors the same three-case closure for the
  trailing array-size/parameter-list pieces.

There is no fourth `else if` for "print a bracketed type-argument list" in either method, and — this is
the key structural point — even if `Datatype` grew a type-parameter-list field tomorrow (per §3.1's option
A), these two methods would still silently ignore it, because the branch is a fixed enumeration of
`type_metatype` values, not a generic "does this type have extra declarator-relevant structure" query.

### Impact

This produces a precise, concrete answer to the ticket's central question:

**What templates/generics output looks like today:** exactly what `varnode-ssa-type-system.md` §3.1's own
improvement option B describes as the *status quo* — a monomorphized instantiation prints as an ordinary,
disconnected `TypeStruct`/`TypeUnion` under whatever name `TypeFactory` gave it (e.g. a demangled name
string like `Vector_int` or `Vector<int>` *as a literal, opaque name* if the upstream name source already
produced that text), with **no** `<...>` declarator syntax generated by the printer itself, no relation
recorded to a sibling instantiation (`Vector<char>`), and no way for the printer to know it is looking at
a generic instantiation at all rather than an ordinary struct that happens to have angle brackets in its
name string.

**Why closing the gap needs two coordinated changes, not one:** adding template/generic *output* is not
achievable by touching `printc.cc` alone, nor by touching `type.hh` alone:

1. Without a `Datatype`-level type-parameter concept (`varnode-ssa-type-system.md` §3.1 option A), there
   is no data for the printer to consume — "the printer has nothing to print even if it wanted to,"
   exactly as the ticket's own example phrasing anticipates.
2. Even with that data added, `buildTypeStack`/`pushTypeStart`/`pushTypeEnd`'s closed three-way
   `type_metatype` switch would not automatically pick it up — a new declarator-layer case (parallel to
   how `ptr_expr`/`array_expr`/`function_call` already work, `expression-cast-printer.md` §2.2) has to be
   added by hand in `printc.cc`, and — per Flaw 3 above — in `PrintJava`'s independently-rewritten
   `pushTypeStart`/`pushTypeEnd` (`printjava.cc:76-119`) as a *second*, separate implementation, since
   Java generics erasure and C++ template instantiation are not the same declarator problem and `PrintJava`
   does not inherit `PrintC`'s declarator logic at all (Flaw 3's override table).

So the full answer is: **the blocker is jointly a representational gap in `Datatype`
(`varnode-ssa-type-system.md` §3.1) and a closed-enumeration gap in `PrintC`'s declarator switch
(`printc.cc:143-164`, `:264-303`, `:313-346`)** — closing only one leaves either data with nothing to
print, or a printer change with no data to drive it.

### Severity

**Medium**, matching `varnode-ssa-type-system.md` §3.1's own severity judgment (most current decompiler
targets are C, where this doesn't arise) — but High-priority as *input to requirements*, since ticket
0017 is explicitly named by both this document and §3.1 as the consumer of this exact finding, and the
finding is now complete on both the type-system and printer sides rather than half-documented.

### Improvement options

(These extend, rather than duplicate, `varnode-ssa-type-system.md` §3.1's three options — that section's
options concern representation only; the options below add the printer-side half each would need.)

1. **Full representation (§3.1 option A) + a genuine fourth declarator-layer kind in `PrintC`.** Add
   `TypeTemplate`/`TypeGeneric` to the type system as §3.1 describes, and add a real
   `type_argument_list`-shaped `OpToken`/branch to `buildTypeStack`/`pushTypeStart`/`pushTypeEnd`,
   reusing the same generic `parentheses()`-driven declarator model that already makes `ptr_expr`/
   `array_expr`/`function_call` work correctly (`expression-cast-printer.md` §2.2).
   - *Pro*: the only option that produces genuinely correct, relationship-preserving template output
     (`Vector<int>` printed *and* known to be an instantiation of `Vector`); reuses a declarator model
     already proven to compose correctly for pointer/array/function nesting.
   - *Con*: inherits §3.1 option A's full "Large" cost (interning scheme, `compare()`/`compareDependency()`,
     the Ghidra type-bridge round-trip, §1.8) *plus* new `PrintC`-side work *plus* a second,
     independent implementation in `PrintJava` (Flaw 3) if Java-style generics are also in scope, since
     `PrintJava` does not share `PrintC`'s declarator code at all. *Size*: Large.
2. **Cosmetic name-pattern printing only (§3.1 option B), fully printer-side, no `Datatype` change.**
   Recognize an already-demangled `Foo<int>`-shaped name string in `buildTypeStack`'s base-type name
   (`ct->getDisplayName()`, used at `printc.cc:284`) and print it as-is rather than routing it through the
   pointer/array/function declarator machinery at all, since a name string requires no new declarator
   layer to print — it is already just the base-type atom.
   - *Pro*: essentially free on the printer side (the base-type name is already printed verbatim,
     `printc.cc:283-285`) — this option's entire cost is in whatever upstream demangling/naming step
     already produces (or could produce) template-shaped name strings, which is outside this ticket's file
     scope. *Con*: purely cosmetic, exactly as §3.1 already says — two instantiations still have zero
     recorded relationship at the `Datatype` level, so nothing downstream of printing (analysis, cast
     logic, a future "these are the same generic" IDE feature) benefits, only the literal printed text
     does. *Size*: Small (matching §3.1's own sizing for the representation half; the printer half is
     effectively zero additional cost beyond what already exists).
3. **Narrow container-only modeling (§3.1 option C) with a matching narrow printer case.** Model only
   single-type-parameter containers by extending `TypeArray`'s existing element-type pattern
   (`type.hh` — `TypeArray` already carries one element `Datatype*`), and add exactly one new declarator
   case to `PrintC` for that narrow shape (print `Container<ElementType>` where `ElementType` is the
   existing element-type pointer's display name) rather than a fully general argument-list mechanism.
   - *Pro*: much smaller than option 1 on both sides — the printer case is nearly as simple as option 2's,
     but backed by a real (if narrow) `Datatype`-level relationship instead of a name-string convention,
     so it is not purely cosmetic for the container case specifically. *Con*: does not generalize to
     multi-parameter templates or non-container generics, matching §3.1's own caveat; the printer-side
     "one narrow case" would likely need to grow into option 1's general mechanism anyway if any
     multi-parameter case is ever wanted, making this a plausible dead end rather than a stepping stone
     if that generalization is needed later. *Size*: Small-Medium.

### Open questions

- Same as `varnode-ssa-type-system.md` §3.1's open question, restated for the printer half: if Ghidra's
  own `DataTypeManager` already carries some template/generic concept the C++-side bridge doesn't
  propagate (§1.8), does that concept already imply a preferred declarator syntax (angle brackets, a
  different bracket convention, name-mangling conventions) that should shape which of the three options
  above is chosen, rather than inventing one from scratch on the C++ side?
- If option 1 is eventually chosen, does the new declarator-layer case need to be designed jointly with
  `PrintJava`'s independent `pushTypeStart`/`pushTypeEnd` (Flaw 3) from the start, given Java generics
  (type-erased at the bytecode level, per the JVM's own model) and C++ templates (fully monomorphized,
  distinct concrete types) are not the same underlying phenomenon and may need genuinely different printer
  logic rather than one shared mechanism?

---

## Summary for the requirements phase (ticket 0017)

1. **The templates/generics blocker (Flaw 5) is now fully specified on both sides**: `Datatype` has no
   type-parameter-list concept (`varnode-ssa-type-system.md` §3.1, `type.hh:80-100`), and `PrintC`'s
   declarator printer is closed over exactly `{TYPE_PTR, TYPE_ARRAY, TYPE_CODE}` with no room for a fourth
   layer without new code (`printc.cc:143-164,264-303,313-346`). Any requirement asking for template/
   generic output must scope both halves, not just "add the type system support" — the printer-side work
   is smaller but not zero, and is a *second*, independent cost for `PrintJava` if Java-style generics are
   also wanted (Flaw 3).
2. **A cheap, real, immediately-shippable output-quality fix exists independent of any refactor**: printing
   `volatile` from `Symbol::isVolatile()` (Flaw 4, option 1) is a small, localized `printc.cc` change with
   no type-system dependency — worth flagging to whoever prioritizes near-term work versus refactor-scale
   work, since it is unusual in this document set for being both real and cheap.
3. **The "well-factored abstraction" question has a nuanced answer**: `PrintLanguage`'s RPN engine and
   generic `parentheses()` algorithm are genuinely reusable and proven (Flaw 3) — but the parts most
   likely to matter for a "rich language features" requirement (statement-level control flow, declarator
   syntax) are exactly the parts with the least multi-language validation, and four of them are
   self-documented as bypassing the abstraction entirely today (Flaw 2). A requirement that assumes
   "just implement `PrintLanguage`" is a cheap extension point for language N should budget for
   Flaw 2/Flaw 3's findings, not just the class hierarchy's surface appearance.

## Related documents

- [`expression-cast-printer.md`](../01-review/expression-cast-printer.md) (review ticket 0009) — primary
  source for Flaws 1-4; all file:line citations in this document re-verify, rather than merely repeat,
  that review's claims.
- [`varnode-ssa-type-system.md`](varnode-ssa-type-system.md) (flaw ticket 0012) — §3.1 is the direct
  cross-reference for Flaw 5's representation-side half; §3.2 is the direct cross-reference for Flaw 4's
  `const`/`volatile` half.
- [`../00-overview/glossary.md`](../00-overview/glossary.md) — shared terminology.
- Consolidated across all four flaw docs: [`00-index.md`](00-index.md) (owned by ticket 0011, not this
  document).
