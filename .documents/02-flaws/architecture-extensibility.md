# Flaws: Overall Architecture, Extensibility & API/Testability

Produced by [`.tasks/0015`](../../.tasks/0015-task-flaws-architecture-extensibility.md), part of the
flaw-analysis epic [`.tasks/0011`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md), under
root [`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md). Builds on the review epic
[`.tasks/0002`](../../.tasks/0002-epic-review-current-implementation.md), specifically
[`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) (0003),
[`java-integration-testing.md`](../01-review/java-integration-testing.md) (0010),
[`signature-calling-conventions.md`](../01-review/signature-calling-conventions.md) (0008), and the
synthesized findings in [`00-index.md`](../01-review/00-index.md). Shared terms:
[`../00-overview/glossary.md`](../00-overview/glossary.md).

This document does not re-review any subsystem's internals — it takes the review epic's output and
asks four cross-cutting questions: can the engine be extended without editing core files, does the
codebase's test coverage make a staged refactor safe, what is the real contract between the C++
engine and its Java caller, and can a change be validated without a full Ghidra/Gradle build. Each
flaw follows the structure Observation → Impact → Severity → Improvement options → Open questions,
per ticket 0015. A dedicated **Refactor Sequencing Risk** table (ticket 0015's required deliverable)
follows the flaw entries.

---

## Flaw 1 — The extension model is two incompatible mechanisms wearing one name

### Observation

The engine advertises a single self-registration mechanism, `CapabilityPoint`, as "how new things plug
in": every concrete extension is a global singleton whose constructor pushes itself onto a static
vector, and `CapabilityPoint::initializeAll()` gives each a chance to finish setup after all static
initializers have run
([`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) §4,
`capability.hh:39-52`, `capability.cc:24-29,40-49`). Four independent hierarchies reuse it —
`ArchitectureCapability`, `GhidraCapability`, `IfaceCapability`, `PrintLanguageCapability` (same doc,
§4 table). This genuinely works as advertised for those four: adding a new instance is "add a `.cc`
file and link it in," no central registry edit needed.

But the two things a plugin author would most want to add — a new `Rule` (target-specific
peephole idiom) or a new scheduled `Action` — do **not** use this mechanism at all. All ~500 `Rule`
classes (162 wired into the default pipeline) are hand-instantiated via literal `new RuleXxx(...)`/
`addRule()` calls inside one ~700-line function, `ActionDatabase::universalAction`
(`coreaction.cc:5835` onward — [`action-rule-engine.md`](../01-review/action-rule-engine.md) §1;
confirmed independently in
[`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) §4). The only
data-driven levers here are `Architecture::extra_pool_rules` — which can only *toggle on* an
already-compiled `Rule` by name from a `.cspec` `<rule>` tag, not add a new one
(`architecture.hh:189`, `decodeDynamicRule`, `architecture.hh:359`) — and the `CPUI_RULECOMPILE`-gated
pattern DSL (`rulecompile.cc`), which is dead code in shipped builds (off by default, one call site
references an undeclared variable — [`00-index.md`](../01-review/00-index.md) "Known-surprising
findings", [`action-rule-engine.md`](../01-review/action-rule-engine.md) §3).

A third inconsistency compounds this: the one `Architecture` subclass Ghidra itself actually uses,
`ArchitectureGhidra`, **bypasses** `ArchitectureCapability::findCapability` — the very discovery
mechanism the codebase presents as "how new architectures plug in" — and is instead constructed
directly with `new ArchitectureGhidra(...)` in `RegisterProgram::rawAction`
(`ghidra_process.cc:181`, [`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md)
§2). Only the three file-based, non-Ghidra-hosted architectures (`Bfd`, `RawBinary`, `Xml`) actually go
through capability discovery.

Compared against [`signature-calling-conventions.md`](../01-review/signature-calling-conventions.md)
§1.2/§6, this is a strong contrast: `ModelRule` (parameter-storage assignment) is a real, proven,
declarative layer — `DatatypeFilter`/`QualifierFilter` → `AssignAction`, driven by `<model_rule>` XML,
genuinely additive on top of a hardcoded fallback, actually used by every modern multi-register-struct
ABI in-tree. The `Rule`/`Action` engine has no equivalent: "declarative extension of the simplification
engine" was attempted (`rulecompile.cc`) and abandoned, not merely not-yet-built.

### Impact

- A refactor that wants "let target-specific idioms be added without recompiling the core engine" (a
  plausible ask for this codebase, given how architecture-specific many `Rule`s already are) has **no
  existing seam to extend** — `CapabilityPoint` doesn't reach `Rule`/`Action` at all, and the one prior
  attempt at a declarative rule layer is unmaintained dead code.
- Anyone reading the codebase's own self-description (four `Capability` hierarchies, all documented the
  same way) would reasonably assume `Rule`s work the same way and be wrong — this is a documentation/
  discoverability problem for the *refactor team itself*, not just external plugin authors.
- `ArchitectureGhidra` bypassing discovery means the extension mechanism's own most important consumer
  doesn't exercise it — any future change to `ArchitectureCapability` risks being validated only against
  the three minor, rarely-used file-based architectures, not the path that matters.
- Even where `CapabilityPoint` genuinely works, it is compile-time, single-process, C++-ABI-only: no
  runtime loading, no independent versioning, no isolation (a crash in one capability kills the whole
  process via `segvHandler`'s `_Exit(1)`, `ghidra_process.cc:503-507`,
  [`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) §4/§6). A
  refactor that wants a genuinely safer "third-party extension" story (the kind a modern plugin
  ecosystem implies) is not just missing a `Rule` seam — the entire mechanism has a much narrower
  ceiling than "self-registering singleton" branding suggests.

### Severity

**High.** This is squarely in ticket 0015's Analysis Focus #1 ("does it actually enable third-party
extension today, or is it mostly used internally?") and the answer for the two extension categories a
refactor is most likely to care about — new `Rule`s and a uniform capability story — is no. It also
directly shapes what "the refactor" can promise as a design goal without first doing foundational work.

### Improvement options

1. **Generalize the `ModelRule` pattern to `Rule`/`Action` registration.** Build a `Rule`-equivalent of
   `DatatypeFilter`/`AssignAction`: a declarative match-pattern → transform pairing, data-driven from a
   spec file, with hand-written C++ `Rule`s as the (still-necessary) fallback for anything the
   declarative layer can't express.
   - *Pros*: reuses a pattern already proven in production on a structurally similar problem (storage
     assignment); avoids inventing a second, unrelated DSL from scratch; naturally gives `Rule`s the same
     "config vs. hardcoded" split the codebase already has for calling conventions.
   - *Cons*: `Rule`s pattern-match against arbitrary `PcodeOp` subgraphs, a much larger and more
     irregular match space than "which `Datatype`/qualifier applies to this parameter" — a naive port
     may not scale to the catalog's real complexity (see the abandoned `rulecompile.cc` attempt, which
     tried exactly this and didn't ship).
2. **Resurrect and finish `rulecompile.cc`'s pattern DSL** instead of building a new mechanism.
   - *Pros*: the design work and matcher infrastructure (`ConstraintGroup`/`UnifyState`) already exist;
     less net-new code than option 1.
   - *Cons*: it was abandoned for reasons this review didn't investigate (out of scope for 0006) —
     resurrecting it without first understanding *why* it was dropped risks repeating whatever made it
     non-viable; the undeclared-variable call site suggests it may not even compile cleanly today.
3. **Extend `CapabilityPoint` itself to cover `Rule`/`Action` registration** (a `RuleCapability`
   singleton that self-registers a `Rule` class, same idiom as the other four hierarchies) — purely
   about *compile-time* registration uniformity, not runtime/declarative extension.
   - *Pros*: smallest, most mechanical change; makes "how do I add a new X to this engine" one consistent
     answer across the whole codebase; doesn't require solving the harder declarative-matching problem.
   - *Cons*: doesn't address the actual gap (no way to add a rule without recompiling `coreaction.cc`-
     adjacent C++) — it only makes the *existing* limitation consistent, not smaller. Should be treated
     as a floor, not a solution, if the refactor's goal is genuine plugin extensibility.
4. **Route `ArchitectureGhidra` through `ArchitectureCapability::findCapability`** regardless of which
   broader option is chosen, so the mechanism's primary consumer actually exercises it.
   - *Pros*: small, isolated, low-risk fix; closes the "the main path doesn't use its own extension
     point" inconsistency independent of any larger decision.
   - *Cons*: `ArchitectureGhidra` is constructed with RPC-specific state (`ghidra_process.cc:181`) that
     the capability-discovery interface (built for file-based bootstrapping) may not cleanly model —
     needs its own small design pass, not a pure mechanical move.

### Open questions

- Does the refactor's scope (per root ticket 0001) actually want third-party `Rule` plugins, or is
  "extensibility" here really about the refactor team's own ability to add/reorder rules safely? The
  answer changes which of the options above is worth the investment.
- Is there an appetite to actually delete `rulecompile.cc` if option 1 or 3 is chosen instead of option
  2, to stop the dead-code/misleading-mechanism problem from persisting?
- Should `ModelRule`'s own gap (undocumented in `cspec.xml`, only forward-direction (`assignMap`) fully
  rule-driven — [`signature-calling-conventions.md`](../01-review/signature-calling-conventions.md) §1.2)
  be fixed *before* it's used as the template for a second declarative layer, so the template itself
  isn't propagating a known documentation debt?

