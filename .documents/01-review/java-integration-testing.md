# Java-Side Decompiler Integration & Test Infrastructure

Produced by [`.tasks/0010`](../../.tasks/0010-task-review-java-integration-testing.md), part of the
review epic [`.tasks/0002`](../../.tasks/0002-epic-review-current-implementation.md). Complements
[`architecture-pipeline-overview.md`](architecture-pipeline-overview.md) (0003), which owns the
authoritative description of the wire protocol's element/attribute model — this document describes
the Java side of that boundary and does not re-derive the marshal format. See also
[`../00-overview/glossary.md`](../00-overview/glossary.md) for shared terminology.

This is a review document only: no source was modified in producing it.

## 1. The Java↔C++ boundary

### 1.1 Process model

The C++ decompiler ships as a **separate native executable** (`decompile` / `decompile.exe`), one OS
process per open program, communicating with Java over the process's stdin/stdout pipes as a byte
stream. It is a different build target from the `ghidra_test` binary described in §3.

- `DecompileProcessFactory.get()` locates the executable via `Application.getOSFile("decompile")`
  (or `decompile.exe` on Windows) and constructs a fresh `DecompileProcess`
  (`Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/DecompileProcessFactory.java:42-75`).
- `DecompileProcess.setup()` (`.../DecompileProcess.java:134-194`) does the actual `Runtime.exec`,
  waits up to 200ms for an early failure, and captures stderr into a user-visible error dialog if the
  process dies immediately (`DecompileProcess.java:150-193`).
- `DecompInterface` (`.../DecompInterface.java`) is the long-lived Java façade an analyzer/UI holds
  onto (one per `Program`, effectively). It is **self-healing**: `verifyProcess()`
  (`DecompInterface.java:350-357`) checks `decompProcess.isReady()` before every call and transparently
  respawns + re-`initializeProcess()`s a dead process, replaying all cached configuration (simplification
  style, syntax-tree/C-code toggles, options, signature settings) — see the sequence of
  `sendCommand2Params("setAction", ...)` calls in `initializeProcess()`
  (`DecompInterface.java:258-348`). Callers never see the respawn; they only see a possibly-slower call.
- `DecompInterface.openProgram(Program)` (`DecompInterface.java:373-416`) is the one-time-per-program
  setup: it builds a `DecompileCallback` and calls `initializeProcess()`, which serializes the
  language's SLEIGH translator spec, the compiler spec (`.cspec`), the processor spec (`.pspec` file
  read straight off disk), and Ghidra's core datatypes to XML strings and sends them via the
  `registerProgram` command (`DecompInterface.java:270-286`, `DecompileProcess.java:478-507`). The C++
  side's `RegisterProgram` command handler builds an `ArchitectureGhidra` from this and returns an
  `archId` integer that tags every subsequent command for that program
  (`Ghidra/Features/Decompiler/src/decompile/cpp/ghidra_process.cc:492`, `DecompileProcess.java:506`).
- `closeProgram()` / `dispose()` send `deregisterProgram` and release the process back to
  `DecompileProcessFactory` (`DecompInterface.java:424-445`, `DecompileProcessFactory.java:48-50`).
  `DecompilerDisposer` runs the actual OS-process teardown off the Swing thread because it "sometimes
  hangs" (`DecompileProcess.java:118-128`, comment at line 126).

### 1.2 Wire format and request/response shape

The byte stream is framed with fixed 4-byte markers Java and C++ both hard-code identically:
`command_start`/`command_end` = `{0,0,1,2}`/`{0,0,1,3}`, `query_response_start/end` = `{0,0,1,8}`/`{0,0,1,9}`,
`string_start/end` = `{0,0,1,14}`/`{0,0,1,15}`, `exception_start/end` = `{0,0,1,10}`/`{0,0,1,11}`,
`byte_start/end` = `{0,0,1,12}`/`{0,0,1,13}` (`DecompileProcess.java:54-63`; mirrored as literal
`out.write("\000\000\001\006",4)`-style writes in
`Ghidra/Features/Decompiler/src/decompile/cpp/ghidra_process.cc:465-479`). A command is
`command_start, "<name>", "<archId>", [params...], command_end`; the process replies with a response
whose body is a length-framed, binary "packed" encoding (`PackedEncode`/`PackedDecode`,
`Ghidra/Features/Decompiler/src/decompile/cpp/marshal.hh:505-593`) — a compact, tag/length-prefixed
binary analogue of the same element/attribute model XML uses (`PackedFormat` bit layout at
`marshal.hh:480-503`). `DecompileProcess.readResponse()` (`DecompileProcess.java:318-463`) is a single
state machine that must additionally handle **inbound callback queries** arriving interleaved with the
response — see §1.3.

Every `DecompileProcess.sendCommand*` variant (`sendCommand`, `sendCommand1Param`, `sendCommand2Params`,
`sendCommandTimeout`) follows the same shape: write frame, flush, drive `readResponse` until the
top-level closing marker, decoding either a `PackedDecode` payload, a plain string (`StringIngest`), or
throwing on an `exception_start` frame (`DecompileProcess.java:295-305`, "alignment" errors become
`IOException`, everything else becomes `DecompileException(type, message)`). `DecompInterface` never
talks to the process directly outside these methods.

