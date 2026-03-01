# Behavioral Correctness Enforcement

**Feature Slug:** `behavioral-correctness-enforcement`
**Status:** Specified
**Date:** 2026-03-01

## Short Summary

Structural safeguards to prevent agents from declaring success based on passing tests that do not actually validate real, observable system behavior. Addresses the systemic gap where agents optimize for green tests rather than behavioral correctness.

---

## Background & Context

### Problem Statement

The agent pipeline in `NewAgents/.github/agents/` has a systemic correctness problem where all defense layers (implementer TDD, verifier 4-tier cascade, adversarial review, evidence gates) operate at the pass/fail signal level without evaluating whether tests validate meaningful behavior. Concrete evidence from the `ui-bug-fixes` pipeline run shows:

- Self-referential tests that assert on local variables rather than production code (patterns F-2)
- Business logic changes with zero test coverage passing verification (patterns F-12)
- Self-referential tests classified as Minor severity, allowing pipeline approval without remediation (patterns F-8)
- Acceptance criteria degrading from structured objects (id + text + test_method) to plain strings, losing traceability (dependencies F-11)

### Root Cause

The same agent (implementer) writes both tests and production code with no structural verification that tests actually invoke production code. Downstream agents (verifier, reviewer) check whether tests pass, not whether they test anything meaningful.

### Research Inputs

| Focus        | File                         | Findings                                                                      |
| ------------ | ---------------------------- | ----------------------------------------------------------------------------- |
| Architecture | `research/architecture.yaml` | 10 — structural gaps in TDD enforcement, verification cascade, evidence gates |
| Impact       | `research/impact.yaml`       | 12 — blast radius across 6 agent files, schema size analysis                  |
| Dependencies | `research/dependencies.yaml` | 12 — cross-reference map, evidence chain gaps, AC traceability loss           |
| Patterns     | `research/patterns.yaml`     | 13 — concrete evidence from ui-bug-fixes, severity misclassification          |

### Key Research Findings (Top 10)

1. **Same agent writes tests AND code** — structural conflict of interest with no external test quality validation (arch F-7)
2. **Verifier validates pass/fail counts, not test quality** — 4-tier cascade is structurally blind to self-referential tests (arch F-2)
3. **Evidence gates are quantity-driven, not quality-driven** — pipeline routing satisfied by trivial signals (arch F-6)
4. **Acceptance criteria narrow from spec → planner with no audit** — structured AC objects become plain strings, losing test_method (arch F-3)
5. **No pipeline step performs behavioral delta verification** — no mutation testing, coverage gating, or independent test generation (arch F-5)
6. **~286 lines of SQLite DDL duplicated** between schemas.md and sql-templates.md (impact F-7)
7. **Spec test_method metadata defined but never consumed downstream** — verification intent lost at spec→planner boundary (arch F-8)
8. **Self-verification checklists are form-over-substance** — check fields exist, not quality (patterns F-7)
9. **No runtime wiring verification** — code can be tested in isolation but never connected to runtime (patterns F-9)
10. **Severity taxonomy caused self-referential tests to be classified Minor** — review system detected problem but severity prevented remediation (patterns F-8)

---

## Architectural Directions Evaluated

### Direction A: Structural Enforcement Within Existing Architecture (Recommended)

Keep current agent roles unchanged. Add behavioral verification checks at implementer (production-code invocation check), verifier (behavioral-coverage check_name), and reviewer (severity recalibration) stages. Enhance AC traceability through additive schema fields. Reduce schemas.md bloat. **Lowest blast radius, highest backward compatibility.**

### Direction B: Independent Test Validation Agent

Introduce a new agent role that independently generates or validates tests from acceptance criteria. Addresses the root conflict of interest but requires new agent definition, schema, pipeline step, dispatch pattern, and evidence gate queries. **Highest correctness guarantee but largest blast radius.**

### Direction C: Verification-Centric Enhancement

Focus changes primarily on the verifier agent with new cascade checks. Minimal schema changes but concentrates behavioral enforcement in a single agent. **Moderate blast radius but single point of enforcement.**

---

## Functional Requirements

### FR-1: Acceptance Criteria Traceability

