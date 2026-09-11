# Architecture & Pipeline Overview

> Produced by [`.tasks/0003`](../../.tasks/0003-task-review-architecture-pipeline-overview.md), part of
> [`.tasks/0002`](../../.tasks/0002-epic-review-current-implementation.md) (review epic) under the root
> [`.tasks/0001`](../../.tasks/0001-epic-decompiler-full-refactor.md). See
> [`00-index.md`](00-index.md) for how this fits with the other review documents, and
> [`../00-overview/glossary.md`](../00-overview/glossary.md) for shared terminology.
>
> This document stays intentionally shallow on the simplification engine itself —
> [`action-rule-engine.md`](action-rule-engine.md) (ticket 0006) covers the ~500-rule catalog in depth —
> and on the Java-side test/integration machinery, covered by
> [`java-integration-testing.md`](java-integration-testing.md) (ticket 0010).

## 1. Process / IPC model

The decompiler engine (`libdecomp`, all of `Ghidra/Features/Decompiler/src/decompile/cpp`) is **not** a
library linked into the Ghidra Java process. It is compiled into a standalone native executable,
`decompile` (`decompile.exe` on Windows), and the Java side talks to it as a **subprocess over its
stdin/stdout pipes**, one process per `DecompInterface` that has been `openProgram`'d
(`Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/DecompileProcessFactory.java:42-46`
always `new`s a fresh `DecompileProcess` and spawns it with `Runtime.exec`,
`Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/DecompileProcess.java:151`). In
practice this means one native process per open Program per `DecompInterface` instance (a
`DecompilerProvider`/analyzer typically keeps one alive for as long as it needs decompilations), not
literally "per Ghidra session" — nothing stops a tool from opening several.

### Framing and wire encoding

Both sides speak the same **hand-rolled framed byte protocol**, defined identically (and only as
matching magic-byte tables, not a shared schema) on each side:
- C++: `ArchitectureGhidra::readToAnyBurst` reads bytes until it finds a `\001` marker followed by a
  type byte (`Ghidra/Features/Decompiler/src/decompile/cpp/ghidra_arch.cc:79-98`); `GhidraCommand::doit`
  wraps every command in `\000\000\001\006` / `\000\000\001\007` response brackets
  (`ghidra_process.cc:124-154`).
- Java: `DecompileProcess` declares the mirror-image byte arrays (`command_start`, `string_start`,
  `query_response_start`, `exception_start`, …) and a `switch` on the type byte that drives its read
  loop (`DecompileProcess.java:54-463`).

**The live query/command payloads are *not* XML on the wire**, despite the pervasive XML-flavored
naming (`<function>`, `<symbol>`, `ELEM_*`/`ATTRIB_*`) throughout the codebase. They use a compact
binary TLV-style encoding, `PackedEncode`/`PackedDecode`, documented in
`Ghidra/Features/Decompiler/src/decompile/cpp/marshal.hh:440-504` (`PackedFormat` namespace: 2-bit
record-type header, 5+7-bit varint element/attribute ids, typed attribute values). Both
`ArchitectureGhidra`'s query methods (e.g. `getPcode`, `ghidra_arch.cc:538-552`) and the Java
`DecompileProcess` (`PackedDecode paramDecoder` / `PatchPackedEncode resultEncoder`,
`DecompileProcess.java:82-84`) use this format for every command and nested query. The separate
`XmlEncode`/`XmlDecode` classes (`marshal.hh:378-459`) exist for a different purpose: `.pspec`/`.cspec`
text passed into `registerProgram` is XML text, and the standalone console's `save`/`restore` commands
persist a whole `Architecture` as an XML file (`consolemain.cc:124-172`). Anyone assuming "the wire
protocol is XML" (a reasonable inference from the class/element naming) would be wrong for the hot
path — worth flagging explicitly since ticket 0001's non-goals freeze "the wire protocol" without being
precise about which of the two encodings that means.

### Request/response cycle for one function