The element/attribute **names and numeric IDs are hand-duplicated** in two independently maintained
source files rather than negotiated at runtime: Java's
`Ghidra/Framework/SoftwareModeling/src/main/java/ghidra/program/model/pcode/ElementId.java` (e.g.
`ELEM_DOC = new ElementId("doc", 229)` at line 351) and the C++ definitions scattered across the
`decompile/cpp` sources (e.g. `ElementId ELEM_DOC = ElementId("doc",229);` at
`Ghidra/Features/Decompiler/src/decompile/cpp/ghidra_process.cc:74`). Both sides call a static
`initialize()` that registers every `ElementId`/`AttributeId` into a name→id map
(`marshal.hh:44,73`), but nothing checks that the two independently-authored lists actually agree — if
one side adds/renames/renumbers an id without updating the other, the mismatch surfaces only as a wrong
decode at runtime, not a version-negotiation failure. See §1.4 for how (little) this is otherwise
guarded against.

### 1.3 Callback protocol (C++ pulls from Java, not the reverse)

The `decompileAt`/`generateSignatures`/`debugSignatures`/`structureGraph` commands are long-running and
**re-entrant**: while decompiling, the C++ process repeatedly queries back into Java for facts it needs
about the specific function/program, rather than Java pre-shipping the whole database up front. This
"pull" model is deliberate — bytes, comments, symbols, injected p-code, datatypes, etc. are fetched
lazily and only for what the current decompilation actually touches.

- On the Java side, `DecompileProcess.readResponse()` recognizes callback-query frames (`type == 4`,
  `DecompileProcess.java:326-411`), decodes a `commandId` and dispatches to one of ~16 private methods
  (`getBytes`, `getComments`, `getPcode`, `getPcodeInject`, `getCPoolRef`, `getExternalRef`,
  `getMappedSymbols`, `getNamespacePath`, `getPcode`, `getRegister`, `getRegisterName`,
  `getStringData`, `getCodeLabel`, `getDataType`, `getTrackedRegisters`, `getUserOpName`,
  `isNameUsed` — `DecompileProcess.java:332-390`), each of which thinly wraps the corresponding
  `DecompileCallback` method and re-encodes the result before looping back into `readResponse`
  (`DecompileProcess.java:409-410`). Any exception from a handler is caught and shipped back to the
  decompiler as an `exception_start`/`exception_end` frame rather than killing the pipe
  (`DecompileProcess.java:393-408`).
- `DecompileCallback` (`.../DecompileCallback.java`, 1207 lines) is where these queries actually resolve
  against the live `Program`/`Listing`/`SymbolTable`. Notable behavior:
  - `getBytes` (`DecompileCallback.java:152-183`) reads raw memory and returns `null` on any exception,
    which the decompiler treats as `DataUnavailError` and keeps going rather than aborting the whole
    function (comment at line 146).
  - `getPcode` (`DecompileCallback.java:214-244`) will **pseudo-disassemble** on the fly
    (`pseudoDisassemble`, lines 370-421) when there's no real `Instruction` at the address yet — this is
    how the decompiler can walk into undefined-function bodies.
  - `getMappedSymbols`/`getExternalRef`/`encodeFunction` (lines 651-935) are the single biggest chunk of
    this file: resolving what lives at an address (function vs. data vs. external reference vs. a
    "hole") and building `HighSymbol`/`HighFunction` objects to describe it — this is where the
    decompiler learns about **other** functions it calls, not just the one it's decompiling.
  - `getPcodeInject` (`DecompileCallback.java:290-341`) is the callback path for call-fixups,
    call-other-fixups, call-mechanisms, and injected executable p-code
    (`PcodeInjectLibrary`/`InjectPayload`) — i.e. how compiler-spec-level pcode patches reach the
    decompiler.
  - `getCPoolRef` (`DecompileCallback.java:349-360`) is the constant-pool hook used for
    Java-bytecode-derived p-code (`ConstantPool`).
  - `isNameUsed` (`DecompileCallback.java:490-532`) does an approximate namespace-collision check,
    explicitly documented as imprecise ("Ghidra is inefficient at calculating this perfectly", line
    483-485) and capped at `MAX_SYMBOL_COUNT = 16` matching symbols (line 51, 514).
  - When a `DecompileDebug` sink is attached (`enableDebug`, `DecompInterface.java:184-186`), most
    callback methods also mirror their answer into an XML debug dump
    (e.g. `DecompileCallback.java:200-204, 335-340, 355-359`) — this is the mechanism behind Ghidra's
    "decompiler debug" feature for diagnosing a specific bad decompilation, not a general trace log.

### 1.4 Error handling, timeouts, versioning

