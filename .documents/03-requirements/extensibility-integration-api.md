# Requirements: Extensibility, API & Integration/Testability

Produced by [`.tasks/0019`](../../.tasks/0019-task-requirements-extensibility-integration-api.md), part
of the requirements-definition epic [`.tasks/0016`](../../.tasks/0016-epic-requirements-definition.md),
under root [`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md). Format follows 0016's
contract: each requirement has a stable `REQ-ARCH-<n>` ID, a MUST/SHOULD/MAY priority (RFC-2119 style),
a one-line statement, rationale tracing to a specific flaw finding from epic 0011, and — where feasible —
an acceptance-test sketch.

Primary source: [`architecture-extensibility.md`](../02-flaws/architecture-extensibility.md) (0015).
Also draws on [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md) (0013)
Flaw 2, [`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) (0003),
[`java-integration-testing.md`](../01-review/java-integration-testing.md) (0010), and
[`../00-overview/glossary.md`](../00-overview/glossary.md). Flaw citations use the form
`(0015 Flaw N; 00-index.md #M)`, where `#M` is the consolidated priority-table row in
[`../02-flaws/00-index.md`](../02-flaws/00-index.md).

**Correction carried forward from 0003/0010**: the Java↔C++ wire protocol's hot path is a compact
**binary** TLV encoding (`PackedEncode`/`PackedDecode`, `marshal.hh:440-504`), not XML. XML is used only
for `.pspec`/`.cspec` spec text and the standalone console's save/restore format. Every requirement below
that says "wire protocol" means the binary encoding unless stated otherwise.

## How to read this document

Requirements are grouped into four clusters matching the ticket's starting list: (A) wire-protocol
compatibility, (B) subsystem testability, (C) extension-model uniformity, (D) standalone build/run
tooling. A traceability table closes the document.

---

## Cluster A — Wire-protocol compatibility

### REQ-ARCH-1 — Wire-protocol compatibility bar (explicit)

**Priority:** MUST

**Statement:** Refactoring the C++ decompiler's *internals* (SSA construction, the Action/Rule engine,
structuring, printing, type system) MUST NOT require any simultaneous change to the Java front end's
wire-protocol handling, **unless** the specific refactor step is explicitly declared — in its own task
description — to bump the protocol. A protocol bump is acceptable only when **all** of the following
hold:

1. The new information genuinely cannot be represented in the existing `PackedEncode`/`PackedDecode`
   element/attribute schema (i.e. it is not satisfiable by adding new `ElementId`/`AttributeId` values
   to the existing scheme — see REQ-ARCH-3).
2. The change is additive: existing `ElementId`/`AttributeId` numeric values are never renumbered or
   reused for a different meaning within a protocol version: new ids are appended, old ones are never
   silently repurposed.
3. The change ships together with the version-handshake mechanism required by REQ-ARCH-2, so an old
   Java client paired with a new `decompile` binary (or vice versa) fails fast with a clear
   version-mismatch error rather than a silent wrong decode.
