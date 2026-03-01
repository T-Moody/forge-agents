# Behavioral Correctness Enforcement — Design

**Feature:** `behavioral-correctness-enforcement`
**Designer:** designer
**Date:** 2026-03-01
**Direction:** A — Structural Enforcement Within Existing Architecture

---

## Summary

This design adds behavioral correctness safeguards to the existing 10-step agent pipeline by modifying 8 files with additive changes. No new agents, pipeline steps, or review categories are introduced. The core strategy is distributed enforcement: the planner propagates structured acceptance criteria with `test_method`, the implementer self-checks for production-code invocation in tests, the verifier adds behavioral-coverage and runtime-wiring checks to Tier 2, the adversarial reviewer gets expanded correctness guidance, and the severity taxonomy reclassifies self-referential tests as Major. Supporting changes deduplicate ~260 lines from schemas.md and add an informational evidence gate (EG-7).

---

## Context & Inputs

| Source                 | File                         | Key Data                                                                                                           |
| ---------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Spec                   | `spec-output.yaml`           | 10 FR groups (27 sub-requirements), 18 ACs, 8 edge cases, 9 common requirements                                    |
| Feature                | `feature.md`                 | Problem statement, root cause analysis, architectural directions                                                   |
| Research: Architecture | `research/architecture.yaml` | 10 structural gaps (TDD enforcement, cascade blindness, evidence gate quality, AC narrowing)                       |
| Research: Impact       | `research/impact.yaml`       | 6 agent files + 2 shared docs impacted (~4,800 total lines), schema cascade risk                                   |
| Research: Dependencies | `research/dependencies.yaml` | Cross-reference map (9 agents × 9 shared docs), evidence chain gaps, AC traceability loss                          |
| Research: Patterns     | `research/patterns.yaml`     | Concrete ui-bug-fixes evidence (self-referential tests, Minor severity classification, zero coverage pass-through) |

---

## High-Level Architecture

The existing pipeline architecture is preserved exactly. Changes are distributed across the implementation→verification→review chain to create layered behavioral enforcement:

```
Planner (Step 4)
  └─ Structured AC objects: {id, text, test_method} propagated from spec
       │
Implementer (Step 5)
  ├─ TDD with production-import enforcement
  ├─ Anti-self-referential assertion rule
  ├─ behavioral_coverage mapping in report
  ├─ Runtime wiring self-check (new files)
  └─ Test selection strategy (behavior-not-implementation)
       │
Verifier (Step 6)
  ├─ Tier 2 check: behavioral-coverage (AC→test mapping, import verification)
  └─ Tier 2 check: runtime-wiring (new files referenced by existing code)
       │
Adversarial Reviewer (Step 7)
  ├─ Correctness guidance: self-referential detection, AC coverage, wiring
  └─ Severity: self-referential tests = Major (triggers needs_revision)
       │
EG-7 (informational) — behavioral evidence exists per code task
```

### Files Modified

| File                            | Lines Before | Est. Lines After | Net Change | Primary FRs      |
| ------------------------------- | ------------ | ---------------- | ---------- | ---------------- |
| `implementer.agent.md`          | 334          | ~380             | +46        | FR-2, FR-6, FR-9 |
| `verifier.agent.md`             | 317          | ~355             | +38        | FR-3, FR-5       |
| `planner.agent.md`              | 337          | ~352             | +15        | FR-1, FR-8       |
| `adversarial-reviewer.agent.md` | 336          | ~348             | +12        | FR-10            |
| `schemas.md`                    | 1422         | ~1192            | -230       | FR-1, FR-5, FR-7 |
| `severity-taxonomy.md`          | 172          | ~190             | +18        | FR-4             |
| `sql-templates.md`              | 699          | ~720             | +21        | FR-5             |
| `review-perspectives.md`        | 143          | ~153             | +10        | FR-10            |