- **Timeouts.** `decompileFunction`/`structureGraph`/`generateSignatures`/`debugSignatures` all take an
  explicit `timeoutSecs` (`DecompInterface.java:769, 727, 1019, 1072`), default suggested value
  `DecompileOptions.SUGGESTED_DECOMPILE_TIMEOUT_SECS = 30`
  (`Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/DecompileOptions.java:296`).
  `DecompileProcess.sendCommandTimeout` arms a `GTimer` that calls `dispose()` and sets
  `DisposeState.DISPOSED_ON_TIMEOUT` if it fires (`DecompileProcess.java:578-616`); `DecompInterface`
  surfaces this via `DecompileResults.isTimedOut()` (`DecompileResults.java:123-125`). A `TaskMonitor`
  cancellation is wired the same way through a `CancelledListener` that calls `stopProcess()`
  (`DecompInterface.java:140-145, 780-782, 805-807`), surfaced as `DecompileResults.isCancelled()`
  (`DecompileResults.java:134-136`). A fourth `DisposeState`, `DISPOSED_ON_STARTUP_FAILURE`
  (`DecompileProcess.java:90`), covers the executable simply not launching, surfaced as
  `DecompileResults.failedToStart()` (`DecompileResults.java:144-146`).
- **Errors.** Two independent channels exist and are easy to conflate: (a) protocol/process-level
  failures throw `IOException` or `DecompileException` out of `DecompInterface` methods and are caught
  there, folded into `errorMessage`/`DecompileResults.getErrorMessage()`; (b) decompiler-level warnings
  and errors about the *function itself* (e.g. "Bad p-code" for one op, a `LowlevelError`) are sent as
  an in-band `<error>`-framed string (`type == 16/17` in `DecompileProcess.readResponse`,
  `DecompileProcess.java:434-448`) that `DecompileCallback.setErrorMessage` stashes
  (`DecompileCallback.java:140-142`) and `DecompInterface` copies out after the call
  (`DecompInterface.java:797, 1040, 1092`) — **decompilation can "succeed" (return a usable
  `HighFunction`/C text) while `errorMessage` is still non-empty**, because
  `DecompileResults.decompileCompleted()` only checks whether `hfunc`/`hparamid` got decoded
  (`DecompileResults.java:107-109`), not whether `errMsg` is blank; `isValid()` is the separate check
  for that (`DecompileResults.java:152-154`). Informational messages (`type == 18`) are logged via
  `Msg.warn` and otherwise dropped (`DecompileProcess.generateWarning`,
  `DecompileProcess.java:307-316`).
- **Versioning.** There is **no explicit wire-protocol version handshake** between the Java client and
  the native process (see the shared-ID-table risk in §1.2). The closest thing to a version query,
  `DecompInterface.getMajorVersion()`/`getMinorVersion()`/`getSignatureSettings()`
  (`DecompInterface.java:911-981`), calls the `getSignatureSettings` command and decodes a
  `<sigsettings><major>/<minor>/<settings></sigsettings>` element
  (`decodeVersionNumber`, `DecompInterface.java:911-926`) — but this major/minor pair is the version of
  the **function-signature/feature-vector module** specifically (`ELEM_MAJOR`/`ELEM_SIGSETTINGS` are
  defined in `Ghidra/Features/Decompiler/src/decompile/cpp/signature.cc:33-40`, and the response is
  produced by `signature_ghidra.cc:83-93`), not a general decompiler-protocol version; it throws if the
  returned `settings` don't match what Java asked for (`DecompInterface.java:923-925`), which is a
  feature-configuration consistency check, not a compatibility gate for the wire protocol as a whole. In
  practice, protocol compatibility is maintained by **always building and shipping the `decompile`
  executable from the same source tree/commit as the Java code** (`DecompileProcessFactory` finds the
  build's own `decompile`/`decompile.exe` via `Application.getOSFile`,
  `DecompileProcessFactory.java:65-67`) — there is no mechanism to detect, at runtime, a Java build
  talking to a decompiler executable built from different sources.

### 1.5 UI consumption (brief — not core logic)

`ghidra/app/decompiler/component/**` (43 files, ~6100 lines total) is the Swing UI layer that renders
`DecompileResults.getCCodeMarkup()` (a `ClangTokenGroup` tree of `ClangToken` leaves — `ClangFuncNameToken`,
`ClangVariableToken`, `ClangOpToken`, `ClangTypeToken`, etc., all in
`ghidra/app/decompiler/*.java`). Key consumers:

- `DecompilerPanel`/`ClangLayoutController`/`ClangTextField` lay the token tree out as text with field
  positions for the field-panel widget.
- `DecompilerHighlightController` and friends (`ClangHighlightController.java`,
  `LocationClangHighlightController.java`) implement token/variable highlighting (e.g. highlighting
  every occurrence of a clicked variable) by walking the token tree, not the decompiler.
- `DecompilerUtils` (`component/DecompilerUtils.java`) has helpers to map a `ClangToken` back to a
  `Varnode`/`PcodeOp`/`HighVariable`/`Function` and vice versa — this is how "right-click → rename" and
  similar actions know what database object a piece of displayed text corresponds to.
- `TokenIterator`/`ClangLine` provide line/token traversal used by search (`DecompilerFindDialog`) and
  hovers (`component/hover/*`, e.g. `DataTypeDecompilerHover`, `ScalarValueDecompilerHover`).