Structured AC flow from spec through task to evidence.

| ID     | Requirement                                                                                                                                         | Priority |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-1.1 | Task schema (Schema 6) must carry structured AC objects with parent spec AC ID, text, and test_method (replacing plain-string list)                 | Must     |
| FR-1.2 | Implementation report (Schema 7) must include behavioral_coverage field mapping each AC ID to test file:line or "not_applicable" with justification | Must     |
| FR-1.3 | Verification report (Schema 8) must include behavioral-coverage check_name finding for code tasks with test_method='test' criteria                  | Must     |
| FR-1.4 | Spec test_method must propagate through planner to task schema and be consumed by implementer for verification approach selection                   | Must     |

### FR-2: Implementer Behavioral Test Enforcement

Structural safeguards against self-referential and hollow tests.

| ID     | Requirement                                                                                                                                 | Priority |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-2.1 | Every test file for code tasks must import at least one production module modified/created by the same task                                 | Must     |
| FR-2.2 | Tests must not assert on local variables that replicate production logic — assertions must reference values from production code invocation | Must     |
| FR-2.3 | Implementer self-verification must include behavioral coverage item for test_method='test' criteria                                         | Must     |
| FR-2.4 | Code tasks with null baseline AND null after test_summary for business logic must trigger self-fix attempt or TDD Fallback justification    | Must     |
| FR-2.5 | TDD red-green cycle must be structurally recorded: test run result before production code must show failures                                | Must     |

### FR-3: Verifier Behavioral Verification

New verification checks for test quality and production wiring.

| ID     | Requirement                                                                                                                        | Priority |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-3.1 | Verifier cascade includes behavioral-coverage check verifying test imports, AC mapping completeness, and file existence            | Must     |
| FR-3.2 | Verifier cross-references behavioral_coverage against task acceptance criteria — missing test_method='test' entries fail the check | Must     |
| FR-3.3 | Runtime-wiring check for tasks creating new files — at least one pre-existing file must reference the new file                     | Must     |
| FR-3.4 | Behavioral-coverage check counts within existing EG-2 minimum signal threshold                                                     | Should   |

### FR-4: Severity Taxonomy Recalibration

Ensure correctness problems trigger remediation.

| ID     | Requirement                                                                     | Priority |
| ------ | ------------------------------------------------------------------------------- | -------- |
| FR-4.1 | Self-referential tests (tests not invoking production code) classified as Major | Must     |
| FR-4.2 | Business logic code changes with zero test coverage classified as Major         | Must     |
| FR-4.3 | Test_method='test' AC with no corresponding automated test classified as Major  | Must     |

### FR-5: Evidence Gate Enhancement

Behavioral evidence integrated into pipeline routing.

| ID     | Requirement                                                                                                    | Priority |
| ------ | -------------------------------------------------------------------------------------------------------------- | -------- |
| FR-5.1 | EG-7 query in sql-templates.md checks for behavioral-coverage evidence per code task (informational initially) | Must     |
| FR-5.2 | Verifier check_name Quick Reference updated with behavioral-coverage and runtime-wiring                        | Must     |
| FR-5.3 | schemas.md check_name Naming Patterns table updated with new check_name values                                 | Must     |

### FR-6: Runtime Wiring Verification

Prevent isolated abstractions.

| ID     | Requirement                                                                       | Priority |
| ------ | --------------------------------------------------------------------------------- | -------- |
| FR-6.1 | Implementer self-verification includes new-file reference check                   | Must     |
| FR-6.2 | Verifier runtime-wiring check uses workspace search to verify new file references | Must     |

### FR-7: Schema Size Reduction

Eliminate duplication in schemas.md.

| ID     | Requirement                                                                                   | Priority |
| ------ | --------------------------------------------------------------------------------------------- | -------- |
| FR-7.1 | SQLite DDL section (~286 lines) replaced with concise cross-references to sql-templates.md §1 | Must     |
| FR-7.2 | check_name Naming Patterns and phase semantics preserved (unique to schemas.md)               | Must     |
| FR-7.3 | schemas.md stays under 1300 lines after all changes                                           | Should   |

### FR-8: Planner Acceptance Criteria Quality

