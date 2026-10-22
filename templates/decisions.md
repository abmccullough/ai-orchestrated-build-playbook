# Implementation decisions

Decisions taken while planning and building that the design does not state. Design changes themselves go into
`design/`; this file records the decision and why. The orchestrator adds an entry whenever it resolves a gap.

**Testing rules are not read from here.** They live, current, in `plan/testing-discipline.md`; testing entries here
point to it and remain as history. A new testing decision edits that file in the same commit.

**How to read this file.** It records the decision and why, not the specification. Once shipped, the specification is
the code, its tests and the module documentation; the entry says what was chosen and what it ruled out, so a later batch
can tell a deliberate choice from an accident.

**Numbers are a stable interface.** Code, briefs and frozen migrations cite entries and sub-clauses by number
(`IMP-17 (3)`). Never reuse or renumber; a superseded entry or clause stays and points at what replaced it.

**Still open for the owner's review:** <none | list of entries marked "for review">

---

## IMP-1 — <title>
*<date> · <the owner | orchestrator; for review | decided while the owner was away; for review>*

<The decision. Numbered sub-clauses if it has parts: (1) … (2) …>

*Why:* <the reason; quote the owner where their words matter>
*Rules out:* <what was considered and rejected>
*Revisit:* <what would make you change it>
*Recorded in:* <files edited: design sections, testing-discipline, briefs>