None of this package re-derives or second-guesses the C++ output; it is a consumer, and out of scope for
deep review per the parent epic's non-goals.

## 2. What Java does with decompiler output beyond display

The decompiler is not purely a read-only "show me the C" service — several Java-side commands use its
output to **write back** into the Ghidra program database, and this is what a refactor must keep
producing compatible data for:

- **Parameter/return commit-back.** `CommitParamsAction`
  (`Ghidra/Features/Decompiler/src/main/java/ghidra/app/plugin/core/decompile/actions/CommitParamsAction.java:63`)
  calls `HighFunctionDBUtil.commitParamsToDatabase(hfunc, true, ReturnCommitOption.COMMIT, ...)` — it
  takes the recovered `FuncProto`/parameter list off the `HighFunction` decoded from decompiler output
  and writes it onto the real `Function` object (signature, storage, names) in the program database.
  `HighFunctionDBUtil` itself lives outside the Decompiler module, at
  `Ghidra/Framework/SoftwareModeling/src/main/java/ghidra/program/model/pcode/HighFunctionDBUtil.java`,
  since it operates on core program-database types.
- **Local variable commit-back.** `CommitLocalsAction`
  (`.../actions/CommitLocalsAction.java:51`) calls `HighFunctionDBUtil.commitLocalNamesToDatabase`,
  writing decompiler-recovered local variable names/storage back onto the function.
- **Retype/rename actions.** `RetypeLocalAction`, `RetypeGlobalAction`, `RetypeReturnAction`,
  `RenameVariableTask`, `ForceUnionAction`, `IsolateVariableTask` (same `actions` package) all operate on
  a `HighSymbol`/`HighVariable` obtained from a `DecompileResults` and push a user edit back through
  `HighFunctionDBUtil` — the decompiler's SSA/variable-merging model (`HighVariable`, see
  `ssa-varnode-heritage.md`) is therefore part of the **editing** model, not just a display model: a
  refactor that changes how varnodes merge into `HighVariable`s changes what these actions can express.
- **Prototype override management.** `OverridePrototypeAction`/`EditPrototypeOverrideAction`/
  `DeletePrototypeOverrideAction` let a user pin a specific call site's prototype; the decompiler is
  re-run to pick the override up, so the override format is part of the Java↔C++ contract too (via the
  `injectoverride`-style commands and `HighFunctionDBUtil`).
- **Automated parameter-ID analysis.** `DecompilerParameterIdCmd`
  (`Ghidra/Features/Decompiler/src/main/java/ghidra/app/cmd/function/DecompilerParameterIdCmd.java`) is
  a background `AutoAnalysisManager` command that runs the decompiler in `"paramid"` simplification style
  (see the style list documented at `DecompInterface.java:447-471`) across many functions and commits the
  resulting parameter data types (`HighFunctionDBUtil.ReturnCommitOption`) — i.e. **auto-analysis**, not
  just interactive UI, depends on decompiler output shape.
- **Structure recovery from use.** `FillOutStructureCmd`
  (`Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/util/FillOutStructureCmd.java`) walks
  a `HighFunction`'s p-code around a variable to infer a struct layout from field accesses, then commits
  a new `Structure` datatype — this depends on the exact shape of `PcodeOp`s the decompiler's simplification
  passes leave behind (e.g. whether an access shows up as `PTRSUB`/`LOAD` with a constant offset), so it
  is sensitive to Action/Rule engine behavior even though it's a Java-side feature.
- **Signature/feature-vector export.** `DecompInterface.generateSignatures`/`debugSignatures`
  (`DecompInterface.java:1019-1115`) drive an entirely separate decompiler output mode
  (`SignatureResult`/`DebugSignature`, package `ghidra.app.decompiler.signature`) used for
  function-similarity/matching features — this is a second, independent consumer of the C++ pipeline
  beyond C-text and syntax-tree generation, with its own versioning (§1.4) and its own wire messages
  (`generateSignatures`, `debugSignatures` commands).

There is no evidence in this module of PDB/DWARF data flowing *through* the decompiler process itself;
PDB/DWARF-derived types and symbols are applied to the program database by separate DWARF/PDB analyzers
before decompilation, and the decompiler only sees them via the normal `DecompileCallback` symbol/datatype
queries (§1.3) like any other program information. A refactor therefore does not need to preserve any
PDB/DWARF-specific protocol surface — only the general symbol/datatype query surface.

## 3. Test infrastructure inventory

There are **three independent test surfaces** for the decompiler, run by different tools, covering
different layers. None of them is a full end-to-end "decompile this real-world binary and diff the
output" regression suite.

### 3.1 `datatests` — data-driven single-function C++ tests (89 files)

Directory: `Ghidra/Features/Decompiler/src/decompile/datatests/*.xml` (89 files at review time). Run by
a dedicated test executable (`ghidra_test`, built from `consolemain.cc` — actually the entry point is
`Ghidra/Features/Decompiler/src/decompile/cpp/test.cc:103-174`), **not** the production `decompile`
binary. Each file is a self-contained `<decompilertest>`:

- `<binaryimage arch="...">` — an inline, hand-crafted raw byte blob (`<bytechunk>`) plus `<symbol>`
  tags, i.e. a tiny synthetic "binary" assembled in the test file itself (see e.g.
  `datatests/condconst.xml:2-28`). Built by `FunctionTestCollection::buildProgram`
  (`Ghidra/Features/Decompiler/src/decompile/cpp/testfunction.cc`, declared at `testfunction.hh:82`).
- `<script>` — a sequence of `<com>` console commands run against the built program, using the same
  command interpreter as the interactive decompiler console (`ifaceterm`/`ifacedecomp`): `parse line`
  (inject C declarations), `map addr` (name a memory location), `lo fu <name>` (load a function),
  `decompile`, `print C`. These are literally normal decompiler console commands, so a test file doubles
  as a repro script a human can paste into the interactive console.
- `<stringmatch name="..." min="N" max="M">regex</stringmatch>` — the assertions. Each one is a regular
  expression (optionally multi-line, split on literal `\n` in the tag content —
  `FunctionTestProperty::restoreXml`, `testfunction.cc` lines ~51-90) searched for across the **printed C
  output** collected from the `print C` command's buffered output
  (`FunctionTestCollection::runTests`, `testfunction.cc:309-350`, "bulk output" captured into
  `bulkout`/`console->fileoptr`). `min`/`max` bound how many times the pattern is allowed to match
  (`FunctionTestProperty::endTest`, `testfunction.cc:45-49`); a test with `max="0"` is asserting a
  pattern must **not** appear (see condconst.xml's `#12` below).

**Walked-through example: `datatests/condconst.xml`** (full file at
`Ghidra/Features/Decompiler/src/decompile/datatests/condconst.xml`) — exercises the "conditional
constant" Action/Rule pass (replacing a variable's use with a known-constant value along a specific
control-flow path):

1. A ~130-byte x86-64 blob is defined at `ram:0x1006fa` containing three small hand-assembled functions
   (lines 8-24), named via `<symbol>` tags at offsets `0x1006fa`, `0x10076a`, `0x1007a4` (lines 25-27).
2. The script (lines 29-46) declares extern prototypes for all three functions, maps four global
   `int4` variables (`glob1`..`glob4`) onto specific addresses, then for each of the three functions:
   loads it (`lo fu ...`), decompiles it, and prints its C.
3. Twelve `<stringmatch>` assertions (lines 47-58) check the printed output line-by-line: e.g.
   `#1` (`min=1 max=1`) requires exactly one line matching `\*ptr = b;` to appear — asserting the
   decompiler resolved a conditionally-assigned pointer target to the constant path's value rather than
   leaving a more general expression; `#8`/`#9` similarly assert `glob3 = 10;`/`glob4 = 10;` (a *global*,
   not just a local, gets the constant substitution); `#12` (`min=0 max=0`) is a negative assertion that
   `iStack_c = 10;` must **not** appear anywhere — guarding against a known-wrong simplification that
   would over-apply the rule to a stack variable that should stay symbolic (contrasted with `#11`'s
   `iStack_c = 0x14;`, which should).
4. If `ghidra_test datatests condconst.xml` is run, `FunctionTestCollection::runTestFiles`
   (`testfunction.cc:356-397`) loads the file, calls `runTests`, and reports
   `numTestsApplied`/`numTestsSucceeded`; a failing `<stringmatch>` shows up only in the aggregate pass
   count, not as a diff — there is no captured "expected vs. actual" text shown for a failure beyond
   which named property failed.

This is representative of the whole directory (see `switchind.xml` for jump-table/switch recovery,
`union_datatype.xml` for union-typed variable access, `forloop1.xml`/`forloop_*.xml` for loop
structuring, `retstruct.xml` for struct-by-value return ABI recovery, `bitfields.xml`/`bitfields2.xml`
for bitfield access, `namespace.xml`/`inline.xml`/`injectoverride.xml` for various other features) — file
names group loosely by decompiler feature but there is no subdirectory structure or manifest tying a
test file to the specific `Rule`/`Action` class(es) it's meant to guard.

### 3.2 `unittests` — C++ unit tests (7 files)

Directory: `Ghidra/Features/Decompiler/src/decompile/unittests/*.cc`, run by the same `ghidra_test`
binary (`test.cc:106-174`, `unittests`/`datatests` are separate CLI subcommands of one executable). Uses
a minimal home-grown framework, not a third-party library: `TEST(name){...}` registers a function pointer
in a static `vector<UnitTest*>` (`Ghidra/Features/Decompiler/src/decompile/cpp/test.hh:55-84`);
`ASSERT`/`ASSERT_EQUALS`/`ASSERT_NOT_EQUALS` `throw 0` on failure and are caught per-test so one failure
doesn't abort the run (`test.hh:87-113`, `test.cc:25-50`).

The 7 files and what they actually cover:

| File | Lines | Covers |
|---|---|---|
| `testcirclerange.cc` | 928 | `CircleRange` — the value-set/interval-arithmetic class used by constant/range propagation and jump-table bound analysis. |
| `testfuncproto.cc` | 682 | `FuncProto`/parameter-matching logic (function signature & calling-convention layer — see `signature-calling-conventions.md`, 0008). |
| `testmarshal.cc` | 557 | The `PackedEncode`/`PackedDecode` byte format itself (marshal.hh, §1.2 above) round-tripped in isolation. |
| `testtypes.cc` | 263 | `Datatype` comparison/ordering (`compare`, `compareDependency`), `CastStrategy`/`TypeOp::getInputCast` decisions (when a cast is inserted), and `TypeEnum` bit-flag-to-name matching (`enum_matching`/`enum_matching2`). Builds a real minimal `Architecture` via the `"xml"` `ArchitectureCapability` (`testtypes.cc:44-65`) rather than mocking types. |
| `testparamstore.cc` | 369 | `ParamEntry`/storage-assignment logic for parameters (part of the calling-convention layer). |
| `testfloatemu.cc` | 520 | Software float emulation (`FloatFormat`) used for constant-folding float operations. |
| `testmultiprec.cc` | 89 | Multi-precision integer arithmetic helper(s). |

Every unit test here targets a narrow, low-level, mostly *stateless* utility class (range arithmetic,
byte marshaling, type comparison, float emulation, parameter-storage matching) reachable without running
the full pipeline. **None of them instantiate a `Funcdata`, run an `Action`/`Rule` pass over real p-code,
or run the block-structuring algorithm** — those are only exercised indirectly, end-to-end, through
`datatests`.

### 3.3 `test.slow/java` — Java-driven integration tests (17 files)

Directory: `Ghidra/Features/Decompiler/src/test.slow/java/**`, run as ordinary JUnit tests
(`AbstractGhidraHeadedIntegrationTest`/`AbstractGenericTest` subclasses) via Ghidra's Java test
infrastructure, i.e. a completely different mechanism from `ghidra_test` and typically requires spawning
the real native `decompile` process end-to-end.

- `ghidra.app.plugin.core.decompile.DecompilerTest` (61 lines) — the simplest smoke test: builds a
  2-byte `ret`-only program with `ToyProgramBuilder`, opens a `DecompInterface`, decompiles it, and
  asserts `getDecompiledFunction().getC()` is non-null (`DecompilerTest.java:52-60`). This is a plumbing
  test (does the process start and round-trip at all), not a correctness test.
- `AbstractDecompilerTest`/`DecompilerTest`/`DecompilerNavigationTest`/`DecompilerEquateTest`/
  `DecompilerEditDataFieldTest`/`DecompilerHighSymbolTest`/`DecompilerToggleButtonTest`/
  `DecompilerFindDialogTest`/`DecompilerFindReferencesToActionTest`/
  `DecompilerFindReferencesToNestedStructureActionTest` — these decompile one or a few pre-built test
  programs once per test class and then exercise **UI/editing behavior** against the resulting
  `ClangTokenGroup`/`HighFunction`: renaming a variable via the GUI, setting an equate, clicking to
  navigate, retyping a field, finding references. They assert on UI state (selected token, panel text,
  dialog contents), not on the shape of the decompiled C.
