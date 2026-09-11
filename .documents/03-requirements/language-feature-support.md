# Requirements: Rich Language Feature Support

> Produced by [`.tasks/0017`](../../.tasks/0017-task-requirements-language-feature-support.md), a child
> of [`.tasks/0016`](../../.tasks/0016-epic-requirements-definition.md) /
> [`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md). Format per the parent epic: each
> requirement gets a stable `REQ-LANG-<n>` ID, an RFC-2119-style MUST/SHOULD/MAY priority, a one-line
> statement, rationale with links back to specific flaw-analysis entries, and — where feasible — an
> acceptance-test sketch. Primary inputs:
> [`varnode-ssa-type-system.md`](../02-flaws/varnode-ssa-type-system.md) (0012) §3 "Type-system
> expressiveness gaps", and [`printer-output-quality.md`](../02-flaws/printer-output-quality.md) (0014)
> Flaws 4-5. Shared terms: [`../00-overview/glossary.md`](../00-overview/glossary.md).

## 0. Scope and how to read this document

This document answers one question: **what "rich language features" must the refactored type system and
printer be able to represent and print, and at what priority?** It covers C++ templates/generics,
qualifier (`const`/`volatile`/`restrict`) fidelity, function-pointer and calling-convention type fidelity,
recursive/forward-declared composite types, anonymous aggregates, and namespace/scoping fidelity — the
starting list from ticket 0017, expanded and justified per requirement below.

**Explicitly out of scope for this document** (belongs to a different requirements domain per the parent
epic's split): graded/confidence-scored type inference (`varnode-ssa-type-system.md` §3.7) is a
*type-inference-quality* concern, not a *language-feature-representation* concern — it does not change
what can be expressed, only how confidently the engine claims to know it — and is left to whichever of
sibling tickets 0018 (output fidelity/readability) or 0019 (extensibility/API) the orchestrator judges the
better fit. It is noted here only so its absence isn't mistaken for an oversight.

### 0.1 The feasibility framework (read this before the templates/generics requirements)

Per the ticket's acceptance criteria, every generics/templates-related requirement below is tagged with
one of two feasibility labels, and the label is load-bearing — it changes what "done" means for that
requirement:

- **[FEASIBLE — single binary]**: achievable using information already recoverable from one binary today,
  specifically the structured demangled-name metadata Ghidra's `GnuDemangler`/`MicrosoftDemangler`
  features already parse out of Itanium- and MSVC-mangled symbol names (§1 below documents exactly what
  exists). This is **not** "re-derive the generic definition from its monomorphized machine code" — that
  is a source-level artifact permanently erased by compilation and is not recoverable from any single
  binary's code bytes, full stop. It is "take template-argument structure the demangler has *already
  parsed out of the mangled name string* and thread it through to the type system and printer instead of
  discarding it at the string-formatting boundary it currently stops at."
- **[REQUIRES EXTERNAL INFO]**: would need external debug metadata — a PDB with MSVC's own template
  instantiation records, or DWARF emitted with `-g` carrying `DW_TAG_template_type_param`/
  `DW_TAG_template_value_param` — to ever be satisfied for binaries where the mangled name itself doesn't
  carry the needed information (e.g. a stripped/name-mangling-suppressed binary, or a case where the
  demangler cannot fully parse a name it doesn't recognize). No amount of pattern-matching over
  monomorphized code alone gets there for the general case; recovering an unknown generic's *original type
  parameter name*, its constraints, or a definition body specialized away by the compiler are the concrete
  examples of information that plain demangling cannot supply and only source-level debug metadata could.

Every requirement below that touches templates/generics states which label applies, and for the
"detecting families" requirement (REQ-LANG-3), states explicitly why full recovery-without-hints is
infeasible even in principle.

## 1. What the demangler already gives the decompiler today (verified in this session)

This section is the concrete evidentiary basis for the feasibility split above — read before REQ-LANG-1/2.

**`GnuDemangler`** (`Ghidra/Features/GnuDemangler/src/main/java/ghidra/app/util/demangler/gnu/`) produces
a `ghidra.app.util.demangler.DemangledDataType` for every demangled type, and that class already carries a
dedicated template concept: an `isTemplate` boolean (`DemangledDataType.java:113`) and a
`DemangledTemplate getTemplate()` accessor whose `toTemplate()` method builds `<T1,T2,...>` syntax from a
`List<DemangledDataType>` of parsed template *parameters* — not a flattened string
(`Ghidra/Features/Base/src/main/java/ghidra/app/util/demangler/DemangledTemplate.java:23-45`). Each
parameter is itself a full `DemangledDataType`, so nested templates (`vector<pair<int,char>>`) are already
structurally parsed, not just textually matched. Critically, `DemangledTemplate.getDataType()`
(`DemangledTemplate.java:52-55`) **throws `UnsupportedOperationException`** with the message *"We cannot
store templated types in the datatype manager!"* — i.e. Ghidra's own Java-side `DataTypeManager` (the
system `TypeFactory`/`TypeFactoryGhidra` bridges against, per `varnode-ssa-type-system.md` §1.8) already
declines to store this structure as a real, related-instance-aware datatype. It is parsed, then thrown
away at exactly the boundary where it would need to become a `Datatype`.

**`MicrosoftDemangler`** (`Ghidra/Features/MicrosoftDemangler/.../MicrosoftDemanglerUtil.java`) goes
through the `mdemangler` library, which independently parses MSVC-mangled template names into a structured
`MDTemplateNameAndArguments`/`MDTemplateArgumentsList` (`MicrosoftDemanglerUtil.java:32,179-180`). The
mapping from that structure into a `DemangledTemplate` is present in the source **only as commented-out
code** with the developer's own admission it's incomplete: `// I think that this is a kludge`
(`:415`), `// NO NAMESPACE for high level template` (`:422`), and `// TODO: Not sure templates are data
types` (`:945,947`). So on the MSVC side, the raw structured argument list exists inside `mdemangler`'s
parse tree, but Ghidra's own bridge code does not yet forward it into `DemangledTemplate` the way the GNU
path does.

**What this means for REQ-LANG-1/2 below:** the hard part — parsing a mangled template instantiation name
into a structured argument list — is **already solved** for Itanium-mangled (GCC/Clang) binaries and
**partially solved** (parsed by `mdemangler`, not yet bridged) for MSVC-mangled binaries. Neither the
decompiler's `Datatype` hierarchy nor `PrintC`'s declarator printer currently consumes any of this — per
`printer-output-quality.md` Flaw 5, `PrintC::buildTypeStack`/`pushTypeStart`/`pushTypeEnd` are closed over
exactly `{TYPE_PTR, TYPE_ARRAY, TYPE_CODE}` (`printc.cc:143-164,264-303,313-346`) and would silently drop
any template-argument data handed to them today, and per `varnode-ssa-type-system.md` §3.1, `Datatype`
itself has no field to hold that data in the first place (`type.hh:80-100,180-198`). The gap is real and
is exactly where the flaw docs say it is — but it is a *plumbing and representation* gap downstream of
already-recovered data, not a *"detect templates from raw bytes"* research problem.

## 2. Requirements

### REQ-LANG-1 — Template-instantiation display syntax from demangler metadata

**Priority: MUST**
**Feasibility: [FEASIBLE — single binary]**

Statement: When a symbol's demangled name (`GnuDemangler` or `MicrosoftDemangler`) carries structured
template-argument data (`DemangledTemplate`/`DemangledDataType.isTemplate()`), the printer MUST render the
corresponding type name using proper `Name<Arg1,Arg2,...>` declarator syntax at every point that type
appears in output (variable declarations, casts, function signatures, `sizeof`-like contexts) — not as an
opaque flattened string, not truncated at the first `<`, and not silently dropped by the declarator
printer's closed `type_metatype` switch.

**Rationale.** Directly closes the printer-side half of `printer-output-quality.md` Flaw 5: `PrintC`'s
declarator switch (`buildTypeStack`/`pushTypeStart`/`pushTypeEnd`, `printc.cc:143-164,264-303,313-346`) is
"structurally closed over exactly three composable layer kinds" and "would silently ignore" template data
even if the type system had it; this requirement is Flaw 5's own improvement option 2 ("cosmetic
name-pattern printing... essentially free on the printer side," since `getDisplayName()` is already
printed verbatim at `printc.cc:283-285`) made concrete and mandatory rather than optional. It also matches
`varnode-ssa-type-system.md` §3.1's improvement option B (cosmetic printer-level grouping), which that
document's own summary table (§5, finding 11) names as the **smallest viable option** for the templates
gap — justifying MUST priority here specifically because it is small, has no type-system dependency, and
directly fixes a demonstrated, verified output-quality defect (§1 above: the data is parsed and discarded
today).

