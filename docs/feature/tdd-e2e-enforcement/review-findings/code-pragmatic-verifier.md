# Adversarial Review: code — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-04T00:00:00Z

## Security Analysis

**Category Verdict:** approve

No security findings. Through the correctness lens, the implementation''s security posture is sound:

- **Structured command format (D-21):** `start_command.executable` + `args` array prevents shell injection. Executor validation against `tier5_command_allowlist` regex is well-defined (tool-access-matrix.md §8.2, 8 patterns).
- **Evidence sanitization pipeline (D-25):** sql_escape, HAR header stripping, path validation, console output cap, env scrubbing — all specified with clear ordering in verifier.agent.md Tier 5 and e2e-integration.md §6.
- **Variation ID validation:** `^[a-zA-Z0-9-]+$` regex + 50-char cap + sql_escape prevents injection via adversarial variation names.
- **Skill content trust model:** Correctly classified as "untrusted" with sanitization requirements documented per trust level table in e2e-integration.md §1.

No logical errors creating security holes detected.

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Per-Phase Timeout Sum Exceeds Hard Cap Without Priority Specification

- **Severity:** Minor
- **Description:** Per-phase maximums in the Tier 5 timeout budget table (verifier.agent.md) sum to 690s (60+300+180+120+30) but the total hard cap is 600s. The e2e-integration.md §8 acknowledges this ("Per-phase maximums sum to 690s > 600s. For tasks with skills in all phases, effective time per phase is reduced by the 600s hard cap") but does not specify a priority order for time allocation when phases compete for the remaining budget. When the hard cap fires mid-phase, the verifier knows to kill + teardown, but the lack of a phase budget reduction formula could lead to non-deterministic behavior (e.g., Exploratory could consume 180s and leave only 10s for Adversarial).
- **Affected artifacts:** [verifier.agent.md](NewAgents/.github/agents/verifier.agent.md#L394-L403) Timeout Budget table, [e2e-integration.md](NewAgents/.github/agents/e2e-integration.md#L645-L660) §8 Per-Phase Limits
- **Recommendation:** Add a note: "Remaining budget = 600s – elapsed. Each phase starts with min(its_max, remaining) as its effective timeout." Or introduce a rolling budget calculation.
- **Evidence:** verifier.agent.md Tier 5 Timeout Budget table. Phases 1-5 max durations: 60+300+180+120+30 = 690 > 600.

### Finding A-2: check_name Cross-File Consistency — EG-10 Names vs. Verifier Output

- **Severity:** Minor (architecture perspective; see C-1 for correctness impact)
- **Description:** The check_names used in EG-10 queries (`baseline-verified`, `integration-tests-passed`) are not documented in the verifier's check_name quick reference table (verifier.agent.md lines 117-139) or any producer documentation. No agent has a documented responsibility to INSERT records with these check_names. This is a producer/consumer contract gap — the gate queries (consumers) reference names that no producer emits.
- **Affected artifacts:** [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L646), [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L661), [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L676) vs. [verifier.agent.md](NewAgents/.github/agents/verifier.agent.md#L117-L139) check_name table
- **Recommendation:** See C-1 for resolution. From an architecture standpoint, add a "Producer/Consumer" comment above each EG query listing which agent INSERT creates the expected records.
- **Evidence:** grep for `baseline-verified` across NewAgents/ returns only sql-templates.md EG-10 queries — zero matches in verifier.agent.md, implementer.agent.md, or e2e-integration.md.

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: EG-10 Queries Reference Non-Existent check_names — Gates Will Never Pass

- **Severity:** Critical
- **Description:** All three EG-10 lane variants query for `check_name IN ('baseline-verified', ...)` but NO agent in the pipeline ever INSERTs a record with `check_name='baseline-verified'`. The verifier produces `baseline-discrepancy` (check_name table, verifier.agent.md line 139). Similarly, EG-10(unit-integration) and EG-10(full-tdd-e2e) query for `integration-tests-passed` — a check_name that no agent produces. The verifier has `tests` (Tier 2, line 120), `import-check` (Tier 3, line 123), and `smoke-execution` (Tier 3, line 124), but nothing called `integration-tests-passed`. Consequence: EG-10(unit-only) expects COUNT≥2 but can match at most 1 (`tdd-compliance`). EG-10(unit-integration) expects COUNT≥3 but can match at most 1. EG-10(full-tdd-e2e) expects COUNT≥4 but can match at most 2 (`tdd-compliance` + `e2e-test-execution`). **All three gate variants are unsatisfiable.** No task can ever pass the EG-10 verification gate, which means Step 6 will fail for every task in every pipeline run.
- **Affected artifacts:** [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L639-L678) (EG-10 all 3 variants), [verifier.agent.md](NewAgents/.github/agents/verifier.agent.md#L117-L139) (check_name table)
- **Recommendation:** Replace phantom check_names with actual verifier-produced names: `baseline-verified` → `baseline-discrepancy` (invert logic: it should be absence of `passed=0` OR mandate a positive `baseline-verified` INSERT from verifier). `integration-tests-passed` → possibly aggregate of `tests` + `import-check` or define a new explicit check_name in the verifier. Alternatively, redefine EG-10 to use a count of ANY passed checks rather than specific named checks.
- **Evidence:** `grep_search` for `baseline-verified` in `NewAgents/.github/agents/**` returns 3 matches, ALL in sql-templates.md EG-10 queries. Zero matches in verifier.agent.md or any other agent file. `grep_search` for `integration-tests-passed` returns 2 matches, both in sql-templates.md.

### Finding C-2: 9 of 10 Code-Type Implementation Reports Missing Required Schema 7 Fields

- **Severity:** Major
- **Description:** Schema 7 (implementation-report) was extended with `verify_phase` (object, "Required for task_type='code'") and `tdd_fallback_reason` (string|null). Only TASK-005 includes both fields. The remaining 9 code-type tasks (TASK-002, 003, 006, 007, 009, 010, 012, 013 + TASK-004 is documentation so exempt) are missing `verify_phase` entirely. TASK-007 uses a non-schema field `note:` under `tdd_red_green` instead of `tdd_fallback_reason`. TASK-002, 003, 006, 009, 010, 012, 013 have no fallback reason at all despite reporting `initial_run_failures: 0`. This is a systematic schema compliance failure — the very schema this feature defines is not enforced in its own implementation reports.
- **Affected artifacts:** Implementation reports for TASK-002, 003, 006, 007, 009, 010, 012, 013 in `docs/feature/tdd-e2e-enforcement/implementation-reports/`
- **Recommendation:** Each code-type implementation report should include either: (a) `verify_phase` with all three boolean fields (when TDD was performed), or (b) `tdd_fallback_reason` (when TDD was skipped for non-runtime files). TASK-007''s `note:` should be renamed to `tdd_fallback_reason`.
- **Evidence:** `grep_search` for `verify_phase:|tdd_fallback_reason:` across `implementation-reports/**` returns only 2 matches, both in TASK-005.yaml.

### Finding C-3: e2e-integration.md Exceeds Line Count Target by 147 Lines

- **Severity:** Major
- **Description:** The e2e-integration.md file is 697 lines. Design-output.yaml estimated 500 lines with a tolerance of ±50 (450-550). The file exceeds the upper bound by 147 lines. The verifier independently flagged this: `line-count-range` check returned `passed=0` for TASK-004 (implementer self-reported 546 lines, actual 697 lines — a 151-line discrepancy from self-report). This matters because the design specifically extracted content into this reference document to keep the orchestrator under its 550-line cap. If the reference document itself is oversized, it creates context pressure for agents instructed to read it.
- **Affected artifacts:** [e2e-integration.md](NewAgents/.github/agents/e2e-integration.md) (697 lines vs 450-550 target)
- **Recommendation:** Trim §8 (Timeout Budget, Parallel Instance Isolation, API/CLI App Testing subsections, and Realistic Budget Example account for most of the excess). Consider moving the API/CLI App Testing and Parallel Instance Isolation content to dispatch-patterns.md or a separate supplementary document.
- **Evidence:** `wc -l` equivalent: e2e-integration.md = 697 lines. Design-output.yaml `estimated_lines: 500`. Verifier TASK-004 `line-count-range` check: `passed=0`, `severity=Major`, `output_snippet: "Actual: 697 lines. Target: 500 +/- 50 (450-550). Delta: +147 lines."`.

### Finding C-4: TDD Fallback Conditions Create Unreachable State for Code-Classified Markdown Tasks

- **Severity:** Minor
- **Description:** The tightened TDD Fallback (implementer.agent.md) states TDD is skippable "ONLY when ALL of: (a) task_type is NOT 'code', AND (b) no production source files modified, AND (c) only docs/config/non-runtime files change." Multiple tasks in this feature are classified `task_type: 'code'` but modify only `.agent.md` markdown files (non-runtime). Per the strict reading, condition (a) fails so TDD is mandatory — but there are no executable tests possible for markdown-only changes. The implementers correctly applied the fallback anyway, but the rule as written creates a logical contradiction. TASK-005 handled this gracefully with a detailed `tdd_fallback_reason`; other tasks simply left `initial_run_failures: 0` with no explanation.
- **Affected artifacts:** [implementer.agent.md](NewAgents/.github/agents/implementer.agent.md#L330-L357) TDD Fallback section
- **Recommendation:** Add an explicit clause: "Condition (a) may also be satisfied when task_type='code' but the task modifies ONLY non-runtime files (e.g., .md, .yaml instruction files with no executable code)." This clarifies intent without weakening the TDD mandate for actual code.
- **Evidence:** TASK-005 (task_type=code, modifies only implementer.agent.md markdown) includes tdd_fallback_reason. TASK-002, 003, 006, 007, 009, 010, 012, 013 are all task_type=code with only markdown changes but no tdd_fallback_reason.

### Finding C-5: TASK-004 Implementer Self-Reported Line Count Discrepancy

- **Severity:** Minor
- **Description:** TASK-004 implementer reported e2e-integration.md as 546 lines, but the actual file is 697 lines (151-line discrepancy). The verifier caught this and recorded `baseline-discrepancy` with `passed=0`. While the verifier correctly identified the issue, the discrepancy suggests the implementer did not run a final line count check before reporting. This is a data integrity issue in the implementation report.
- **Affected artifacts:** TASK-004 implementation report (`docs/feature/tdd-e2e-enforcement/implementation-reports/TASK-004.yaml`)
- **Recommendation:** No code change needed — the verifier's cross-check mechanism worked as designed. The finding is logged for adversarial review completeness.
- **Evidence:** Verifier TASK-004 `baseline-discrepancy` check: `passed=0`, output: "File confirmed NEW. However, implementer claimed 546 lines but actual file is 697 lines — 151 lines over self-reported count."

## Summary

The implementation is broadly well-structured, with comprehensive E2E lifecycle design and good security sanitization. However, the EG-10 evidence gate queries reference check_names (`baseline-verified`, `integration-tests-passed`) that no agent ever produces, making all three lane variants unsatisfiable — this is a Critical correctness defect that would prevent any task from passing verification in production. Additionally, 9 of 10 code-type implementation reports violate the Schema 7 extensions that this very feature defines (missing `verify_phase` and `tdd_fallback_reason`). The e2e-integration.md exceeds its design-specified line budget by 147 lines. Recommend: fix EG-10 check_names (Critical), add missing schema fields to reports (Major), and trim e2e-integration.md (Major).