All agent files stay under 400 lines (NFR-2). schemas.md drops to ~1192 (under 1300 target, NFR-5/FR-7.3).

---

## Data Models & DTOs

### Task Schema (Schema 6) — Updated `acceptance_criteria` Field

**Current format (list of strings):**

```yaml
acceptance_criteria:
  - "Email validation rejects malformed addresses"
  - "Rate limiting returns 429 after 5 attempts per minute"
```

**New format (list of structured objects):**

```yaml
acceptance_criteria:
  - id: "AC-3"
    text: "Email validation rejects malformed addresses"
    test_method: "test"
  - id: "AC-5"
    text: "Rate limiting returns 429 after 5 attempts per minute"
    test_method: "test"
  - id: "AC-7"
    text: "Error messages use consistent color token"
    test_method: "inspection"
```

Fields:

- `id` (string, required): Parent spec AC ID for traceability
- `text` (string, required): Criterion text
- `test_method` (string, required): `test` | `inspection` | `demonstration` | `analysis` — propagated from spec

### Implementation Report (Schema 7) — New Fields

```yaml
payload:
  # ... existing fields ...
  behavioral_coverage: # NEW (required for task_type='code')
    - ac_id: "AC-3"
      test_file: "src/auth/__tests__/validator.test.ts"
      test_name: "rejects malformed email addresses"
      status: "covered" # covered | not_applicable
    - ac_id: "AC-7"
      status: "not_applicable"
      justification: "test_method=inspection, visual-only criterion"
  tdd_red_green: # NEW (required for task_type='code' when tests written)
    tests_written_first: true
    initial_run_failures: 3 # count of failures on first run after writing tests
    initial_run_exit_code: 1 # exit code of test run before production code
```

### New Fields Summary

| Schema   | Field                                         | Type    | Required         | Default  |
| -------- | --------------------------------------------- | ------- | ---------------- | -------- |
| Schema 6 | `acceptance_criteria[].id`                    | string  | Yes              | —        |
| Schema 6 | `acceptance_criteria[].text`                  | string  | Yes              | —        |
| Schema 6 | `acceptance_criteria[].test_method`           | string  | Yes              | `"test"` |
| Schema 7 | `payload.behavioral_coverage`                 | list    | Yes (code tasks) | `[]`     |
| Schema 7 | `payload.behavioral_coverage[].ac_id`         | string  | Yes              | —        |
| Schema 7 | `payload.behavioral_coverage[].test_file`     | string  | Conditional      | —        |
| Schema 7 | `payload.behavioral_coverage[].test_name`     | string  | Conditional      | —        |
| Schema 7 | `payload.behavioral_coverage[].status`        | string  | Yes              | —        |
| Schema 7 | `payload.behavioral_coverage[].justification` | string  | Conditional      | —        |
| Schema 7 | `payload.tdd_red_green`                       | object  | Conditional      | `null`   |
| Schema 7 | `payload.tdd_red_green.tests_written_first`   | boolean | Yes              | —        |
| Schema 7 | `payload.tdd_red_green.initial_run_failures`  | integer | Yes              | —        |
| Schema 7 | `payload.tdd_red_green.initial_run_exit_code` | integer | Yes              | —        |

All new fields are additive. Schema version bumps from `1.0` to `1.1` per evolution strategy.

---

## APIs & Interfaces

### New check_name Values

| check_name            | Phase   | Tier | Used By  | Description                                                                                      |
| --------------------- | ------- | ---- | -------- | ------------------------------------------------------------------------------------------------ |
| `behavioral-coverage` | `after` | 2    | Verifier | Behavioral coverage verification — checks AC-to-test mapping, production imports, file existence |
| `runtime-wiring`      | `after` | 2    | Verifier | Runtime wiring verification — checks new files are referenced by existing code                   |

### Evidence Gate Query: EG-7

