# Plan: Behavioral Correctness Enforcement

## Feature Overview

Decompose the behavioral correctness enforcement design (8 decisions, 8 files) into dependency-aware implementation tasks. The design adds layered behavioral verification across 4 pipeline agents (planner, implementer, verifier, reviewer) plus 4 shared reference documents, enforcing that tests invoke production code, ACs are traceable, and new files are wired into runtime.

**Planning Mode:** Initial (no prior plan exists)

---

## Success Criteria (from spec-output.yaml)

| AC    | Summary                                                         | Test Method |
| ----- | --------------------------------------------------------------- | ----------- |
| AC-1  | Schema 6 structured AC objects with id, text, test_method       | inspection  |
| AC-2  | Schema 7 behavioral_coverage field with AC-to-test mapping      | inspection  |
| AC-3  | Verifier behavioral-coverage check_name in verification reports | test        |
| AC-4  | Test files import production modules                            | test        |
| AC-5  | Implementer 3 self-verification items for behavioral checks     | inspection  |
| AC-6  | severity-taxonomy.md 3 Major entries for behavioral violations  | inspection  |
| AC-7  | schemas.md DDL replaced with cross-reference, -200+ lines       | test        |
| AC-8  | Planner self-verification for observable AC behavior            | inspection  |
| AC-9  | test_method propagation from spec through planner to task       | inspection  |
| AC-10 | EG-7 behavioral evidence query in sql-templates.md §6           | inspection  |
| AC-11 | tdd_red_green recording and TDD Fallback handling               | test        |
| AC-12 | Verifier runtime-wiring check for new-file tasks                | test        |
| AC-13 | No TBD/TODO/FIXME in modified files                             | test        |
| AC-14 | Verifier check_name Quick Reference updated                     | inspection  |
| AC-15 | Implementer Test Selection Strategy subsection                  | inspection  |
| AC-16 | Reviewer correctness guidance + pragmatic-verifier priorities   | inspection  |
| AC-17 | schemas.md check_name Naming Patterns updated                   | inspection  |
| AC-18 | All 10 schema examples preserved                                | inspection  |

---

## Ordered Task Index

| #   | Task ID | Title                                                       | Risk | Size     | Wave |
| --- | ------- | ----------------------------------------------------------- | ---- | -------- | ---- |
| 1   | task-01 | schemas.md DDL Deduplication                                | 🟡   | Standard | 1    |
| 2   | task-02 | severity-taxonomy.md & review-perspectives.md Updates       | 🟡   | Standard | 1    |
| 3   | task-03 | sql-templates.md EG-7 Query                                 | 🟡   | Standard | 1    |
| 4   | task-04 | schemas.md Schema 6 & 7 Field Updates + check_name Patterns | 🟡   | Standard | 2    |
| 5   | task-05 | planner.agent.md Structured AC Propagation                  | 🟡   | Standard | 3    |
| 6   | task-06 | implementer.agent.md Behavioral TDD & Self-Verification     | 🟡   | Standard | 3    |
| 7   | task-07 | verifier.agent.md Behavioral Checks                         | 🟡   | Standard | 3    |
| 8   | task-08 | adversarial-reviewer.agent.md Correctness Guidance          | 🟡   | Standard | 2    |

---

## Execution Waves

### Wave 1 — Foundation (Independent)

**Max concurrent:** 3

| Task    | File(s)                                      | Dependencies |
| ------- | -------------------------------------------- | ------------ |
| task-01 | schemas.md                                   | none         |
| task-02 | severity-taxonomy.md, review-perspectives.md | none         |
| task-03 | sql-templates.md                             | none         |

**Rationale:** These tasks modify independent files with no cross-dependencies. task-01 creates line-count headroom in schemas.md for subsequent field additions. task-02 establishes severity entries referenced by downstream reviewer guidance. task-03 adds an independent evidence gate query.

### Wave 2 — Schema Fields & Reviewer Guidance

**Max concurrent:** 2

| Task    | File(s)                       | Dependencies |
| ------- | ----------------------------- | ------------ |
| task-04 | schemas.md                    | task-01      |
| task-08 | adversarial-reviewer.agent.md | task-02      |

**Rationale:** task-04 adds behavioral fields to schemas.md after DDL removal creates headroom. task-08 expands reviewer correctness guidance after severity entries exist (so referenced severities are defined).

### Wave 3 — Agent Definitions

**Max concurrent:** 3

| Task    | File(s)              | Dependencies |
| ------- | -------------------- | ------------ |
| task-05 | planner.agent.md     | task-04      |
| task-06 | implementer.agent.md | task-04      |
| task-07 | verifier.agent.md    | task-04      |

**Rationale:** All three agent definitions reference Schema 6/7 field definitions from schemas.md. They must wait for task-04 to establish the canonical schema structure so inline schemas and behavioral checks reference the correct field names and types.

---

## Dependency Graph

```
task-01 ─────────────────┐
                         ├──→ task-04 ──┬──→ task-05
task-02 ──→ task-08      │              ├──→ task-06
                         │              └──→ task-07
task-03 (independent)    │
```

