# Flaws: Varnode/SSA Representation & Type System

> Produced by [`.tasks/0012`](../../.tasks/0012-task-flaws-varnode-ssa-type-system.md), a child of
> [`.tasks/0011`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md) /
> [`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md). Primary inputs:
> [`ssa-varnode-heritage.md`](../01-review/ssa-varnode-heritage.md) (ticket 0004),
> [`type-system.md`](../01-review/type-system.md) (ticket 0005); also cites
> [`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) (ticket 0003) for
> the `Architecture` god-object comparison the ticket asks for. Shared terms:
> [`../00-overview/glossary.md`](../00-overview/glossary.md).

Format per the parent epic: each finding is **Observation → Why it's a problem (concretely) →
Severity/impact → Improvement options (2-4, pros/cons, rough size) → Open questions**. Every claim below
cites a review-doc section or a `file:line` verified directly against source in this session. Severity is
High/Medium/Low, judged by correctness risk and/or refactor-blocking cost, not by how interesting the
finding is.

## Contents
1. [Datatypes to merge or simplify](#1-datatypes-to-merge-or-simplify) *(required subsection)*
2. [God-object risk: `Funcdata` and `Architecture`](#2-god-object-risk-funcdata-and-architecture)
3. [Type-system expressiveness gaps](#3-type-system-expressiveness-gaps) *(required subsection, feeds
   ticket 0017 directly)*
4. [Correctness-adjacent risk areas (SSA/merge heuristics)](#4-correctness-adjacent-risk-areas-ssamerge-heuristics)
5. [Summary table](#5-summary-table)

---

## 1. Datatypes to merge or simplify

### 1.1 The `TypePartial*` triad — three parallel "byte-range slice" types

**Observation.** `TypePartialStruct`, `TypePartialUnion`, and `TypePartialEnum` (`type.hh:681`,
`type.hh:708`, `type.hh:659`) all model the same concept — "a byte range that is provably inside a
bigger aggregate but whose exact sub-field is not (yet) resolvable" — with the same three-part shape (a
`stripped` fallback `Datatype*`, a `container`/`parent` pointer back to the aggregate, and a byte
`offset`). Only `TypePartialEnum` actually inherits from its non-partial counterpart (`TypeEnum`,
`type-system.md` §2.1, "Notable shape observations" bullet 1); `TypePartialStruct` and
`TypePartialUnion` inherit directly from `Datatype`, not `TypeStruct`/`TypeUnion`, despite being
conceptually struct/union-shaped. This is corroborated by `type-system.md` §7 item 1, which the review
already flagged for this ticket to verify — verified: `type.hh:681-733` shows each class independently
declaring its own `stripped`/`container`/`offset` triple and its own `getStripped()`/`getPartialBase()`
overrides rather than sharing a base.

**Why it's a problem (concretely).** Any future change to "how does a partial-aggregate type behave"
(e.g. fixing a `compare()` bug, adding a new query like `nearestArrayedComponentForward`) has to be made
three times, in three classes, with no shared base to enforce they stay consistent. `TypePartialUnion`
additionally duplicates real resolution logic against `TypeUnion` (see §1.3) rather than delegating to
it, compounding the risk that the two drift.

**Severity.** Medium. Not a correctness bug today (the review found no evidence of the three actually
disagreeing), but a maintainability tax that grows every time one of the three is touched, and a
concrete example of "merge candidate" the user's stated goal asks for by name.

**Improvement options.**
- **A — Introduce a shared `TypePartialBase` mixin/abstract class** carrying `stripped`/`container`/
  `offset` and the common `getStripped()`/`getPartialBase()` logic; `TypePartialStruct`,
  `TypePartialUnion`, and `TypePartialEnum` (dropping its separate `TypeEnum` inheritance, or using
  multiple inheritance) derive from it. *Pros*: single implementation of the shared 80%, still lets each
  subclass differ where it genuinely must (union field resolution vs. struct field lookup vs. enum-value
  lookup). *Cons*: touches every `dynamic_cast`/type-switch in the codebase that currently
  distinguishes these three by concrete class; C++ multiple inheritance interacts awkwardly with
  `TypePartialEnum`'s existing `TypeEnum` base. *Size*: Medium.
- **B — Make `TypePartialStruct`/`TypePartialUnion` genuinely inherit from `TypeStruct`/`TypeUnion`**
  (matching the `TypePartialEnum` precedent) rather than from `Datatype` directly, so field-lookup code
  is inherited instead of re-implemented. *Pros*: closer structural honesty ("a partial struct is-a
  struct-like thing"), reuses more code than option A's mixin. *Cons*: `TypeStruct`/`TypeUnion`'s own
  invariants (full field vector, `assignFieldOffsets`) may not hold for a *partial* view, so this risks
  spreading "is this the whole type or a slice" conditionals throughout the parent classes instead of
  containing them. *Size*: Large.
- **C — Leave the three classes as-is, but extract only the duplicated `getStripped()`-family logic**
  into free functions/helpers all three call. *Pros*: small, low-risk, immediately removes the worst
  duplication without a hierarchy change. *Cons*: doesn't fix the conceptual "three near-identical
  classes" problem, just its worst symptom. *Size*: Small.

**Open questions.** Does the Ghidra-side `DataTypeManager` model partial/sliced types at all, or is this
purely a decompiler-internal propagation concept with no Java-side analogue to keep in sync (relevant to
whichever option is chosen, since option B's `TypeStruct` base carries encode/decode logic that talks to
that bridge, §1.8)?

### 1.2 `compare()` vs. `compareDependency()` — two orderings, currently identical at the base

**Observation.** Every `Datatype` subclass independently overrides both `compare(op, level)` (orders
types for *propagation* — "bigger/more specific types come earlier", `type-system.md` §3) and
`compareDependency(op)` (orders types for *storage in `TypeFactory::tree`*, must not consult `id`,
`type-system.md` §3). Per `type-system.md` §7 item 2, the base-class defaults (`type.cc:229`,
`type.cc:244`) are "currently textually identical". Verified against `type.hh:293-294`: both are
declared as independent `virtual` functions with no default relationship enforced between them (a
subclass can override one without the other, and nothing checks they stay behaviorally consistent for
the subset of comparisons where they should agree).

**Why it's a problem (concretely).** ~14 concrete `Datatype` subclasses (`type-system.md` §2.1 class
list) each carry two hand-written comparators. Because the base defaults are identical, most subclasses
that override one plausibly *should* override the other identically too, but the two virtuals give no
static/structural signal that they've fallen out of sync — a subtle propagation-vs-interning
inconsistency (e.g. two types compare distinct for storage but equal for propagation ordering, or vice
versa) would be a silent correctness bug in type propagation (`ActionInferTypes`, `type-system.md` §5.1)
that is not obviously connected to its root cause.

**Severity.** Low-Medium. No evidence found of current divergence; this is a latent-bug-class risk from
the design shape, not a demonstrated bug.

**Improvement options.**
- **A — Collapse to one `compare(op, level)` with `level` distinguishing the two use sites** (e.g. a
  sentinel `level` value meaning "for interning, ignore `id`"), removing `compareDependency` entirely.
  *Pros*: one comparator per subclass instead of two; the `type-system.md` §3 note that the *only* real
  difference is "must not look at `id`" suggests this is achievable. *Cons*: `compare()`'s existing
  `level` parameter already has propagation-specific meaning (recursion-depth limiting per
  `type.cc:229`'s doc); overloading it risks conflating two unrelated concerns into one signature, and
  every override site must be re-audited to confirm no subclass's two comparators silently differ today
  in a way collapsing would break. *Size*: Medium.
- **B — Keep both, but add a debug-only assertion/test that for every registered concrete subclass,
  `compare()` and `compareDependency()` agree on the subset of inputs where they're supposed to** (i.e.,
  make the invariant checkable instead of implicit). *Pros*: small, non-invasive, catches drift without
  a hierarchy change; fits the existing `testtypes.cc` unit-test investment noted as the type system's
  strong point (`00-index.md` "Test coverage" table). *Cons*: doesn't reduce the duplication itself,
  only guards against it silently worsening. *Size*: Small.
- **C — Leave as-is**, given no demonstrated divergence. *Pros*: zero risk/cost now. *Cons*: the
  duplication and the risk described above persist into whatever refactor architecture is chosen next.
  *Size*: None.

**Open questions.** Is there a documented (even historical/commit-message) reason the two were split in
the first place, beyond "storage lookup must ignore `id`"? If that's genuinely the only difference,
option A is likely correct; if there's a second, undocumented reason, it needs to surface before
collapsing them.

### 1.3 Union/struct-field resolution logic duplicated across `TypeUnion`, `TypePointer`, `TypePartialUnion`

**Observation.** `resolveInFlow`/`findResolve`/`findCompatibleResolve`/`resolveTruncation` are each
independently implemented on `TypeUnion` (`type.cc:2617,2635,2645,2725`), `TypePointer`
(`type.cc:1361,1382,1394`), and `TypePartialUnion` (`type.cc:3036,3076`, plus its own
`findCompatibleResolve`) — `type-system.md` §7 item 3. All three ultimately funnel into the same
`Funcdata::getUnionField`/`ScoreUnionFields` machinery (`type-system.md` §6), but each independently
re-derives its own `baseType`/pointer-vs-value handling before getting there.

**Why it's a problem (concretely).** This is the same "three near-identical entry points into one
underlying algorithm" pattern as §1.1, but for *behavior* (resolution correctness) rather than mostly
data layout — a higher-stakes duplication, since `ScoreUnionFields` is already an admittedly heuristic,
capped-effort search (`maxPasses=6`, `threshold=256`, `maxTrials=1024`, `unionresolve.cc:131-133`,
`type-system.md` §6) whose correctness depends on all three call sites feeding it consistent
`baseType`/offset context. A bug in how e.g. `TypePointer::resolveInFlow` derives its `baseType` for a
pointer-to-union, versus how `TypePartialUnion` derives its own, could produce a resolution that's
internally consistent within one call path but disagrees with another path resolving logically the same
union field from a different type wrapper.

**Severity.** Medium-High. This sits directly upstream of union-field-choice correctness (§4.3 below),
which the type-system review already flags as heuristic and capped, not exhaustive.

**Improvement options.**
- **A — Extract a single free function `resolveUnionEntry(baseType, offset, op, slot, ...)`** that all
  three `resolveInFlow` overrides call after each does only its own type-specific unwrapping (pointer
  dereference, partial-slice offset math). *Pros*: keeps each class's legitimately-different "how do I
  get to a `baseType`+offset" step, unifies everything after that; moderate risk since the shared core
  logic is centralized once instead of three times. *Cons*: still requires each class to get its own
  unwrapping step right; doesn't remove all duplication. *Size*: Medium.
- **B — Route `TypePointer`-to-union and `TypePartialUnion` resolution entirely through
  `TypeUnion::resolveInFlow`** by first normalizing to a `(TypeUnion*, offset)` pair, making `TypeUnion`
  the single owner of resolution logic and the other two thin adapters. *Pros*: strongest deduplication,
  matches the "TypeUnion is authoritative" mental model implied by `needsResolution()`
  (`type-system.md` §6). *Cons*: larger change; `TypePartialUnion`'s "slice" semantics (a byte-range,
  not necessarily union-aligned) may not normalize cleanly to a plain union+offset in all cases,
  requiring careful edge-case audit. *Size*: Large.
- **C — Add cross-path consistency tests** (feed the same logical union access through the
  `TypePointer`, `TypeUnion`, and `TypePartialUnion` entry points and assert identical resolutions)
  without restructuring the code. *Pros*: cheap, directly targets the correctness risk described above.
  *Cons*: doesn't reduce maintenance cost, only catches divergence after the fact. *Size*: Small.

**Open questions.** Does `ScoreUnionFields`'s per-`(op,slot)` and address-based caching
(`Funcdata::getUnionField`/`getAddressBasedUnionField`, `type-system.md` §6) already implicitly paper
over inconsistencies between the three entry points by caching the *first* resolution and reusing it —
i.e., is this duplication risk actually being masked today by caching rather than being genuinely absent?
This needs a targeted read of `unionresolve.cc`'s cache-key derivation before committing to option B.

### 1.4 Two independently-coded "ephemeral type, here's the real one" mechanisms

**Observation.** `TypePointerRel` marks itself `is_ptrrel` and caches a `has_stripped`-flagged plain
`TypePointer` via `markEphemeral()` (`type.hh:1037-1054`) so that formal declarations print/store the
plain pointer instead of the context-carrying relative one (`type-system.md` §2.1). Independently, each
of `TypePartialStruct`/`TypePartialUnion`/`TypePartialEnum` implements its *own* `stripped` field and
`getStripped()` override (`type.hh:661,683,711`, `type-system.md` §7 item 6). Both mechanisms answer the
same question — "this type instance exists only to carry extra propagation context; here is the type to
actually use in formal output" — but neither shares implementation or storage with the other.

**Why it's a problem (concretely).** A future consumer of "does this `Datatype` have a stripped/formal
form" (e.g. the printer, or a serializer) has to know about *two* unrelated flag/field conventions
(`is_ptrrel`+`has_stripped` vs. each partial type's own `stripped` member) rather than one, and any new
"ephemeral, propagation-only" type added later (a plausible outcome of closing the expressiveness gaps
in §3) has no obvious single pattern to follow.

**Severity.** Low-Medium — a design-consistency issue more than a demonstrated bug, but directly
relevant to how any new type kinds from §3's options should be built.

**Improvement options.**
- **A — Generalize `hasStripped()`/`getStripped()` into the single sanctioned mechanism**: give
  `Datatype` itself a protected `stripped` field and let `TypePointerRel` and the `TypePartial*` family
  all populate it through one shared code path (constructor helper or mixin), rather than each managing
  its own flag/field. *Pros*: one pattern, one invalidation story, easy to document and reuse for future
  ephemeral types. *Cons*: `Datatype` is already a wide base class (§3 discussion of expressiveness
  gaps will likely want to add fields too) — growing it further has its own cost; touches every
  subclass's constructor. *Size*: Medium.
- **B — Leave both mechanisms but document the convention explicitly** ("ephemeral types must expose
  `getStripped()`; storage is per-class") so future additions at least follow a known contract even
  without code sharing. *Pros*: near-zero cost. *Cons*: doesn't reduce existing duplication, relies on
  future authors reading docs rather than the type system enforcing the pattern. *Size*: Small.

**Open questions.** If option A from §1.1 (a `TypePartialBase` mixin) is adopted, does folding this
stripped-form logic into the same mixin make sense, or is the union of "partial-slice" concerns and
"ephemeral-stripped-form" concerns too unrelated to combine into one base class?

### 1.5 `TypeSpacebase` — a pseudo-struct standing in for address-space-relative symbol lookup

**Observation.** `TypeSpacebase` (`type.hh:816`) deliberately "treats a specific `AddrSpace` as a
structure that will get indexed into" (`type.hh:813`, `type-system.md` §7 item 5) purely to reuse
struct-like field/offset lookup machinery for stack/global symbol resolution. This is not cosmetic:
`propagateSpacebaseRef` (`coreaction.cc:5638`, `type-system.md` §8) walks a `TypeSpacebase` as if it
were an ordinary aggregate as part of real type-propagation logic.

**Why it's a problem (concretely).** This conflates two different concerns inside one `Datatype`
subclass: "the type of a value" (what every other `Datatype` subclass models) and "an index into a
memory region" (an addressing/symbol-table concern). Any change to how `Datatype::getSubType`/
`nearestArrayedComponentForward` etc. behave for genuine aggregates risks unintentionally changing
stack/global symbol-lookup behavior too, since both paths share the same virtual dispatch surface.

**Severity.** Low. No correctness issue identified — the review doc frames this as a design-smell worth
questioning, not a bug — but it's a real complication for anyone trying to reason cleanly about
`Datatype`'s contract ("is a `Datatype*` always a value's type, or sometimes an addressing helper?").

**Improvement options.**
- **A — Give stack/global symbol lookup its own dedicated indexing structure**, independent of the
  `Datatype` hierarchy, and have `propagateSpacebaseRef` and its callers use that instead of walking a
  `Datatype` subtype chain. *Pros*: restores "`Datatype` = value type" as an exceptionless invariant,
  which simplifies every other piece of code that pattern-matches on `Datatype` subclasses expecting
  only genuine value types. *Cons*: `TypeSpacebase` reuse of `getSubType`/offset-walking is exactly what
  makes the current code short; a dedicated structure duplicates that walk logic rather than borrowing
  it. *Size*: Large — touches `Heritage`/`ScopeLocal` (`ssa-varnode-heritage.md` §1.3) as well as
  `ActionInferTypes`.
- **B — Keep the reuse, but stop routing it through the public `Datatype` virtual interface** (e.g. make
  `TypeSpacebase`'s walk a private helper that happens to share code with `TypeStruct`'s offset-walking
  utility function, rather than making `TypeSpacebase` itself a `Datatype` subclass reachable through
  generic `Datatype*` code paths). *Pros*: keeps the code-sharing benefit, removes the conflation from
  the public type hierarchy. *Cons*: still a nontrivial refactor to identify every place `TypeSpacebase`
  currently participates in `Datatype`-generic logic today. *Size*: Medium.
- **C — Leave as-is and document the dual role explicitly** at the class declaration. *Pros*: zero risk.
  *Cons*: the conflation and its downstream risk (shared virtual dispatch surface) remain. *Size*: None.

**Open questions.** How deep does `TypeSpacebase` reuse actually go — is it only `getSubType`/offset
math, or does propagation-comparison logic (`compare`/`compareDependency`, §1.2) also run over
`TypeSpacebase` instances in practice? This affects whether option A or B is more tractable.

### 1.6 Three overlapping "what kind of type is this" enums

**Observation.** `type_class` (`type.hh:132-142` — storage-class classification: general/float/pointer/
vector/hidden-return plus 4 architecture-specific slots), `type_metatype` (`type.hh:80-100` — the
size-disregarding coarse kind used for identity/dispatch), and `sub_metatype` (`type.hh:104-129` — a
finer propagation-ordering-only refinement of `type_metatype`) are three separate small enums, each
answering "what kind of type is this" at a different granularity and for a different purpose
(`type-system.md` §7 item 8: parameter-storage-class assignment vs. propagation ordering vs. structural
identity, respectively).

**Why it's a problem (concretely).** A newcomer asking "how do I check what kind of type this is" has
three different, non-interchangeable answers depending on which subsystem they're touching (parameter
storage, `fspec.hh`/ticket 0008's territory; type propagation, `ActionInferTypes`; type identity/
interning, `TypeFactory`), with no single source of truth and no automatic derivation of one from
another shown in the type-system review — `Datatype::base2sub[18]` (`type.hh:178`) does map
`type_metatype → sub_metatype` as a *default*, but `type_class` is derived independently elsewhere
(signature/calling-convention code per ticket 0008's scope, not reviewed in depth here).

**Severity.** Low. This is architectural clarity debt, not a demonstrated bug — the type-system review
explicitly frames it as "not obviously wrong ... worth a closer look" rather than a flaw.

**Improvement options.**
- **A — Document the three explicitly as a deliberate layered design** (coarse identity →
  propagation-order refinement → storage-class classification) with a single comparison table, and add
  a named conversion function for each pairwise derivation that exists today only ad hoc
  (`base2sub` already does one direction). *Pros*: very low cost, directly addresses the "newcomer
  confusion" problem without touching working code. *Cons*: doesn't reduce actual duplication if any
  exists. *Size*: Small (documentation-only).
- **B — Audit whether `type_class` could be derived entirely from `type_metatype` instead of maintained
  independently**, collapsing to two enums instead of three if the derivation turns out to be total and
  mechanical. *Pros*: real simplification if the audit confirms redundancy. *Cons*: `type_class`'s
  4 architecture-specific slots (`type.hh:132-142`) suggest it encodes target-ABI information
  `type_metatype` structurally cannot — the audit may well conclude the three are *not* redundant, in
  which case this option is a dead end investigated for no gain. *Size*: Medium (mostly investigation,
  small if it turns out mergeable).

**Open questions.** This finding's disposition depends on ticket 0008's signature/calling-convention
flaw analysis (sibling ticket 0013's territory per the epic's task split) — `type_class` is primarily a
parameter-storage concern; a joint read with that ticket's findings before committing to option B is
recommended.

### 1.7 `Varnode`/`HighVariable`/`Symbol` — duplicated `type` field and mirrored lock/volatility flags

**Observation.** All three classes independently store a notion of "this thing's data-type" and "this
thing's lock/volatility state", verified directly against the headers:

| Concept | `Varnode` | `HighVariable` | `Symbol` |
|---|---|---|---|
| Datatype pointer | `type` (`varnode.hh:147`) | `type` (`variable.hh:135`, derived+cached, dirty-bit `typedirty`) | `type` (`database.hh:239`) |
| Type-lock | `typelock` flag bit `varnode_flags` (`varnode.hh:90`), tested `isTypeLock()` | derived from member `Varnode` flags via `updateFlags()`, tested `isTypeLock()` (`variable.hh:220`) | `flags` bit, comment "Varnode-like properties... only typelock,namelock,..." (`database.hh:243-245`), tested `isTypeLocked()` (`database.hh:257`) |
| Name-lock | `namelock` (`varnode.hh:91`) | derived, `isNameLock()` (`variable.hh:221`) | `flags` bit, `isNameLocked()` (`database.hh:258`) |
| Volatile | `volatil` (`varnode.hh:93`) | not independently tracked (inherited via `Varnode` flags union, `updateFlags()`) | `flags` bit, `isVolatile()` (`database.hh:260`) |
| Address-tied / persistent | `addrtied`/`persist` (`varnode.hh:97-98`) | derived, `isAddrTied()`/`isPersist()` (`variable.hh:216-217`) | inherited from `Scope` per `database.hh:244` comment, not stored independently |

`ssa-varnode-heritage.md` §1.3 and §4.3 already describe `HighVariable` as caching "several things that
are really properties of its constituent `Varnode`s" behind nine hand-maintained dirty bits
(`variable.hh:119-131`). `Symbol`'s `flags` field comment (`database.hh:243-245`) is explicit that it
duplicates a *subset* of `Varnode`'s flag bits by design ("only typelock, namelock, readonly, externref"
— i.e. someone already had to hand-pick which `Varnode` flags a `Symbol` needs to mirror). This is
exactly the "real duplication between `Varnode` metadata and `HighVariable`/`Symbol` metadata" the
ticket's Analysis Focus item 1 asks about.

**Why it's a problem (concretely).** Three independent copies of "is this locked" exist for what is,
per `Varnode::copySymbol()` (`varnode.cc:517-529`, `ssa-varnode-heritage.md` §1.3 context), supposed to
be *one* logical fact propagated between the three representations at specific transfer points
(symbol-linking, `HighVariable` construction). Because propagation is by explicit copy rather than a
single source of truth, a code path that sets a lock on one representation without remembering to
propagate it to the other two produces a real, silent inconsistency — e.g. a `Symbol` marked
type-locked in the database whose `Varnode`s haven't yet had `setSymbolProperties()`
(`varnode.cc:434-448`) called on them would report *unlocked* if queried at the `Varnode` level, locked
at the `Symbol` level. This is squarely in the same risk class `ssa-varnode-heritage.md` §4.3 already
flags for `HighVariable`'s nine dirty bits ("a missed dirty-mark is a latent stale-cache bug class, not
something the type system catches").

**Severity.** Medium-High. Type-lock/name-lock correctness is load-bearing for type propagation
(`type-system.md` §5.2: "a lock is the system's only notion of confidence... a type is either freely
overwritable or completely frozen") — a silent lock-state mismatch between representations is exactly
the kind of "silently produce wrong (not just ugly) output" risk item 3 of the ticket's Analysis Focus
asks to look for.

**Severity.** Medium-High — see rationale above; downgraded from High only because the review docs found
no *demonstrated* instance of the three disagreeing in practice, only the structural risk that they
could.

**Improvement options.**
- **A — Make `Symbol` the single source of truth for lock state, and have `Varnode`/`HighVariable`
  query through a pointer/reference rather than storing their own copies.** *Pros*: eliminates the
  propagation-timing risk entirely for locked `Varnode`s (the common, most consequential case — user- or
  database-declared types). *Cons*: not every `Varnode` has a `Symbol` (temporaries, anonymous
  intermediate values, `type-system.md` §5.1's `getLocalType()` inference targets) — the lock concept
  has to remain independently representable for unattached `Varnode`s regardless, so this only removes
  duplication for the subset that *are* symbol-backed, not the general case. *Size*: Large — touches the
  hot path of type propagation (`propagateTypeEdge`, `coreaction.cc:5445`) and `HighVariable`
  construction.
- **B — Add an explicit, checked "sync" operation** (rather than ad hoc call sites like
  `setSymbolProperties`/`copySymbol`) that is the *only* sanctioned way to move lock/type state between
  the three levels, with debug-mode assertions that flag/type state actually agrees after every merge
  (`Merge::merge()`, `ssa-varnode-heritage.md` §4.2) and symbol-link
  (`Funcdata::linkSymbol`, `funcdata.hh:435`) operation. *Pros*: much smaller than option A, directly
  targets "missed propagation" as a class of bug via defensive checking rather than eliminating the
  duplication. *Cons*: doesn't reduce the duplication itself, only catches drift after the fact (same
  trade-off as §1.2 option B and §1.3 option C — a recurring cheap-mitigation pattern across this
  document). *Size*: Small-Medium.
- **C — Document the intended propagation points precisely** (which call sites are responsible for
  syncing which flags, in what order) as an explicit invariant, without code changes. *Pros*: minimal
  cost; makes the *existing* convention-based discipline legible instead of implicit
  (`ssa-varnode-heritage.md` §1.5 already names this general pattern — "managed almost entirely by
  convention" — as a top-level flaw across the whole `Varnode`/`HighVariable`/`Symbol` object graph).
  *Cons*: doesn't prevent future violations, only documents the current ones. *Size*: Small.

**Open questions.** Is there a reason `HighVariable` can't simply *always* delegate flag/type queries to
its member `Varnode`s live (no caching, no dirty bits) rather than caching-with-invalidation? The
`ssa-varnode-heritage.md` §4.3 framing suggests performance was the original motivation (avoiding
recomputation across, e.g., repeated `Cover` intersection tests) — worth a profiling-informed answer
before committing to any option that removes caching (option A partially does).

### 1.8 Two independently-maintained type systems: `TypeFactory` vs. Ghidra's `DataTypeManager`

**Observation.** The decompiler's `TypeFactory`/`TypeFactoryGhidra` and Ghidra's Java-side
`DataTypeManager` are, per `type-system.md` §4, "two **independently-maintained, DB-backed type
systems** that must be kept in sync for the duration of one decompile," bridged one-directionally and
lazily: a cache miss in `TypeFactoryGhidra::findById()` (`typegrp_ghidra.cc:20`) triggers a
`COMMAND_GETDATATYPE` round-trip, and the result is cached **indefinitely, across every subsequently
decompiled function**, with the only invalidation being the Java side's coarse `FlushNative` command
that wipes the *entire* non-core cache (`type-system.md` §4.1, `ghidra_process.cc:255-266`).

**Why it's a problem (concretely).** This isn't intra-process class duplication like §1.1-1.7 — it's
duplication *across a process boundary*, and the two copies can genuinely diverge: `type-system.md` §4.1
lists concrete failure modes — stale-type risk if the Java side edits a structure without a full
`FlushNative`, `findAdd`'s "Trying to alter definition of type" hard-abort (`type.cc:4067`) when a
same-name+id type reappears with a different structural definition, and an informal (not enforced)
partition between Ghidra-sourced and locally-synthesized ids (`type.cc:812`, high-bit convention).
Because there is genuinely one C++-side interning tree (`TypeFactory::tree`/`nametree`,
`type-system.md` §3) standing in for what should be a live mirror of the Java-side source of truth, this
is the type system's largest-blast-radius "merge/reconcile" candidate even though the fix is
necessarily cross-language (Java `DataTypeManager` is out of this ticket's file-list scope but not out
of the *problem's* scope).

**Severity.** High. Unlike the intra-process items above, this one has a demonstrated failure mode
(`findAdd`'s abort) and a documented gap (no per-type invalidation) rather than only a structural risk —
`type-system.md` explicitly calls it out as a correctness trap in §3 ("this is a real correctness trap,
see §8").

**Improvement options.**
- **A — Add per-type invalidation to the Java→native bridge** (a `COMMAND_INVALIDATETYPE(name, id)`
  companion to the existing coarse `FlushNative`), fired whenever Ghidra's `DataTypeManager` commits an
  edit to a type that has a live decompiler-side cache entry. *Pros*: directly closes the stale-cache
  gap without the blunt "wipe everything" cost of `FlushNative`; incremental cost proportional to actual
  edits. *Cons*: requires Java-side instrumentation on every `DataTypeManager` mutation path (out of
  this ticket's C++ file-list scope; likely intersects ticket 0010's Java-integration findings) and
  careful handling of types already embedded in in-flight decompiles. *Size*: Large — spans both
  languages and the wire protocol.
- **B — Make the redefinition conflict (`type.cc:4067`) recoverable instead of a hard `LowlevelError`
  abort**: on a detected same-id structural mismatch, evict and re-fetch rather than aborting the whole
  decompile. *Pros*: much smaller than option A, turns a hard failure into a soft retry; doesn't require
  Java-side protocol changes since the *detection* already exists, only the *response* changes.
  *Cons*: doesn't fix the root staleness, just makes its symptom less disruptive; a type that changes
  mid-decompile (not just between decompiles) could still produce a result inconsistent with either the
  old or new definition depending on timing. *Size*: Medium.
- **C — Document the one-directional, session-cached-until-flush model explicitly as an accepted
  trade-off**, and rely on the front end always calling `FlushNative` conservatively (e.g. on every
  program-open / after any bulk retype operation) rather than trying to make the native side smarter.
  *Pros*: zero code change; may already reflect actual Java-side practice (unverified — flagged as an
  open question in `type-system.md` §4.1 itself: "Whether the Java side *always* flushes on every
  relevant edit is ... a follow-up question for ticket 0010's Java-integration review"). *Cons*: pushes
  all correctness responsibility onto Java-side discipline with no native-side backstop; the "one shot,
  wipes everything" granularity is also a performance cost on large programs with frequent edits.
  *Size*: None (process/documentation only).

**Open questions.** This is explicitly a joint concern with ticket 0010 (Java-integration review) — does
the Java front end in fact call `FlushNative` on every `DataTypeManager` edit today? That answer
determines whether this is a live bug (option A/B needed urgently) or a latent risk under an
undocumented-but-currently-sufficient discipline (option C might be acceptable short-term). Recommend
cross-referencing with whichever ticket in epic 0011 covers Java-integration flaws.

---

## 2. God-object risk: `Funcdata` and `Architecture`

### 2.1 `Funcdata`

**Observation.** `Funcdata` (`funcdata.hh:56-630`) is, by its own class comment, the per-function
integration point owning `VarnodeBank`, `PcodeOpBank`, raw and structured `BlockGraph`s, `Heritage`,
`Merge`, `ScopeLocal`, `FuncProto`, call-site specs, jump-table state, in-progress return recovery, and
per-function overrides — the full member table is reproduced in `ssa-varnode-heritage.md` §3.1
(`funcdata.hh:75-101`). Its public API spans ~70 `PcodeOp`-manipulation methods, a dozen-plus
search/traversal overloads, `Varnode` creation, block-structuring mutation, symbol linking, and
union-field resolution (`ssa-varnode-heritage.md` §3.1, `funcdata.hh:285-594`), implemented across
~6300 lines in four files that all define `Funcdata::` methods (`funcdata.cc`, `funcdata_op.cc`,
`funcdata_varnode.cc`, `funcdata_block.cc` — `ssa-varnode-heritage.md` §3.3). It is declared `friend` of
both `Varnode` (`varnode.hh:160`) and `PcodeOp` (`op.hh:65`), reaching into their private setters
directly rather than through a narrow public contract (`ssa-varnode-heritage.md` §3.2-3.3).

**Why it's a problem (concretely).** Per `ssa-varnode-heritage.md` §3.3: every subsystem that transforms
a function (`Rule`s, block structuring, signature recovery, the printer) is coupled to the *concrete*
`Funcdata` type with no narrower interface available (e.g. "just the op-mutation API"), so testing or
reusing any one concern in isolation means dragging in the whole class — directly explaining
`00-index.md`'s observation that SSA/heritage construction has "no direct unit coverage at all" and the
action/rule engine has "zero unit-level tests" (`00-index.md` "Test coverage" table): there is no
sub-`Funcdata`-sized seam to test against. The `friend`-based access additionally means the `Varnode`/
`PcodeOp` class boundary is syntactic, not semantic (`ssa-varnode-heritage.md` §3.3) — any invariant
those classes are supposed to protect can be violated by `Funcdata` code without the compiler noticing a
contract breach.

**Severity.** High for refactorability (this *is* "the central architectural fact a refactor has to
reckon with," `ssa-varnode-heritage.md` §3.3's own words), Medium for correctness directly (the coupling
is a testability/maintainability cost more than a demonstrated bug source, though untested code is
inherently higher correctness risk by omission).

**Is the coupling essential or historical?** `ssa-varnode-heritage.md` §3.2 gives the honest
justification directly from source comments (`varnode.hh:76-77`): "nearly every transformation the
decompiler performs ... needs to *simultaneously* touch a `PcodeOp`, its `Varnode` operands, the
`VarnodeBank`/`PcodeOpBank` indexes those live in, and often the basic-block graph, so keeping them all
reachable from one object avoids threading five separate references through every helper function
call." This is a real data-dependency argument, not pure accident — but the review also notes concrete
*accretion* evidence that isn't essential: the `#ifdef OPACTION_DEBUG` instrumentation block living on
the production object (`funcdata.hh:597-629`, ~15 fields/methods, `ssa-varnode-heritage.md` §3.3) and
nested helper classes (`PcodeEmitFd`, `CloneBlockOps`, `AncestorRealistic`) declared in the same header
rather than owning their own scope are historical accretion, not fundamental to the "many things need
simultaneous access" argument.

**Improvement options.**
- **A — Extract narrow read/mutate interfaces** (e.g. an `IVarnodeQuery`/`IOpMutator`-style abstract
  interface covering only the `beginLoc`/`endLoc`/`op*` methods a given `Rule` actually needs) that
  `Funcdata` implements, and have `Action`/`Rule` (ticket 0006's territory), block structuring (0007),
  and the printer (0009) depend on the narrow interface instead of the concrete class. *Pros*: directly
  enables the missing unit-test coverage `00-index.md` flags, without requiring the "everything
  reachable from one object" access pattern to be abandoned internally — `Funcdata` still exists,
  concretely, as the one real implementation. *Cons*: C++ virtual-dispatch overhead on what is today a
  hot-path concrete-type call in a tight per-opcode loop (`ActionPool`, `action-rule-engine.md`); the
  interfaces have to be designed carefully to avoid becoming "narrow" in name only if most `Rule`s
  genuinely need broad access anyway (the review notes the coupling is largely real, not accidental).
  *Size*: Large.
- **B — Peel off the debug/instrumentation state first** (`OPACTION_DEBUG` block,
  `funcdata.hh:597-629`) into a separate optional companion object, and move the nested helper classes
  (`PcodeEmitFd`, `CloneBlockOps`, `AncestorRealistic`) into their own headers/translation units.
  *Pros*: low-risk, addresses the *non*-essential accretion the review specifically identifies as
  historical rather than load-bearing, immediately shrinks the "what does `Funcdata` do" surface a
  newcomer has to read without touching the essential coupling. *Cons*: does nothing for the deeper
  testability problem (option A's target); a purely cosmetic win by itself. *Size*: Small.
- **C — Accept the coupling as essential and instead invest in integration-level test coverage**
  (build the missing SSA/heritage and Action/Rule test suites `00-index.md` flags as gaps, using
  `Funcdata` directly rather than trying to narrow its interface) — i.e., solve the *symptom*
  (untested code) without solving the *cause* (god-object shape). *Pros*: fastest path to closing the
  test-coverage gap that's the most concrete, cited cost of the current design; doesn't require
  redesigning a class whose coupling the review found to be substantially justified. *Cons*: doesn't
  reduce the long-term maintenance/reuse cost of the god-object shape; large test-authoring effort in
  its own right given `00-index.md`'s note that heritage/Action-Rule have *zero* current coverage to
  build from. *Size*: Large (test-authoring effort, not `Funcdata` refactor effort).

**Open questions.** Given the explicit note in `ssa-varnode-heritage.md` §3.3 that this is "the central
architectural fact a refactor has to reckon with," and that the eventual `architecture-options.md`
(ticket 0022) is where a staged decomposition plan would actually be designed — should this ticket's
recommendation be narrower than "pick A/B/C" and instead be "any architecture option in 0022 must
include an explicit `Funcdata`-decomposition story, because option A (the only one that structurally
fixes the coupling) is large enough to need its own dedicated design pass"?

### 2.2 `Architecture`

**Observation.** `Architecture` (`architecture.hh:165-379`) is independently identified as "the
God-object of the engine" by the architecture-pipeline review (`architecture-pipeline-overview.md` §2,
verbatim), owning as raw member pointers: `TypeFactory *types`, `Database *symboltab`, the `ProtoModel`
registry (`map<string,ProtoModel*> protoModels` plus three more `ProtoModel*` fields), `Translate
*translate`/`LoadImage *loader`, `PcodeInjectLibrary*`/`CommentDatabase*`/`StringManager*`/
`ConstantPool*`, `PrintLanguage *print` plus a `printlist`, `OptionDatabase *options`, a `UserOpManage`,
`vector<Rule*> extra_pool_rules`, and the entire `ActionDatabase allacts` (the whole Action/Rule engine)
— `architecture-pipeline-overview.md` §2 bullet list, verified against `architecture.hh:165-379`'s field
declarations. It is abstract, with a battery of `build*` factory methods (`buildDatabase`,
`buildTranslator`, `buildLoader`, `buildTypegrp`, etc., `architecture.hh:263-347`) invoked in a fixed
order from `Architecture::init`.

**Why it's a problem (concretely).** Every one of the ~10 major subsystems reviewed across epic 0002
(type system, symbol database, calling conventions, printer, options, rule engine, ...) is reachable —
and in Ghidra-hosted operation, RPC-backed — through this one object. Unlike `Funcdata` (per-function,
recreated per decompile), `Architecture` is per-program and longer-lived, so its god-object shape
compounds the `TypeFactory`-vs-`DataTypeManager` staleness risk from §1.8: `Architecture` is exactly the
object whose long lifetime is *why* `TypeFactoryGhidra`'s cache persists "indefinitely, across every
function subsequently decompiled by that same `libdecomp` process" (`type-system.md` §4.1). The review
also flags a concrete extensibility inconsistency living on this same object:
`ArchitectureGhidra` — the subclass Ghidra itself actually uses — is constructed directly
(`new ArchitectureGhidra(...)`, `ghidra_process.cc:181`) rather than through the
`ArchitectureCapability::findCapability` self-registration mechanism the rest of the class's design
implies is "how new architectures plug in" (`architecture-pipeline-overview.md` §2, "Asymmetry worth
noting").

**Severity.** Medium-High. Primarily a refactorability/testability concern (mirroring `Funcdata`'s), but
with one demonstrated inconsistency (the capability-registration bypass) that indicates the class's own
documented extension story doesn't match how it's actually used in the one configuration (Ghidra-hosted)
that matters for this refactor's audience.

**Improvement options.**
- **A — Split `Architecture` into a small immutable "session context" plus a set of independently
  constructible service objects** (type system, symbol database, calling-convention registry, print
  engine, action/rule engine) that `Architecture` composes rather than *is*. *Pros*: the biggest
  structural win — makes each subsystem separately testable/mockable (directly relevant to the same
  test-coverage gaps `Funcdata`'s options target) and clarifies which parts are genuinely
  program-global vs. incidentally attached. *Cons*: very large — nearly every `build*` factory method
  and every call site that reaches a subsystem via `Architecture::` (a very high fan-in point per the
  review) needs updating; `ArchitectureGhidra`'s RPC-backed overrides for each `build*` (`ghidra_arch.hh:
  82-169`) would need re-plumbing too. *Size*: Large.
- **B — Fix the capability-registration asymmetry first, as a smaller independent step**: either route
  `ArchitectureGhidra` construction through `ArchitectureCapability::findCapability` like the file-based
  architectures (`BfdArchitectureCapability`, `RawBinaryArchitectureCapability`,
  `XmlArchitectureCapability`), or explicitly document that Ghidra-hosted operation is a deliberately
  separate, non-capability-based construction path and the capability mechanism only governs the
  console tool. *Pros*: small, resolves a concrete documented inconsistency without touching the god-
  object shape itself; clarifies the extension story for whoever eventually works on ticket 0006's
  Action/Rule extensibility findings (a related "extension model: inconsistent across the codebase"
  theme flagged in `00-index.md`). *Cons*: doesn't address the god-object coupling itself, only one
  symptom of it. *Size*: Small.
- **C — Treat `Architecture` and `Funcdata` as one combined decomposition problem** rather than two
  separate ones, since `Funcdata` already holds a back-reference into program-global state via its
  `functionSymbol`/`Architecture` reach-through, and a staged refactor plan (ticket 0022) may get more
  leverage from designing both objects' seams together (e.g. a shared "service locator" pattern both
  could use) than from two independent decompositions that might disagree on interface shape.
  *Pros*: avoids designing two incompatible decomposition strategies for what are, in practice, the two
  most central objects in the whole pipeline. *Cons*: couples this finding's resolution to `Funcdata`'s
  (§2.1), delaying either until both are designed together; larger up-front design cost before any
  code change lands. *Size*: Large (primarily a planning/sequencing recommendation, not a code-size
  estimate).

**Open questions.** Same sequencing question as §2.1: should this ticket recommend that ticket 0022
(architecture options) treat `Funcdata` and `Architecture` decomposition as one joint design exercise
(option C) rather than two independent ones? Also: is the `ArchitectureGhidra` capability-registration
bypass (option B) deliberate (Ghidra-hosted operation was never meant to go through the generic
capability path) or an oversight — worth a direct question back to whoever authored
`architecture-pipeline-overview.md` (ticket 0003) if that's not already answered there.

---

## 3. Type-system expressiveness gaps

Per the ticket's Analysis Focus item 3, this section is deliberately exhaustive — it is direct input to
requirements ticket 0017. Each gap is a concrete "the `Datatype` hierarchy cannot represent X" claim,
verified against `type.hh`/`type.cc` in this session beyond what the type-system review already
documents.

### 3.1 No templates, generics, or parametric types

**Observation.** `type-system.md` §6.2, verified: every `Datatype` is fully concrete — `TypeStruct`/
`TypeUnion` fields are resolved, fixed `Datatype*` pointers (`type.hh:580,633`). There is no type
parameter, no unresolved/deferred type variable, and no template-instantiation identity distinct from
plain structural identity (§3 of `type-system.md`, the `id`/interning scheme). A C++
`template<T> struct Foo` monomorphized for two different `T`s becomes two entirely unrelated
`TypeStruct` objects with **no recorded relationship** that they came from "the same" generic
definition.

**Impact.** Any decompiler output involving C++ templates, Rust generics, or similar (a large share of
modern C++ binaries) can only ever show monomorphized, disconnected concrete types — there's no way to
say "these five structs are `Vector<T>` for five different `T`s" even if the analysis could in principle
detect the pattern. This is a pure representational ceiling, not a heuristic-quality issue: no algorithm
improvement closes this gap without a new `Datatype` concept.

**Severity.** Medium today (most binaries the decompiler targets are C, where this doesn't arise), but
directly relevant to ticket 0017's scope per the type-system review's own forward-reference ("one of the
specific gaps the root epic's requirements phase — ticket 0016/0017 — is expected to address explicitly,"
`type-system.md` §6.2).

**Improvement options.**
- **A — Introduce a `TypeTemplate`/`TypeGeneric` concept**: a named generic definition plus an explicit
  instantiation-argument list per concrete instance, with the interning key (§`type-system.md` §3)
  extended to include the argument list so two instantiations of the same generic over the same
  arguments canonicalize to one object, and two different argument lists are recognized as related-but-
  distinct. *Pros*: directly closes the gap; gives future display/analysis code a real hook ("this is
  `Vector<int>`, an instance of generic `Vector`"). *Cons*: large — touches the interning scheme (§3),
  `compare()`/`compareDependency()` (§1.2), the Ghidra type bridge (§1.8, since Ghidra's own
  `DataTypeManager` would need an analogous concept for round-tripping), and every place that currently
  assumes "structurally equal `Datatype*` implies fully interchangeable type." *Size*: Large.
- **B — Keep struct+opaque-pointer convention (status quo), but add purely cosmetic name-pattern
  detection at the printer layer** (recognize `Foo<int>`-shaped mangled/demangled names and group them
  for *display* only, with no underlying `Datatype`-level relationship). *Pros*: small, no core type-
  system change, immediately improves readability for the common case (a demangled C++ name already
  often contains the template arguments as text). *Cons*: purely cosmetic — doesn't help any analysis
  that would benefit from actually *knowing* two types are related (e.g. simplifying casts between
  instantiations, or matching a known-generic-container's method against a database of standard-library
  signatures). *Size*: Small.
- **C — Model only the common special case (containers with one size/type parameter, e.g.
  fixed-size arrays and `TypeArray`'s existing element-type field) as "good enough" generics-lite**,
  without a general parametric-type mechanism. *Pros*: `TypeArray` already carries an element
  `Datatype*` (`type.hh:78` diagram) — this is closer to "extend what exists" than "add a new concept."
  *Cons*: doesn't generalize to multi-parameter templates, non-container generics, or template
  specialization — a narrow fix for a narrow slice of the actual gap. *Size*: Small-Medium.

**Open questions.** Does this need to interoperate with Ghidra's own `DataTypeManager` template/generic
support (if any exists there) — i.e., is this purely a decompiler-internal representational gap, or does
Ghidra's Java side already have *some* concept here that the bridge (§1.8) simply doesn't propagate?
Worth a direct question to whoever holds ticket 0017.

### 3.2 No composable const/volatile qualifier fidelity

**Observation.** Verified directly: `Datatype`'s full flag set (`type.hh:180-198`) has **no `const` bit
at all**, at any granularity. `volatile` exists, but only one level away from where C models it: as a
`Varnode`-level flag (`volatil`, `varnode.hh:93`) and a `Symbol`-level flag (`isVolatile()`,
`database.hh:260`) — i.e. volatility is a property of a *storage location/symbol*, never of a
`Datatype` object itself. There is no way to represent, at the type level, the distinction C makes
between `const int*` (pointer to const int), `int *const` (const pointer to int), and
`const int *const` (both) — `TypePointer` (`type.hh:460`) has no qualifier field on either itself or its
`ptrto`.

**Impact.** Two concrete, demonstrable consequences: (1) `const`-ness can never appear in printed output
derived purely from the type system (only from whatever heuristic, if any, the printer applies based on
other signals — not reviewed here, out of this ticket's file scope, but the type system itself supplies
no data for it to use); (2) volatility is tied to *where a value lives* (a `Varnode`/`Symbol`), not to
*what its type is* — so a `Datatype*` shared between a volatile-storage access and a non-volatile one
(plausible under the interning scheme, §`type-system.md` §3, since identical structural types share one
canonical `Datatype*`) cannot itself distinguish the two, unlike real C semantics where `volatile` is
part of the type, not the storage.

**Severity.** Medium. Missing `const` is primarily a display/readability gap (the emitted C is less
informative than it could be, but not incorrect); missing type-level `volatile` composability is closer
to a modeling gap that could affect optimization-sensitive `Rule`s (ticket 0006's territory) if any of
them make correctness-relevant assumptions based on type alone rather than checking the `Varnode`/
`Symbol` flag directly — this ticket did not find evidence either way and flags it as an open question.

**Improvement options.**
- **A — Add a `const`/`volatile` qualifier bitmask to `Datatype` itself** (mirroring how C's type system
  actually works: qualifiers attach to the type, and a pointer's qualifier is independent of its
  pointee's), extending the interning key so `const int` and `int` intern separately but share structure
  where possible. *Pros*: closes the gap at its root; matches source-language semantics exactly.
  *Cons*: qualifier-aware interning multiplies the number of distinct `Datatype*` instances for
  practically every primitive/pointer/struct combination, with performance and cache-locality
  implications for `TypeFactory::typecache[9][8]` (`type.hh:859`, `type-system.md` §2.2) and every
  `compare()`/`compareDependency()` override; every existing pointer-equality-as-type-equality
  assumption throughout propagation (`type-system.md` §3, "pointer equality is used everywhere as type
  equality") needs re-auditing for whether it should now also check/ignore qualifiers. *Size*: Large.
- **B — Model `const` as display-only metadata on `Symbol`/`SymbolEntry`, matching how `volatile`
  already partially works, without touching `Datatype`.** *Pros*: much smaller, consistent with the
  existing (if inconsistent) precedent that `volatile` already lives partly outside `Datatype`; doesn't
  risk the interning-explosion problem of option A. *Cons*: perpetuates the existing inconsistency
  (`volatile` split between `Symbol` and `Varnode`, and now `const` living somewhere else entirely
  rather than either) instead of resolving it; still can't express "pointer to const" vs. "const
  pointer" since that distinction is inherently about the *type*, not the storage location it's
  ultimately assigned to. *Size*: Small-Medium.
- **C — Add qualifiers only to `TypePointer`/`TypePointerRel`** (the case where C's qualifier placement
  genuinely matters for correctness of output, i.e. pointer chains), leaving scalar/struct-level
  `const`/`volatile` as future work. *Pros*: targets the highest-value case (pointer qualifier placement
  is the classic C readability/correctness issue) without the full interning-explosion risk of
  qualifying every type. *Cons*: still incomplete relative to full C semantics; a struct field or local
  variable's own `const`-ness remains unrepresentable at the type level. *Size*: Medium.

**Open questions.** Does any existing `Rule` (ticket 0006/0013's territory) implicitly rely on type
identity (pointer equality, `type-system.md` §3) meaning "these two values are truly interchangeable,"
in a way that adding qualifiers (which would make `const int*` and `int*` distinct `Datatype*`
instances) could silently break by making a rule's type-equality check newly fail where it used to
succeed? This needs a cross-check against ticket 0013's findings before committing to option A.

### 3.3 No first-class "function pointer with context" type (member-function pointers, closures)

**Observation.** `type-system.md` §6.3, verified against `type.hh:787-809`: `TypeCode` models a
function/function-pointer target purely as a `FuncProto*` (`type.hh:790`). A `FuncProto` can be marked
constructor/destructor and carries a calling convention through `ProtoModel`
(`fspec.hh:1362`, confirmed this session — `TypeCode::getPrototype()` returns `const FuncProto*`, and
`FuncProto` in turn owns a `ProtoModel *model`), but there is **no `Datatype` subclass that pairs a
`TypeCode` with a separate offset/adjustor** the way `TypePointerRel` pairs a `TypePointer` with a
struct offset (§2.1's diagram). A C++ member-function pointer (`R (C::*)(Args...)`, which on most ABIs
needs both a code address *and* an implicit `this`-adjustment/vtable-context) has no representational
analogue.

**Impact.** Member-function-pointer values (a real, if less common, feature of C++ binaries — used for
callback tables, some virtual-dispatch-adjacent idioms) can only be represented as either a raw
`TypeCode`/function pointer (losing the adjustment context) or an opaque blob, not as a distinct,
analyzable type. This directly limits how well the decompiler can present C++-heavy binaries, which is
plausibly in-scope for the refactor's target audience given the codebase's own C++ implementation
language and Ghidra's general use on compiled-C++ targets.

**Severity.** Low-Medium — a real, named gap (per the review) but a narrower audience-impact than §3.1's
generics gap, since member-function pointers are a smaller fraction of typical binaries than templated
types are.

**Improvement options.**
- **A — Introduce a `TypeCodeRel` (or similarly named) subclass of `TypeCode`**, directly mirroring
  `TypePointerRel`'s existing pattern (a "pointer + fixed offset into container" specialization,
  `type.hh:740-779`) but for code pointers plus an adjustor/context value. *Pros*: follows an existing,
  proven pattern in the same codebase (`TypePointerRel`) rather than inventing a new one — lowest-novelty
  option. *Cons*: member-function-pointer ABI representations vary significantly across compilers
  (Itanium ABI's `{ptr, adjustment}` pair vs. MSVC's multiple representations depending on inheritance
  shape) — a single `TypeCodeRel` shape may not generalize across target ABIs without ABI-specific
  variants, unlike `TypePointerRel`'s comparatively uniform "offset into container" semantics.
  *Size*: Medium.
- **B — Treat member-function pointers as an opaque, sized `struct`-like blob (status quo) and rely on
  target-ABI-specific `Rule`s to recognize and specially print the idiom**, without a dedicated
  `Datatype`. *Pros*: zero core type-system change; matches how some other ABI-specific idioms are
  already handled at the `Rule` layer per `action-rule-engine.md`'s territory. *Cons*: pushes real type
  information into pattern-matching heuristics instead of the type system, the same "should this be a
  type or a rule" tension the review's `TypeSpacebase` finding (§1.5) already raises elsewhere in this
  document. *Size*: Small.
- **C — Defer entirely, scoped as future work contingent on §3.1's generics decision**, since both gaps
  ultimately point toward the same underlying need (a more expressive parametric/composite type
  mechanism) and solving them independently risks two incompatible ad hoc extensions. *Pros*: avoids
  building a narrow, ABI-specific mechanism that might be subsumed by whatever general mechanism §3.1
  adopts. *Cons*: leaves the gap entirely open in the meantime. *Size*: None (a sequencing decision, not
  a change).

**Open questions.** How common are member-function-pointer values in the actual binaries this refactor's
stakeholders care about decompiling? Without usage data, it's hard to rank this against §3.1/§3.4/§3.5
for priority — worth a question back to whoever scopes ticket 0017's requirements.

### 3.4 No union-level bitfields

**Observation.** `type-system.md` §6.1, verified: `TypeBitField` (`type.hh:339`) exists **only as a
member of `TypeStruct::bitfield`** (`type.hh:581`). `TypeUnion` has no bitfield vector at all
(`type.hh:630-656`) — a bitfield inside a union field is representable only indirectly, as an ordinary
integer-typed union field, with no bit-level breakdown modeled by the type system itself. The actual
bitfield shift/mask idiom recognition is a separate `Rule`-based layer (`bitfield.hh:67,124`,
`BitFieldTransform`) operating after the fact on decoded p-code, independent of whether the underlying
`Datatype` is a struct or union.

**Impact.** A union containing a bitfield-typed member (legal, if unusual, C — e.g. hardware register
overlay unions with bitfield sub-views) cannot have that bitfield structure represented at the type
level at all, only at the separate expression-rewriting layer, and only for whatever access pattern that
layer's `Rule`s happen to recognize — the type itself just says "this union field is an integer."

**Severity.** Low. A narrow, uncommon C idiom (though not rare in embedded/hardware-register-heavy
reverse-engineering targets, which are plausibly relevant to this decompiler's actual user base).

**Improvement options.**
- **A — Add a `bitfield` vector to `TypeUnion`, mirroring `TypeStruct::bitfield`** (`type.hh:581`),
  reusing the existing `TypeBitField` class as-is (it's already just name + underlying type + `BitRange`,
  not struct-specific in its own definition). *Pros*: small, symmetric with the existing struct
  mechanism, reuses `TypeBitField` unchanged. *Cons*: `assignFieldOffsets`
  (`type.cc:2440`, `type-system.md` §6) and `assignContiguousBitfields`'s "position counter reaches the
  next bitfield-adjacent field" logic is written for struct's *sequential* layout model — unions don't
  have a sequential offset accumulator (every field starts at offset 0), so the layout algorithm would
  need real (if likely small) union-specific logic, not just reused field-vector storage. *Size*: Small-
  Medium.
- **B — Leave as-is; this is adequately covered by the existing Rule-based bitfield-idiom recognition
  layer regardless of the declared type**, since the actual shift/mask rewriting doesn't depend on
  whether the container is nominally a struct or union at the type-declaration level. *Pros*: zero cost;
  the review found the *expression-level* handling to already be independent of struct-vs-union.
  *Cons*: the type itself still can't declare "this union field is a 3-bit flag," which matters for
  anyone consuming the type information directly (e.g. structure-editing UI, scripted analysis) rather
  than only reading printed output. *Size*: None.

**Open questions.** How does Ghidra's own `DataTypeManager` model union bitfields, if at all — is this
purely a decompiler-side gap, or does the bridge (§1.8) already lose this information on the Ghidra side
too, making this moot until Ghidra's own type model supports it?

### 3.5 Function types carry calling convention only via full `FuncProto`, not as a lightweight attribute

**Observation.** Per §3.3 above and `fspec.hh:1362` (`ProtoModel *model` member of `FuncProto`), a
calling convention is only attachable to a function type by fully instantiating a `FuncProto` (which
also demands parameter/return recovery machinery, `signature-calling-conventions.md`'s territory per
ticket 0008). There is no lighter-weight way to say "this is a function type using calling convention
X" without going through the complete prototype-recovery apparatus — e.g. for a raw function-pointer
*declaration* known only by calling convention and signature shape (common in headers: `__stdcall`,
`__fastcall`, `__thiscall` annotations on a function pointer type before any specific function is
analyzed).

**Impact.** This narrows where calling-convention information can enter the type system to exactly the
places `FuncProto` recovery already runs — it cannot be attached earlier/independently as a bare
type-level fact about a function-pointer typedef, for instance. Whether this is a real practical
limitation depends on ticket 0008's territory (how/where `ProtoModel` selection actually happens today)
more than on `type.hh` itself, so this finding is narrower and more speculative than §3.1-3.4.

**Severity.** Low. Flagged for completeness per the ticket's explicit ask ("function types with
calling-convention attributes attached") but this ticket's source scope (`type.hh`, `type.cc`,
`typeop.hh`) cannot fully assess whether it's a real gap or whether `signature-calling-conventions.md`
(ticket 0008, not required reading for this ticket) already shows a lighter-weight mechanism this
document missed.

**Improvement options.**
- **A — Add an optional, standalone calling-convention tag directly on `TypeCode`**, usable without a
  full `FuncProto` (only populated when a fuller prototype isn't yet recovered/available). *Pros*: closes
  the gap identified above if it's real. *Cons*: risks two divergent calling-convention representations
  on the same class (`TypeCode`'s own tag vs. `FuncProto::model`) needing reconciliation once full
  prototype recovery does run — a new instance of the same "two representations of one fact" pattern
  flagged repeatedly elsewhere in this document (§1.7, §1.8). *Size*: Medium.
- **B — Defer this finding to ticket 0013/0008's territory for confirmation before any type-system
  change**, since it may already be adequately handled by machinery this ticket's file scope didn't
  cover. *Pros*: avoids solving a problem that might not exist as stated. *Cons*: leaves the question
  open. *Size*: None.

**Open questions.** This finding needs direct confirmation from `signature-calling-conventions.md`
(ticket 0008) or its corresponding flaw-analysis sibling before being acted on — flagged here per the
ticket's explicit instruction to be exhaustive on expressiveness gaps, not because this ticket's own
source scope fully substantiates it.

### 3.6 No packed/explicit-alignment-override representation

**Observation.** A targeted grep of `type.hh`/`type.cc` in this session for "packed" found **zero
matches** in either file. `TypeStruct::assignFieldOffsets` (`type.cc:2440`, `type-system.md` §6) is
described as "straightforward C-struct layout (accumulate offset, align to each field's
`getAlignment()`, round the final size up to `newAlign`)" — a naturally-aligned layout model with no
flag or mechanism found for a field or struct explicitly overriding natural alignment (the C/C++
`__attribute__((packed))` / `#pragma pack` idiom).

**Impact.** A packed struct sourced from Ghidra's `DataTypeManager` (which likely *does* model packing,
since it's a common binary-format concern) would presumably arrive with pre-resolved field offsets via
the wire bridge (§1.8) rather than being locally recomputed by `assignFieldOffsets` — meaning this may
be a non-issue in practice for Ghidra-hosted operation specifically (Ghidra does the packing-aware
layout, the decompiler only consumes the result) but would be a real gap for the standalone/console tool
path (`BfdArchitectureCapability` et al., §2.2) doing its own struct layout without a Ghidra-side type
database to defer to.

**Severity.** Low. Likely masked in the primary (Ghidra-hosted) use case by the wire bridge; a genuine
but narrower gap for the standalone tool.

**Improvement options.**
- **A — Add a `packed`/explicit-alignment flag to `TypeStruct` and teach `assignFieldOffsets` to honor
  it** (skip per-field alignment padding when set). *Pros*: closes the gap for the standalone-tool case.
  *Cons*: low value if, per the impact analysis above, the primary Ghidra-hosted path never exercises
  local `assignFieldOffsets` for packed structs anyway — needs confirmation before investing here.
  *Size*: Small.
- **B — Confirm whether Ghidra-sourced struct field offsets bypass `assignFieldOffsets` entirely
  (arrive pre-resolved via the wire bridge) before treating this as a live gap.** *Pros*: cheap,
  resolves the open question directly. *Cons*: investigation only, no functional improvement by itself.
  *Size*: None.

**Open questions.** Does `TypeFactoryGhidra::decodeType()` (`type.cc:4842`, §`type-system.md` §4) ever
call `assignFieldOffsets` for a wire-received struct, or does it always set field offsets directly from
the decoded wire data? This determines whether option A is worth doing at all.

### 3.7 No graded/confidence-scored type inference — lock is binary

**Observation.** `type-system.md` §5.2, verified: `typelock`/`namelock` (`varnode.hh:90-91`) are the
system's *only* notion of type confidence — "there is no numeric/graded confidence score anywhere in
propagation; a type is either freely overwritable or completely frozen." `Varnode::updateType(ct)`
(`varnode.cc:480-483`) enforces exactly this binary rule at the single commit point.

**Impact.** This isn't a missing-representation gap in the same sense as §3.1-3.6 (it's not that some
*type* can't be expressed — every concrete type the propagation engine might infer is expressible); it's
that the *inference result itself* can't carry a confidence gradient, only a lock bit. Two Varnodes both
inferred (not locked) as `int*` — one from a single, strong piece of evidence (a locked-parameter's
propagated type) and one from a weak, speculative chain of `ScoreUnionFields`-style heuristic reasoning
— are indistinguishable to any downstream consumer. A refactor wanting to, e.g., visually flag
low-confidence inferred types in the UI, or make merge/propagation decisions confidence-aware rather than
binary, has no data to work from.

**Severity.** Medium. This is squarely adjacent to (not identical to) the correctness-adjacent risk
areas in §4 — the *lack* of graded confidence is itself part of why some of those risk areas (§4.2, §4.3)
can only be "best effort" with no way to communicate degree of certainty downstream.

**Improvement options.**
- **A — Add a confidence enum/score alongside the existing lock bit** (e.g. `unlocked` /
  `inferred-strong` / `inferred-weak` / `locked`), threaded through `propagateTypeEdge`
  (`coreaction.cc:5445`) and `ScoreUnionFields`'s existing internal scoring (`unionresolve.cc:131-133`,
  which *already computes* a numeric score internally, per `type-system.md` §6, but currently discards
  it once the best field is picked). *Pros*: `ScoreUnionFields` already does the hard part (numeric
  scoring) — surfacing it rather than discarding it is a comparatively contained change; enables future
  UI/tooling to distinguish confident vs. speculative types without re-deriving confidence from scratch.
  *Cons*: touches the propagation fixed-point loop's convergence logic (`type-system.md` §5.1's
  `typeOrder()`-based "strictly more specific" acceptance test) — adding a confidence dimension to an
  already-nontrivial convergence proof (7-round empirical cap, `coreaction.cc:5763`) needs care that it
  doesn't introduce new non-termination or oscillation cases. *Size*: Medium-Large.
- **B — Leave the binary lock model, but expose `ScoreUnionFields`'s discarded numeric score
  specifically for union-field-resolution confidence only** (a narrower version of option A scoped to
  the one place a real internal score already exists and is currently thrown away). *Pros*: much smaller
  than a general confidence system; directly reuses existing computation. *Cons*: doesn't generalize to
  non-union type-propagation confidence (the broader half of this gap). *Size*: Small.
- **C — Leave as-is.** *Pros*: zero cost/risk. *Cons*: the gap, and its downstream effect on §4.2/§4.3's
  "best effort, silently degrades" risk areas, persists. *Size*: None.

**Open questions.** Would a graded-confidence model actually change any *decision* the decompiler makes
today (propagation acceptance, merge/cover heuristics), or would it only add *displayable* metadata with
no behavioral effect? If purely the latter, option B (small, display-only) is clearly sufficient; if the
former, option A's larger convergence-logic risk needs explicit sign-off from whoever owns ticket 0017's
requirements before being scoped as more than "future work."

---

## 4. Correctness-adjacent risk areas (SSA/merge heuristics)

Per the ticket's Analysis Focus item 3 ("merge/cover heuristics that are 'best effort' — where does the
design admit it can silently produce wrong (not just ugly) output?"). This section is deliberately
shorter than §§1-3 since `ssa-varnode-heritage.md` §4.3 and `type-system.md` §§5-6 already enumerate the
concrete admissions in detail; this section synthesizes them as flaw entries rather than re-deriving them.

### 4.1 `Merge`'s speculative-merge heuristics are abandoned, not corrected, on conflict

**Observation.** `ssa-varnode-heritage.md` §4.2: speculative merges (opportunistic, e.g.
`mergeByDatatype`, `merge.cc:359`) are "simply *abandoned*, with no dataflow change, if a `Cover`
intersection is found" — this is explicitly the *safe* half of the design (forced merges, by contrast,
insert `COPY`s to force disjointness rather than risk wrong output, §4.2). The risk instead sits in
`Symbol::isolate` (`database.hh:265,300-301`) — "an explicit user/heuristic override to stop speculative
auto-merging for a `Symbol` altogether — an acknowledgment that the speculative heuristics sometimes
produce a *plausible but wrong* grouping that a human has to veto" (`ssa-varnode-heritage.md` §4.3).

**Impact.** A *plausible but wrong* grouping is worse than an obviously-failed one: it doesn't crash or
warn, it presents as a normal-looking single variable that actually elides a real distinction in the
underlying SSA values — exactly the "silently produce wrong (not just ugly) output" case the ticket asks
about. The existence of `isolate` as an escape hatch is itself the design's admission that this happens
in practice, not just in theory.

**Severity.** Medium-High — this is the clearest, most directly-cited "admits it can be wrong" finding
in the source material for this ticket.

**Improvement options.**
- **A — Add heuristic-confidence output to speculative merges** (tying into §3.7's confidence-gradient
  gap): when a speculative merge is accepted, record *how* confidently (e.g. single-datatype-match vs.
  multiple corroborating signals) so `isolate`-worthy cases could in principle be flagged proactively
  rather than only reactively (after a human notices wrong output and manually isolates). *Pros*: turns
  a purely reactive safety valve into a partially proactive one. *Cons*: depends on §3.7's broader
  confidence-model change landing first; "how confident was this speculative merge" isn't obviously
  reducible to a single score without design work. *Size*: Large (contingent on §3.7).
- **B — Instrument/log speculative-merge decisions during a test/audit pass** (not shipped runtime
  behavior — a diagnostic mode) to empirically characterize how often `isolate` actually gets used across
  a representative corpus, informing whether this is a rare edge case or a frequent-enough issue to
  justify option A's investment. *Pros*: cheap, evidence-gathering, doesn't commit to a larger change
  before knowing if it's warranted. *Cons*: doesn't fix anything by itself. *Size*: Small.

**Open questions.** Is there existing telemetry/data (even informal, from Ghidra's issue tracker or user
reports) on how often speculative-merge mis-grouping is actually observed and reported by users? That
data would materially change this finding's priority relative to the type-system expressiveness gaps in
§3, which are more clearly cost-free-to-defer by comparison.

### 4.2 Aliasing conservatism trades wrong-output risk for over-conservative output

**Observation.** `ssa-varnode-heritage.md` §4.3: because `guardStores`/`guardLoads` force `INDIRECT`
ops wherever a computed pointer *might* touch a range (§2.4), the resulting `Cover` intersections "can
block a merge that would have been semantically fine had the alias been provably absent" — i.e. the
design explicitly chose over-conservatism (more distinct displayed variables than necessary) specifically
*to avoid* the wrong-output failure mode, with `LoadGuard` refinement via value-set analysis
(`heritage.cc:834-909`) as "the main lever for recovering precision."

**Impact.** This is the inverse of §4.1 — here the design is explicitly *not* silently wrong, it's
silently *worse-looking* (variable-count bloat) as the deliberate safety trade-off. Worth recording
precisely because it's the correct-by-design counterexample to keep in mind alongside §4.1's genuine
risk: not every "best effort" heuristic in this subsystem is a correctness risk; some are conservative-
by-design precision costs instead.

**Severity.** Low as a flaw (it's working as intended), but High-value as context: any refactor
"improving" merge precision by relaxing this conservatism without also strengthening `LoadGuard`
refinement would reintroduce real silent-wrong-output risk, not just restore lost precision.

**Improvement options.**
- **A — Invest further in `LoadGuard` refinement (value-set analysis precision)** as the sanctioned way
  to recover precision, rather than relaxing the underlying conservatism. *Pros*: stays on the safe side
  of the existing, deliberate trade-off; directly targets the actual bottleneck (`LoadGuard::
  establishRange`/`finalizeRange`, `heritage.hh:151-152`) the review identifies as "the main lever."
  *Cons*: value-set analysis precision work is itself nontrivial and possibly already near its practical
  ceiling for irreducible aliasing cases (pointers whose range genuinely can't be narrowed statically).
  *Size*: Medium-Large, and out of this ticket's scope to estimate precisely (value-set analysis internals
  weren't part of the required reading).
- **B — No change; document this explicitly as a "working as intended" boundary** so future refactor
  proposals don't mistake the variable-count cost for a bug to "fix" by relaxing conservatism.
  *Pros*: zero cost, directly prevents a plausible future mistake. *Cons*: doesn't improve anything.
  *Size*: None.

**Open questions.** None beyond what's already answered in the review doc — this entry is included
primarily to make the "conservative-by-design, not a flaw" distinction explicit and citable for later
tickets, per the ticket's Analysis Focus asking specifically where the design *admits* it can be wrong
(here, it explicitly does not — it admits the opposite trade-off).

### 4.3 Union-field resolution is a bounded heuristic search, not exhaustive

**Observation.** `type-system.md` §6, verified: `ScoreUnionFields` is "a heuristic, capped-effort local
search, not an exhaustive or provably-optimal resolution" — bounded by `maxPasses=6`, `threshold=256`,
`maxTrials=1024` (`unionresolve.cc:131-133`) — and "a union field choice can be wrong, and once cached
(locked or not) it is not automatically revisited as more of the function is analyzed unless something
explicitly invalidates it."

**Impact.** This is a direct, source-confirmed "can silently produce wrong output" case distinct from
§4.1 (which is about *merging distinct SSA values into one display variable*; this is about *which
struct/union field a memory access actually means*) — a wrong field pick changes the actual semantic
content of the printed expression (e.g. printing `.x` where the real access was `.y`), not just its
variable-naming/grouping.

**Severity.** High. Of everything in this section, a wrong union-field resolution most directly changes
what the output *means*, not merely how it's grouped or displayed — squarely the ticket's "silently wrong,
not just ugly" bar. Compounded by §1.3's finding that the same underlying heuristic is entered from three
independently-coded call sites, raising (unconfirmed, per §1.3's open question) the possibility of
cross-path inconsistency on top of the heuristic's own inherent imprecision.

**Severity.** High — see above; not downgraded, since (unlike §4.2) this is a genuine "can be wrong"
admission from the source docs, not a deliberate conservative trade-off.

**Improvement options.**
- **A — Surface `ScoreUnionFields`'s confidence/margin (best score vs. runner-up) to the user/printer**
  (e.g. a visual or textual indicator when a union-field resolution was a close call), reusing the same
  underlying mechanism as §3.7's confidence-gradient proposal. *Pros*: doesn't require improving the
  heuristic's accuracy, just its honesty about uncertainty — directly actionable and much smaller than
  trying to make the search exhaustive. *Cons*: doesn't fix wrong resolutions, only flags likely-uncertain
  ones; "close call" isn't the same as "wrong" (a heuristic can be simultaneously confident and wrong on
  a genuinely ambiguous case). *Size*: Medium (shares scope with §3.7 option B).
- **B — Add a revisit/invalidation trigger** so a cached (non-locked) resolution is re-scored when
  materially new evidence becomes available later in analysis (addressing the specific "not automatically
  revisited... unless something explicitly invalidates it" gap). *Pros*: directly closes a named,
  specific weakness rather than only surfacing uncertainty. *Cons*: risks resolution *instability*
  (a field choice flip-flopping across passes) if not carefully bounded — the existing 6-pass/1024-trial
  caps exist precisely to bound cost/oscillation, and a revisit mechanism needs equally careful bounding.
  *Size*: Medium-Large.
- **C — Raise the search bounds** (`maxPasses`/`threshold`/`maxTrials`, `unionresolve.cc:131-133`) to
  trade compile-time cost for resolution accuracy, without structural change. *Pros*: trivial to
  implement. *Cons*: the bounds were presumably chosen empirically for a performance/accuracy trade-off
  already (mirroring the type-propagation "7 rounds... arrived at empirically" pattern, `type-system.md`
  §8) — raising them blindly risks a meaningful compile-time regression for uncertain accuracy gain
  without profiling data to justify the new values. *Size*: Small (to change), Medium (to validate
  properly).

**Open questions.** Is there a corpus of known-correct union-field resolutions (e.g. from hand-verified
`datatests`) that could be used to measure `ScoreUnionFields`'s actual accuracy rate today, to establish
a baseline before investing in any of the above options? `00-index.md`'s test-coverage table doesn't
call out union-resolution specifically, so this may not currently exist.

---

## 5. Summary table

| # | Finding | Section | Severity | Smallest viable option |
|---|---|---|---|---|
| 1 | `TypePartial*` triad duplication | §1.1 | Medium | C (extract shared `getStripped()` helper) |
| 2 | `compare()`/`compareDependency()` duplication | §1.2 | Low-Medium | B (consistency assertions) |
| 3 | Union-resolution logic tripled | §1.3 | Medium-High | C (cross-path consistency tests) |
| 4 | Two "ephemeral/stripped" mechanisms | §1.4 | Low-Medium | B (document convention) |
| 5 | `TypeSpacebase` pseudo-struct conflation | §1.5 | Low | C (document dual role) |
| 6 | `type_class`/`type_metatype`/`sub_metatype` overlap | §1.6 | Low | A (document + name conversions) |
| 7 | `Varnode`/`HighVariable`/`Symbol` field & flag duplication | §1.7 | Medium-High | B (checked sync operation) |
| 8 | `TypeFactory` vs. Ghidra `DataTypeManager` split-brain | §1.8 | High | B (recoverable conflict, not hard abort) |
| 9 | `Funcdata` god object | §2.1 | High (refactor) / Medium (correctness) | B (peel off debug/nested-class accretion) |
| 10 | `Architecture` god object | §2.2 | Medium-High | B (fix capability-registration asymmetry) |
| 11 | No templates/generics | §3.1 | Medium | B (cosmetic printer-level grouping) |
| 12 | No composable const/volatile | §3.2 | Medium | C (qualify pointers only) |
| 13 | No function-pointer-with-context type | §3.3 | Low-Medium | C (defer, contingent on §3.1) |
| 14 | No union bitfields | §3.4 | Low | B (leave as-is; Rule layer suffices) |
| 15 | Calling convention needs full `FuncProto` | §3.5 | Low (unconfirmed) | B (defer to ticket 0008) |
| 16 | No packed/alignment-override | §3.6 | Low | B (confirm whether it's live first) |
| 17 | No graded type-inference confidence | §3.7 | Medium | B (surface `ScoreUnionFields`'s existing score only) |
| 18 | Speculative merge: plausible-but-wrong grouping | §4.1 | Medium-High | B (instrument/measure first) |
| 19 | Aliasing conservatism (working as intended) | §4.2 | Low (context, not a flaw) | B (document the trade-off) |
| 20 | Union-field resolution: bounded heuristic, can be wrong | §4.3 | High | A (surface confidence/margin) |