```sql
-- EG-7: Behavioral evidence exists per code task (INFORMATIONAL)
-- Expected: ≥1 for each code task; logged, not blocking
SELECT task_id, COUNT(*) AS behavioral_signals
FROM anvil_checks
WHERE run_id = '{run_id}'
  AND phase = 'after'
  AND check_name = 'behavioral-coverage'
  AND passed = 1
GROUP BY task_id;
```

---

## Sequence / Interaction Notes

### Behavioral Enforcement Flow (Happy Path)

1. **Spec** emits AC with `test_method` field (existing behavior, no change)
2. **Planner** reads spec ACs, creates task with structured AC objects preserving `{id, text, test_method}` (**D-1**)
3. **Implementer** reads task ACs, consults `test_method`:
   - `test` → writes automated test importing production module, records in `behavioral_coverage`
   - `inspection` → maps AC as `not_applicable` with justification
   - `demonstration` → captures runtime evidence (screenshot/log)
   - `analysis` → runs static analysis or metric check
4. **Implementer** self-verifies: production import check ✓, anti-self-referential ✓, behavioral_coverage populated ✓, runtime wiring ✓ (**D-2**)
5. **Verifier** Tier 2 runs `behavioral-coverage` check: reads `behavioral_coverage` from implementation report, verifies completeness for `test_method='test'` ACs, checks test files exist and import production modules (**D-3**)
6. **Verifier** Tier 2 runs `runtime-wiring` check (if new files created): searches for references to new files from existing code (**D-3**)
7. **Adversarial Reviewer** correctness analysis: checks for self-referential tests, AC coverage, wiring (**FR-10**)
8. **Severity taxonomy**: self-referential tests classified Major → needs_revision routing triggered if found (**D-4**)

### Self-Referential Test Detection (Caught at Implementer)

1. Implementer writes test declaring `bool cancelWasCalled = false` and asserts `Assert.False(cancelWasCalled)` without calling production code
2. Self-verification check "Tests invoke production code, not local variable replications" → **FAILS**
3. Self-fix loop: implementer rewrites test to call production `OnCancel()` and assert on production state
4. Corrected test passes self-verification → proceeds to verifier
5. Problem caught at Step 5, not Step 7 (review) — saving 2 pipeline steps of wasted work

### Edge Case: Visual-Only Task (EC-1)

1. All task ACs have `test_method: "inspection"`
2. Implementer maps all ACs to `status: "not_applicable"` in `behavioral_coverage`
3. Verifier behavioral-coverage check: all ACs mapped → **passed=1** (inspection-only criteria exempt)
4. No false failure for legitimate visual tasks

### Edge Case: Pipeline Resume (EC-4)

1. Tasks verified before behavioral enforcement won't have `behavioral-coverage` records
2. EG-7 is informational (non-blocking) → pipeline proceeds normally
3. Only new tasks going through the updated agents get behavioral checks

---

## Security Considerations

**No security implications.** All changes are to agent instruction text and schema definitions — no runtime code, no authentication changes, no data flow changes, no secrets handling changes.

Justification:

- New schema fields are optional/additive with no security-sensitive data
- behavioral-coverage and runtime-wiring checks use existing read-only tools (grep_search, read_file)
- EG-7 query follows existing sql_escape() and stdin piping patterns (SEC-2)
- No new tool access grants (verifier already has grep_search and read_file)
- No changes to the Security Blocker Policy or escalation logic

---

## Failure & Recovery