4. The change is scoped to the smallest surface that needs it (e.g. one new command's response shape),
   not a wholesale re-encoding of the marshal format.

Any refactor step that does not meet this bar MUST keep the wire format byte-for-byte compatible: same
element/attribute ids, same command names, same response shapes.

**Rationale:** Directly answers ticket 0019's required "state the bar for when a version bump is
acceptable." Grounded in 0015 Flaw 3's finding that the practical contract today is "same-commit build,"
not a negotiated interface (`java-integration-testing.md` §1.4), and in root ticket 0001's ambiguous
"wire protocol freeze" non-goal, which 0015 Flaw 3 flags as needing disambiguation
(`architecture-extensibility.md` Flaw 3, Impact; 00-index.md #9). Without this bar, a refactor task could
plausibly justify a protocol change by convenience rather than necessity, defeating the "refactor
internals without forcing lockstep Java changes" goal the ticket asks for.

**Acceptance-test sketch:** For any refactor PR touching `decompile/cpp` marshal-adjacent code, CI
diffs the `ElementId`/`AttributeId` table (per REQ-ARCH-3) before/after; a diff that removes or
renumbers an existing id, without an accompanying version bump per REQ-ARCH-2, fails the build. A
manual review checklist item ("does this PR require a protocol version bump? If yes, cite which of the
four conditions above justify it") is required in the PR template for any change under
`decompile/cpp/*marshal*`, `ghidra_process.cc`, or `ElementId.java`.

---

### REQ-ARCH-2 — Wire-protocol version handshake

**Priority:** MUST

**Statement:** `registerProgram` MUST exchange an explicit wire-protocol version (distinct from the
existing signature/feature-vector `getMajorVersion()`/`getMinorVersion()` check, which only covers the
function-signature module per `signature.cc:33-40`) between the Java client and the `decompile` process,
and the C++ side MUST reject the connection with a clear, catchable error when the versions are
incompatible, rather than proceeding to decode with mismatched `ElementId` tables.

**Rationale:** Directly answers 0015 Flaw 3 (00-index.md #9): "no general wire-protocol version
handshake at all... in practice, compatibility is maintained purely by always building and shipping the
`decompile` executable from the same source tree/commit" (`architecture-extensibility.md` Flaw 3,
Observation). The failure mode today is **silent corruption**, not a version error — a strictly worse
outcome for any refactor that might stage Java/C++ changes across separate commits or rollouts. This is
listed as Improvement option 3 in the source flaw document.

**Acceptance-test sketch:** A new `unittests`/`test.slow` case starts a `DecompileProcess` against a
`decompile` binary built to report a deliberately-mismatched protocol version and asserts
`registerProgram` throws a typed, descriptive exception (not a downstream decode failure). A second test
confirms a matched-version pair registers successfully with no behavior change from today.

---

### REQ-ARCH-3 — Cross-checked or single-sourced `ElementId`/`AttributeId` tables

**Priority:** SHOULD

**Statement:** The Java (`ElementId.java`) and C++ (scattered globals, e.g. `ghidra_process.cc:74`)
element/attribute id tables SHOULD be generated from one shared schema, or, at minimum, cross-checked by
an automated build/CI step that fails loudly on any name/id mismatch between the two lists.

**Rationale:** Directly answers 0015 Flaw 3 (00-index.md #9): "the element/attribute names and numeric
IDs behind it are hand-duplicated in two independently maintained source files... nothing cross-checks
that the two lists agree" (`architecture-extensibility.md` Flaw 3, Observation, citing
`java-integration-testing.md` §1.2). This is a prerequisite for REQ-ARCH-1's "additive-only" bar to be
enforceable rather than aspirational — without a diffable table, "did this PR remove/renumber an id" is
a manual, error-prone judgment call. Marked SHOULD rather than MUST because the *build-time check*
(Flaw 3 option 2, the cheaper mitigation) is sufficient to satisfy REQ-ARCH-1's enforcement need; the
*single-source-of-truth codegen* (Flaw 3 option 1) is the more complete fix but carries real migration
cost across every call site on both sides, and the choice between them is an implementation decision
this requirements doc does not need to force.

**Acceptance-test sketch:** A CI job (or `make`/Gradle task) extracts both tables (e.g. by parsing
`ElementId.java` and grepping/parsing the C++ globals, or by consuming a generated schema if option 1 is
chosen) and fails if any name→id pair present on one side is absent, differently-numbered, or
differently-named on the other.

---

## Cluster B — Subsystem testability

### REQ-ARCH-4 — Subsystem boundaries MUST be independently unit-testable

**Priority:** MUST

**Statement:** Each of the four highest-risk subsystems named in 0015's Refactor Sequencing Risk table —
**(Rank 1) SSA/Varnode/Heritage**, **(Rank 2) Action/Rule engine**, **(Rank 3)
Architecture/pipeline/wire-protocol**, and **(Rank 4) Java↔C++ integration** — MUST expose a boundary
through which its behavior can be exercised and asserted on by a unit-level test, without requiring a
full `decompileAt` round trip through a live Ghidra program, before any refactor step that changes that
subsystem's internals is merged.

Concretely:
- **SSA/Heritage (Rank 1):** a minimal `Funcdata`/`Heritage` test fixture MUST be able to construct a
  handful of manually-created `PcodeOp`/`Varnode`s and assert on phi-node (`MULTIEQUAL`) placement and
  `HighVariable` merge decisions directly — today "no direct unit coverage at all... a plausible-but-
  different variable split... has no downstream test surface positioned to catch it specifically"
  (Refactor Sequencing Risk table, Rank 1 row).
- **Action/Rule engine (Rank 2):** see REQ-ARCH-5 (its own requirement, below).
- **Architecture/pipeline (Rank 3):** `Architecture`'s `build*` factory methods and `ActionDatabase`
  wiring MUST be testable against a minimal, file-based `Architecture` (the existing `"xml"`
  `ArchitectureCapability` path `testtypes.cc` already uses is the proven precedent), independent of a
  live `ArchitectureGhidra` RPC connection.
- **Java↔C++ integration (Rank 4):** the commit-back path (`HighFunctionDBUtil.commitParamsToDatabase`,
  `commitLocalNamesToDatabase`) MUST be exercisable against a synthetic `HighFunction` in a JUnit test,
  independent of a running `decompile` subprocess where feasible (a subprocess-backed integration test
  remains valuable in addition, not instead).

**Rationale:** This is the ticket's explicit instruction to "tie directly to 0015's Refactor Sequencing
Risk table" and "fix what that table calls out." Per 0015 Flaw 2 (00-index.md #4): "SSA/heritage
construction has no direct unit coverage of phi-placement or `HighVariable` merge decisions at all; the
Action/Rule engine... has zero unit-level tests of any individual `Rule`... nothing checks that the
hand-duplicated Java/C++ `ElementId` tables actually agree" (`architecture-extensibility.md` Flaw 2,
Observation). The table itself states ranks 1-3 "should not be refactored without characterization tests
in place first" — this requirement is the unit-test half of that; REQ-ARCH-6/7 are the
characterization-test half.

**Acceptance-test sketch:** For Rank 1, a new `unittests/testheritage.cc` (mirroring `testtypes.cc`'s
structure) builds a synthetic `Funcdata` with a small hand-constructed CFG (e.g. a diamond with two
predecessors writing the same stack slot) and asserts a `MULTIEQUAL` is placed at the join block. For
Rank 3, a new test constructs an `Architecture` via the `"xml"` capability with a minimal `.cspec`/
`.pspec` pair and asserts `ActionDatabase::universalAction()` wires the expected six root actions without
requiring a live RPC connection.

---

### REQ-ARCH-5 — A single `Rule` MUST be testable in isolation with a synthetic P-code snippet

**Priority:** MUST

**Statement:** It MUST be possible to construct a minimal synthetic `PcodeOp`/`Varnode` graph (without
loading a full binary image or running the entire ~162-rule pipeline) and directly call one `Rule`'s
`applyOp()`, then assert whether it fired and what transformation it produced, using the existing
`test.hh` `TEST()` framework (the same harness `testtypes.cc`/`testfuncproto.cc` already use).

**Rationale:** This is the direct answer to 0013 Flaw 2 (00-index.md #2), required reading for this
ticket. Per that document: "`Rule::applyOp(PcodeOp *op, Funcdata &data)`... needs a live `PcodeOp*`
embedded in a live `Funcdata`... there is no lightweight, dependency-free way to build 'one `PcodeOp`
plus its operand `Varnode`s' and hand it to a single rule's `applyOp()`"
(`rule-engine-block-structuring.md` Flaw 2, Observation). That document's own improvement option 1 (a
minimal synthetic-`Funcdata` test fixture) is adopted here as a MUST, not left optional, because it is a
named prerequisite of REQ-ARCH-4's Rank 2 coverage and of REQ-ARCH-6's coverage bar for the Action/Rule
engine — without it, "the Action/Rule engine is unit-testable" (REQ-ARCH-4) has no concrete
implementation path. This requirement deliberately does not mandate retrofitting all ~162 existing rules
with isolated tests in this refactor (0013 Flaw 2's own open question flags this as a separately
schedulable effort) — only that the harness exists and that any *new or refactored* rule is covered by
it going forward.

**Acceptance-test sketch:** A new `unittests/testrule.cc` fixture builds a minimal `Architecture` (same
pattern as `testtypes.cc:44-65`), a `Funcdata` with hand-inserted `PcodeOp`s for a target pattern (e.g.
`INT_ADD` of a pointer and a constant, matching `RulePtrArith`'s trigger shape), calls
`RulePtrArith::applyOp()` directly, and asserts the op's opcode becomes `PTRADD` with correct operands —
without decompiling any function or touching `datatests`. Any refactor task that rewrites or splits a
`Rule` subclass MUST add or update a test of this shape for that rule as part of the same PR.

---

### REQ-ARCH-6 — Characterization-test coverage bar before touching a high-risk subsystem

**Priority:** MUST

**Statement:** Before any refactor step modifies the internals of a **Rank 1-3** subsystem (SSA/Varnode/
Heritage, Action/Rule engine, or Architecture/pipeline/wire-protocol, per the 0015 Refactor Sequencing
Risk table), the following characterization-test bar MUST be met and enforced in CI:

> **100% of the `datatests` corpus (89 files as of this writing — the full corpus, not a sample) MUST
> produce byte-identical full-function printed-C output, captured by a golden-snapshot harness
> (REQ-ARCH-7), comparing the state immediately before and immediately after each individual refactor
> commit/PR.** Any diff MUST be explicitly reviewed and justified in the PR description as either (a) an
> intentional, documented behavior change (with before/after examples) or (b) a bug fix (with a linked
> issue) — an unexplained diff blocks merge.

For **Rank 4** (Java↔C++ integration), the same byte-identical bar applies at the `test.slow/java` suite
level (17 files) instead of `datatests`. For **Rank 5-8** subsystems (control-flow structuring,
printer/cast logic, signature/calling-convention recovery, type system), the existing `datatests`/
`unittests` suites passing at 100% (not full golden-diff) remains the floor; the golden-snapshot harness
is available to those tasks but not mandatory.

**Rationale:** This is the ticket's explicitly-required "explicit bar... a number or concrete method, not
vague language," directly answering 0015 Flaw 2 (00-index.md #4): "a refactor that touches a subsystem
with only black-box coverage can silently change *which* equally-plausible variable split, cast
placement, or structuring shape comes out, without any test noticing" (`architecture-extensibility.md`
Flaw 2, Observation). The bar operationalizes Flaw 2's improvement option 1 ("characterization-test
generation before touching a subsystem... capture full-function output... as a golden baseline") and
option 3 ("sequence the refactor itself around this table... require characterization tests as an
explicit prerequisite task"). Anchoring the bar to the full 89-file corpus (not a sampled subset) and to
byte-identical full-function text (not the existing `<stringmatch>` regex fragments) closes the specific
gap 0010's review calls "the single biggest structural gap": "there is no test anywhere in this tree...
that pins the full, exact printed-C text of any function" (`java-integration-testing.md` §4). This bar is
conceptually aligned with — and should not be read as contradicting — the golden/snapshot-testing
approach ticket 0018's output-fidelity requirements are expected to also rely on for determinism
guarantees; this document does not depend on 0018's specific wording.

**Acceptance-test sketch:** A CI gate (`make characterize` or equivalent) runs the full `datatests`
corpus through `ghidra_test`, captures each scenario's complete `print C` output to a golden file per
scenario (not just the `<stringmatch>` fragments), and on a subsequent run diffs against the baseline. A
refactor PR touching `heritage.cc`, `coreaction.cc`/`ruleaction.cc`, or `architecture.cc`/
`ghidra_process.cc` that produces any unreviewed diff fails CI.

---

### REQ-ARCH-7 — Full-function golden-output snapshot harness

**Priority:** MUST

**Statement:** A golden-snapshot test harness MUST exist that captures the **complete, exact printed-C
text** of every `datatests` scenario (not the existing `<stringmatch>` regex-with-occurrence-bounds
assertions) and can diff a subsequent run's output against a stored baseline, reporting a full
expected-vs-actual text diff on mismatch. It MUST be runnable via the standalone Makefile loop
(REQ-ARCH-13) without requiring a Gradle/JVM build.

**Rationale:** This is the infrastructure REQ-ARCH-6 depends on, and is called out independently because
it also closes a distinct, separately-cited gap: 0015 Flaw 2's improvement option 1 (build "once and
reused for every subsequent subsystem" — `architecture-extensibility.md` Flaw 2) and 0010's finding that
`datatests`' regex-based failures report "which named property failed, not an expected-vs-actual diff,
which slows down characterizing what a refactor actually changed" (`java-integration-testing.md` §3.1,
also flagged in Flaw 2 Impact). Building this once, generically, is cheaper than re-deriving a
per-subsystem characterization method for each of the four high-risk subsystems in REQ-ARCH-6.

**Acceptance-test sketch:** Run the harness twice with no code changes between runs; it reports zero
diffs. Introduce a deliberate one-line change to a `Rule`'s output (e.g. flip a cast-insertion
condition); the harness reports exactly which `datatests` scenario(s) changed and shows the specific
line-level diff, not just a pass/fail count.

---

## Cluster C — Extension-model uniformity

### REQ-ARCH-8 — Unified self-registering extension mechanism for Rules and Actions

**Priority:** MUST

**Statement:** New `Rule` and `Action` classes MUST become addable via the same self-registering
`CapabilityPoint` idiom already used by `ArchitectureCapability`, `GhidraCapability`, `IfaceCapability`,
and `PrintLanguageCapability` — i.e. adding a new `Rule` MUST become "add a `.cc` file implementing a
self-registering singleton and link it into the build," not "add a `new RuleXxx(...)` call inside
`ActionDatabase::universalAction()`." The existing hand-wired ~162 rules MUST be migrated to this
mechanism (mechanical, not behavioral, migration — same rules, same default wiring, new registration
path) as part of adopting this requirement, so `coreaction.cc`'s ~700-line function is no longer the sole
way to extend the catalog. This requirement establishes uniform **compile-time** registration; it does
NOT by itself mandate a runtime/data-driven rule-authoring layer (see Open Questions Deferred, below).

**Rationale:** Directly answers 0015 Flaw 1 (00-index.md #3), the extension-model inconsistency this
ticket names explicitly: "the two things a plugin author would most want to add — a new `Rule`... or a
new scheduled `Action` — do **not** use this mechanism at all... hand-instantiated via literal
`new RuleXxx(...)`/`addRule()` calls inside one ~700-line function" (`architecture-extensibility.md`
Flaw 1, Observation). This adopts Flaw 1's improvement option 3 (`RuleCapability`-style self-registration)
as the floor requirement, explicitly per that option's own framing: "smallest, most mechanical change;
makes 'how do I add a new X to this engine' one consistent answer across the whole codebase... should be
treated as a floor, not a solution, if the refactor's goal is genuine plugin extensibility." Option 1
(a `ModelRule`-style declarative match-pattern layer) is the more ambitious answer to "genuine plugin
extensibility" but is deliberately **not** mandated as a MUST here: 0015 Flaw 1's own Open Questions ask
whether third-party `Rule` plugins are actually in scope for this refactor, or whether "extensibility"
here really means the refactor team's own ability to add/reorder rules safely — that scoping decision
belongs to the epic(s) that plan the rule-engine refactor itself (0021/0024), not to this requirements
document. Making option 3 a MUST and option 1 a SHOULD-for-later (see REQ-ARCH-8a below) keeps this
document from prematurely committing to an unresolved design question while still fixing the
inconsistency the ticket asks for.

**REQ-ARCH-8a — Declarative rule-authoring layer (SHOULD, forward-looking):** SHOULD be designed as a
follow-on to REQ-ARCH-8, modeled on the proven `ModelRule`/`AssignAction` pattern from parameter-storage
assignment (`signature-calling-conventions.md` §1.2/§6) rather than resurrecting `rulecompile.cc` (see
REQ-ARCH-12), once REQ-ARCH-8's compile-time uniformity and REQ-ARCH-5's per-rule test harness exist to
validate it incrementally. Scoping and design of this layer is explicitly deferred to a later epic.

**Acceptance-test sketch:** After migration, adding a new `Rule` subclass requires zero edits to
`coreaction.cc`: a new `.cc` file defines a `RuleXxx` and a self-registering
`RuleCapability`-derived singleton; linking that file into the build is sufficient for the rule to appear
in the default pipeline with its declared group tag. A test asserts that `ActionDatabase::universalAction()`'s
resulting rule set is unchanged (same 162 rules, same groups, same pool assignment) before/after the
migration, using REQ-ARCH-7's golden-output harness to confirm zero behavior change.

---

### REQ-ARCH-9 — Architecture and calling-convention extension points preserved for out-of-tree consumers

**Priority:** MUST

**Statement:** The refactor MUST preserve (not regress) the ability for an out-of-tree consumer to add:
(a) a new file-based `Architecture` bootstrap via `ArchitectureCapability` (the existing `Bfd`/
`RawBinary`/`Xml` pattern), and (b) a new calling convention/ABI via the declarative `ModelRule`/
`AssignAction` mechanism (`DatatypeFilter`/`QualifierFilter` → `AssignAction`, driven by `.cspec`
`<model_rule>` XML) without editing core engine files.

**Rationale:** Directly answers the ticket's "extension model MUST remain (or become) usable by
out-of-tree consumers for... new architectures/ProtoModels." `ArchitectureCapability` is confirmed
working as advertised today for the three file-based architectures (`architecture-pipeline-overview.md`
§4), and `ModelRule` is independently confirmed as "a real, proven, declarative layer... genuinely
additive on top of a hardcoded fallback, actually used by every modern multi-register-struct ABI in-tree"
(`architecture-extensibility.md` Flaw 1, Observation, contrasting it favorably against the `Rule`/`Action`
gap). Because both mechanisms already work, this requirement's job is to name them as things a refactor
MUST NOT silently regress while restructuring `Architecture`'s internals (a Rank 3 subsystem per
REQ-ARCH-4) — not to propose new design work.