1. **One-time per process**: Java sends `registerProgram` with four XML strings (pspec, cspec, a
   trimmed `<sleigh>` tag, core datatypes) — `DecompInterface.java:284-286`,
   `ghidra_process.cc:156-204`. The C++ side constructs one `ArchitectureGhidra` and calls
   `Architecture::init` (`RegisterProgram::rawAction`, `ghidra_process.cc:170-195`), returning an
   integer `archid` that is echoed on every subsequent command
   (`GhidraCommand::loadParameters`, `ghidra_process.cc:86-102`, reads it back out of a global
   `vector<ArchitectureGhidra *> archlist`, `ghidra_process.cc:76`).
2. **Per function**: `DecompInterface.decompileFunction` encodes the entry address and calls
   `sendCommandTimeout("decompileAt", …)` (`DecompInterface.java:769-834`). On the native side,
   `DecompileAt::rawAction` (`ghidra_process.cc:286-330`) looks up the `Funcdata` via
   `symboltab->queryFunction(addr)` and, if not already processed, runs
   `ghidra->allacts.getCurrent()->reset(*fd); ghidra->allacts.getCurrent()->perform(*fd);` — see §3.
3. **Nested queries, same pipe, same command**: while building/simplifying `fd`, the engine pulls
   whatever it doesn't yet have cached straight from the Ghidra client, *synchronously, inside the same
   `decompileAt` call*: mapped symbols (`ArchitectureGhidra::getMappedSymbolsXML`,
   `ghidra_arch.cc:561-575`), single-instruction p-code (`getPcode`, `ghidra_arch.cc:538-552`),
   registers, comments, string data, datatypes, constant-pool entries, call-fixup injections, etc. (full
   list of commands: `ghidra_arch.hh:26-44`). Each is a full nested request/response burst
   (`\000\000\001\004` … `\000\000\001\005`) that the Java `DecompileProcess` read loop recognizes by
   type byte 4 mid-command and dispatches to `DecompileCallback` by command name
   (`DecompileProcess.java:320-411`; the `getPcode`/`getMappedSymbols`/`getDataType`/… implementations
   live in `Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/DecompileCallback.java:194-
   758`). So a single `decompileAt` round trip is really dozens of smaller round trips; the decompiler is
   best understood as a **pull-based client of the live Ghidra database for the duration of one
   function**, not a one-shot batch job that receives everything up front.
4. **Result**: `DecompileAt::rawAction` encodes a `<doc>` element containing the `Funcdata` (optionally
   with the raw syntax tree), optionally `ParamIDAnalysis` measures, and — if C output was requested —
   invokes the `PrintLanguage` to emit marked-up C (`ghidra_process.cc:306-330`,
   `ghidra->print->docFunction(fd)`). Java decodes this back into a `HighFunction` plus a
   `ClangTokenGroup` (`DecompileResults.decodeStream`,
   `DecompileResults.java:211-260`). That method's own comment calls out an "ugly kludge": the response
   reuses the `ELEM_FUNCTION` tag twice — once for the `HighFunction` and once (when C code was
   requested) for the root of the Clang markup tree, disambiguated only by whether `hfunc` has already
   been set (`DecompileResults.java:225-235`).

### Commands available over the protocol

`GhidraDecompCapability::initialize` registers the full command set the Ghidra client can issue
(`ghidra_process.cc:489-499`):

| Command | C++ class | Purpose |
|---|---|---|
| `registerProgram` | `RegisterProgram` | Create an `ArchitectureGhidra` for a program (§ above) |
| `deregisterProgram` | `DeregisterProgram` | Free it; also issues meta-command `1` to terminate the process |
| `flushNative` | `FlushNative` | Drop cached symbols/types/comments/strings/cpool so they re-fetch |
| `decompileAt` | `DecompileAt` | Decompile one function (the main path, above) |
| `structureGraph` | `StructureGraph` | Structure an arbitrary externally-supplied CFG (no `Funcdata` involved) |
| `setAction` | `SetAction` | Switch the *root* `Action` and/or toggle what gets sent back (§3) |
| `setOptions` | `SetOptions` | Push an `<optionslist>` document into `Architecture::options` |