**Acceptance-test sketch.** Feed the decompiler an Itanium-mangled symbol for a function taking a
`std::vector<int>&` parameter (or an equivalent synthetic `_datatests` XML fixture using a
`GnuDemangler`-recognized mangled name). Assert the printed signature contains the literal text
`vector<int>` (or the project's chosen container-name convention) with balanced angle brackets — not the
mangled name, not a name truncated at the first template argument, and not an unrelated placeholder type.
Repeat for a nested case (`vector<pair<int,char>>`) to confirm recursive argument printing.

**Before/after example.** See §3 (Worked Example) below for a full signature-level illustration.

---

### REQ-LANG-2 — First-class `Datatype`-level template/generic representation

**Priority: SHOULD**
**Feasibility: [FEASIBLE — single binary, for the *representation-linking* half only; REQUIRES EXTERNAL
INFO for anything beyond what the demangled argument list itself encodes]**

Statement: The `Datatype` hierarchy SHOULD gain a template/generic concept (e.g. a `TypeTemplate`/
`TypeGeneric` kind) that records, for a monomorphized instantiation whose demangled name carries template
arguments, an explicit link to a shared "generic definition" identity plus its own argument list — so that
`Vector<int>` and `Vector<char>` recovered from the same binary are represented as two *related* instances
of one generic, not two structurally-unrelated `TypeStruct` objects, and downstream consumers (casts,
struct-editing UI, cross-reference tooling) can query that relationship directly instead of re-parsing
display-name text.

**Rationale.** This is `varnode-ssa-type-system.md` §3.1's improvement option A (full representation) and
`printer-output-quality.md` Flaw 5's improvement option 1 (the "only option that produces genuinely
correct, relationship-preserving template output... reuses a declarator model already proven to compose
correctly," at the flaw doc's own admitted "Large" cost — interning-scheme changes, `compare()`/
`compareDependency()` updates per §1.2, and a second independent `PrintJava` implementation per Flaw 3).
SHOULD rather than MUST because: (a) REQ-LANG-1 already delivers the readability win most users need at
MUST-priority and near-zero cost; (b) this requirement's own "Large" sizing (both source flaw docs agree)
means it competes for scarce refactor budget against other MUST items with demonstrated correctness
impact (e.g. REQ-LANG-4/5's qualifier fixes, or 0012's own higher-severity findings like union-field
resolution); (c) per §1 above, the *argument-list* data needed to drive this already exists in the
demangler bridge for Itanium names and is one bridging fix away for MSVC names — so this requirement's
representation half is feasible from single-binary demangled metadata precisely because it reuses that
already-parsed structure as its interning key, exactly as `varnode-ssa-type-system.md` §3.1's option A
describes ("the interning key... extended to include the argument list").