- `DecompilerClangTest` (2584 lines) / `DecompilerClang2Test` (283 lines) — the largest files in this
  directory; despite the name, these are still UI-token tests: token/field iteration
  (`assertToken`/`assertTokenIndex`), copy-to-clipboard text, highlighting propagation across duplicate
  variable tokens, cursor/location tracking. They incidentally depend on the decompiler producing a
  *stable* token sequence for their fixture functions, but they assert against specific token text they
  themselves expect (i.e. they'd need updating if legitimate output changed) rather than asserting
  general correctness properties.
- `DecompilerSwitchAnalyzerTest` (214 lines) — tests `DecompilerSwitchAnalyzer`, an auto-analyzer that
  consumes decompiler jump-table output to add flow/labels for switch statements; builds a synthetic
  x86-64 program with `ProgramBuilder` and asserts on the resulting `Program` flow/references — this is
  the one file in this directory that most directly exercises control-flow/jump-table recovery output,
  but through the analyzer's interpretation of it, not through the printed C text.
- `DecompilerCachingTest`, `DecompilerPspecVolatilityTest`, `DecompilerSpecExtensionTest` — narrower
  integration checks (result caching behavior in the panel, `.pspec` volatile-register handling,
  `.cspec`/callfixup extension mechanisms).

## 4. Explicit test-coverage gaps relevant to a refactor

This is the section that feeds `.tasks/0015`/`0019` directly. For each subsystem named by the sibling
review tasks (0004-0009), here is what — concretely — would catch a regression today, and what would
not.

- **Architecture / pipeline / wire protocol (0003 — `Architecture`, `Capability`, the marshal format).**
  *Covered*: `testmarshal.cc` round-trips the `PackedEncode`/`PackedDecode` byte format in isolation
  (§3.2). *Not covered*: nothing checks that the **Java and C++ `ElementId`/`AttributeId` tables agree**
  (§1.2) — a rename/renumber on one side with a stale update on the other produces a silent runtime
  decode mismatch, not a build or test failure, because the two tables are independently-typed source
  files (a Java `record` list and scattered C++ globals) with no shared source of truth or
  cross-checking test. There is also no test that starts a real `DecompileProcess`, kills it mid-command,
  and asserts the auto-respawn in `verifyProcess()`/`initializeProcess()` (§1.1) actually reproduces
  identical state (options, simplification style, signature settings) on the new process — that logic is
  currently only exercised implicitly whenever a `datatests`/`test.slow` run happens not to crash it.

- **SSA / varnode / heritage construction (0004 — `Funcdata`, `Varnode`, `Heritage`, `Cover`/merge).**
  *Covered*: only indirectly, through whichever `datatests` cases happen to depend on correct SSA
  construction to produce their expected printed text (e.g. `partialmerge.xml`, `partialsplit.xml`,
  `varcross.xml`, `revisit.xml`, `dupptr.xml` by name suggest merge/coalescing scenarios). *Not
  covered*: no unit test in `unittests/` builds a `Funcdata`/runs `Heritage` directly and asserts on the
  resulting SSA form, phi-node (`MULTIEQUAL`) placement, or `HighVariable` merge decisions — a change
  here can only be caught by whichever handful of `datatests` regex assertions happen to depend on the
  exact resulting variable split/merge, and a regression that changes *which* variables get merged
  without changing any of the specific matched text (e.g. producing an equally-plausible but
  differently-named/differently-split variable) would very likely pass silently.

- **Type system (0005 — `Datatype` hierarchy, union resolution, composite/function types).**
  *Covered best of any subsystem at the unit level*: `testtypes.cc` directly unit-tests `Datatype`
  ordering/comparison, cast-insertion decisions, and enum bit-matching against a real (if minimal)
  `Architecture` (§3.2). `datatests/union_datatype.xml`, `piecestruct.xml`, `packstructaccess.xml`,
  `threedim.xml`/`twodim.xml`, `impliedfield.xml`, `offcut.xml` exercise composite/array field-access
  recovery end-to-end. *Not covered*: **union field-resolution disambiguation** (choosing which union
  member an access "means") and cross-function type propagation through calls have only the
  `union_datatype.xml`/whatever-named `datatests` cases as a safety net, all string-regex based; there is
  no test asserting the actual `Datatype*` graph/identity produced for a function (e.g. that two
  accesses got unified onto the *same* structure object, not just visually-similar printed text).

- **Action/Rule simplification engine (0006 — `Action`, `Rule`, `RuleAction`, `coreaction.cc`,
  `condexe.cc`, `subflow.cc`; ~500-rule catalog per the epic summary).** *Covered*: entirely by
  `datatests` — every file in the directory is, in effect, a regression pin on *some* subset of the rule
  catalog reachable from the default `"decompile"` action's rule list, exercised as a black box (raw
  bytes in, regex-matched C text out). *Not covered*: there is **no unit-level test of any individual
  `Rule` subclass** — no test instantiates a `RuleAction` and asserts it fires (or doesn't) on a
  constructed `PcodeOp` pattern in isolation. With ~500 rules and 89 `datatests` files, most individual
  rules have **no dedicated test at all**; coverage is whatever those 89 hand-picked bug-repro-style
  scenarios happen to touch, and a change to rule ordering/interaction (a well-known risk category —
  see the companion flaw task 0013) would only be caught if it happens to flip one of the ~89 files'
  regex outcomes. There is no fuzzing, no property-based testing, and no test that asserts the
  simplification engine reaches a fixed point.

- **Control-flow structuring (0007 — `BlockGraph`, `BlockAction`, `JumpTable`, goto elimination).**
  *Covered*: `datatests` files named for loop/switch/if shapes (`forloop1.xml`, `forloop_loaditer.xml`,
  `forloop_thruspecial.xml`, `forloop_varused.xml`, `forloop_withskip.xml`, `noforloop_alias.xml`,
  `noforloop_globcall.xml`, `noforloop_iterused.xml`, `ifswitch.xml`, `ifnoexit.xml`, `elseif.xml`,
  `switchind.xml`, `switchhide.xml`, `switchloop.xml`, `switchmask.xml`, `switchmulti.xml`,
  `switchreturn.xml`, `orcompare.xml`) plus `DecompInterface.structureGraph`
  (`DecompInterface.java:727-759`) is itself a distinct, directly-callable entry point for structuring a
  caller-supplied `BlockGraph`, but nothing in `test.slow/java` or `unittests` calls it — it is untested
  Java-side. `DecompilerSwitchAnalyzerTest` (§3.3) tests the *analyzer* built on top of jump-table output,
  not the structuring algorithm itself. *Not covered*: no test asserts that structuring **never
  regresses to `goto`-heavy output** for a case that used to structure cleanly (the closest proxy is
  whatever specific text a `datatests` regex happens to require); there's no coverage tracking how many
  of the 89 `datatests` programs currently require goto-fallback vs. clean structuring, so a structuring
  regression that still produces *technically valid but uglier* C (more gotos, worse loop shape) can
  pass every existing test.

- **Function signature & calling-convention recovery (0008 — `FuncProto`, `ProtoModel`, `ParamID`,
  override).** *Covered*: `testfuncproto.cc` and `testparamstore.cc` are real unit-level coverage here —
  the best-covered subsystem outside the type system. `datatests/retstruct.xml`, `multiret.xml`,
  `retspecial.xml`, `stackreturn.xml`, `stackspill.xml`, `indproto.xml`, `overridedest.xml`,
  `injectoverride.xml` cover ABI/return/override scenarios end-to-end. *Not covered*: the Java-side
  commit-back path (§2 — `HighFunctionDBUtil.commitParamsToDatabase`, `DecompilerParameterIdCmd`'s
  bulk `"paramid"`-style analysis) has no `test.slow` coverage found in this module's own test tree; a
  refactor that changes what `FuncProto`/parameter info looks like on the wire could silently break
  auto-analysis commit-back without any test here catching it (that plumbing may be tested elsewhere in
  the analysis test suite, outside this module's scope, but nothing under `Decompiler/src/test.slow`
  exercises it).

- **Expression/cast logic & the C pretty-printer (0009 — `PrintLanguage`, `PrintC`, `CastStrategy`,
  `PrettyPrint`).** *Covered*: `testtypes.cc`'s `cast_*`/`type_ordering`/`cast_integertoken` tests
  directly unit-test `CastStrategy`/`TypeOp::getInputCast` decisions (§3.2) — this is real, targeted
  coverage of *when* a cast is emitted. `datatests/displayformat.xml`, `floatprint.xml`, `floatcast.xml`,
  `floatconv.xml`, `nan.xml`, `longdouble.xml`, `sbyte.xml`, `mixfloatint.xml` cover printer/format edge
  cases end-to-end. Java's `PrettyPrinter`
  (`Ghidra/Features/Decompiler/src/main/java/ghidra/app/decompiler/PrettyPrinter.java`) — which turns
  the `ClangTokenGroup` into the final flat/indented text most callers actually consume via
  `DecompileResults.getDecompiledFunction()` — has **no dedicated unit test** anywhere in this module;
  its correctness is only ever checked incidentally by whatever `test.slow` UI tests happen to compare
  small substrings of its output. *Not covered, and the single biggest structural gap of this whole
  area*: **there is no test anywhere in this tree — `datatests`, `unittests`, or `test.slow` — that pins
  the full, exact printed-C text of any function.** Every `datatests` assertion is a small regex over
  one or a few lines with an occurrence-count bound (§3.1); nothing diffs a complete function's output
  against a golden/expected file. That means any change that is locally plausible per-line but shifts
  overall formatting, statement ordering, brace/indentation style, or variable-declaration grouping
  across a whole function has **no test surface at all** to catch it, however severe the cumulative
  effect on readability. Similarly, **there is no performance/timing regression benchmark** anywhere in
  this test tree — no test asserts an upper bound on decompile time or rule-engine iteration count for a
  fixed input, so a refactor that makes the simplification engine converge slower (or fail to converge
  and hit a rule-count safety cutoff) would not be caught by any existing test, only by a human noticing
  the UI feels slower or a real-world decompile timing out at the 30-second default (§1.4).

### Summary table

| Subsystem (review task) | Unit-level (`unittests`) | End-to-end (`datatests`) | Java integration (`test.slow`) | Weakest point |
|---|---|---|---|---|
| Architecture/pipeline/protocol (0003) | marshal format only | indirect | process plumbing smoke test only | Java/C++ `ElementId` table sync is unchecked |
| SSA/varnode/heritage (0004) | none | indirect, incidental | none | no direct SSA-shape assertions anywhere |
| Type system (0005) | strong (`testtypes.cc`) | good | none | union/cross-call type identity unchecked |
| Action/Rule engine (0006) | none | only coverage that exists | none | ~500 rules, ~89 opaque black-box scenarios |
| Control-flow structuring (0007) | none | good, by shape name | one analyzer test | no goto-fallback/quality regression tracking |
| Signature/calling convention (0008) | strong (`testfuncproto.cc`, `testparamstore.cc`) | good | none found in-module | Java commit-back path untested here |
| Expression/cast/printer (0009) | targeted cast coverage | edge-case coverage | incidental only | **no full-text golden output test at all; no perf benchmark** |

## Related documents

- [`.tasks/0010`](../../.tasks/0010-task-review-java-integration-testing.md) — the ticket this document
  satisfies.
- [`architecture-pipeline-overview.md`](architecture-pipeline-overview.md) (0003) — wire-protocol/element
  model detail this document builds on rather than re-deriving.
- [`ssa-varnode-heritage.md`](ssa-varnode-heritage.md) (0004),
  [`type-system.md`](type-system.md) (0005), [`action-rule-engine.md`](action-rule-engine.md) (0006),
  [`control-flow-structuring.md`](control-flow-structuring.md) (0007),
  [`signature-calling-conventions.md`](signature-calling-conventions.md) (0008),
  [`expression-cast-printer.md`](expression-cast-printer.md) (0009) — the subsystem docs the gap
  analysis in §4 is keyed against.
- [`../00-overview/glossary.md`](../00-overview/glossary.md) — shared terminology.