**Acceptance-test sketch:** A regression test builds an `Architecture` via each of the three existing
`ArchitectureCapability` implementations after a refactor step touching `Architecture`'s internals, and
confirms all three still construct successfully. A `ModelRule`-based `.cspec` `<model_rule>` scenario
already covered by `datatests` (e.g. `retstruct.xml`) is included in REQ-ARCH-6's golden-output gate for
any Rank 3 change.

---

### REQ-ARCH-10 — `ArchitectureGhidra` routed through capability discovery

**Priority:** SHOULD

**Statement:** `ArchitectureGhidra` — the one `Architecture` subclass Ghidra itself actually uses —
SHOULD be constructed via `ArchitectureCapability::findCapability` (the same discovery mechanism the
three file-based architectures use) rather than being directly `new`'d in
`RegisterProgram::rawAction`, so that the extension mechanism's primary real-world consumer actually
exercises it.

**Rationale:** Directly answers a specific sub-finding of 0015 Flaw 1 (00-index.md #3): "the one
`Architecture` subclass Ghidra itself actually uses, `ArchitectureGhidra`, **bypasses**
`ArchitectureCapability::findCapability`... Only the three file-based, non-Ghidra-hosted architectures
actually go through capability discovery" (`architecture-extensibility.md` Flaw 1, Observation). Per that
document's Impact section: "any future change to `ArchitectureCapability` risks being validated only
against the three minor, rarely-used file-based architectures, not the path that matters." Marked SHOULD,
not MUST, because the source flaw document's own improvement option 4 (which this requirement adopts)
flags that `ArchitectureGhidra` carries RPC-specific construction state the discovery interface (built for
file-based bootstrapping) may not cleanly model — it "needs its own small design pass, not a pure
mechanical move," so this document states the goal without mandating a specific mechanical
implementation.

**Acceptance-test sketch:** After the change, a test confirms
`ArchitectureCapability::findCapability("ghidra")` (or equivalent identifier) returns an
`ArchitectureGhidra`-producing capability, and that `RegisterProgram::rawAction`'s behavior (archid
assignment, `Architecture::init` sequencing) is unchanged per REQ-ARCH-7's golden-output harness.