**Feasibility caveat, stated explicitly per the ticket's acceptance criteria:** this requirement is
feasible *as a name-driven grouping mechanism* — two instantiations whose demangled names share a base
template name and differ only in argument lists get linked. It is **not** feasible, from a single binary,
to recover the *original, unspecialized* generic definition (its source-level type-parameter names,
constraints, or a body that the compiler discarded during monomorphization) — only PDB/DWARF template
metadata (`DW_TAG_template_type_param` et al.) or the MSVC PDB's own template-instantiation records could
ever supply that. This requirement's acceptance criterion is therefore scoped to "instances are correctly
linked and printed with correct argument syntax," not "the generic's original declaration is reconstructed
verbatim."

**Acceptance-test sketch.** Given two synthetic functions in one `_datatests` fixture, one taking
`Vector<int>*` and one taking `Vector<char>*` (both Itanium-mangled with the same base template name),
assert: (1) both print with correct `Vector<T>` declarator syntax (REQ-LANG-1); (2) a query against the
type system (e.g. `TypeFactory` lookup or an equivalent test hook) confirms the two `Datatype` instances
report the same generic-definition identity and different argument lists, distinguishing this from two
independently-named, unrelated structs that merely happen to share a textual prefix.

---

### REQ-LANG-3 — Monomorphized-function-family detection (stretch)

**Priority: MAY** (explicit stretch goal, not a MUST — see justification)
**Feasibility: [FEASIBLE — single binary, heuristic-only, opt-in] for the demangled-name-driven case;
[infeasible in principle, any information source] for reconstructing an unmangled/stripped binary's
generic families purely from code-shape similarity with guaranteed correctness**

Statement: The decompiler MAY detect families of near-identical functions that plausibly originated from
one generic/template definition — driven primarily by shared demangled base names with differing template
arguments (the reliable signal, when present), and only secondarily by structural code-shape similarity
across functions with no demangled relationship (the unreliable signal, when mangling/demangling is
unavailable) — and present them grouped with an explicit "possibly generated from a common definition,
confidence: heuristic" annotation, never as a flat assertion of fact.

**Rationale and priority justification.** This is the ticket's own explicitly-flagged stretch item.
Priority is capped at MAY, not SHOULD, for three concrete reasons grounded in the flaw docs: (1) when a
demangled name is available, REQ-LANG-2 already gives an exact, non-heuristic answer — this requirement's
only genuine value-add is the code-shape-similarity fallback for stripped/unmangled binaries, which is a
narrower and much less reliable case; (2) `varnode-ssa-type-system.md` explicitly warns, in an adjacent
context (§3.7, graded confidence), that "a plausible-but-wrong grouping is worse than an obviously-failed
one" (§4.1's finding on `Merge`'s speculative heuristics) — the same risk class applies here: two
genuinely-unrelated functions that happen to share code shape (e.g. two different small helper functions
compiled similarly by an optimizer) could be wrongly presented as one generic family, which is a
readability nicety turning into a misleading correctness claim if not clearly labeled as speculative; (3)
building a reliable structural-similarity detector is real, open-ended heuristic-search work with no
existing scaffolding in the reviewed codebase to build from (unlike REQ-LANG-1/2, which extend
already-existing demangler infrastructure).

**Feasibility, stated per the ticket's acceptance criteria:** for the demangled-name-driven case, this is
achievable from one binary today — no external info needed beyond what REQ-LANG-2 already extracts. For
the fully-general case (recovering that an unrelated, unmangled/stripped set of functions originated from
one generic definition, with no textual hint at all), **this is not reliably achievable from any single
binary's code, and is not fully achievable even with external debug info** unless that debug info itself
records the instantiation relationship (which is exactly the demangled-name/PDB-template-record case
already covered by REQ-LANG-2) — pure code-shape clustering is an unsound heuristic in the general case
(convergent optimization can make unrelated functions look similar; template specialization can make
related functions look different) and must never be presented as a fact rather than a suggestion.

