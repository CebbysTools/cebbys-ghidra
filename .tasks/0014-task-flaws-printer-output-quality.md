# 0014 — [TASK] Flaws: Expression/Cast Logic & Printer Output Quality

## Type
Task

## Status
`todo`

## Parent
0011 (epic) / 0001 (root)

## Objective
Critically assess cast-insertion logic and the C pretty-printer for correctness risk, readability
issues, and language-extensibility limits (this feeds directly into the "rich language features"
requirement, since the printer is where language-specific syntax ultimately has to land).

## Inputs (read first)
- `.documents/01-review/expression-cast-printer.md` (0009)
- Source as needed: `cast.hh`, `printc.hh`, `printlanguage.hh`.

## Analysis Focus
1. **Cast correctness/readability tension**: known cases where `CastStrategy` over- or under-inserts
   casts (check source comments, TODOs, and issue-shaped comments). Is the strategy pattern actually
   swappable today, or is `PrintC` still C-specific in ways that leak past the abstraction?
2. **PrintLanguage as an abstraction**: given the PrintC vs. PrintJava comparison from 0009, is
   `PrintLanguage` a genuinely reusable base, or C-shaped with Java bolted on? Concrete evidence either
   way.
3. **Declaration printing for complex types**: function pointers, arrays of pointers, nested
   const/volatile — does the current recursive declarator printer handle the full space correctly, or
   are there known gaps (again: search source comments/TODOs)?
4. **What would templates/generics output even look like** in a C-family pretty printer, given the
   current design? Identify the specific architectural blocker (e.g. "the printer has no concept of a
   type parameter list because Datatype has none" — cross-reference 0012).
5. **Improvement options**: e.g. separate "cast necessity" (semantic) from "cast style" (presentation)
   more cleanly, introduce a proper visitor/AST intermediate between structured blocks and text instead
   of direct emission, etc.

## Deliverable
`.documents/02-flaws/printer-output-quality.md`, structured per the parent epic's format.

## Acceptance Criteria
- [ ] Document exists, follows required structure, cites file:line/review-doc throughout.
- [ ] Direct, explicit link established between "no template/generic support" and the specific
      printer/type-system limitation causing it (cross-ref 0012).
- [ ] At least 3 concrete improvement options with pros/cons.

## Dependencies
0009 (and benefits from 0012 for the type-system side of the cross-reference).