| Failure Mode                                                               | Likelihood | Impact | Recovery Strategy                                                                                                                                                                                                                            |
| -------------------------------------------------------------------------- | ---------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Implementer produces behavioral_coverage with incorrect AC IDs             | Medium     | Medium | Verifier cross-references AC IDs from task schema against implementation report; mismatch → behavioral-coverage check fails → NEEDS_REVISION                                                                                                 |
| behavioral-coverage check too strict (false positives on legitimate tasks) | Medium     | High   | Check has explicit exemptions for test_method='inspection' (EC-1), TDD Fallback (EC-2), and modification-only tasks (EC-3 for runtime-wiring). If still too strict, EG-7 is informational so won't block pipeline while tuning               |
| Language-specific import analysis fails (dynamic imports, XAML)            | Low        | Medium | Production-import check supports multi-language patterns (import/require/using/include). EC-6 allows justification override for difficult cases                                                                                              |
| schemas.md DDL removal breaks agent understanding                          | Low        | Low    | Agents already reference sql-templates.md for canonical DDL. The illustrative DDL was marked as non-authoritative. Cross-reference section provides table names and descriptions                                                             |
| Planner fails to propagate test_method to task ACs                         | Medium     | High   | Verifier behavioral-coverage check expects test_method in task ACs. Missing test_method defaults to 'test' (safe default). Self-verification checklist item in planner catches omission before output                                        |
| Self-verification items ignored by agent                                   | Medium     | Medium | This is the fundamental limitation of instruction-based enforcement (patterns F-7). Mitigated by layered enforcement — implementer self-check + verifier cascade check + reviewer correctness analysis create 3 independent detection layers |

### Graceful Degradation

If any single enforcement layer fails:

- **Implementer self-check missed** → Verifier behavioral-coverage catches it in Tier 2
- **Verifier behavioral-coverage missed** → Reviewer correctness analysis with Major severity catches it
- **Reviewer missed** → EG-7 (informational) logs the absence for post-mortem detection by Knowledge Agent
- All three layers must fail for a self-referential test to pass undetected (vs. current state where all three layers are structurally blind)

---

## Non-Functional Requirements

| NFR                                 | Design Approach                                                                                                                           | Compliance   |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| NFR-1: <50% execution time increase | Behavioral checks use static file analysis (grep_search, read_file), not runtime execution. No mutation testing, no additional test runs. | ✅ Compliant |
| NFR-2: No agent file >400 lines     | Largest addition is implementer (+46 lines → ~380). All agents stay under 400. Orchestrator exempt.                                       | ✅ Compliant |
| NFR-3: YAML parser compatibility    | All schema changes are additive optional fields with defaults. No field removals or type changes.                                         | ✅ Compliant |
| NFR-4: Multi-language support       | Production import check supports C# (using), TypeScript/JavaScript (import/require), Python (import/from). EC-6 handles edge cases.       | ✅ Compliant |
| NFR-5: schemas.md <1300 lines       | DDL removal (-260) + additions (+30) → ~1192 lines.                                                                                       | ✅ Compliant |

---

## Migration & Backwards Compatibility

### Schema Migration

- **Schema 6 (task-schema):** `acceptance_criteria` changes from `list of strings` to `list of objects`. This is technically a breaking change for the field type. **Mitigation:** Planner (sole producer) and Implementer/Verifier (consumers) are all updated synchronously. No external consumers exist.
- **Schema 7 (implementation-report):** New optional fields `behavioral_coverage` and `tdd_red_green`. Additive — existing reports without these fields remain valid. Consumers (Verifier) handle missing fields as absent.
- **Schema version:** Both schemas bump from `1.0` to `1.1`.

### Pipeline Resume Compatibility

- **EG-7 is informational:** Tasks verified before behavioral enforcement have no `behavioral-coverage` records. EG-7 logs the absence but does not block (EC-4).
- **New check_names are additive:** `behavioral-coverage` and `runtime-wiring` add to the verification signal count within existing EG-2 thresholds. Tasks without them still pass EG-2 via existing checks.
- **Severity taxonomy entries are additive:** New Major examples don't change the treatment of existing findings.

### Synchronized Update Order

Files must be updated in this order to prevent intermediate inconsistency:

1. `schemas.md` (DDL deduplication + new fields) — central registry first
2. `sql-templates.md` (EG-7 query) — evidence infrastructure
3. `severity-taxonomy.md` (Major entries) — classification infrastructure
4. `review-perspectives.md` (behavioral priorities) — reviewer guidance
5. `planner.agent.md` (structured ACs) — AC producer
6. `implementer.agent.md` (behavioral enforcement) — code producer
7. `verifier.agent.md` (behavioral checks) — verification
8. `adversarial-reviewer.agent.md` (correctness guidance) — review

---

## Testing Strategy

| Test Type           | What                               | How                                                                                                   |
| ------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Schema inspection   | Schema 6 structured AC objects     | Inspect schemas.md and planner inline schema for {id, text, test_method} fields                       |
| Schema inspection   | Schema 7 behavioral_coverage field | Inspect schemas.md and implementer inline schema for field definition                                 |
| Schema inspection   | check_name entries                 | Inspect schemas.md Naming Patterns + verifier Quick Reference for behavioral-coverage, runtime-wiring |
| Content inspection  | Implementer checklist              | Inspect implementer.agent.md Self-Verification for 3 behavioral items                                 |
| Content inspection  | Planner checklist                  | Inspect planner.agent.md Self-Verification for observable behavior item                               |
| Content inspection  | Severity taxonomy                  | Inspect severity-taxonomy.md Major section for 3 new entries                                          |
| Content inspection  | Reviewer guidance                  | Inspect adversarial-reviewer.agent.md + review-perspectives.md for behavioral items                   |
| Line count          | schemas.md DDL reduction           | Count lines after DDL removal — must be ≤1222 (1422 - 200)                                            |
| Query validation    | EG-7 SQL syntax                    | Validate query against anvil_checks schema                                                            |
| Integration trace   | AC propagation                     | Trace AC from spec-output.yaml → task.yaml → implementation-report.yaml — test_method preserved       |
| Behavioral scenario | Self-referential test detection    | Simulate: test with no production import → implementer self-check fails → self-fix                    |
| Behavioral scenario | New file wiring                    | Simulate: new file created → no existing import → runtime-wiring check fails                          |

---

## Tradeoffs & Alternatives Considered

### D-1: Acceptance Criteria Format — Structured Objects in Task Schema

**Context:** FR-1 requires structured AC flow with spec AC ID, text, and test_method. Currently `task.acceptance_criteria` is a list of strings, losing the `test_method` metadata defined in spec-output (architecture F-3, F-8).

| Alternative                                   | Pros                                                                                    | Cons                                                               |
| --------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Alt 1: Structured objects (SELECTED)**      | Full traceability, test_method consumed downstream, enables behavioral_coverage mapping | Modifies Schema 6 field type (mitigated: synchronized update)      |
| Alt 2: Keep strings + parallel metadata field | No change to existing field                                                             | Two fields to maintain, drift risk, positional correlation fragile |

**Selected:** Alt 1 — Structured objects with {id, text, test_method}
**Confidence:** High — clear spec requirement, comparable pattern exists in spec-output schema, all consumers updated synchronously
**Risk:** 🟡 — modifies business logic (schema definition consumed by multiple agents)

---

### D-2: Behavioral Test Enforcement — Inline Additions to Implementer

**Context:** FR-2 requires structural guards against self-referential tests. The implementer currently has prose TDD rules with no structural check that tests actually invoke production code (patterns F-1).

| Alternative                                    | Pros                                                                                        | Cons                                                          |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Alt 1: Inline additions (SELECTED)**         | Minimal footprint (~46 lines), stays under 400-line budget, uses existing checklist pattern | Adds to already-full file                                     |
| Alt 2: Dedicated Behavioral Validation section | Clean separation                                                                            | ~60 lines, risks 400-line budget, breaks section organization |

**Selected:** Alt 1 — inline additions to Step 4, Output Schema, Operating Rules, and Self-Verification
**Confidence:** High — follows established agent structure, concrete change locations identified, line budget verified
**Risk:** 🟡 — modifying core implementer TDD workflow

---

### D-3: Verifier Behavioral Checks — Tier 2 Placement