Behavioral testability at the planning stage.

| ID     | Requirement                                                              | Priority |
| ------ | ------------------------------------------------------------------------ | -------- |
| FR-8.1 | Planner self-verification includes observable behavior check for AC text | Must     |
| FR-8.2 | Planner propagates spec AC IDs into task-level criteria                  | Must     |
| FR-8.3 | Planner propagates test_method from spec AC to task criteria             | Must     |

### FR-9: Principled Test Selection Strategy

Codified rules for what to test and how.

| ID     | Requirement                                                                                                                                                 | Priority |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-9.1 | Implementer Operating Rules include test selection strategy (behavior-not-implementation, no type-system tests, public interface only, avoid brittle tests) | Must     |
| FR-9.2 | Implementer includes test_method-based selection guide (test→automated, inspection→review, demonstration→runtime evidence, analysis→metrics)                | Must     |
| FR-9.3 | UI behavior validation strategy: Playwright only for cross-component/JS-interop/gesture flows; unit tests for logic/state; inspection for CSS               | Must     |

### FR-10: Reviewer Correctness Enhancement

Strengthen behavioral detection within existing 3-category structure.

| ID      | Requirement                                                                                                 | Priority |
| ------- | ----------------------------------------------------------------------------------------------------------- | -------- |
| FR-10.1 | Correctness analysis guidance includes self-referential test detection, AC-to-test coverage, runtime wiring | Must     |
| FR-10.2 | Pragmatic-verifier perspective priorities include self-referential test detection and runtime wiring        | Must     |

---

## Non-functional Requirements

