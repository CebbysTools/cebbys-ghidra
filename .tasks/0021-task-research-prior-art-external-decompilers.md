# 0021 — [TASK] Research: Prior Art in Other Decompilers & Academic Literature

## Type
Task

## Status
`done`

## Parent
0020 (epic) / 0001 (root)

## Objective
Survey how comparable problems (P-code/IR-to-C structuring, type recovery, generics/template handling
in decompiled output) are solved elsewhere, as grounding for the architecture options in 0022.

## Method
Use WebSearch/WebFetch. For each source, capture: what it does differently from Ghidra's current
approach, and what specifically is reusable as an idea (not code — license/IP: do not copy code,
only architectural/algorithmic ideas, and note license of anything referenced).

## Suggested Sources (verify current URLs; add others found)
- RetDec (Avast) — its LLVM-IR-based pipeline; architecture docs/readme.
- angr / pypcode-based tooling — Python ecosystem approaches to P-code-based analysis.
- reko decompiler — open-source, C#, structuring approach.
- Hex-Rays (IDA) — publicly available talks/blog posts on their microcode/ctree pipeline (no source
  access, but their public design talks are a legitimate reference).
- Snowman / nocode / other historical open decompilers, for structuring-algorithm variety.
- Academic literature: "No More Gotos" (Yakdan et al., NDSS 2015), the classic Cifuentes
  "Reverse Compilation Techniques" thesis, DREAM/DREAM++ papers, "A Combinatorial Approach to
  Control-Flow Structuring" or similar interval-analysis papers, and any modern (2020+) paper on type
  recovery or generics/templates recovery from binaries.
- Ghidra's own upstream project: check if NationalSecurityAgency/ghidra has open issues/discussions or
  a design doc about decompiler modernization plans, to avoid duplicating or conflicting with upstream
  direction.

## Deliverable
`.documents/04-solution-research/prior-art-external-decompilers.md`: one section per source, each with
a link, a 3-6 sentence summary, and an explicit "relevance to our requirements" note referencing
specific REQ-IDs from 0016 where applicable.

## Acceptance Criteria
- [x] Document exists, at least 6 distinct sources covered (mix of tools and papers), every source
      linked.
- [x] Every source's summary ends with an explicit relevance note tied to REQ-IDs or flaw entries.
- [x] A closing synthesis paragraph: "the common thread across prior art that's most applicable here
      is..." — this feeds 0022/0023 directly.

## Dependencies
0016 (requirements must exist to make relevance notes meaningful); benefits from 0011.