---

## Flaw 2 — Test-coverage gaps make the refactor's own ordering a source of risk

### Observation

Per [`java-integration-testing.md`](../01-review/java-integration-testing.md) §3-4 (already
synthesized in [`00-index.md`](../01-review/00-index.md)'s coverage table), the three independent test
surfaces — `unittests` (7 files, narrow stateless utilities), `datatests` (89 files, black-box
regex-over-printed-C), `test.slow/java` (17 files, mostly UI-behavior tests) — leave whole subsystems
with **no characterization of their actual behavior**, only incidental protection from whichever
`datatests` scenario happens to depend on them still passing. Concretely: SSA/heritage construction has
no direct unit coverage of phi-placement or `HighVariable` merge decisions at all; the Action/Rule
engine (500 classes) has zero unit-level tests of any individual `Rule`; there is no full-function
golden-output test anywhere in the tree, so a change that's locally plausible per matched line can
silently degrade whole-function formatting; and nothing checks that the hand-duplicated Java/C++
`ElementId` tables actually agree (§1.2, detailed in Flaw 3 below).

This matters specifically *for sequencing*, which is ticket 0015's Analysis Focus #2: a refactor that
touches a subsystem with only black-box coverage can silently change *which* equally-plausible variable
split, cast placement, or structuring shape comes out, without any test noticing — the failure mode
isn't "tests fail," it's "tests keep passing while behavior quietly changes," which is worse than a
visible gap for planning purposes because the risk doesn't show up as a red build.

### Impact

- Refactoring SSA/heritage or the Action/Rule engine **first**, before adding characterization tests,
  means the refactor's own correctness is being judged by test surfaces admitted (in the docs already
  produced by this review) to not exercise the thing being changed.