---

## Risk Summary

### Overall Risk: 🟡

All 8 tasks modify agent definition or shared reference files (🟡 Business Logic). No files touch auth, crypto, data deletion, or public API surface (🔴 criteria). The primary risk is cross-file consistency: Schema 6/7 changes in schemas.md must be reflected consistently in planner, implementer, and verifier inline schemas.

### Per-Task Risk Breakdown

| Task    | File Risk                               | Task Risk | Rationale                                                 |
| ------- | --------------------------------------- | --------- | --------------------------------------------------------- |
| task-01 | schemas.md: 🟡                          | 🟡        | Large deletion (~286 lines); must preserve unique content |
| task-02 | severity-taxonomy: 🟡, review-persp: 🟢 | 🟡        | Severity entries affect routing decisions                 |
| task-03 | sql-templates: 🟡                       | 🟡        | New SQL query; must be syntactically valid                |
| task-04 | schemas.md: 🟡                          | 🟡        | Breaking type change (string→object), version 2.0         |
| task-05 | planner.agent.md: 🟡                    | 🟡        | Inline schema must match schemas.md canonical definition  |
| task-06 | implementer.agent.md: 🟡                | 🟡        | Largest line addition (~66 lines); must stay under 400    |
| task-07 | verifier.agent.md: 🟡                   | 🟡        | Unconditional framing critical (review finding major)     |
| task-08 | adversarial-reviewer.agent.md: 🟡       | 🟡        | Must reference correct severity entries                   |

### Design Review Constraints (Incorporated)

| Finding                                            | Severity | Affected Tasks   | Mitigation                                                                |
| -------------------------------------------------- | -------- | ---------------- | ------------------------------------------------------------------------- |
| Schema 6 type change = breaking, not additive      | Major    | task-04          | Version bump 2.0 (not 1.1); document as breaking with synchronized update |
| Tier 2 conditional framing mismatch                | Major    | task-07          | Explicit unconditional sub-heading for behavioral checks (Steps 3e–3f)    |
| behavioral_coverage status enum gap (TDD Fallback) | Major    | task-04, task-06 | Document `not_applicable` + justification covers TDD Fallback scenarios   |
| F-3 narrowing gap overclaimed                      | Major    | task-05          | AC notes planner propagation "narrows" gap, not "closes" it               |
| EG-7 promotion criteria undefined                  | Minor    | task-03          | Add promotion criteria note in query comment                              |
| tdd_red_green self-attested                        | Minor    | task-08          | Reviewer guidance flags suspicious tdd_red_green patterns                 |

---

## Pre-Mortem Analysis

| Task    | Failure Scenario                                                               | Likelihood | Impact | Mitigation                                                                   |
| ------- | ------------------------------------------------------------------------------ | ---------- | ------ | ---------------------------------------------------------------------------- |
| task-01 | DDL removal accidentally deletes check_name Naming Patterns or phase semantics | M          | H      | AC explicitly requires preserving unique content; impl reads before deleting |
| task-02 | Major severity entries use vague language that doesn't trigger detection       | L          | M      | ACs require exact named entries matching design spec                         |
| task-03 | EG-7 SQL syntax error against anvil_checks schema                              | L          | L      | Query follows existing EG-1..6 patterns; validator can check syntax          |
| task-04 | Schema 6 example inconsistent with field table after type change               | M          | H      | AC checks example matches field definitions; cross-referenced                |
| task-05 | Planner inline Schema 6 diverges from schemas.md canonical                     | M          | H      | Explicit dependency on task-04; AC requires matching field definitions       |
| task-06 | Implementer exceeds 400-line budget after additions                            | M          | M      | Estimated 334+66=400 lines; implementer must check final count               |
| task-07 | Behavioral checks placed under conditional Tier 2 heading                      | H          | H      | AC requires unconditional sub-heading; review finding incorporated           |
| task-08 | Reviewer guidance references undefined severity entries                        | L          | L      | Explicit dependency on task-02 ensures entries exist first                   |

### Overall Risk Level: Medium

**Justification:** All changes are to instruction text and schema definitions (no runtime code), limiting blast radius. The primary risk vectors are cross-file consistency (Schema 6/7 definitions referenced by 4 agents) and the breaking Schema 6 type change requiring synchronized updates. Both are mitigated by dependency ordering (schemas.md first, consumers second).

### Key Assumptions

1. **schemas.md DDL section is ~lines 961-1246** — If section boundaries have shifted, task-01 must locate the correct boundaries. Tasks task-04..task-07 depend on this.
2. **Implementer budget is 400 lines** — The implementer agent file is currently 334 lines. If it has grown, task-06 additions (~66 lines) may exceed budget.
3. **Schema 6 consumers are only Planner, Implementer, Verifier** — Per schemas.md dependency table. If additional consumers exist, the breaking change has broader impact.
4. **Verifier Tier 2 heading is at ~line 171** — task-07 must locate and amend this section. If structure has changed, the unconditional sub-heading placement needs adjustment.