---

### REQ-ARCH-11 — Output-language extension point preserved for new `PrintLanguage` consumers

**Priority:** MUST

**Statement:** The `PrintLanguageCapability` self-registration mechanism MUST remain the way a new output
language (a `PrintLanguage` subclass, e.g. beyond today's `PrintC`/`PrintJava` pair) is added, without
requiring edits to `Architecture` or other core dispatch code.

**Rationale:** Directly answers the ticket's "extension model MUST remain (or become) usable by
out-of-tree consumers for... new output languages (`PrintLanguage` subclasses)." `PrintLanguageCapability`
is one of the four confirmed-working `CapabilityPoint` hierarchies
(`architecture-pipeline-overview.md` §4), so this requirement is preservation, not new design — but it is
stated explicitly because the sibling flaw document 0014 separately notes that `PrintLanguage` reusability
beyond the two shipped implementations is unproven, since `PrintJava` is "not an independent
`PrintLanguage` implementation; it's a `PrintC` subclass overriding only 9 methods"
(`00-index.md`, "Known-surprising findings"; consolidated at 00-index.md #16). This requirement does not
mandate fixing that reusability gap (out of scope for this ticket — it belongs to whichever epic acts on
0014's findings) but does require that the registration *mechanism itself* is not weakened by this
refactor.

**Acceptance-test sketch:** A minimal third `PrintLanguage` subclass (e.g. a stub that overrides only
what's needed to compile and register) added as a new `.cc` file with a self-registering
`PrintLanguageCapability` singleton is discoverable via `Architecture::printlist` and selectable via
`setPrintLanguage`, with no other file edited.

---

### REQ-ARCH-12 — Retire the dead rule-pattern DSL; no dual-tracked extension mechanisms

**Priority:** SHOULD

**Statement:** `rulecompile.cc`/`.hh`'s pattern-matching DSL and its broken `decodeDynamicRule` runtime-
loading path SHOULD be deleted (or, at minimum, formally marked unsupported and decoupled from any
codegen-convenience use) rather than resurrected or left in an ambiguous present-but-dead state, and the
extension-model work in REQ-ARCH-8/8a MUST NOT reintroduce a second, syntactically distinct
rule-authoring mechanism running alongside it.

**Rationale:** Directly answers 0013 Flaw 3 (00-index.md #14), required reading: "this entire mechanism
is dead in shipped builds... `RuleGeneric::build()`... cannot currently compile
[undeclared-variable call site]... `CPUI_RULECOMPILE`... off by default... zero [wired rules] are
`RuleGeneric` instances" (`rule-engine-block-structuring.md` Flaw 3, Observation). That document's own
explicit recommendation is adopted verbatim here: "replace/retire — do not dual-track... if a declarative
rule layer is wanted later, it should be designed as part of [the ordering/dependency] work, not
resurrected from `rulecompile.cc`" (Flaw 3, "Explicit recommendation" section). This is directly relevant
to REQ-ARCH-8a's future declarative-layer design: building that layer on `ModelRule`'s proven pattern
instead of reviving `rulecompile.cc`'s unification-engine approach avoids adding a second undeclared-
ordering risk surface on top of the one 0013 Flaw 1 already documents. Marked SHOULD rather than MUST
because the narrower carve-out (keep only the `parse rule`/`UnifyCPrinter` codegen convenience, decoupled
from the broken runtime path) is a legitimate alternative per Flaw 3's own improvement options, and the
choice between full deletion and the narrow carve-out is an implementation decision, not a requirements-
level one.

**Acceptance-test sketch:** A code-search gate (or manual audit checklist item) confirms
`decodeDynamicRule` and its `CPUI_RULECOMPILE`-gated runtime path are absent (or explicitly marked
`@deprecated`/unsupported with no live call site) after this work lands, and that no new `Rule`-plugin
mechanism introduced under REQ-ARCH-8a shares `rulecompile.cc`'s `ConstraintGroup`/`UnifyState`
machinery.

---

## Cluster D — Standalone build/run tooling

### REQ-ARCH-13 — Standalone build-and-run loop preserved and documented

**Priority:** MUST

**Statement:** The existing Makefile-based, Gradle/JVM-independent local-iteration loop
(`Ghidra/Features/Decompiler/src/decompile/cpp/Makefile`) — specifically `make decomp_opt`/`decomp_dbg`
(interactive console REPL via `consolemain.cc`), `make test`/`decomp_test_dbg` (runs the full
`unittests` + `datatests` corpus), and `make ghidra_dbg`/`ghidra_opt` (RPC binary, debug variant
including console modules) — MUST continue to build and run correctly throughout the refactor, AND MUST
be documented (a README or CONTRIBUTING note under `Ghidra/Features/Decompiler/src/decompile/`, or
equivalent onboarding doc reachable from the repo root) as the recommended fast local-iteration path for
C++-only changes, distinct from the Gradle-built production binary. "Preserved" alone is insufficient to
satisfy this requirement — the documentation half is required equally.

**Rationale:** Directly answers the ticket's instruction to state this as "preserve AND document," tied
to 0015 Flaw 4 (00-index.md #19): "a fast standalone iteration loop exists (console driver) but isn't
discoverable... nothing in either build file, nor any README under `Ghidra/Features/Decompiler/`,
documents that the Makefile exists" (`architecture-extensibility.md` Flaw 4, Observation). The flaw
document's own bottom line makes the same point this requirement encodes: "a fast, standalone,
full-rebuild-avoiding local iteration loop exists today... and should be the refactor team's default
inner loop for C++-only changes — but it needs to be documented... before the team can rely on it with
confidence" (Flaw 4, "Can the decompiler be built/run standalone" section). This is adopted as a MUST
(not SHOULD) because ticket 0015 itself frames a fast local loop as a precondition the refactor
specifically depends on (Analysis Focus #4), and because documentation is a near-zero-cost fix per Flaw
4's own improvement option 1 ("nearly free; immediately raises discoverability").

**Acceptance-test sketch:** A new contributor following only in-repo documentation (no prior Ghidra
decompiler history) can run `make decomp_opt` and `make test` successfully within, say, 15 minutes of
cloning, without reading `Makefile` source directly. CI includes a smoke-check that `make test` still
builds and the `unittests`/`datatests` binary still runs (independent of whether its *output* is gated
by REQ-ARCH-6/14) after each refactor step, to catch the Makefile itself silently breaking.

---

### REQ-ARCH-14 — Standalone test loop wired into CI

**Priority:** SHOULD

**Statement:** `make test` (or equivalent) SHOULD be invoked from CI on every change touching
`decompile/cpp`, so that `unittests`/`datatests` runs are an enforced gate rather than a manual/ad hoc
step, and so that any future drift between the Makefile's and `buildNatives.gradle`'s source-file lists
surfaces as a build failure.

**Rationale:** Directly supports REQ-ARCH-6's characterization-test bar (which needs `make test`/the
golden-snapshot harness to actually run somewhere enforced) and answers 0015 Flaw 4's improvement option
2: "wire the Makefile's `test` target into CI... closes the duplicated-list risk the same way Flaw 3's
option 2 does for `ElementId`; turns the existing `datatests`/`unittests` suites... into an enforced
regression gate" (`architecture-extensibility.md` Flaw 4, Improvement options). Marked SHOULD rather than
MUST because Flaw 4's own Open Questions note this may require investigating CI
platform/toolchain availability (`bison`/`flex` dependencies) not diagnosable from this document, and
because REQ-ARCH-13 (documentation) is the higher-priority, lower-risk fix that must land regardless.

**Acceptance-test sketch:** A CI workflow runs `make test` on every PR touching
`Ghidra/Features/Decompiler/src/decompile/**` and fails the PR on any `unittests`/`datatests` failure;
a deliberately-introduced Makefile/Gradle source-list mismatch (e.g. a new `.cc` file added to one but
not the other) is caught by this gate before merge.

---

## Traceability table

| REQ ID | Priority | One-liner | Primary flaw source |
|---|---|---|---|
| REQ-ARCH-1 | MUST | Explicit wire-protocol compatibility bar; version bump only under 4 stated conditions | 0015 Flaw 3 (00-index.md #9) |
| REQ-ARCH-2 | MUST | `registerProgram` MUST exchange and check a real protocol version | 0015 Flaw 3 (00-index.md #9) |
| REQ-ARCH-3 | SHOULD | `ElementId`/`AttributeId` tables single-sourced or cross-checked in CI | 0015 Flaw 3 (00-index.md #9) |
| REQ-ARCH-4 | MUST | Rank 1-4 subsystem boundaries (SSA/Heritage, Rule engine, Architecture, Java integration) independently unit-testable | 0015 Flaw 2 + Refactor Sequencing Risk table (00-index.md #4) |
| REQ-ARCH-5 | MUST | A single `Rule` testable in isolation via synthetic P-code, using `test.hh` | 0013 Flaw 2 (00-index.md #2) |
| REQ-ARCH-6 | MUST | 100%/89-file byte-identical golden `datatests` diff gate before touching Rank 1-3 subsystems | 0015 Flaw 2 (00-index.md #4); 0010 §4 |
| REQ-ARCH-7 | MUST | Full-function golden-output snapshot harness (infra for REQ-ARCH-6) | 0015 Flaw 2 (00-index.md #4); 0010 §3.1/§4 |
| REQ-ARCH-8 | MUST | Unified `CapabilityPoint`-style self-registration for `Rule`/`Action` | 0015 Flaw 1 (00-index.md #3) |
| REQ-ARCH-8a | SHOULD | Future declarative rule-authoring layer modeled on `ModelRule` | 0015 Flaw 1 (00-index.md #3) |
| REQ-ARCH-9 | MUST | Preserve `ArchitectureCapability` + `ModelRule` extensibility for out-of-tree architectures/ABIs | 0015 Flaw 1 (00-index.md #3) |
| REQ-ARCH-10 | SHOULD | Route `ArchitectureGhidra` through capability discovery | 0015 Flaw 1 (00-index.md #3) |
| REQ-ARCH-11 | MUST | Preserve `PrintLanguageCapability` for new output languages | 0015 Flaw 1 (00-index.md #3); cross-ref 0014 (00-index.md #16) |
| REQ-ARCH-12 | SHOULD | Retire dead `rulecompile.cc` DSL; no dual-tracked rule-authoring mechanisms | 0013 Flaw 3 (00-index.md #14) |
| REQ-ARCH-13 | MUST | Preserve AND document the standalone Makefile build/run loop | 0015 Flaw 4 (00-index.md #19) |
| REQ-ARCH-14 | SHOULD | Wire `make test` into CI | 0015 Flaw 4 (00-index.md #19) |

## Open questions deferred (explicitly out of scope for this document)

- The exact shape of REQ-ARCH-8a's declarative rule-authoring layer (grammar, matcher, how it expresses
  ordering/dependency metadata) is a design question for the epic that plans the rule-engine refactor
  itself, not a requirements-level decision — see 0015 Flaw 1's own Open Questions.
- Whether `rulecompile.cc` is fully deleted or narrowly preserved as a codegen convenience (REQ-ARCH-12)
  is an implementation choice within the SHOULD.
- Rule/Action ordering-dependency declarations (0013 Flaw 1) are related to, but distinct from, this
  ticket's testability scope (0013 Flaw 2) and are not required here; REQ-ARCH-5's per-rule test harness
  is a prerequisite for validating any future ordering-declaration work, but designing that work is out
  of scope for this document.
- The specific golden-snapshot harness's relationship to any determinism requirements ticket 0018 may
  define is intentionally left uncoordinated in detail — REQ-ARCH-6/7 are written so as not to contradict
  a golden/snapshot-testing approach there, but this document does not depend on 0018's exact wording.

## Related documents

- [`.tasks/0019`](../../.tasks/0019-task-requirements-extensibility-integration-api.md) — the ticket this
  document satisfies.
- [`.tasks/0016`](../../.tasks/0016-epic-requirements-definition.md) — parent epic; owns the requirement
  ID/format contract and the consolidated `00-index.md` (not edited by this document).
- [`architecture-extensibility.md`](../02-flaws/architecture-extensibility.md) (0015) — primary source.
- [`rule-engine-block-structuring.md`](../02-flaws/rule-engine-block-structuring.md) (0013) — source for
  REQ-ARCH-5 and REQ-ARCH-12.
- [`architecture-pipeline-overview.md`](../01-review/architecture-pipeline-overview.md) (0003),
  [`java-integration-testing.md`](../01-review/java-integration-testing.md) (0010) — wire-protocol and
  test-infrastructure facts this document builds on.
- [`../02-flaws/00-index.md`](../02-flaws/00-index.md) — consolidated flaw priority table (row numbers
  cited throughout).
- [`../00-overview/glossary.md`](../00-overview/glossary.md) — shared terminology.