- The printer's "no golden full-text test" gap ([`java-integration-testing.md`](../01-review/java-integration-testing.md)
  §4, Expression/cast/printer entry) means any refactor to `PrintC`/`Emit`/`EmitPrettyPrint` — even one
  with no functional regressions — has no way to confirm it *didn't* change formatting output at scale
  short of manual diffing.
- The `datatests` format itself (regex + occurrence bounds over printed text,
  [`java-integration-testing.md`](../01-review/java-integration-testing.md) §3.1) means even the
  *existing* protection for untested subsystems is coarse: a failure reports which named property
  failed, not an expected-vs-actual diff, which slows down characterizing what a refactor actually
  changed even when a test does catch something.
- This finding is the direct input to ticket 0015's required Refactor Sequencing Risk table below —
  every row's "coverage" axis is sourced from this gap analysis.

### Severity

**High.** Untested code isn't just a quality gap here — given the refactor's own stated need to make
"safe, incremental change" possible, it is a direct blocker on ordering: subsystems in this state cannot
be refactored first without extra up-front work, which changes the refactor's critical path.

### Improvement options

1. **Characterization-test generation before touching a subsystem.** For each subsystem ranked
   high-risk in the table below, run the existing `datatests` corpus (or a purpose-built larger corpus)
   through the *current* engine and capture full-function output (not just the `<stringmatch>`
   fragments) as a golden baseline, then diff post-refactor output against it.
   - *Pros*: directly closes the "no full-text golden test" gap; doesn't require understanding *why*
     current output is correct, only that it doesn't change unintentionally — a good fit for a large,
     poorly-specified legacy engine; can be automated as a one-time harness built once and reused for
     every subsequent subsystem.
   - *Cons*: a golden-diff baseline pins current behavior including any existing bugs, so it needs a
     documented process for "this diff is an intentional fix" vs. "this diff is a regression"; doesn't
     substitute for real unit tests of internal invariants (e.g. phi-placement correctness), only for
     detecting *that something changed*.
2. **Add targeted unit tests for the two most acute unit-level gaps** — a `Funcdata`/`Heritage` harness
   that asserts on resulting SSA shape (phi placement, `HighVariable` merge decisions) directly, and a
   per-`Rule` harness that instantiates one `Rule` and asserts it fires/doesn't fire on a constructed
   `PcodeOp` pattern in isolation (mirroring `testtypes.cc`'s approach for the type system, the
   subsystem's own best-covered precedent —
   [`java-integration-testing.md`](../01-review/java-integration-testing.md) §3.2).
   - *Pros*: real regression protection at the unit level, not just a diff; follows an existing,
     working pattern in this codebase rather than inventing one.
   - *Cons*: building a `Funcdata` or driving a `Rule` in isolation from `unittests/` may require
     non-trivial test scaffolding (a minimal `Architecture`, synthetic p-code) that doesn't currently
     exist for these two subsystems specifically; higher up-front cost per subsystem than option 1.