`consolemain.cc` and `ifacedecomp.cc`/`ifaceterm.cc` are the **other** front end to the same engine: a
line-oriented interactive console (`decomp>` prompt) used by developers and by the `datatests`/
`unittests` harnesses, built around `IfaceStatus`/`IfaceCommand`
(`Ghidra/Features/Decompiler/src/decompile/cpp/interface.hh:170-298`) and a decompiler-specific command
set registered by `IfaceDecompCapability` (`ifacedecomp.hh`, ~100 `Ifc*` classes,
`ifacedecomp.hh:111-621`). Its `decompile` command, `IfcDecompile::execute`
(`ifacedecomp.cc:889-916`), calls the **exact same two lines** —
`dcp->conf->allacts.getCurrent()->reset(*dcp->fd); dcp->conf->allacts.getCurrent()->perform(*dcp->fd);`
— as `DecompileAt::rawAction`. This is worth knowing precisely because it is the one genuine seam
between "used from Ghidra" and "used from tests/scripts/command line": everything below `Action::perform`
is identical in both front ends; everything above it (how `Funcdata` got populated, how results get
reported) differs.

## 2. The `Architecture` object

`Architecture` (`architecture.hh:165-379`) is the God-object of the engine: one instance is constructed
per program/executable and owns essentially every other major subsystem as a raw member pointer:

- `TypeFactory *types` — the datatype system (detailed in `type-system.md`, ticket 0005).
- `Database *symboltab` — the global `Scope` tree (functions, globals, namespaces).
- `map<string,ProtoModel *> protoModels`, `ProtoModel *defaultfp`,
  `ProtoModel *evalfp_current`/`evalfp_called` — the calling-convention registry parsed from `.cspec`
  (detailed in `signature-calling-conventions.md`, ticket 0008).
- `const Translate *translate`, `LoadImage *loader` — disassembly/byte access; for the Ghidra-hosted
  case these are `GhidraTranslate` (`ghidra_translate.hh:36-57`, queries p-code and register names from
  the client over the same protocol as §1) rather than a real local SLEIGH translator.
- `PcodeInjectLibrary *pcodeinjectlib`, `CommentDatabase *commentdb`, `StringManager *stringManager`,
  `ConstantPool *cpool` — auxiliary databases, all similarly backed by RPC queries when hosted by
  Ghidra.
- `PrintLanguage *print` plus `vector<PrintLanguage *> printlist` — the pretty-printer(s); which one is
  "current" is itself switchable at runtime (`setPrintLanguage`, `architecture.hh:236`) — detailed in
  `expression-cast-printer.md`, ticket 0009.
- `OptionDatabase *options` — the name-keyed `ArchOption` registry driving the `setOptions` command
  (`options.hh:76-354`, ~40 concrete `Option*` subclasses, one per configurable knob).
- `UserOpManage userops` — registry of target-specific "user-defined" p-code operations (`CALLOTHER`).
- `vector<Rule *> extra_pool_rules` — architecture/cspec-specific extra `Rule`s (see §4).
- `ActionDatabase allacts` — the *entire* Action/Rule simplification engine for this architecture (§3).

`Architecture` is abstract: it declares a battery of `build*` factory methods (`buildDatabase`,
`buildTranslator`, `buildLoader`, `buildPcodeInjectLibrary`, `buildTypegrp`, `buildCoreTypes`,
`buildCommentDB`, `buildStringManager`, `buildConstantPool`, `buildContext`, `buildSymbols`,
`buildSpecFile`, `modifySpaces`, `resolveArchitecture`; `architecture.hh:263-347`), all invoked in a
fixed order from `Architecture::init` (`architecture.cc`, driven from `parseProcessorConfig`/
`parseCompilerConfig`/`restoreFromSpec` — see `architecture.cc:587-660` for `buildAction`'s place in that
sequence). `ArchitectureGhidra` (`ghidra_arch.hh:82-169`) is the concrete subclass used whenever the
engine is driven by the Ghidra client: every `build*` override wires the subsystem up to issue RPC
queries back over the same pipe (e.g. `ScopeGhidra`, `database_ghidra.hh:37-117`, is a `Scope`
implementation that forwards symbol lookups to `ArchitectureGhidra::getMappedSymbolsXML` and caches the
result in an internal `ScopeInternal`). Three other, much simpler, `ArchitectureCapability`
implementations exist for the non-Ghidra-hosted console tool: `BfdArchitectureCapability`
(`bfd_arch.hh`), `RawBinaryArchitectureCapability` (`raw_arch.hh`), and `XmlArchitectureCapability`
(`xml_arch.hh`) — these build a self-contained `Architecture` straight from a file on disk, no subprocess
protocol involved.