**Context:** FR-3 requires new behavioral-coverage and runtime-wiring verification checks. The existing 4-tier cascade has no behavioral quality assessment (architecture F-2, patterns F-3).

| Alternative                         | Pros                                                           | Cons                                                                   |
| ----------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------- |
| **Alt 1: New Tier 2.5**             | Clear separation                                               | Introduces new tier concept, renumbers documentation, over-engineering |
| **Alt 2: Tier 2 checks (SELECTED)** | Natural home, counts toward EG-2, unconditional for code tasks | Tier 2 grows larger                                                    |
| Alt 3: Tier 3 checks                | Tier 3 is runtime-focused                                      | Tier 3 is conditional, behavioral checks must always run               |

**Selected:** Alt 2 — Tier 2 as Steps 3e (behavioral-coverage) and 3f (runtime-wiring)
**Confidence:** High — follows existing Tier 2 patterns, checks use existing tools, no cascade restructuring needed
**Risk:** 🟡 — modifying verification cascade logic

---

### D-4: Severity Recalibration — Explicit Major Entries

**Context:** FR-4 requires self-referential tests classified as Major. Currently they're implicitly Minor, preventing remediation (patterns F-8 — concrete evidence from ui-bug-fixes).

| Alternative                                      | Pros                                                 | Cons                                                                             |
| ------------------------------------------------ | ---------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Alt 1: Add explicit Major entries (SELECTED)** | Minimal change, uses existing verdict infrastructure | None significant                                                                 |
| Alt 2: New severity sub-category                 | Clear delineation                                    | Breaks 4-level taxonomy, requires updating all verdict logic and SQL constraints |

**Selected:** Alt 1 — Add 3 named entries to severity-taxonomy.md Major section
**Confidence:** High — straightforward additive change, existing verdict routing handles Major automatically
**Risk:** 🟢 — additive to reference document

---

### D-5: schemas.md DDL Deduplication — Cross-Reference Replacement

**Context:** CR-4/FR-7 requires replacing ~286 lines of duplicated SQLite DDL with cross-references. schemas.md contains illustrative DDL already marked as non-authoritative (impact F-7).

| Alternative                                | Pros                                                                       | Cons                                                      |
| ------------------------------------------ | -------------------------------------------------------------------------- | --------------------------------------------------------- |
| **Alt 1: Full cross-reference (SELECTED)** | Removes all duplication, consistent with evidence gate pattern, -260 lines | Less inline DDL visibility                                |
| Alt 2: Abbreviated DDL (column names only) | Some inline visibility                                                     | Still duplicated, still takes lines, still has drift risk |

**Selected:** Alt 1 — Replace with ~25-line cross-reference section listing table names
**Confidence:** High — follows existing pattern (evidence gate queries already cross-reference sql-templates.md §6)
**Risk:** 🟢 — removing duplication, no functional change

---

### D-6: Planner AC Quality — Checklist Items and test_method Propagation

**Context:** FR-8 requires planner to emit observable behavior criteria and propagate test_method from spec ACs.

| Alternative                                                   | Pros                                         | Cons                                                 |
| ------------------------------------------------------------- | -------------------------------------------- | ---------------------------------------------------- |
| **Alt 1: Checklist items + rules (SELECTED)**                 | Uses existing patterns, low line count (~15) | Self-checks are self-reported                        |
| Alt 2: Structural validation with vague-term pattern matching | More structural                              | Brittle — unbounded vague term list, over-engineered |

**Selected:** Alt 1 — Self-verification item + workflow propagation rules
**Confidence:** High — follows existing planner pattern, concrete additions identified
**Risk:** 🟢 — additive to planner instructions

---

### D-7: Test Selection Strategy — Inline in Implementer Operating Rules

**Context:** FR-9 requires codified test selection rules so the implementer makes principled choices about what to test and how.

