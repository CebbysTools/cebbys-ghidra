# Architecture Decision Record: Decompiler Refactor Approach

Epic: [`.tasks/0020`](../../.tasks/0020-epic-solution-research-and-selection.md) · Ticket:
[`.tasks/0023`](../../.tasks/0023-task-solution-selection-decision-record.md) · Root:
[`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md)

This is the ADR referenced by the root epic's acceptance criteria ("a single Architecture Decision
Record states the selected refactor approach") and by epic 0020's deliverable list. It is the final
document of the current planning pass. Primary evidence base:
[`architecture-options.md`](architecture-options.md) (ticket 0022), whose two comparison matrices are
cited directly throughout the Rationale section below rather than re-derived. Also consulted in full:
[`../03-requirements/00-index.md`](../03-requirements/00-index.md) (ticket 0016),
[`../02-flaws/00-index.md`](../02-flaws/00-index.md) (ticket 0011), and
[`prior-art-external-decompilers.md`](prior-art-external-decompilers.md) (ticket 0021).

## Status

**Accepted.**

## Context

Ghidra's decompiler back end (`Ghidra/Features/Decompiler/src/decompile/cpp`, ~230 files, ~150k lines)
converts P-code into structured, typed pseudo-C through a `Funcdata`/`Action`/`Rule` pipeline that has
accreted for over a decade. Epic 0011's flaw analysis identified 22 findings, seven of them High
severity: undeclared ordering between the rule engine and block structuring (flaw #1), zero unit-test
coverage for the ~500-class `Rule` catalog (flaw #2), two incompatible extension-registration mechanisms
wearing one name (flaw #3), test-coverage gaps that directly gate safe refactor sequencing (flaw #4), a
split-brain type system between `TypeFactory` and Ghidra's Java-side `DataTypeManager` (flaw #5), a
~6,300-line `Funcdata` god object with no narrow interfaces (flaw #6), and a bounded-heuristic
union-field resolver that can silently pick the wrong field (flaw #7). Epic 0016 translated these,
alongside the initiative sponsor's explicit call for richer language features, into 36 prioritized
requirements (REQ-LANG/REQ-OUT/REQ-ARCH) with RFC-2119 priorities, of which 19 are MUST — see
[`../03-requirements/00-index.md`](../03-requirements/00-index.md) for the full table.

Ticket 0021 surveyed prior art across five other decompilers and one recent post-processing approach
(DIRTY) and converged, independently of any of Ghidra's own findings, on the same conclusion Ghidra's
own flaw analysis reached from the inside: the load-bearing move is making pipeline stage boundaries
explicit, named, and independently testable before touching any individual algorithm inside a stage.
Ticket 0022 took that conclusion seriously, factored it out as a mandatory Foundation Layer applied
identically under every option (which is why the Foundation-Layer-dependent MUST rows read identically
across Options 1–3 in its comparison matrix), and then developed four genuinely distinct options for
*what changes inside the pipeline* on top of that shared foundation: Option 1 (in-place incremental
refactor), Option 2 (new intermediate IR/AST layer ahead of the printer), Option 3 (staged replacement
behind a feature-flag cutover — a rollout discipline, not a target shape), and Option 4 (a narrow,
independently-combinable structuring-algorithm swap). 0022 scored all four against the full 19-row MUST
table and the 7-row High-impact-flaw table and closed by stating a reading of that data without making
the decision itself — that is this ticket's job.

## Decision

**Adopt a composite architecture, not a single option**, structured as four layered commitments rather
than a menu choice:

1. **Foundation Layer (mandatory, applies everywhere, built first).** Named, ordered pipeline stages
   plus a full-text golden-snapshot characterization-test harness over the 89-file `datatests` corpus,
   including the declared-order-vs-registration-order assertion for flaw #1's mechanism. This is
   infrastructure, not a design choice among Options 1–4 — 0022 established that all three canonical
   options depend on it identically, and the requirements index independently flags it (REQ-OUT-6 +
   REQ-ARCH-6/7) as "a near-zero-regret first step regardless of which architecture option is chosen."
2. **Option 1 (in-place incremental refactor) as the default treatment for SSA/heritage, the
   `Action`/`Rule` engine, the type-system core, `Funcdata`, and the extension-registration model.**
   This subsystem set stays inside its existing `Funcdata`/`Action`/`Rule` shape; flaws are fixed
   subsystem-by-subsystem behind the Foundation Layer's tests.
3. **Option 2 (new typed IR/AST layer) scoped specifically to the printer/output subsystem**, applied
   only where it is strictly the better-evidenced fit: template printing, composable qualifiers, and
   cast-decision unification. This is additive downstream of `Funcdata`, not a replacement for it —
   `Funcdata` decomposition itself stays under Option 1's incremental track, per the comparison matrix's
   own finding that Option 2 adds one more consumer of `Funcdata`'s surface rather than reducing it.
4. **Option 3's staged feature-flag cutover discipline, applied narrowly to the Option-2 printer
   rewrite only** — not broadly to every Option-1 change. The printer/IR work is the one piece of this
   composite that is a wholesale rewrite rather than an incremental patch, making it the single
   highest-regression-risk piece being introduced; Option 3's flag-gated dual-implementation rollout is
   reserved for exactly that piece, self-funded alongside REQ-ARCH-2's version-handshake work at the
   same `registerProgram` touch-point. Option-1 changes do not get this treatment — they are local,
   additive, and reversible by ordinary code review, so paying Option 3's double-maintenance and
   REQ-ARCH-1 wire-touch cost for them would be waste.

**Option 4 (structuring-algorithm swap) is accepted as an independently-scheduled track**, not blocked
on and not a prerequisite for 1–3, to be picked up opportunistically once the Foundation Layer exists —
most plausibly after the initial Option-1/Foundation baseline lands, since it needs its own
characterization baseline (current goto-fallback frequency across `datatests`) before a replacement
algorithm can be evaluated against it.

This is a real, scoped decision, not a restatement of 0022's closing paragraph: where 0022 says Option
3 is "optionally" wrapped around "whichever of those carries the highest regression risk," this decision
names that piece explicitly (the printer/IR rewrite) and explicitly excludes the rest of the composite
from that treatment, for the reasons in the Rationale below.

## Rationale

**Why not Option 1 alone.** The MUST-requirement matrix gives Option 1 the broadest, most even
coverage — 12 of 19 rows ✅, 0 ❌ — and it is "the only option with a non-zero ✅ on every single
High-impact flaw row that's in scope for any option" (flaws #3 and #4), per the matrix narrative. But
its 7 🟡 rows cluster exactly on the requirements the requirements index itself calls out as most likely
to force a real design choice: REQ-LANG-5 (composable qualifiers) is rated 🟡 specifically because
"regression risk is real and hard to bound without auditing all ~500 rule classes" for
pointer-equality-as-type-equality assumptions against the same live, mutating `TypeFactory` interning
tree; REQ-OUT-1 (cast safety floor) and REQ-ARCH-4/5 (unit-testability) score 🟡 for the same
structural reason — no new seam is created to make the highest-risk subsystems easier to isolate.
Choosing Option 1 alone accepts open-ended audit risk on exactly the MUSTs the requirements index
flagged as decision-forcing (REQ-ARCH-8/8a and REQ-LANG-1/2 per
[`../03-requirements/00-index.md`](../03-requirements/00-index.md) "Cross-domain notes for epic 0020").

**Why not Option 2 alone.** Option 2 has the strongest MUST profile of any option (13/19 ✅, only 1 ❌)
and is the architecture-options document's own "strongest fit" for REQ-LANG-1 (templates), the "best
fit" for REQ-OUT-1 (a single cast-decision point at the lowering boundary), and "lower audit risk" for
REQ-LANG-5 versus Option 1. But its High-impact-flaw row is 0 ✅ / 2 🟡 / 5 ❌ — five of the seven
High-severity flaws (#1 rule ordering, #2 rule testability, #3 extension-model split, #5 type-system
split-brain, #6 `Funcdata` god object) sit entirely upstream of where a printer-facing IR layer changes
anything, and the matrix is explicit that this option "provides zero help at all" for REQ-ARCH-5 (single
`Rule` testability). The comparison-matrix narrative states plainly: "Choosing Option 2 as the only
architectural move would leave the codebase's most-flagged risks completely unaddressed." Since those
five flaws are the ones the flaw index itself ranks highest-severity and highest-refactor-risk (the
Refactor Sequencing Risk table ranks SSA/heritage, the rule engine, and `Architecture`/pipeline — all
outside Option 2's reach — as ranks 1–3), Option 2 alone is disqualified as a sole answer regardless of
its output-fidelity strength.

**Why the split is drawn where it is (Option 1 for engine/type-system/extension-model, Option 2 for
printer only), rather than some other division.** The two matrices are close to complementary along
exactly this seam. Every row where Option 2 beats Option 1 outright (REQ-LANG-1 "strongest fit,"
REQ-LANG-5 "lower audit risk," REQ-OUT-1 "best fit," REQ-ARCH-9/10/11 "arguably improved") is a
printer/output-facing row. Every High-impact flaw where Option 1 scores ✅ and Option 2 scores ❌ (#3
extension model, #4 test coverage — both ✅ under Option 1) or where Option 1 is at least 🟡 against
Option 2's ❌ (#1, #2, #5, #6, #7) is a flaw rooted upstream of the printer, inside `Funcdata`,
`Heritage`, `ruleaction.cc`, or the `TypeFactory`/`DataTypeManager` bridge — precisely the subsystems
this decision keeps under Option 1's incremental track. This is not coincidence: the architecture-options
document's own "Reading the matrices together" section observes the same pattern and states the
resulting composite explicitly as the shape the evidence points toward. This decision adopts that shape
as the actual call rather than leaving it as a reading of the data.

**Why Option 3 is scoped to the printer rewrite specifically, not applied broadly.** The matrix's own
narrative is explicit that Option 3 "is not a competitor to Options 1/2 on either table" — its row
totals are dominated by "🟡 inherits," and its only two genuinely independent, option-specific facts are
(a) it is uniquely *weaker* on REQ-ARCH-1 (the only option with any wire-protocol touch at all — adding
a feature-flag parameter to `registerProgram` is a Java-visible change, even if small and additive), and
(b) it is uniquely *stronger* on REQ-ARCH-6/flaw #4 (forced cutover-gating discipline) and on the
"old-implementation-as-comparison-oracle" testability bonus. Both of those facts matter most exactly
where a wholesale rewrite risks silent regression against `datatests` and needs a fallback — which is
Option 2's printer retarget (described in 0022 as "a full rewrite (not incremental patch)" of
`printc.cc`/`printc.hh`/`printjava.cc`/`printjava.hh`), not Option 1's local, additive patches. Paying
Option 3's REQ-ARCH-1 wire-touch cost and double-maintenance burden (0022's Risk section, points 1–2)
for Option 1's changes, which never need a second parallel implementation to be safe, would be pure
overhead with no matching benefit on either matrix row.

**Why Option 4 stays an independent track rather than blocking or being folded in.** The MUST matrix
marks 14 of 19 rows N/A for Option 4 — "correctly reflecting its narrow, complementary scope," per the
matrix's own summary line — and the flaw-table narrative states it is "the only option that directly,
specifically answers REQ-OUT-3 (goto-minimization) better than any combination of the other three could
without also adopting it, and it is combinable with whichever of Options 1/2/3 is chosen for everything
else." Nothing in the evidence base makes Option 4 a prerequisite for 1–3, and nothing in 1–3 blocks it;
scheduling it as parallel, opportunistic work is a direct reading of its own "N/A by genuine scope
exclusion" profile, not a compromise.

## Consequences

**What becomes easier.** REQ-LANG-4 (`volatile` printing) and REQ-ARCH-8 (`CapabilityPoint`
unification) become the cheapest possible wins under Option 1's in-place track and can land almost
immediately behind the Foundation Layer. REQ-LANG-1/5 and REQ-OUT-1 become structurally easier to
satisfy with lower audit risk than a pure Option 1 approach would carry, because they get a dedicated IR
boundary instead of extending a closed `PrintC` switch and a live, mutating interning tree read by ~500
rule classes. The printer rewrite gets a genuine safety net (Option 3's flag + comparison-oracle
property) precisely where it is most needed. Output-language extensibility (a future third `PrintX`) is
arguably improved, since it would target the new IR's node types instead of subclassing 3,500 lines of
`PrintC`.

**What becomes harder.** The composite has more moving pieces than any single option: two genuinely
different subsystem-treatment philosophies (incremental-in-place vs. new-layer-plus-staged-cutover) must
coexist in the same codebase and the same epic 0024 backlog, and the boundary between "printer-facing,
goes through Option 2/3" and "everything else, goes through Option 1" must be actively maintained rather
than being a natural consequence of a single uniform approach. `Funcdata`'s god-object decomposition
(flaw #6) and the ~500-class rule-testability audit (flaw #2) remain exactly as large and slow-burn as
Option 1's own Risk section describes — this composite does not make those two hardest problems smaller,
it only avoids compounding them with an unrelated printer rewrite happening inside the same undifferentiated
effort.

**Deferred requirements (explicitly, not dropped).** The following SHOULD/MAY requirements from 0016 do
not fit inside the near-term scope this decision implies and are deferred to later planning (epic
0024's later slices or epic 0025), not abandoned:

- **REQ-LANG-2** (SHOULD, first-class `Datatype`-level template/generic representation) — the MUST bar
  (REQ-LANG-1) is satisfiable with a minimal template tag on `Datatype` plus the new IR's own node type;
  the fuller, general-purpose representation REQ-LANG-2 describes is deferred until the walking-skeleton
  IR slice (Epic 0024 scope, below) proves out on real `datatests` cases and the actual representational
  needs are better known.
- **REQ-LANG-3** (MAY, monomorphized-function-family grouping) — explicit stretch goal by its own text;
  deferred until REQ-LANG-1/2 are shipped and there is real data on false-positive grouping risk.
- **REQ-LANG-7** (SHOULD, standalone calling-convention type attribute) — its own requirement text
  flags the need as unconfirmed; deferred pending verification, not scheduled speculatively.
- **REQ-LANG-8** (SHOULD, member-function-pointer-with-context type) — deferred past the first 3–6
  slices; revisit once the Option-1 type-system track has room after the qualifier/template work.
- **REQ-LANG-10** (SHOULD, anonymous struct/union/enum parity) and **REQ-LANG-12/13** (MAY, union
  bitfields / packed-alignment override) — lower-priority type-system expressiveness gaps (flaw #21);
  deferred to a later epic-0024 slice or epic 0025, revisited once the core template/qualifier work
  stabilizes the type-system surface they'd extend.
- **REQ-OUT-2** (SHOULD, cast readability ceiling) and **REQ-OUT-5** (SHOULD, merge readability
  ceiling) — both are explicitly framed in 0016 as refinements *on top of* their corresponding MUST
  safety floors (REQ-OUT-1, REQ-OUT-4); deferred until those floors are shipped and validated by the
  Foundation Layer harness, so readability tuning doesn't get entangled with correctness work.
- **REQ-OUT-3** (SHOULD, goto-minimization) — this is Option 4's namesake requirement; deferred from
  Epic 0024's first slices specifically because it rides on the independently-scheduled structuring
  track (Decision, point 4), not because it is lower priority in absolute terms.
- **REQ-ARCH-3** (SHOULD, single-sourced/CI-cross-checked `ElementId`/`AttributeId` tables) — CI/tooling
  work orthogonal to this decision's architectural choice; deferred to whenever wire-protocol tooling is
  next touched (naturally bundled with the Option-3 `registerProgram` flag work if timing allows, but
  not required by it).
- **REQ-ARCH-8a** (SHOULD, `ModelRule`-style declarative rule-registration layer) — explicitly named in
  0016 itself as "a later, explicitly deferred SHOULD" on top of the REQ-ARCH-8 MUST floor; this
  decision preserves that deferral rather than pulling it forward.

Two SHOULD items are *not* deferred and are folded into the near-term scope because they are cheap and
directly adjacent to work already planned: **REQ-ARCH-12** (retire the dead `rulecompile.cc` DSL) rides
along with the REQ-ARCH-8 `CapabilityPoint` unification slice since both touch extension registration,
and **REQ-ARCH-14** (wire `make test` into CI) rides along with the Foundation Layer slice since it is
the natural way to make the new harness load-bearing rather than optional.

## Scope for Epic 0024

The following is guidance text only — an ordered list of the first implementation slices this decision
implies, each scoped to be its own future ticket. No ticket files are created by this task; ticket
creation for 0024 is future work per that epic's own non-goals.

1. **Foundation Layer.** Stand up the golden-snapshot characterization-test harness (full-function text
   diff, extending `testfunction.hh`/`FunctionTestProperty`) over the 89-file `datatests` corpus;
   declare pipeline stage names and order; add the declared-order-vs-registration-order assertion to
   `ActionDatabase::universalAction()` (flaw #1's mechanism); wire `make test` into CI (REQ-ARCH-14).
   Mandatory, near-zero-regret, blocks nothing else and is blocked by nothing.
2. **Cheapest high-value in-place wins.** Ship `volatile` keyword printing (REQ-LANG-4) directly in
   `printc.cc`, and unify cast-necessity decisions into one `CastStrategy` method with a role parameter
   (REQ-OUT-1) in `cast.cc` — both small, local, immediately verifiable against the new harness.
3. **Extension-model unification.** Unify `Rule`/`Action` registration onto `CapabilityPoint` as a
   `RuleCapability` addition (REQ-ARCH-8, flaw #3), and retire the dead `rulecompile.cc` DSL in the same
   pass (REQ-ARCH-12) since both touch the same registration surface.
4. **Rule-testability seam.** Build a minimal synthetic-`Funcdata`/synthetic-P-code test fixture
   (REQ-ARCH-5, flaw #2) and apply it to the highest-risk rule-ordering cases documented by flaw #1's
   eight examples first — this fixture is a prerequisite for every deeper Option-1 fix that follows and
   should not be deferred past this point.
5. **Printer/IR walking skeleton.** Design and build the smallest possible new IR (variable reference,
   binary op, call, basic statement forms — no templates or qualifiers yet) sufficient to reproduce a
   small, real subset of `datatests` scenarios (straight-line integer arithmetic, no control flow)
   byte-for-byte through the new lowering+print path, gated behind an Option-3 feature flag threaded
   through `Architecture`'s option database, combined with REQ-ARCH-2's real version handshake at the
   same `registerProgram` touch-point.
6. **Templates and qualifiers on the IR.** Extend the walking-skeleton IR with template-instantiation
   and qualifier-composition node types (REQ-LANG-1/5, REQ-LANG-9 for recursive/forward-declared
   composites feeding it), plumb the existing `DemangledTemplate` data through instead of discarding it,
   and cut the new printer over for the covered subset once 100% byte-identical parity is confirmed
   under the flag against the corresponding `datatests` cases.

## Scope note for Epic 0025

"Done" for the basic-feature milestone should mean: the Foundation Layer harness is wired into CI and
treated as a hard merge gate, not optional tooling; the printer/IR flag from slices 5–6 has been flipped
to default-on for its covered subset with zero `datatests` regressions recorded under either flag state
during the transition; REQ-LANG-1/4/5 and REQ-OUT-1 are shipped and demonstrably exercised by golden
snapshots, not just implemented; REQ-ARCH-8's `CapabilityPoint` unification is complete with the dead
`rulecompile.cc` DSL removed; and the synthetic-`Funcdata`/synthetic-P-code fixture from slice 4 has been
proven on at least the highest-risk rule-ordering cases, establishing the pattern epic 0025 can then
scale to the remaining rule catalog. Epic 0025 should not be scoped to start until these hold, and should
treat the explicitly deferred requirements list above (REQ-LANG-2/3/7/8/10/12/13, REQ-OUT-2/3/5,
REQ-ARCH-3/8a) plus the still-unscheduled Option 4 structuring-algorithm track as its primary backlog
input, alongside `Funcdata`'s full decomposition and the remaining ~500-class rule-testability audit that
this milestone deliberately did not attempt to finish.