**Asymmetry worth noting**: the three file-based architectures above are all discovered through
`ArchitectureCapability::findCapability` (`architecture.hh:151-152`, used by the console's `load`
command, `consolemain.cc:61-64`), but `ArchitectureGhidra` is **not** — `RegisterProgram::rawAction`
constructs it directly with `new ArchitectureGhidra(...)` (`ghidra_process.cc:181`). The
self-registering-capability discovery mechanism that the rest of the file describes as "how new
architectures plug in" simply isn't used on the path Ghidra itself takes to create an `Architecture`.

## 3. The `Action` pipeline stages, at a glance

*(Depth on individual `Rule`s and `Action`s is out of scope here — see
[`action-rule-engine.md`](action-rule-engine.md), ticket 0006. This section only locates where the major
phases sit in the sequence and how the "current root Action" is selected.)*

`ActionDatabase` (`action.hh:298-324`) holds:
- `actionmap`: name → root `Action*` (currently only one is ever registered, named `"universal"`, built
  once by `ActionDatabase::universalAction` — see below).
- `groupmap`: name → `ActionGroupList`, i.e. a named *slice* of group-tags out of the universal tree.
- `currentact`/`currentactname`: which sliced-and-derived root `Action` is active right now.

`ActionDatabase::universalAction` (`Architecture::buildAction` calls it,
`architecture.cc:587-591`; the body itself is ~700 lines in
`Ghidra/Features/Decompiler/src/decompile/cpp/coreaction.cc:5835` onward) hand-assembles one giant
`ActionRestartGroup` containing every `Action` and rule-`Pool` the engine knows about, each tagged with a
`"group"` string (`"base"`, `"protorecovery"`, `"localrecovery"`, `"typerecovery"`, `"merge"`,
`"blockrecovery"`, `"cleanup"`, `"casts"`, …). `ActionDatabase::buildDefaultGroups`
(`coreaction.cc:5791-5831`) then defines six *named subsets* of those groups as the actual root Actions a
client can select via the `setAction` command (`ghidra_process.hh:184-218` documents the same six names
in the RPC surface):

| Root action | Groups included (highlights) | Used for |
|---|---|---|
| `decompile` | everything — full group list, `coreaction.cc:5797-5805` | The default, full pipeline |
| `jumptable` | `base`, `noproto`, `localrecovery`, … (no `protorecovery`) | Just enough to resolve a switch's jump table |
| `normalize` | full list minus type/param-heavy groups, plus `normalanalysis` | Normalization-oriented decompilation (e.g. for diffing) |
| `paramid` | `base`…`typerecovery`…`siganalysis`, no block/printing groups | Parameter-ID recovery only (`ParamIDAnalysis`) |
| `register` | `base`, `analysis`, `subvar` | One register-only analysis pass, no stack variables |
| `firstpass` | `base` only | Build the raw, unsimplified syntax tree |

The engine's own Doxygen main page narrates the conceptual work flow in 15 steps
(`Ghidra/Features/Decompiler/src/decompile/cpp/docmain.hh:105-423`); mapping that narrative onto the
group/Action names above (approximately — see ticket 0006 for the authoritative mapping):

1. **Specify entry point / generate raw p-code / build CFG** (`docmain.hh:132-169`) — happens *before*
   `Action::perform` is ever called, seeded via `Funcdata`'s flow-following (out of this ticket's scope;
   see `ssa-varnode-heritage.md`, ticket 0004).