| Alternative                                        | Pros                                                  | Cons                                                                      |
| -------------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------- |
| **Alt 1: Inline subsection (SELECTED)**            | Co-located with consumer, single reference, ~20 lines | Only visible in implementer file                                          |
| Alt 2: Separate shared document (test-strategy.md) | Reusable                                              | Only one real consumer (implementer), adds context window pressure, YAGNI |

**Selected:** Alt 1 — New "Test Selection Strategy" subsection in Operating Rules
**Confidence:** High — single consumer, no reuse case, YAGNI principle
**Risk:** 🟢 — additive to implementer instructions

---

### D-8: Evidence Gate EG-7 — Informational Behavioral Query

**Context:** FR-5.1 requires a new evidence gate query checking behavioral evidence per code task. Must handle pipeline resume (EC-4).

| Alternative                               | Pros                                          | Cons                                                     |
| ----------------------------------------- | --------------------------------------------- | -------------------------------------------------------- |
| **Alt 1: Informational query (SELECTED)** | Low risk, handles EC-4, can be promoted later | Not blocking — gaps can still pass                       |
| Alt 2: Required gate immediately          | Immediate enforcement                         | Breaks pipeline resume (EC-4), sudden threshold increase |

**Selected:** Alt 1 — Informational EG-7, logged but not blocking
**Confidence:** High — spec explicitly says "informational initially" (FR-5.1), handles EC-4 gracefully
**Risk:** 🟢 — additive query, non-blocking

---

## Implementation Checklist & Deliverables

### Per-File Change Specification

#### 1. schemas.md (FR-1, FR-5, FR-7)

- [ ] Replace SQLite Schemas section (~lines 961-1246) with ~25-line cross-reference section → **AC-7**
- [ ] Update Schema 6 `task-schema` fields: `acceptance_criteria` from `list of strings` to `list of objects` with `id`, `text`, `test_method` → **AC-1**
- [ ] Update Schema 6 example to show structured AC objects → **AC-1, AC-18**
- [ ] Add `behavioral_coverage` and `tdd_red_green` fields to Schema 7 field table → **AC-2**
- [ ] Update Schema 7 example to include `behavioral_coverage` → **AC-2, AC-18**
- [ ] Add `behavioral-coverage` and `runtime-wiring` to check_name Naming Patterns table → **AC-17**
- [ ] Verify all 10 schema examples preserved → **AC-18**
- [ ] Verify total line count <1300 → **AC-7**

#### 2. sql-templates.md (FR-5)

- [ ] Add EG-7 query to §6 → **AC-10**

#### 3. severity-taxonomy.md (FR-4)

- [ ] Add Major entry: "Tests that do not invoke production code (self-referential tests)" → **AC-6**
- [ ] Add Major entry: "Business logic code changes with zero test coverage (null baseline AND null after)" → **AC-6**
- [ ] Add Major entry: "Acceptance criterion with test_method='test' lacking corresponding automated test" → **AC-6**

#### 4. review-perspectives.md (FR-10)

- [ ] Add to §4 Pragmatic Verifier priorities: "Self-referential test detection" → **AC-16**
- [ ] Add to §4 Pragmatic Verifier priorities: "Runtime wiring verification" → **AC-16**

#### 5. planner.agent.md (FR-1, FR-8)

- [ ] Update inline Schema 6 definition: `acceptance_criteria` field as structured objects → **AC-1**
- [ ] Update inline Schema 6 example → **AC-1**
- [ ] Add workflow rule: propagate spec AC IDs and test_method to task ACs → **AC-9**
- [ ] Add Self-Verification item: "Every acceptance criterion specifies an observable behavior with a clear pass/fail definition" → **AC-8**
- [ ] Add Self-Verification item: "Spec AC IDs and test_method propagated to task-level criteria" → **AC-9**

#### 6. implementer.agent.md (FR-2, FR-6, FR-9)