| ID    | Requirement                                                                                                                                                           | Priority |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| NFR-1 | Per-task execution time increase from behavioral checks must not exceed 50% over baseline (keep checks as static file analysis, not runtime execution)                | Should   |
| NFR-2 | No agent definition file may exceed 400 lines after changes (current max: implementer at 334, orchestrator at 492 — orchestrator is exempt due to routing complexity) | Should   |
| NFR-3 | All changes must be parseable by existing YAML parsers without version bumps to consumer agents                                                                       | Must     |
| NFR-4 | Behavioral checks must work across all target languages/frameworks supported by the pipeline (C#, TypeScript, Python at minimum)                                      | Should   |
| NFR-5 | schemas.md must remain under 1300 lines total to maintain context window headroom for agents that reference it                                                        | Should   |

---

## Constraints & Assumptions

### Constraints

- Windows environment, VS Code GitHub Copilot agent framework
- SQLite verification-ledger.db with WAL mode + busy_timeout=5000 for 5 concurrent writer agents
- All SQL must use stdin piping and sql_escape() per SEC-2
- run_id namespace filter required on all SQL queries
- Adversarial reviewer limited to exactly 3 review categories
- Bounded feedback loops: design max 1, impl-verify max 3, code review max 2
- Planner task size: max 3 files, ~500 lines per task
- Schema changes must be additive per Schema Evolution Strategy

### Assumptions

- Direction A (structural enforcement within existing architecture) is the selected approach unless the designer identifies blocking constraints
- Behavioral checks are static analysis (file content inspection), not runtime execution or mutation testing
- The existing 4-tier verifier cascade structure is preserved; new checks are added as additional checks within existing tiers or as a new sub-tier
- EG-7 (behavioral evidence) starts as informational/non-blocking and may be promoted to required in a future change

---

## Acceptance Criteria

| ID    | Criterion                                                                        | Test Method | Pass/Fail Definition                                                                                                        |
| ----- | -------------------------------------------------------------------------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------- |
| AC-1  | Task schema carries structured AC objects with spec AC ID, text, and test_method | Inspection  | Pass: Schema 6 definition and planner inline schema both show structured object. A planner output has all three sub-fields. |
| AC-2  | Implementation report has behavioral_coverage mapping AC IDs to tests            | Inspection  | Pass: Schema 7 definition shows field. Sample report maps every test_method='test' AC to a test file.                       |
| AC-3  | Verification report has behavioral-coverage check_name for code tasks            | Test        | Pass: Verifier output for a code task contains check_name='behavioral-coverage' with correct passed value.                  |
| AC-4  | Test files import production modules from the same task                          | Test        | Pass: grep of test files shows import/require/using for production modules. Zero production imports = fail.                 |
| AC-5  | Implementer checklist has 3 behavioral items                                     | Inspection  | Pass: All three items present in Self-Verification section of implementer.agent.md.                                         |
| AC-6  | Severity taxonomy classifies self-referential tests as Major                     | Inspection  | Pass: Explicit named entry exists in severity-taxonomy.md.                                                                  |
| AC-7  | schemas.md SQLite DDL section replaced with ≤30-line cross-reference             | Test        | Pass: Line count reduced by ≥200 from 1422 baseline.                                                                        |
| AC-8  | Planner checklist has observable behavior check                                  | Inspection  | Pass: Item present in planner.agent.md Self-Verification.                                                                   |
| AC-9  | test_method propagated from spec AC to task AC                                   | Inspection  | Pass: Tracing an AC from spec-output.yaml → tasks/\*.yaml shows matching test_method.                                       |
| AC-10 | EG-7 query exists in sql-templates.md §6                                         | Inspection  | Pass: Valid SQL query present checking behavioral-coverage evidence per code task.                                          |
| AC-11 | Null-test code tasks don't silently pass                                         | Test        | Pass: Code task with null baseline + null after test_summary either has tests or has tdd_fallback_reason.                   |
| AC-12 | Runtime-wiring check exists for new-file tasks                                   | Test        | Pass: New file with zero references from existing files shows check_name='runtime-wiring' with passed=0.                    |
| AC-13 | No TBD/TODO/FIXME in modified files                                              | Test        | Pass: grep across modified files returns 0 matches.                                                                         |
| AC-14 | Verifier Quick Reference has new check_names                                     | Inspection  | Pass: Both 'behavioral-coverage' and 'runtime-wiring' entries present with tier and description.                            |
| AC-15 | Implementer has codified test selection strategy                                 | Inspection  | Pass: Four sub-items present in Operating Rules.                                                                            |
| AC-16 | Reviewer correctness guidance includes behavioral items                          | Inspection  | Pass: All three items (self-referential detection, AC coverage, wiring) present.                                            |
| AC-17 | schemas.md Naming Patterns table has new check_names                             | Inspection  | Pass: Both entries present with convention and context.                                                                     |
| AC-18 | Schema examples preserved after changes                                          | Inspection  | Pass: 10 example blocks remain in schemas.md.                                                                               |

---

## Edge Cases & Error Handling

| ID   | Input/Condition                                                             | Expected Behavior                                                                                 | Severity if Missed                                  |
| ---- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| EC-1 | CSS/visual-only task, all ACs have test_method='inspection'                 | behavioral-coverage check auto-passes with "not_applicable" mapping                               | High — false failures block legitimate visual tasks |
| EC-2 | TDD Fallback: no test framework, task creates framework itself              | behavioral-coverage check accepts TDD Fallback justification as valid                             | High — blocks bootstrapping tasks                   |
| EC-3 | Task modifies existing files only (no new files created)                    | runtime-wiring check is not run or auto-passes                                                    | Medium — false failures on modification tasks       |
| EC-4 | Pipeline resume: tasks verified before behavioral enforcement enabled       | EG-7 skips tasks with pre-existing verification records                                           | Critical — blocks pipeline resume, data loss risk   |
| EC-5 | Planner scopes out an AC from a task (covered by different task)            | behavioral_coverage maps only this task's assigned ACs, not all spec ACs                          | High — false failures on partial-coverage tasks     |
| EC-6 | Language with difficult import analysis (dynamic imports, XAML code-behind) | Production-import check supports language-specific patterns or manual override with justification | Medium — false failures in specific tech stacks     |
| EC-7 | Agent context window pressure from added instructions                       | Per-agent instruction increase capped at ~50 lines                                                | Medium — degraded agent performance if exceeded     |
| EC-8 | Mixed test_methods: some ACs are 'test', some are 'inspection'              | behavioral-coverage checks only 'test' criteria, maps 'inspection' as not_applicable              | High — false failures on mixed tasks                |

---

## User Stories / Flows

### Story 1: Bug Fix with Behavioral Verification

1. Spec defines AC: "Button click navigates to settings page" with test_method='test'
2. Planner creates task with structured AC carrying ID, text, and test_method='test'
3. Implementer writes test that imports the production NavigationService and asserts navigation occurred
4. Implementer's self-verification confirms: test imports production module ✓, assertion references production output ✓
5. Implementer records behavioral_coverage: AC-ID → test file:line in implementation report
6. Verifier runs behavioral-coverage check: AC mapping present ✓, test file exists ✓, test imports production module ✓ → passed=1
7. Adversarial reviewer's correctness analysis confirms tests invoke production code ✓

### Story 2: Detecting Self-Referential Test (Caught Early)

1. Implementer writes test for "cancel action does not trigger callback" — test declares local `bool cancelWasCalled = false` and asserts `Assert.False(cancelWasCalled)` without calling production code
2. Implementer self-verification check "Tests invoke production code, not local variable replications" → FAILS
3. Implementer's self-fix loop rewrites test to call production `OnCancel()` method and assert on production state
4. Corrected test passes self-verification → proceeds to verifier
5. Problem caught at implementer stage, not at adversarial review stage

### Story 3: New Module Wiring Check

1. Task creates a new `ThemeService.cs` file with unit tests
2. Implementer self-verification checks "New source files referenced by existing code" → finds no existing file imports ThemeService → FAILS
3. Implementer adds `using ThemeService` to existing `AppShell.xaml.cs`
4. Self-verification passes → proceeds to verifier
5. Verifier runs runtime-wiring check: `ThemeService.cs` referenced by `AppShell.xaml.cs` → passed=1

### Story 4: Visual-Only Task (No Automated Tests)

1. Task has all ACs with test_method='inspection' (CSS color change)
2. Implementer maps all ACs as "not_applicable: visual-only per test_method=inspection"
3. behavioral_coverage field is populated with not_applicable entries
4. Verifier behavioral-coverage check sees all ACs mapped as not_applicable → auto-passes
5. No false failure for legitimate visual tasks

---

## Test Scenarios

| ID    | Scenario                                                                                    | Linked ACs  | Expected Result                                                  |
| ----- | ------------------------------------------------------------------------------------------- | ----------- | ---------------------------------------------------------------- |
| TS-1  | Code task with test_method='test' ACs, implementer produces tests importing production code | AC-3, AC-4  | behavioral-coverage check passed=1                               |
| TS-2  | Code task where test file has zero imports of production modules                            | AC-4, AC-6  | behavioral-coverage check passed=0, classified Major by reviewer |
| TS-3  | Code task with behavioral_coverage field missing entirely                                   | AC-2, AC-3  | behavioral-coverage check passed=0                               |
| TS-4  | Visual-only task, all ACs test_method='inspection'                                          | AC-1, AC-3  | behavioral-coverage auto-passes, no tests required               |
| TS-5  | New file task, new file referenced by existing code                                         | AC-12       | runtime-wiring check passed=1                                    |
| TS-6  | New file task, new file NOT referenced by existing code                                     | AC-12       | runtime-wiring check passed=0                                    |
| TS-7  | Code task with null baseline + null after test_summary                                      | AC-11       | Self-check fails or tdd_fallback_reason is present               |
| TS-8  | schemas.md after DDL section removal                                                        | AC-7        | Line count ≤ 1222 (1422 - 200)                                   |
| TS-9  | Task schema with structured AC objects                                                      | AC-1, AC-9  | AC has id, text, and test_method fields                          |
| TS-10 | EG-7 query executed against anvil_checks with behavioral-coverage records                   | AC-10       | Query returns count ≥ 1 for code tasks                           |
| TS-11 | Mixed test_method task (some 'test', some 'inspection')                                     | AC-3, AC-8  | Only 'test' criteria checked for automated tests                 |
| TS-12 | TDD Fallback task with justification                                                        | AC-11       | tdd_fallback_reason populated, self-check passes                 |
| TS-13 | Modified files contain no TBD/TODO/FIXME                                                    | AC-13       | grep returns 0 matches                                           |
| TS-14 | Implementer self-verification has all behavioral items                                      | AC-5, AC-15 | All items present in checklist                                   |
| TS-15 | Adversarial reviewer finds self-referential test                                            | AC-6, AC-16 | Finding classified as Major, triggers needs_revision             |

---

## Dependencies & Risks

### Component Dependencies

| Component                       | Change Type                                                            | Risk                                     |
| ------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------- |
| `implementer.agent.md`          | Modified — self-verification, TDD enforcement, test selection strategy | Medium — heaviest instruction changes    |
| `verifier.agent.md`             | Modified — new check_names, cascade enhancement                        | Medium — new verification logic          |
| `schemas.md`                    | Modified — DDL removal, new schema fields, check_name table updates    | High — 8 of 9 agents reference this file |
| `sql-templates.md`              | Modified — EG-7 query, new INSERT templates                            | Medium — 5 agents use SQL templates      |
| `planner.agent.md`              | Modified — AC quality check, structured AC propagation                 | Low — additive self-verification item    |
| `adversarial-reviewer.agent.md` | Modified — correctness guidance expansion                              | Low — guidance text only                 |
| `review-perspectives.md`        | Modified — pragmatic-verifier priority additions                       | Low — priority list additions            |
| `severity-taxonomy.md`          | Modified — new Major severity entries                                  | Low — additive entries                   |
| `global-operating-rules.md`     | Unchanged                                                              | N/A — no changes needed                  |
| `tool-access-matrix.md`         | Unchanged                                                              | N/A — no new tools required              |

### Risks

| Risk                                                                   | Likelihood | Impact   | Mitigation                                                                   |
| ---------------------------------------------------------------------- | ---------- | -------- | ---------------------------------------------------------------------------- |
| Schema drift between schemas.md and agent inline schemas after changes | Medium     | High     | Update schemas.md first, then each agent file. Include version notes.        |
| Behavioral checks cause false failures on legitimate no-test tasks     | Medium     | High     | EC-1, EC-2, EC-3 edge case handling with not_applicable mapping              |
| Context window pressure from expanded instructions                     | Low        | Medium   | Cap per-agent increase at ~50 lines; reduce schemas.md first (FR-7)          |
| EG-7 blocks pipeline resume for pre-existing tasks                     | Low        | Critical | EC-4 handling: EG-7 starts informational, not blocking                       |
| Languages with difficult import analysis cause false failures          | Medium     | Medium   | EC-6 handling: language-specific patterns and manual override                |
| Self-referential test detection is subjective in edge cases            | Medium     | Medium   | Focus on structural checks (imports production module) not semantic analysis |

---

## Pushback Log

| #   | Severity | Concern                                                                               | Recommendation                                                                         | Disposition    |
| --- | -------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | -------------- |
| 1   | Major    | Implementation complexity increase (~30-50% per task from behavioral checks)          | Keep checks lightweight (static file analysis, not runtime). Accept as necessary cost. | Proceed (auto) |
| 2   | Major    | Schema change cascade requires synchronized updates across schemas.md + 3 agent files | Make all changes additive. Update schemas.md first as central registry.                | Proceed (auto) |
| 3   | Minor    | schemas.md at 1422 lines approaching context window limits before additions           | Remove ~286 lines of duplicated SQLite DDL first (CR-4).                               | Proceed (auto) |

---

## Requirement-to-Criteria Traceability

| Functional Requirement         | Acceptance Criteria |
| ------------------------------ | ------------------- |
| FR-1 (AC Traceability)         | AC-1, AC-2, AC-9    |
| FR-2 (Implementer Enforcement) | AC-4, AC-5, AC-11   |
| FR-3 (Verifier Behavioral)     | AC-3, AC-12, AC-14  |
| FR-4 (Severity Recalibration)  | AC-6                |
| FR-5 (Evidence Gates)          | AC-10, AC-17        |
| FR-6 (Runtime Wiring)          | AC-12               |
| FR-7 (Schema Reduction)        | AC-7, AC-18         |
| FR-8 (Planner Quality)         | AC-1, AC-8, AC-9    |
| FR-9 (Test Selection)          | AC-15               |
| FR-10 (Reviewer Enhancement)   | AC-16               |