2. **Inspect sub-functions / adjust p-code** (`docmain.hh:171-227`) — `"base"`/`"protorecovery"` group
   Actions such as `ActionDefaultParams`, `ActionFuncLink` (`coreaction.cc:5853-5858`).
3. **Main simplification loop** (`docmain.hh:229-... `, the bulk of the work):
   - *Generate SSA form* → `ActionHeritage` (`"base"`, `coreaction.cc:5865`) — see
     `ssa-varnode-heritage.md`.
   - *Dead code elimination* → `ActionDeadCode` (`"deadcode"`, `coreaction.cc:5876`).
   - *Propagate local types* → `ActionInferTypes` (`"typerecovery"`, `coreaction.cc:5880`) — see
     `type-system.md`.
   - *Term rewriting* → the `ActionPool` instances `"oppool1"`/`"oppool2"`
     (`coreaction.cc:5884, 6035`), which apply the large `Rule` catalog to a fixed point — this is the
     heart of ticket 0006.
   - *Adjust CFG / recover structure* → `ActionBlockStructure`, `ActionStructureTransform`
     (`"blockrecovery"`, `coreaction.cc:6032, 6095`) — see `control-flow-structuring.md`, ticket 0007.
4. **Exit SSA / merge variables** (`docmain.hh:339-378`) → the `"merge"` group: `ActionMergeRequired`,
   `ActionMarkExplicit`, `ActionMergeMultiEntry`, … (`coreaction.cc:6097-6106`).
5. **Casts, prototype, naming** (`docmain.hh:380-405`) → `"casts"`, `"fixateproto"` groups.
6. **Final control-flow structuring / emit C tokens** (`docmain.hh:407-423`) → final `"blockrecovery"`
   passes, then `PrintLanguage`/`PrintC` token emission — see `expression-cast-printer.md`, ticket 0009.

Both call sites that actually run this (`DecompileAt::rawAction` and `IfcDecompile::execute`, §1) reduce
to the same pair of calls, `allacts.getCurrent()->reset(*fd)` then `->perform(*fd)`
(`Action::perform` walks the `ActionGroup`/`ActionPool` tree recursively, applying each `Rule`/`Action`
in turn, with `rule_repeatapply` groups iterating to a fixed point — see `action.hh:44-92` for the
flag/status bits that drive this, and ticket 0006 for the traversal algorithm itself).

## 4. Extension points

### `Capability` — the self-registration mechanism

`CapabilityPoint` (`capability.hh:39-52`, implemented in `capability.cc`) is a deliberately tiny base
class: its constructor pushes `this` onto a function-local `static vector<CapabilityPoint *>`
(`capability.cc:24-29`, classic Meyers-singleton-for-a-container idiom, chosen so the vector is
guaranteed constructed before any static `CapabilityPoint` subclass registers into it, sidestepping the
C++ static-initialization-order fiasco for the *list itself*). Every concrete extension is a **global
singleton instance of a subclass**, so it self-registers purely as a side effect of static
initialization; nothing calls a registration function explicitly. `CapabilityPoint::initializeAll()`
(`capability.cc:40-49`) is called exactly once by `startDecompilerLibrary`
(`libdecomp.cc:20-57`) / directly by `main` in the two executables (`consolemain.cc:206`,
`ghidra_process.cc:524`), giving every registered instance a chance to do the second half of its setup
in `initialize()` — *after* all static initializers across every translation unit have already run, so
ordering between unrelated capabilities is not observable, only "some point after link-time static init,
before first command".

Four independent hierarchies reuse this exact mechanism for four different kinds of plug-in:

| Hierarchy | Purpose | Example concrete singletons |
|---|---|---|
| `ArchitectureCapability` (`architecture.hh:117-157`) | New ways to bootstrap an `Architecture` from a file | `BfdArchitectureCapability` (`bfd_arch.hh:30`), `RawBinaryArchitectureCapability` (`raw_arch.hh:29`), `XmlArchitectureCapability` (`xml_arch.hh:29`) |
| `GhidraCapability` (`ghidra_process.hh:45-53`) | New RPC command sets the `decompile` process understands | `GhidraDecompCapability` (`ghidra_process.hh:59-66`, the core command set, §1), `GhidraSignatureCapability` (`signature_ghidra.hh:29`) |
| `IfaceCapability` (`interface.hh:170-180`) | New command modules for the interactive console | `IfaceDecompCapability` (`ifacedecomp.hh:34`), `IfaceAnalyzeSigsCapability` (`analyzesigs.hh:27`), `IfaceCodeDataCapability` (`codedata.hh:23`) |
| `PrintLanguageCapability` (`printlanguage.hh:42`) | New output-language pretty-printers | (backs `PrintC`; see `expression-cast-printer.md`) |

Because registration is driven purely by whether a singleton's translation unit is linked in, adding a
new instance of any of the four kinds above is genuinely just "add a `.cc` file implementing the
subclass and link it into the `decompile`/`ghidraDec`/console build target" — no central registry file
to edit. The cost is that this is a **compile-time, single-process, C++-ABI-only extension mechanism**:
there is no notion of loading a capability at runtime, versioning one independently of the engine, or
isolating a misbehaving one (see `segvHandler`, `ghidra_process.cc:503-507`: a crash anywhere kills the
whole process for whatever `Architecture`s happen to be registered in `archlist` at the time).

### How `Rule`/`Action` plug in — *not* the same mechanism

Despite living in the same file family, the ~500-entry `Rule` catalog and the `Action` tree are **not**
`CapabilityPoint`-based. They are hand-instantiated, by literal `new ActionXxx(...)`/`new RuleXxx(...)`
calls, inside the single large function `ActionDatabase::universalAction`
(`coreaction.cc:5835` onward — see §3). The only *data-driven* extension points for this part of the
engine are:
- `Architecture::extra_pool_rules` (`architecture.hh:189`), populated by `parseExtraRules` from `<rule>`
  tags inside a `.cspec` file (`architecture.cc`, decoded via `decodeDynamicRule`,
  `architecture.hh:359`) — lets a processor/compiler spec opt a *pre-existing, compiled-in* `Rule`
  subclass into the pool by name, with a group and an "experimental" flag, not add genuinely new
  behavior.
- A `CPUI_RULECOMPILE`-gated experimental-rules XML file, wired up only from the standalone console
  (`IfcLoadFile::execute`, `consolemain.cc:69-84`), for a rule-*compiler* feature that is conditionally
  compiled and not part of the normal Ghidra-hosted build.

**Implication for refactor-ability**: today's "extension points" are really two unrelated mechanisms
wearing the same name (`Capability`) for one of them — a coarse, compile-time, whole-subsystem
plug-in system with no runtime story, and a separate, much narrower, data-driven toggle that only
reorders/enables already-compiled `Rule`s. Neither offers a way to add a genuinely new `Rule` or
`Action` without editing and recompiling `coreaction.cc` (or another `.cc` that calls
`ActionDatabase::registerAction`/`ActionGroup::addAction` directly). Any refactor that wants real
plug-in `Rule`s (e.g. for target-specific idioms) has no existing seam to build on — it would need a new
mechanism, not an extension of `CapabilityPoint`.

## 5. Sequence diagram: one `decompile <func>` request