- [ ] Add to Step 4 (TDD): structural rule — tests must import production modules → **AC-4**
- [ ] Add to Step 4 (TDD): structural rule — assertions must reference production code output, not local variables → **AC-4**
- [ ] Add to Step 4 (TDD): record initial test run result (tdd_red_green) → **AC-11**
- [ ] Add `behavioral_coverage` and `tdd_red_green` to Output Schema → **AC-2**
- [ ] Add Operating Rules subsection: Test Selection Strategy (4 rules + test_method guide + UI strategy) → **AC-15**
- [ ] Add Self-Verification: "Tests invoke production code, not local variable replications" → **AC-5**
- [ ] Add Self-Verification: "Every test_method='test' AC has a corresponding automated test in behavioral_coverage" → **AC-5**
- [ ] Add Self-Verification: "New source files are imported/referenced by at least one existing source file" → **AC-5**
- [ ] Handle edge cases: inspection-only tasks (EC-1), TDD Fallback (EC-2), modification-only tasks (EC-3) → **EC-1, EC-2, EC-3**

#### 7. verifier.agent.md (FR-3, FR-5, FR-6)

- [ ] Add Tier 2 Step 3e: behavioral-coverage check → **AC-3, AC-14**
- [ ] Add Tier 2 Step 3f: runtime-wiring check (new-file tasks only) → **AC-12, AC-14**
- [ ] Update check_name Quick Reference table with behavioral-coverage (Tier 2) and runtime-wiring (Tier 2) → **AC-14**
- [ ] Add Self-Verification items for behavioral cascade completeness → **AC-3**

#### 8. adversarial-reviewer.agent.md (FR-10)

- [ ] Expand Step 3c Correctness Analysis guidance: self-referential test detection, AC-to-test coverage, runtime wiring → **AC-16**

### Acceptance Criteria Traceability

| AC    | Decision | File(s)                                               | Implementation Path                                     |
| ----- | -------- | ----------------------------------------------------- | ------------------------------------------------------- |
| AC-1  | D-1, D-6 | schemas.md, planner.agent.md                          | Schema 6 field update + planner inline schema + example |
| AC-2  | D-2      | schemas.md, implementer.agent.md                      | Schema 7 field addition + implementer output schema     |
| AC-3  | D-3      | verifier.agent.md                                     | Tier 2 behavioral-coverage check + SQL INSERT           |
| AC-4  | D-2      | implementer.agent.md                                  | Step 4 TDD structural rules                             |
| AC-5  | D-2      | implementer.agent.md                                  | 3 Self-Verification checklist items                     |
| AC-6  | D-4      | severity-taxonomy.md                                  | 3 Major section entries                                 |
| AC-7  | D-5      | schemas.md                                            | DDL section replacement with cross-reference            |
| AC-8  | D-6      | planner.agent.md                                      | Self-Verification checklist item                        |
| AC-9  | D-1, D-6 | planner.agent.md                                      | Workflow rule + Self-Verification item                  |
| AC-10 | D-8      | sql-templates.md                                      | EG-7 query in §6                                        |
| AC-11 | D-2      | implementer.agent.md                                  | tdd_red_green field + TDD Fallback handling             |
| AC-12 | D-3      | verifier.agent.md                                     | Tier 2 runtime-wiring check                             |
| AC-13 | —        | All modified files                                    | Implementer grep verification (no TBD/TODO/FIXME)       |
| AC-14 | D-3      | verifier.agent.md                                     | check_name Quick Reference table update                 |
| AC-15 | D-7      | implementer.agent.md                                  | Test Selection Strategy subsection                      |
| AC-16 | D-2, D-3 | adversarial-reviewer.agent.md, review-perspectives.md | Correctness guidance + pragmatic-verifier priorities    |
| AC-17 | D-3      | schemas.md                                            | check_name Naming Patterns table entries                |
| AC-18 | D-5      | schemas.md                                            | All 10 examples preserved after DDL removal             |