**Acceptance-test sketch.** Given a fixture with three Itanium-mangled instantiations of the same template
and two unrelated, independently-compiled functions that happen to share a superficially similar
instruction pattern, assert the grouping mechanism links the three genuine instantiations (via
REQ-LANG-2's mechanism) and does **not** merge either of the two unrelated functions into that group
without a demangled relationship — i.e., the acceptance bar for this requirement is explicitly about *not
overclaiming*, not about maximizing recall.

---

### REQ-LANG-4 — Print `volatile` from existing `Symbol` data (immediate fix)

**Priority: MUST**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: The printer MUST emit the `volatile` keyword in a declaration's text whenever
`Symbol::isVolatile()` is true, using the same underlying flag `PrintC::pushSymbol` already consults for
syntax-highlight coloring (`printc.cc:1971`) — not merely represent volatility as a GUI-only color cue.

**Rationale.** This is `printer-output-quality.md` Flaw 4's own headline finding, and the flaw doc's own
severity judgment ("Medium... a real, demonstrable information-loss bug with a small, well-localized fix")
plus its improvement-option-1 sizing ("Small... close to the smallest possible fix in this entire
document") justify MUST priority directly: the underlying data already exists and is already consulted at
the exact call site that needs to also emit text (`pushSymbol`, `printc.cc:1971`), a direct string search
of `printc.cc`/`printc.hh` found **zero** occurrences of `"volatile"` as printable output today (Flaw 4,
verified), and — per the flaw doc's own framing — "the information exists and is discarded specifically at
the print boundary, not lost earlier in the pipeline," making this the rare finding in the source material
that is both real and unambiguously cheap to fix. This requirement is independent of, and should not be
blocked by, REQ-LANG-5's larger type-system work (Flaw 4's improvement option 3 makes exactly this
sequencing argument).

**Acceptance-test sketch.** A `_datatests` fixture with a symbol explicitly marked volatile (e.g. a
memory-mapped hardware register access) must produce printed C containing the literal token `volatile` in
that symbol's declaration; a matching non-volatile symbol of the same type must not.

---

### REQ-LANG-5 — Composable `const`/`volatile`/`restrict` qualifiers on `Datatype`

**Priority: MUST**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: The `Datatype` hierarchy MUST support `const`, `volatile`, and `restrict` as composable
type-level qualifiers — attachable independently to a pointer and to its pointee, so that `const int*`
(pointer to const int), `int *const` (const pointer to int), and `const int *const` (both) are
distinguishable `Datatype` instances, matching real C/C++ qualifier semantics — and the printer MUST place
the qualifier keyword(s) at the correct declarator position for each case.

**Rationale.** Directly closes `varnode-ssa-type-system.md` §3.2 ("No composable const/volatile qualifier
fidelity" — verified: `Datatype`'s full flag set at `type.hh:180-198` has **no `const` bit at all**, at any
granularity, and `volatile` exists only one level removed, as a `Varnode`/`Symbol` flag never a `Datatype`
property) and the representational half of `printer-output-quality.md` Flaw 4 (`const` "is a pure
representational ceiling, not a printer bug... the fix has to start upstream in the type system"). MUST
priority because: this is a plain, everyday C construct (unlike templates, which are a narrower C++-heavy
audience concern per 0012 §3.1's own severity note) — `const`-correctness is core to how real-world C/C++
headers are written, and its complete absence from `Datatype` is a representational ceiling affecting
*every* decompiled binary with `const`-qualified data, not an edge case. `volatile`'s current
storage-location-not-type modeling additionally means a `Datatype*` shared between a volatile and
non-volatile access (plausible under the interning scheme) cannot itself distinguish the two, which is a
correctness-adjacent modeling gap per 0012 §3.2's own impact analysis, not merely cosmetic. `restrict` is
included because it is the third C-standard type qualifier and the ticket's own scope names it explicitly;
this ticket's flaw-analysis inputs did not audit `restrict` specifically, so its concrete gap should be
re-verified against `type.hh` during implementation rather than assumed identical to `const`'s, but the
requirement is stated for completeness of the qualifier set.

**Sizing note carried forward from the source flaw doc:** `varnode-ssa-type-system.md` §3.2 sizes the full
option (its option A) as Large, chiefly because qualifier-aware interning multiplies `Datatype*` instance
counts and requires re-auditing every place propagation code currently treats pointer-equality as
type-equality (§3.2's own open question, itself flagging a needed cross-check against ticket 0013's rule
findings before implementation). This MUST requirement does not waive that sizing or that open question —
it states the *outcome* required, not that it is cheap; implementation planning (a later epic, per this
epic's scope) should treat §3.2's option C (qualify pointers only, the flaw doc's own "smallest viable
option" per its §5 summary table) as an acceptable staged first increment toward satisfying this
requirement in full, provided the staged plan is explicit that it is partial.

**Acceptance-test sketch.** A `_datatests` fixture declaring a parameter of type `const int *` and another
of type `int *const` must print with the qualifier at the textually correct position for each, and the two
must be distinguishable `Datatype` objects at the type-system level (not merely distinguished by printer
string formatting). A third case, `const int *const`, must print both qualifiers correctly placed.

---

### REQ-LANG-6 — Preserve function-pointer / calling-convention type fidelity through refactor

**Priority: MUST (non-regression)**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: Any refactor of the type system or printer MUST NOT regress the calling-convention fidelity
`ProtoModel`/`FuncProto` already provide today — function-pointer types and declarations that carry a
non-default calling convention (e.g. `__fastcall`, `__stdcall`, `__thiscall`) MUST continue to print that
convention's decoration correctly after any refactor, exactly as they do today.

**Rationale.** Per the ticket's own framing, this is explicitly a *preserve, don't regress* requirement —
`ProtoModel`-based calling-convention modeling is existing, reviewed functionality
(`signature-calling-conventions.md`, review ticket 0008) that the 0012/0014 flaw docs did not identify as
broken; `varnode-ssa-type-system.md` §3.3/§3.5 instead flag *adjacent, narrower* gaps (below, REQ-LANG-7/8)
rather than a defect in the core mechanism. MUST priority because calling-convention fidelity is
correctness-relevant output (a `__fastcall` function pointer cast to an incompatible calling-convention
type is a real bug class in generated C, not a readability nicety), and because an explicit non-regression
requirement is the correct way to guard genuinely-working functionality during a large refactor — it gives
the refactor's own test suite (characterization tests per the sequencing risk table in
`architecture-extensibility.md`, flaw ticket 0015) an explicit target to check against.

**Acceptance-test sketch.** Re-run the existing calling-convention-focused fixtures already covered by
`testfuncproto.cc`/`testparamstore.cc` (per `00-index.md`'s test-coverage table, real unit tests already
exist here) after any refactor lands, plus a printer-level `_datatests` fixture asserting a `__fastcall`
function-pointer *type* (not just a function definition) prints its convention keyword correctly in a
variable declaration and in a cast.

---

### REQ-LANG-7 — Standalone calling-convention tag on a bare function-pointer type

**Priority: SHOULD**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: The type system SHOULD support attaching a calling-convention tag to a function-pointer
`Datatype` (`TypeCode`) independently of a fully-recovered `FuncProto`, so a function-pointer *type*
declaration known only by calling convention and signature shape (the common case for header-declared
callback types before any specific function using that type has been analyzed) can carry that information
without requiring the full parameter/return-recovery machinery `FuncProto` currently demands.

**Rationale.** This is `varnode-ssa-type-system.md` §3.5, which that document's own severity judgment
scores Low and — unusually among that document's findings — explicitly flags as **unconfirmed**: "this
ticket's source scope (`type.hh`, `type.cc`, `typeop.hh`) cannot fully assess whether it's a real gap or
whether `signature-calling-conventions.md` (ticket 0008)... already shows a lighter-weight mechanism this
document missed." SHOULD rather than MUST reflects that unresolved status directly — this requirement
should not be treated as confirmed-necessary until cross-checked against ticket 0008/0013's territory as
§3.5's own open question requests; it is included at SHOULD, not omitted, because the ticket's own
suggested requirement list names function-pointer/calling-convention fidelity explicitly and the gap is
plausible even if unconfirmed. Implementers should re-verify §3.5's open question before committing
resources here.

**Acceptance-test sketch.** Deferred pending the confirmation `varnode-ssa-type-system.md` §3.5 itself asks
for — an acceptance test cannot be soundly written until it's established whether the gap is real; this is
noted as a prerequisite rather than skipped.

---

### REQ-LANG-8 — First-class function-pointer-with-context type (member-function pointers)

**Priority: SHOULD**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: The type system SHOULD provide a `Datatype` subclass that pairs a function/function-pointer
type with an explicit offset/adjustor context (mirroring how `TypePointerRel` already pairs a `TypePointer`
with a struct offset, `type.hh:740-779`), so that C++ member-function pointer values (`R (C::*)(Args...)`)
— which on most ABIs carry both a code address and an implicit `this`-adjustment or vtable-context value —
can be represented as a distinct, analyzable type rather than an opaque blob or a plain function pointer
that silently loses the adjustment context.

**Rationale.** Directly closes `varnode-ssa-type-system.md` §3.3 ("No first-class 'function pointer with
context' type"), which frames the gap as real but of narrower audience-impact than the templates gap
("Low-Medium... member-function pointers are a smaller fraction of typical binaries than templated types
are"). SHOULD rather than MUST reflects that severity judgment plus a concrete implementation risk §3.3
itself names: "member-function-pointer ABI representations vary significantly across compilers (Itanium
ABI's `{ptr, adjustment}` pair vs. MSVC's multiple representations depending on inheritance shape)" — a
single mechanism may need ABI-specific variants, raising real cost the flaw doc could not fully size within
its own scope. §3.3's own option C (defer, contingent on the templates/generics decision in §3.1, since
both point toward "a more expressive parametric/composite type mechanism") is a reasonable sequencing
choice an implementation plan may adopt; this requirement states the target outcome, not the sequencing.

**Acceptance-test sketch.** A `_datatests` fixture representing a member-function-pointer value under one
concrete ABI (e.g. Itanium's `{ptr, adjustment}` pair) must print as a recognizable member-function-pointer
declaration distinct from a plain function pointer, and the type system must expose the adjustment value as
queryable structure, not only as opaque extra bytes.

---

### REQ-LANG-9 — Recursive/self-referential and forward-declared composite types

**Priority: MUST**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: The type system and printer MUST correctly represent and print self-referential composite types
(e.g. a linked-list node struct containing a pointer to its own type) and mutually-recursive composite
types (struct `A` holding a pointer to struct `B`, and `B` holding a pointer back to `A`), emitting
whatever forward declarations are needed for the output to be valid, compilable C — at minimum preserving
the behavior that exists today, and closing any gap found in the mutual-recursion case specifically (see
verification note below).

**Rationale and verification note.** Unlike REQ-LANG-1 through REQ-LANG-8, this requirement is **not**
pre-documented as a gap in either 0012 or 0014 — it is one of the ticket's own explicitly-requested items,
and this session verified the starting state directly rather than citing an existing flaw entry: `type.hh`
already defines a `type_incomplete` flag explicitly documented as "Set if this (**recursive**) data-type
has not been fully defined yet" (`type.hh:191`, with `TypeStruct`/`TypeUnion`'s default constructors
starting in that state, `type.hh:598,638`), and `PrintC::docTypeDefinitions` already calls
`typegrp->dependentOrder(deporder)` to "put things in resolvable order" before emitting type definitions
(`printc.cc:2465-2476`) — i.e. **substantial existing machinery for this already exists** and the
self-referential single-struct case (the common one — a `next`/`prev`-style linked structure) is plausibly
already handled correctly via pointer types not requiring a complete pointee. MUST priority reflects that
this is ordinary, extremely common C (every linked data structure uses it) — a regression here during
refactor would be far more visible and impactful than most gaps in this document. The mutual-recursion
case (two named structs each needing the other, requiring a true standalone `struct B;` forward
declaration emitted ahead of `struct A`'s body) was **not traced to a definitive conclusion in this
session** — `dependentOrder`'s actual tie-breaking behavior for a true two-struct cycle needs direct
verification during implementation planning; this requirement is written to cover that verification
explicitly rather than assume the answer either way.

**Acceptance-test sketch.** Two `_datatests` fixtures: (1) a single self-referential struct (linked-list
node) must print correctly with no missing/incorrect forward reference; (2) two mutually-referential named
structs (`A` holding `B*`, `B` holding `A*`) must print as valid, compilable C — including an explicit
`struct B;` (or equivalent) forward declaration if the chosen definition order requires one. Test (2) is
the one this document could not pre-verify and is therefore the higher-value test to write first.

---

### REQ-LANG-10 — Anonymous struct/union/enum parity with modern C

**Priority: SHOULD**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: The printer SHOULD correctly emit anonymous (unnamed) struct, union, and enum types — both as
standalone type definitions and, per modern C (C11 §6.7.2.1) and common compiler extensions, as anonymous
members nested inside an enclosing struct/union whose fields are promoted into the enclosing scope — at
parity with how a hand-written modern C compiler would accept such a declaration.

**Rationale.** Ticket 0017's own starting list names this explicitly. This session verified partial
existing support directly: `PrintC` already special-cases anonymous types at multiple points —
`buildTypeStack`'s "Some other anonymous type" fallthrough (`printc.cc:161-162`), `pushTypeStart`'s
matching case (`printc.cc:342`), and an explicit anonymous-type check at `printc.cc:280` (`if
(ct->getName().size()==0)`) — and `TypeFactory::saveXml`-adjacent code explicitly skips saving anonymous
types by name (`type.cc:4912`, "Don't save anonymous types"). This indicates the *concept* of an anonymous
type is already recognized throughout the pipeline; what neither the flaw docs nor this session's targeted
checks establish is whether the *nested-anonymous-member-with-field-promotion* idiom (a `union { int x; }`
declared with no member name, whose fields become directly accessible on the enclosing struct — common in
real-world headers, e.g. Linux kernel and Windows SDK styles) is specifically supported, since that
requires the printer to omit a member/variable name at the field-declaration level in addition to the
type-name level. SHOULD (not MUST) because this is a narrower, more idiom-specific gap than REQ-LANG-5's
qualifier fix or REQ-LANG-9's recursive-type case, and was not independently confirmed as broken by either
source flaw document — it is included and prioritized on the strength of the ticket's explicit ask, not a
demonstrated defect.

**Acceptance-test sketch.** A `_datatests` fixture with a struct containing an anonymous nested union (two
overlapping fields accessible without a union member name) must print valid modern C reproducing that
promotion — not a synthetic named union standing in for it, which would change the field-access syntax a
human or downstream tool would see.

---

### REQ-LANG-11 — Namespace/scoping fidelity for C++-originated symbols

**Priority: MUST**
**Feasibility: N/A (not a templates/generics requirement, though closely related to REQ-LANG-1/2's naming
plumbing)**

Statement: (a) The printer MUST continue to correctly emit the minimal-or-full namespace-qualification
path for *symbols* (functions, variables) that already works today via `PrintC::pushSymbolScope`/
`emitSymbolScope`'s `MINIMAL_NAMESPACES`/`ALL_NAMESPACES` strategies — this is a non-regression
requirement, since this session verified the mechanism already exists and functions
(`printc.cc:199-259`). (b) The type system SHOULD (extending beyond pure non-regression) additionally
support a structural namespace/scope path for *type* names (struct/class/enum), analogous to what already
exists for `Symbol`/`Scope`, so a type's namespace can be queried and selectively qualified the same way a
symbol's can — today, a struct/class name recovered from a demangled C++ name is a flat display string
that may or may not textually contain `::` depending on what upstream demangling produced, with no
structural `Scope`-based representation the way `Symbol` has.

**Rationale.** This session verified both halves directly rather than relying on a pre-existing flaw
citation (this item, like REQ-LANG-9/10, is one of the ticket's own explicitly-requested items not
pre-covered by 0012/0014). The symbol-level mechanism is real and working: `Scope::getFullName`
(`database.cc:1510`) and `PrintC`'s `pushSymbolScope`/`emitSymbolScope` (`printc.cc:199-259`) compute a
"resolution depth" against the current scope and print exactly the namespace elements needed to
disambiguate a symbol reference, under two selectable strategies. This is genuine, non-trivial existing
functionality that a refactor could easily regress if the `Scope` hierarchy or `PrintC`'s scope-walking
logic is touched without preserving this behavior — hence MUST for the non-regression half. The
type-name half is different in kind: `Datatype` names are plain strings (`getDisplayName()`,
`getName()`), so a demangled `Foo::Bar` struct name is only namespace-qualified if the upstream demangler
happened to bake `::` into the literal string — there is no `Scope`-typed field on `Datatype` the way
`Symbol` has, so the type system cannot answer "what namespace is this struct in" as a structured query,
only "what string does this struct's name happen to contain." This is a genuine asymmetry between how
well-modeled symbol-scoping is versus type-scoping, surfaced by this ticket's own investigation rather
than the prior flaw docs — hence SHOULD (an improvement) layered on top of the MUST (preserve what
exists).

**Acceptance-test sketch.** (a) Non-regression: a `_datatests` fixture with two functions of the same name
in different namespaces, called from a third scope, must continue to print the minimal disambiguating
namespace path for each call site exactly as today. (b) New capability: a `_datatests` fixture with a
demangled nested-namespace struct name (e.g. `outer::inner::Widget`) should allow a test hook to query the
type's namespace path structurally (not by re-parsing the display string) once REQ-LANG-11(b) is
implemented.

---

### REQ-LANG-12 — Union-level bitfields (low-priority completeness item)

**Priority: MAY**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: The type system MAY gain a bitfield vector on `TypeUnion` (mirroring `TypeStruct::bitfield`,
`type.hh:581`) so a union member with bit-level sub-structure (common in hardware-register overlay idioms)
can be represented at the type level, not only recognized after the fact by the separate
expression-rewriting bitfield-idiom `Rule` layer (`bitfield.hh`).

**Rationale.** Directly closes `varnode-ssa-type-system.md` §3.4 ("No union-level bitfields"), whose own
severity judgment is explicitly Low ("A narrow, uncommon C idiom (though not rare in embedded/
hardware-register-heavy reverse-engineering targets...)") and whose own summary-table smallest-viable
option is "B — leave as-is; adequately covered by the existing Rule-based bitfield-idiom recognition layer
regardless of declared type" (§5, finding 14). MAY priority follows that judgment directly — this is
included for completeness against the ticket's "type-system expressiveness gaps" source material, not
because either flaw doc found it to be a high-value gap to close.

**Acceptance-test sketch.** Deferred — per the flaw doc's own severity judgment, this is not worth an
acceptance test unless and until it is actually scheduled for implementation.

---

### REQ-LANG-13 — Packed / explicit-alignment-override representation (completeness item)

**Priority: MAY**
**Feasibility: N/A (not a templates/generics requirement)**

Statement: `TypeStruct` MAY gain a packed/explicit-alignment flag honored by `assignFieldOffsets`
(`type.cc:2440`) for the standalone/console-tool code path, so a `#pragma pack`/`__attribute__((packed))`
struct laid out locally (without a Ghidra-side `DataTypeManager` to defer to) computes correct,
non-naturally-aligned field offsets.

**Rationale.** Directly closes `varnode-ssa-type-system.md` §3.6, whose own analysis concludes this is
"likely masked in the primary (Ghidra-hosted) use case by the wire bridge" (Ghidra's `DataTypeManager`
already resolves packed-struct offsets before they cross the wire, per §1.8's bridge description) and is
"a genuine but narrower gap for the standalone tool" only. §3.6's own improvement option B ("confirm
whether Ghidra-sourced struct field offsets bypass `assignFieldOffsets` entirely... before treating this as
a live gap") is a documented open question this ticket does not resolve. MAY priority, and this
requirement should not be scheduled for implementation until that open question is answered — implementing
option A (the flag) has "low value if... the primary Ghidra-hosted path never exercises local
`assignFieldOffsets` for packed structs anyway" per the source flaw doc's own words.

**Acceptance-test sketch.** Deferred pending §3.6's own open question (whether
`TypeFactoryGhidra::decodeType()` ever calls `assignFieldOffsets` for wire-received structs) — write the
open-question investigation as a prerequisite task before an acceptance test can be meaningfully scoped.

## 3. Worked example — before/after (REQ-LANG-1, REQ-LANG-2)

**Scenario.** A C++ binary, compiled with GCC/Clang (Itanium mangling), exports a function:

```cpp
// Original C++ source (not recoverable verbatim — shown for context only)
template<typename T>
T& Vector<T>::at(size_t index) { ... }

// Instantiated for two different T in this binary:
int&  Vector<int>::at(size_t);
char& Vector<char>::at(size_t);
```

The mangled symbols (Itanium ABI) are roughly `_ZN6VectorIiE2atEm` and `_ZN6VectorIcE2atEm` — the `I...E`
segment is the ABI's own template-argument-list delimiter (confirmed against the
[Itanium C++ ABI mangling specification](https://itanium-cxx-abi.github.io/cxx-abi/abi-mangling.html), see
References below), and Ghidra's `GnuDemangler` already parses it into a `DemangledFunction` whose
containing-class `DemangledDataType` has `isTemplate()==true` and a `DemangledTemplate` holding one
parameter (`int` / `char` respectively) — this parsing already happens today, per §1 above.

**Today's decompiler output (verified against the printer-side closed switch, `printer-output-quality.md`
Flaw 5):**

```c
// Two apparently-unrelated, monomorphized functions — no way to tell
// from the printed C alone that these came from "the same" template.
Vector_int * Vector_int_at(Vector_int *this, size_t index)
{
    return &this->_data[index];
}

Vector_char * Vector_char_at(Vector_char *this, size_t index)
{
    return &this->_data[index];
}
```

*(The exact flattened spelling depends on whatever name-sanitization step turns a demangled `<`/`>`/`::`
into identifier-safe text upstream of the printer — the substantive point, per Flaw 5's Observation, is
that the printer's declarator switch never sees or emits real `<...>` syntax, and the two `Datatype`
objects for `Vector<int>` and `Vector<char>` are today two entirely unrelated `TypeStruct`s with no
recorded relationship, per `varnode-ssa-type-system.md` §3.1.)*

**Desired output after REQ-LANG-1 (MUST, printer-only fix):**

```c
Vector<int> * Vector<int>::at(Vector<int> *this, size_t index)
{
    return &this->_data[index];
}

Vector<char> * Vector<char>::at(Vector<char> *this, size_t index)
{
    return &this->_data[index];
}
```

Correct template-argument syntax is now visible — a direct readability win requiring only that the
printer stop discarding the `DemangledTemplate` data GnuDemangler already computed (§1).

**Desired output after REQ-LANG-2 (SHOULD, full representation) — same printed text, but now backed by
real structure:** a test/tooling query against the type system for either `Vector<int>` or `Vector<char>`
returns "instance of generic `Vector`, argument list `[int]`" (or `[char]`), and both instances report the
same generic-definition identity — enabling, for instance, a future "jump to all instantiations of
`Vector`" navigation feature, or a cast-simplification rule that knows `Vector<int>*` and `Vector<char>*`
are "the same shape, different argument" rather than two coincidentally-similar unrelated structs. This is
the concrete difference between REQ-LANG-1 (fixes the *text*) and REQ-LANG-2 (fixes the *model* the text is
derived from) that the ticket's acceptance criteria ask this document to distinguish explicitly.

## 4. External references

1. **Itanium C++ ABI — Mangling specification**
   (<https://itanium-cxx-abi.github.io/cxx-abi/abi-mangling.html>, mirrored at
   <https://itanium-cxx-abi.github.io/cxx-abi/abi.html>) — the formal grammar GCC/Clang use to encode
   template-argument lists (the `I...E` production) into symbol names; the structural basis for what
   `GnuDemangler` already parses out (§1 above) and what REQ-LANG-1/2 build on rather than re-derive.
2. **MARX: Uncovering Class Hierarchies in C++ Programs** (Pawlowski et al., NDSS 2017,
   <https://www.ndss-symposium.org/wp-content/uploads/2017/09/ndss2017_05B-3_Pawlowski_paper.pdf>) —
   demonstrates recovering C++ class hierarchies from vtable/RTTI structure in stripped binaries at high
   precision (93.2%/88.4% reported on two large real-world targets), directly relevant to REQ-LANG-8's
   member-function-pointer/vtable-context work and to framing what *is* realistically recoverable from
   binary structure alone (vtable shape, inheritance) versus what fundamentally is not (original template
   source, per §0.1's feasibility framework) — a useful calibration point for how much C++-semantics
   recovery is realistic without external debug info.
3. **Reconstruction of Class Hierarchies for Decompilation of C++ Programs** (Fokin et al., 2010 17th
   Working Conference on Reverse Engineering, IEEE, <https://ieeexplore.ieee.org/document/5714442/>) — an
   earlier, complementary paper on the same problem (RTTI-based when present, vtable/constructor/destructor
   pattern-based fallback when absent), supporting the same "structural recovery has real limits; lean on
   RTTI/mangling when present rather than re-deriving from scratch" framing this document adopts throughout
   §0.1 and §1.

## 5. Traceability summary

| ID | Priority | One-liner | Primary source |
|---|---|---|---|
| REQ-LANG-1 | MUST | Print template-instantiation syntax from demangler metadata | 0014 Flaw 5 (opt. 2); 0012 §3.1 (opt. B) |
| REQ-LANG-2 | SHOULD | `Datatype`-level template/generic representation, argument-linked | 0014 Flaw 5 (opt. 1); 0012 §3.1 (opt. A) |
| REQ-LANG-3 | MAY | Monomorphized-function-family detection (stretch, labeled heuristic) | 0012 §3.1; §4.1 risk-class analogy |
| REQ-LANG-4 | MUST | Print `volatile` keyword from existing `Symbol` data | 0014 Flaw 4 |
| REQ-LANG-5 | MUST | Composable `const`/`volatile`/`restrict` on `Datatype` | 0012 §3.2; 0014 Flaw 4 |
| REQ-LANG-6 | MUST (non-regression) | Preserve `ProtoModel` calling-convention fidelity | Review 0008; 0012 §3.3/§3.5 (adjacent) |
| REQ-LANG-7 | SHOULD | Standalone calling-convention tag without full `FuncProto` | 0012 §3.5 (unconfirmed) |
| REQ-LANG-8 | SHOULD | Function-pointer-with-context type (member-function pointers) | 0012 §3.3 |
| REQ-LANG-9 | MUST | Recursive/self-referential and forward-declared composites | Ticket 0017 ask; verified this session |
| REQ-LANG-10 | SHOULD | Anonymous struct/union/enum parity with modern C | Ticket 0017 ask; verified this session |
| REQ-LANG-11 | MUST + SHOULD | Namespace/scoping fidelity (symbols: preserve; types: extend) | Ticket 0017 ask; verified this session |
| REQ-LANG-12 | MAY | Union-level bitfields | 0012 §3.4 |
| REQ-LANG-13 | MAY | Packed/explicit-alignment override | 0012 §3.6 |

Every MUST-priority requirement above traces to at least one 0011 flaw-analysis entry (REQ-LANG-1,
REQ-LANG-4, REQ-LANG-5) or an explicit statement in epic 0016's own summary / ticket 0017's scope
(REQ-LANG-6, REQ-LANG-9, REQ-LANG-11(a)), per the parent epic's acceptance criteria.

## 6. Explicitly out of scope (see §0)

- **Graded/confidence-scored type inference** (`varnode-ssa-type-system.md` §3.7) — an inference-quality
  concern, not a language-feature-representation concern; not a REQ-LANG item by design.
- **Re-deriving a template's original source-level definition from monomorphized machine code with no
  demangled-name or debug-info hint at all** — stated as infeasible in principle throughout §0.1/REQ-LANG-2/
  REQ-LANG-3, not merely deferred.