3. **Sequence the refactor itself around this table** (i.e. treat the table below as a gating input to
   epics 0020/0023, not just an artifact): require characterization tests as an explicit prerequisite
   task for any subsystem in the top risk tier before its refactor task is scheduled.
   - *Pros*: turns this finding into an actual process control rather than a one-time recommendation;
     cheap to adopt (it's a planning decision, not new code).
   - *Cons*: only as effective as the discipline to actually enforce it across later epics — a
     documentation-only mitigation if not paired with 1 or 2.

### Open questions

- Should characterization-test generation (option 1) be its own dedicated task in a later epic, given
  its cross-cutting value, rather than repeated per-subsystem?
- Is there budget/appetite to build the `Funcdata`/`Heritage` and per-`Rule` unit-test scaffolding
  (option 2) before refactor work starts, or is option 1's cheaper diff-based approach the realistic
  floor given the refactor's timeline?
- The `datatests` format's own weakness (regex fragments, no diff on failure) is itself arguably a flaw
  in the *existing* test infrastructure, separate from coverage gaps — is fixing that in scope for this
  refactor, or purely a "nice to have" alongside it?

---

## Flaw 3 — No wire-protocol version handshake; the Java/C++ contract is enforced only by same-build discipline

### Observation

Per [`java-integration-testing.md`](../01-review/java-integration-testing.md) §1.4 and
[`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) §1: the C++
decompiler and its Java caller speak a hand-rolled framed byte protocol using the `PackedEncode`/
`PackedDecode` binary encoding (not XML, despite XML-flavored naming — `marshal.hh:440-504`), and the
element/attribute names and numeric IDs behind it are **hand-duplicated in two independently maintained
source files**: Java's `ElementId.java` (e.g. `ELEM_DOC = new ElementId("doc", 229)`) and scattered C++
globals (e.g. `ghidra_process.cc:74`). Nothing cross-checks that the two lists agree — a rename/renumber
on one side with a stale update on the other produces a silent wrong decode at runtime, not a build or
test failure ([`java-integration-testing.md`](../01-review/java-integration-testing.md) §1.2). There is
**no general wire-protocol version handshake at all**: the closest thing,
`DecompInterface.getMajorVersion()`/`getMinorVersion()`, only covers the separate
signature/feature-vector module (`signature.cc:33-40`), not the protocol as a whole
(`DecompInterface.java:911-981`, same doc §1.4). In practice, compatibility is maintained purely by
**always building and shipping the `decompile` executable from the same source tree/commit as the Java
code** (`DecompileProcessFactory.java:65-67`) — there is no runtime detection of a Java build talking to
a decompiler binary built from different sources.

This is exactly the ABI-stability question ticket 0015's Analysis Focus #3 asks about: the practical
contract is "same-commit build," not a negotiated or versioned interface, and the same
hand-duplicated-list risk pattern recurs in the `Makefile`'s source-file list vs. `buildNatives.gradle`'s
source-file list for the C++ build itself — see Flaw 4.

### Impact

- A refactor that changes anything in the wire format (adds/renames an `ElementId`, changes a command's
  argument shape) has **no automated check** that both sides were updated consistently — the failure
  mode is a silent wrong decode at runtime, discovered (if at all) by a confusing downstream symptom,
  not a clear protocol error. This is a materially worse failure mode than a version-mismatch exception.
- Any refactor phase that wants to ship the C++ engine and Java client on different cadences (e.g.
  incremental rollout, a staged migration) is currently unsupported by the protocol itself — "always the
  same commit" is a hard operational constraint baked into the architecture, not a policy that could be
  relaxed without protocol changes.
- Root ticket 0001's non-goals reportedly "freeze the wire protocol" without distinguishing which of the
  two encodings (the binary `PackedEncode`/`PackedDecode` hot path vs. the XML spec-file/console-
  save format) that covers — [`00-index.md`](../01-review/00-index.md) "What's genuinely config-driven"
  section flags this ambiguity already; it directly affects how much latitude a refactor has here and
  should be resolved before subsystem refactor tasks that touch marshaling begin.

### Severity

**Medium-High.** The silent-corruption failure mode (not just "no version check") elevates this above a
routine testing gap — see [`00-index.md`](../01-review/00-index.md)'s own framing: "a silent-corruption
risk, not just a coverage gap." It is not immediately blocking (the current same-build discipline works
today), but it constrains how the refactor can be staged and rolled out.

### Improvement options

1. **Generate both `ElementId`/`AttributeId` tables from one source of truth** (a shared schema file,
   code-generated into both the Java `ElementId.java` records and the C++ globals) instead of hand
   duplication.
   - *Pros*: eliminates the drift risk at the root rather than adding a check for it; a one-time
     investment that also documents the wire format in one place (which today exists only as scattered
     source, per the same finding).
   - *Cons*: real migration cost — every current call site on both sides references the existing
     generated/hand-written symbols; needs a compatibility bridge or a coordinated one-time cutover.
2. **Add a build-time or startup-time consistency check** between the two tables (e.g. C++ emits its
   table, a test/tool diffs it against Java's) without unifying the source of truth.
   - *Pros*: much smaller lift than option 1; converts "silent wrong decode" into "loud build/CI
     failure," which is the single biggest severity reduction available here.
   - *Cons*: doesn't reduce the ongoing maintenance burden of keeping two lists in sync by hand, only
     catches mistakes after the fact; still two lists to edit for every protocol change.
3. **Add a real protocol version handshake** (distinct from the existing signature-module version
   check) exchanged on `registerProgram`, with an explicit compatibility policy (exact match required,
   or a documented forward/backward-compat window).
   - *Pros*: directly enables any future decoupling of Java/C++ release cadence; turns "wrong build
     paired together" into a clear, actionable error instead of undefined behavior.
   - *Cons*: only prevents *gross* mismatches (different source trees) — it doesn't, by itself, catch a
     same-version-but-drifted-table bug the way option 1/2 would, since the table drift can happen
     within what both sides consider "the same version" if the bump isn't paired with every ID change.
4. **Explicitly scope root ticket 0001's "wire protocol freeze"** to name `PackedEncode`/`PackedDecode`
   specifically (vs. the XML spec/save-file format), resolving the ambiguity
   [`00-index.md`](../01-review/00-index.md) already flags.
   - *Pros*: essentially free (a documentation/scoping fix); removes a source of confusion for every
     later refactor task that touches marshaling.
   - *Cons*: doesn't address the underlying technical risk (options 1-3 do); this is a prerequisite
     clarification, not a fix.

### Open questions

- Is decoupling Java/C++ release cadence actually a goal of this refactor, or is "always same-commit" an
  acceptable permanent constraint? This determines whether option 3 is worth the investment.
- Which of options 1/2 is more compatible with the "wire protocol is frozen" non-goal — does generating
  both tables from one schema count as changing the protocol (the bytes on the wire don't change, only
  how the tables are authored), or does the freeze need to be interpreted narrowly enough to exclude
  this kind of internal-authoring-mechanism change?
- Should this be resolved before or in parallel with Flaw 1's extension-model work, given both involve
  touching `Capability`-adjacent registration code (`GhidraCapability`'s command set overlaps with the
  protocol surface)?

---

## Flaw 4 — Two independent, undocumented C++ build paths; the fast local-iteration loop exists but isn't discoverable

### Observation

See the dedicated section below ("Can the decompiler be built/run standalone for fast local iteration?")
for the full evidence. In short: the repository contains **two entirely separate build systems** for the
same C++ source tree, and neither documents the other's existence or purpose anywhere in-repo:

- **Gradle** (`Ghidra/Features/Decompiler/build.gradle` → `buildNatives.gradle`, a `NativeExecutableSpec`
  model) builds exactly the two executables Ghidra ships and Java invokes: `decompile` (via
  `ghidra_process.cc`'s RPC-driven `main`) and `sleigh`. Its C++ source list for `decompile`
  **explicitly excludes** the interactive console/test front end — `callgraph.cc`, `ifacedecomp.cc`,
  `ifaceterm.cc`, and `interface.cc` are present in the source list only as commented-out lines marked
  `// uncomment for debug` (`buildNatives.gradle:136-139`). This is the only build path exercised by
  the project's normal Gradle-based build/CI flow.
- **A classic, hand-maintained `Makefile`** (`Ghidra/Features/Decompiler/src/decompile/cpp/Makefile`)
  defines a completely independent set of targets, invoked directly with `make` (`g++`/`bison`/`flex`,
  no Gradle or JVM involved at all): `decomp_opt`/`decomp_dbg` link `consolemain.cc` into a standalone
  interactive console binary (`main`, `consolemain.cc:176`, builds an `IfaceTerm("[decomp]> ", ...)`
  REPL, `consolemain.cc` ~213); `decomp_test_dbg` (target alias `test`, `Makefile:264-268`) links
  `test.cc` plus everything under `../unittests/*.cc` into the binary that runs both `unittests` and
  `datatests` (`Makefile:67-69,128-130,150`, matching the harness
  [`java-integration-testing.md`](../01-review/java-integration-testing.md) §3.1-3.2 describes); and
  `ghidra_dbg`/`ghidra_opt` build the RPC-driven binary Makefile-style, with the debug variant
  additionally linking in the console command modules (`callgraph ifacedecomp testfunction ifaceterm
  interface`, `Makefile:132`) that Gradle's build permanently excludes.

The Makefile's own source-file lists (`CORE`/`DECCORE`/`GHIDRA` macros, `Makefile:86-101`) were checked
against `buildNatives.gradle`'s `decompile` source list and currently agree (every file Gradle compiles
also appears in the Makefile's variable set) — but, as with the `ElementId` tables in Flaw 3, this is
two hand-maintained lists describing the same file set with **nothing enforcing they stay in sync**
(the Makefile is not invoked anywhere in the Gradle build or any CI config found in this repo — confirmed
by searching for `decompile/cpp` references outside `srcDir`/`source` declarations in `.gradle`/CI
files). Nothing in either build file, nor any README under `Ghidra/Features/Decompiler/`, documents that
the Makefile exists, what `decomp_opt`/`test` do, or that it is the fast local-iteration path — a
developer discovering it has to read the Makefile itself or infer it from citations in the review docs
(e.g. [`action-rule-engine.md`](../01-review/action-rule-engine.md) already cites `Makefile:114,125,133`
for unrelated reasons, incidentally proving the file is live and current, not vestigial).

### Impact

- The fast, Gradle/JVM-independent local-iteration loop this refactor needs (per ticket 0015's Analysis
  Focus #4) **does exist**, but is effectively undiscoverable without either already knowing Ghidra's
  decompiler development history or reading raw source — a real onboarding/velocity cost for a refactor
  that specifically depends on being able to iterate quickly and validate incrementally.
- The two build systems' file lists are a duplicated-source-of-truth risk structurally identical to
  Flaw 3's `ElementId` tables: currently in sync, unchecked, and only as reliable as whoever remembers to
  update both when adding a new `.cc` file. A refactor that adds/removes/splits source files (plausible
  for a subsystem restructuring effort) must remember to update both lists, with nothing to catch a
  miss beyond a Makefile-only build breaking (silently, since it's not in CI) at some later, unrelated
  point.
- Gradle's `decompile` target's exclusion of `ifacedecomp.cc`/`interface.cc` means the RPC-driven
  production binary and the interactive console/test binary are **not validated against each other by
  the normal build** — the debug Makefile target that includes console modules (`ghidra_dbg`) is a
  different, less-used configuration than what Gradle ships, so a change that only breaks the
  console/test integration could pass a Gradle-only validation loop.

### Severity

**Medium.** This does not block the refactor (a working fast loop exists), but it is a real
"build/tooling constraint on incremental validation" per ticket 0015's own framing, and the
undocumented-ness compounds every other subsystem refactor task's ramp-up cost.

### Improvement options

1. **Document the Makefile-based loop** (a short `README`/`CONTRIBUTING` note under
   `Ghidra/Features/Decompiler/src/decompile/`, or a section in this refactor's own working docs)
   covering `make decomp_opt` (console REPL for manual exploration), `make test` (runs `unittests` +
   `datatests`), and `make ghidra_dbg` (RPC binary with debug/console modules linked in) as the
   recommended fast-iteration entry points, distinct from the Gradle-built production binary.
   - *Pros*: nearly free; immediately raises discoverability without touching any build logic; a good
     candidate for a very early, low-risk task in the refactor's own execution.
   - *Cons*: doesn't address the underlying duplicated-file-list risk, only the discoverability problem.
2. **Wire the Makefile's `test` target into CI** (or a Gradle task that shells out to it) so
   `unittests`/`datatests` actually run on every change, and so a Makefile/Gradle file-list drift would
   surface as a build failure rather than staying latent.
   - *Pros*: closes the duplicated-list risk the same way Flaw 3's option 2 does for `ElementId`;
     turns the existing `datatests`/`unittests` suites (currently, per
     [`java-integration-testing.md`](../01-review/java-integration-testing.md) §3, apparently only
     run manually/ad hoc) into an enforced regression gate — independently valuable for Flaw 2 as well.
   - *Cons*: requires investigating why this isn't already wired in (platform/toolchain availability in
     CI, `bison`/`flex` dependencies, etc. — out of scope for this review to diagnose); moderate,
     not trivial, effort.
3. **Generate the Makefile's file lists from the same source as `buildNatives.gradle`'s** (or vice
   versa) rather than maintaining both by hand — the same "single source of truth" idea as Flaw 3's
   option 1, applied to build-file source lists instead of wire-protocol IDs.
   - *Pros*: eliminates the drift risk at the root.
   - *Cons*: Gradle's `NativeExecutableSpec` DSL and Make's variable-substitution syntax are different
     enough that a shared generator adds real tooling complexity for a relatively low-frequency change
     (new `.cc` files are added rarely); may not be worth it unless option 2 already surfaces actual
     drift incidents.

### Open questions

- Is the Makefile path still officially supported by the Ghidra project (vs. a legacy artifact kept
  working by convention), and does that affect whether the refactor should invest in wiring it into CI
  (option 2) vs. simply documenting it (option 1) as an unofficial convenience?
- Should validating a subsystem refactor task standardize on `make test` (fast, C++-only) as a required
  first gate before a full Gradle/Ghidra integration build, given the latter is dramatically slower and
  requires the JVM toolchain? This seems like a natural process recommendation for later planning epics.

---

## Refactor Sequencing Risk

Required by ticket 0015: rank the eight subsystems reviewed in epic 0002 by **(test coverage) x
(coupling)**. Coverage is pulled directly from
[`java-integration-testing.md`](../01-review/java-integration-testing.md) §4's summary table
(reproduced in [`00-index.md`](../01-review/00-index.md)); coupling is this document's own judgment,
cited against the relevant review-doc sections. **Risk rank 1 = highest risk** (refactor last, and only
after adding characterization tests per Flaw 2); higher-numbered ranks are progressively safer to
refactor earlier.

| Rank | Subsystem (review ticket) | Test coverage today | Coupling (why) | Why this rank |
|---|---|---|---|---|
| 1 | **SSA / Varnode / Heritage** (0004) | None at unit level; only incidental, indirect `datatests` coverage — "no direct unit coverage at all" ([`java-integration-testing.md`](../01-review/java-integration-testing.md) §4) | **Very high, foundational.** `Varnode`→`HighVariable`→`Symbol` is consumed by the type system, the Action/Rule engine, structuring, the printer, *and* the Java-side editing model (`HighFunctionDBUtil` commit-back operates directly on `HighVariable`/`Cover`/`Merge` decisions — [`java-integration-testing.md`](../01-review/java-integration-testing.md) §2). A silent regression here (a plausible-but-different variable split) has no downstream test surface positioned to catch it specifically as an SSA/heritage bug — it would surface, if at all, as an unrelated-looking `datatests` failure elsewhere in the pipeline. | Worst combination: zero direct coverage of the subsystem every other subsystem's correctness implicitly assumes. |
| 2 | **Action/Rule engine** (0006) | Zero unit-level tests of any individual `Rule`; entire coverage is 89 black-box `datatests` files exercising *some* subset of ~500 classes ([`java-integration-testing.md`](../01-review/java-integration-testing.md) §4) | **Very high, by surface area and ordering fragility.** ~500 classes, 162 hand-wired into one function (`coreaction.cc:5835`); rule *ordering/interaction* is a known risk category flagged for the companion flaw task 0013; `ActionPool` re-scans from index 0 whenever a rule changes an op's opcode within one visit ([`action-rule-engine.md`](../01-review/action-rule-engine.md) §1), so effects compose in ways that are easy to perturb. | Same "no unit coverage" floor as SSA/heritage, but with far more individual moving parts (500 vs. one conceptual pass) and a fixed-point/ordering-dependent execution model that makes isolated verification harder even after the fact. |
| 3 | **Architecture / pipeline / wire protocol** (0003) | `testmarshal.cc` covers the byte format in isolation; ElementId-table agreement is unchecked; no auto-respawn/state-replay test ([`java-integration-testing.md`](../01-review/java-integration-testing.md) §4) | **Very high.** `Architecture` is a God object owning every other major subsystem as a raw member pointer ([`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) §2); it also *is* the Java/C++ ABI boundary (Flaw 3) — a change here has the widest possible blast radius by construction, touching startup order, every subsystem's factory wiring, and the wire format simultaneously. | Coupling is arguably the single highest of any subsystem (literally everything else hangs off it), but coverage, while thin, is not *zero* the way ranks 1-2 are (the marshal format itself is unit-tested) — placed just below the two subsystems with no unit coverage at all. |
| 4 | **Java↔C++ integration & test infra itself** (0010) | Process-plumbing smoke test only (`DecompilerTest`, 61 lines); most `test.slow` files assert UI/token behavior, not decompiler correctness ([`java-integration-testing.md`](../01-review/java-integration-testing.md) §3.3-4) | **High.** Crosses the process/language boundary that every other subsystem's output must pass through to reach the program database (commit-back, retype/rename, `FillOutStructureCmd` — [`java-integration-testing.md`](../01-review/java-integration-testing.md) §2); changing this layer risks breaking auto-analysis and UI editing actions even if the C++ core is untouched. | High coupling (it's the seam every subsystem's result must cross) but somewhat more locally testable than ranks 1-3 (JUnit infrastructure already exists, just under-used for correctness) — hence one tier down. |
| 5 | **Control-flow structuring** (0007) | Good "by shape name" `datatests` coverage (loop/switch/if scenario files) but no goto-fallback/structuring-quality regression tracking; `structureGraph`'s direct Java entry point is untested ([`java-integration-testing.md`](../01-review/java-integration-testing.md) §4) | **Medium.** Deliberately isolated by design — `ActionBlockStructure` copies the CFG into a disposable second graph (`BlockCopy` wrappers) rather than mutating `Funcdata::bblocks` in place ([`control-flow-structuring.md`](../01-review/control-flow-structuring.md) §1.1) — but it consumes `JumpTable` output and directly determines the printer's structural input. | Coverage is meaningfully better than ranks 1-4 (real scenario-shaped tests exist), and the two-graph isolation genuinely bounds coupling compared to subsystems that mutate shared SSA/type state directly. |
| 6 | **Expression / cast logic & printer** (0009) | Targeted `CastStrategy`/cast-insertion unit tests (`testtypes.cc`) plus edge-case `datatests`, but **no full-function golden-output test anywhere in the tree** — the single biggest structural gap called out in [`java-integration-testing.md`](../01-review/java-integration-testing.md) §4 | **Medium.** Purely downstream/terminal — nothing later in the pipeline depends on `PrintC`'s output, so a printer bug can't cascade *into* other subsystems' correctness the way an SSA/rule-engine bug can. But `PrintJava` is only a 9-method subclass of the ~3500-line `PrintC` ([`00-index.md`](../01-review/00-index.md) "Known-surprising findings"), so any `PrintC` change risks two output languages at once, not one. | Downstream-only coupling keeps this out of the top tier despite the printer's own coverage gap being uniquely severe in kind (whole-function formatting has zero regression protection) — the risk here is real but structurally contained to "wrong-looking output," not "wrong program semantics." |
| 7 | **Signature / calling-convention recovery** (0008) | Real, substantial unit coverage — `testfuncproto.cc` (682 lines), `testparamstore.cc` (369 lines); good `datatests` coverage of ABI/return/override scenarios; only the Java-side commit-back path is untested in-module ([`java-integration-testing.md`](../01-review/java-integration-testing.md) §4) | **Medium-high.** Touches the type system (`DatatypeFilter`), the Action/Rule engine (`ActionDefaultParams` consumes `FuncProto`), and Java commit-back ([`signature-calling-conventions.md`](../01-review/signature-calling-conventions.md) §1.1, §4) — genuinely cross-cutting — but cleanly layered behind the `ProtoModel`/`FuncProto` split, and the layer that varies per-ABI (`ModelRule`) is already the more extensible, declarative half of this subsystem (same doc §6). | The best-tested subsystem outside the type system pulls this down from where its coupling alone would place it — real unit coverage substantially de-risks touching it. |
| 8 | **Type system** (0005) | Best-covered subsystem at the unit level — `testtypes.cc` unit-tests `Datatype` ordering/comparison, cast-insertion decisions, and enum matching against a real (if minimal) `Architecture`; good `datatests` coverage of composite/array access ([`java-integration-testing.md`](../01-review/java-integration-testing.md) §4) | **High but well-encapsulated.** `TypeFactory` is referenced broadly (typelock semantics consumed by the rule engine, `CastStrategy` consumed by the printer, `DatatypeFilter` consumed by signature recovery — [`type-system.md`](../01-review/type-system.md) §2 and cross-references throughout) but accessed through one consistent, canonicalizing interface (`TypeFactory`) rather than scattered ad hoc coupling. | Lowest risk of the eight *despite* broad usage: the combination of the strongest unit coverage in the codebase and a single well-defined access point (rather than diffuse coupling) makes this the safest subsystem to refactor first, or to use as the template for characterization-test approaches applied to the riskier subsystems above (Flaw 2, option 2). |

**How to read this table for sequencing**: ranks 1-3 (SSA/heritage, Action/Rule engine,
Architecture/pipeline) should not be refactored without characterization tests in place first (Flaw 2);
rank 4 (Java integration) is the seam that validates *whether* a change to any of ranks 1-3 broke
something externally visible, so it benefits from investment in parallel rather than after; ranks 5-8 are
progressively safer to sequence earlier, with the type system (rank 8) the strongest candidate for an
early refactor pass or as a worked example for extending unit coverage to the higher-risk subsystems.

---

## Can the decompiler be built/run standalone for fast local iteration?

**Yes — but only via an undocumented, non-Gradle build path**, and this is itself Flaw 4 above. Evidence:

- Gradle's `buildNatives.gradle` builds the *production* `decompile` binary — the one
  `DecompileProcessFactory.get()` actually spawns (`DecompileProcessFactory.java:65-67`,
  [`java-integration-testing.md`](../01-review/java-integration-testing.md) §1.1) — from a fixed,
  explicit source list that **excludes** the interactive console/test modules
  (`callgraph.cc`/`ifacedecomp.cc`/`ifaceterm.cc`/`interface.cc`, all present only as commented-out
  `// uncomment for debug` lines, `buildNatives.gradle:55-145`, exclusions at lines 136-139). Building
  and running *this* binary standalone (outside the full Ghidra/Gradle distribution) is possible but
  gives you only the RPC server half — it has no REPL, no `datatests`/`unittests` runner, and expects a
  Ghidra client speaking the framed byte protocol on its stdin/stdout, not a human at a terminal
  (`ghidra_process.cc:511`, `main`).
- The standalone, Gradle/JVM-independent loop is the classic `Makefile` at
  `Ghidra/Features/Decompiler/src/decompile/cpp/Makefile`, run directly with `make <target>` (`g++`
  `-std=c++11`, `bison`, `flex` — no Java toolchain involved at all):
  - `make decomp_opt` (or `decomp_dbg` for a `-DCPUI_DEBUG`/`__TERMINAL__` build,
    `Makefile:109,124-129,257-261`) links `consolemain.cc` — an interactive `[decomp]>` console REPL
    (`main`, `consolemain.cc:176`, `IfaceTerm`, `consolemain.cc` ~213) — against the full core+SLEIGH
    engine, independent of any Ghidra process or Java code. This is a genuine fast local-iteration loop:
    edit a `.cc` file, `make decomp_opt`, exercise the engine interactively from the console.
  - `make test` (alias for `decomp_test_dbg`, `Makefile:264-268`) links `test.cc` plus every file under
    `../unittests/*.cc` and, when run, executes both the `unittests` suite and the `datatests` corpus
    (`Makefile:67-69,128-130,150`) — the exact harness
    [`java-integration-testing.md`](../01-review/java-integration-testing.md) §3.1-3.2 documents. This
    is the fastest way to run the existing regression suite: a native C++ build and run, no Gradle, no
    JVM, no Ghidra installation.
  - `make ghidra_dbg`/`make ghidra_opt` build the RPC-driven binary the Makefile's own way; the debug
    variant additionally links the console command modules Gradle excludes (`callgraph ifacedecomp
    testfunction ifaceterm interface`, `Makefile:132`), making it the closest thing to "the shipped
    binary, but with the debug console attached."
- This path is **not wired into the Gradle build or any CI configuration found in this repository** —
  searching all `.gradle`/CI files for `decompile/cpp` references outside plain `srcDir`/`source`
  declarations found nothing that invokes `make`. It is also not documented anywhere under
  `Ghidra/Features/Decompiler/` (no README references it); its existence and continued relevance are
  only inferable from the Makefile itself and from other review documents' incidental citations of it
  (e.g. [`action-rule-engine.md`](../01-review/action-rule-engine.md) cites `Makefile:114,125,133` for
  an unrelated finding, which is corroborating evidence the file is current, not vestigial).

**Bottom line for the refactor**: a fast, standalone, full-rebuild-avoiding local iteration loop exists
today (`make decomp_opt` for interactive exploration, `make test` for the regression suite), and should
be the refactor team's default inner loop for C++-only changes — but it needs to be documented (Flaw 4,
option 1) and ideally wired into CI (Flaw 4, option 2) before the team can rely on it with confidence
that it stays in sync with the Gradle-built production binary.

---

## Related documents

- [`.tasks/0015`](../../.tasks/0015-task-flaws-architecture-extensibility.md) — the ticket this document
  satisfies.
- [`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) (0003),
  [`java-integration-testing.md`](../01-review/java-integration-testing.md) (0010),
  [`signature-calling-conventions.md`](../01-review/signature-calling-conventions.md) (0008),
  [`action-rule-engine.md`](../01-review/action-rule-engine.md) (0006),
  [`ssa-varnode-heritage.md`](../01-review/ssa-varnode-heritage.md) (0004),
  [`type-system.md`](../01-review/type-system.md) (0005),
  [`control-flow-structuring.md`](../01-review/control-flow-structuring.md) (0007),
  [`expression-cast-printer.md`](../01-review/expression-cast-printer.md) (0009) — the eight subsystem
  reviews this document draws its coverage/coupling judgments from.
- [`00-index.md`](../01-review/00-index.md) — the synthesized review narrative; source of the
  test-coverage table and the initial extension-model/config-vs-hardcoded findings this document
  verifies and deepens.
- [`../00-overview/glossary.md`](../00-overview/glossary.md) — shared terminology.
- Sibling flaw documents (0012-0014, when produced) and
  [`.tasks/0011`](../../.tasks/0011-epic-flaw-analysis-and-improvements.md)'s
  `.documents/02-flaws/00-index.md` — cross-flaw prioritization, not owned by this document.