```mermaid
sequenceDiagram
    participant UI as CodeBrowser / DecompilerProvider
    participant DI as DecompInterface (Java)
    participant DP as DecompileProcess (Java, per Program)
    participant CB as DecompileCallback (Java)
    participant NP as decompile (native process)

    Note over DI,NP: openProgram() — one time per DecompInterface
    DI->>DP: registerProgram(pspec, cspec, tspec, coretypes)
    DP->>NP: spawn subprocess; write "registerProgram" + 4 XML strings
    NP->>NP: new ArchitectureGhidra(...); conf->init(store)
    NP-->>DP: archid

    Note over UI,NP: decompileFunction(func, timeout)
    UI->>DI: decompileFunction(func)
    DI->>DP: sendCommandTimeout("decompileAt", entryAddr)
    DP->>NP: archid + "decompileAt" + <addr> (packed encoding)

    NP->>NP: Funcdata *fd = symboltab->queryFunction(addr)

    rect rgba(200,200,200,0.15)
    Note over NP,CB: nested queries, same pipe, same command
    NP->>DP: query getMappedSymbols(addr)
    DP->>CB: getMappedSymbols(addr)
    CB-->>DP: <symbol>/<function>/<hole>
    DP-->>NP: response
    NP->>DP: query getPcode(addr) (repeated per instruction)
    DP->>CB: getPcode(addr)
    CB-->>NP: p-code ops
    NP->>DP: query getComments / getDataType / getCPoolRef / ...
    DP->>CB: (dispatched by command name)
    CB-->>NP: response
    end

    NP->>NP: allacts.getCurrent()->reset(*fd)
    NP->>NP: allacts.getCurrent()->perform(*fd)
    Note right of NP: Action/Rule pipeline — SSA, typing,<br/>rule simplification, structuring (§3, ticket 0006)
    NP->>NP: print->docFunction(fd)  (if C requested)

    NP-->>DP: <doc><function>...</function>[clang markup]</doc>
    DP-->>DI: Decoder over packed response bytes
    DI->>DI: new DecompileResults(...).decodeStream()
    DI-->>UI: HighFunction + ClangTokenGroup (displayed C text)
```

## 6. Summary of surprising / undocumented patterns

- **The live wire format is a compact binary encoding, not XML**, despite XML-flavored naming
  throughout (`marshal.hh:440-504` vs. `marshal.hh:378-459`) — see §1.
- **`ArchitectureGhidra` bypasses the `ArchitectureCapability` discovery mechanism** that the rest of
  the engine advertises as "how new architectures plug in"; it is constructed directly by
  `RegisterProgram::rawAction` (`ghidra_process.cc:181`) — see §2.
- **The `decompileAt` RPC command is not one round trip** but a command that itself issues an unbounded
  number of nested synchronous query/response round trips back to the same Java client while it builds
  and simplifies the function — see §1.
- **Global, manually-indexed process state**: `vector<ArchitectureGhidra *> archlist`
  (`ghidra_process.cc:76`) and `map<string,GhidraCommand *> GhidraCapability::commandmap`
  (`ghidra_process.cc:78`) are plain global mutables; the protocol nominally supports multiple
  registered programs multiplexed over one process/pipe via `archid`, but the Java client
  (`DecompileProcessFactory.get()`, always `new`s a fresh process and never reuses one across programs)
  never actually exercises that — see §1, §2.
- **A crash takes down the whole process.** `segvHandler` deliberately calls `_Exit(1)` with no cleanup
  on `SIGSEGV`, purely to stop the OS popping up a Windows error dialog
  (`ghidra_process.cc:503-507`); the Java side's resilience is entirely a *client-side* restart
  (`DecompileProcess.DisposeState`, fresh process next time it's needed) — see §2.
- **`Rule`/`Action` extension is not `Capability`-based**, unlike everything else called an "extension
  point" in this codebase — see §4.
- **A tag-reuse "kludge"** in the Java-side result decoder: the response document uses `ELEM_FUNCTION`
  for two structurally different things (the `HighFunction` and, when present, the root of the C markup
  tree), disambiguated only by decode order (`DecompileResults.java:225-235`, the code's own comment
  calls this "an ugly kludge") — see §1.
- **Two structurally identical, independently-implemented front ends.** The Ghidra RPC path
  (`DecompileAt::rawAction`) and the interactive console path (`IfcDecompile::execute`) both reduce to
  the identical `reset()`/`perform()` pair on `allacts.getCurrent()`, but everything around that call —
  how the `Funcdata` got populated, how results are reported, error handling — is duplicated rather than
  shared through a common driver function — see §1, §3.
