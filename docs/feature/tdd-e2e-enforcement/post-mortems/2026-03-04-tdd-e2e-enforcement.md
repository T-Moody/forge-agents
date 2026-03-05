# Post-Mortem — TDD & E2E Testing Enforcement

**Run ID:** `2026-03-04T00:00:00Z`
**Feature:** `tdd-e2e-enforcement`
**Date:** 2026-03-04
**Overall Risk:** 🔴
**Pipeline Outcome:** ✅ Success

---

## Executive Summary

Pipeline completed successfully: 16/16 tasks verified, all reviews converged to approval in Round 2, zero errors, zero retries. The feature touched 15 files (14 modified + 1 new) across the entire agent system. Two critical security findings and one critical architectural gap were caught and resolved during review cycles.

---

## What Worked

1. **Wave decomposition** — 4-wave structure with proper dependency ordering meant all 16 tasks could proceed without blocking on unresolved dependencies. Each wave's verification completed cleanly before the next wave's implementation.

2. **Design review caught a critical gap early** — architecture-guardian identified that the original design did not specify how Playwright CLI would perform step-by-step interaction. This was resolved in design revision R2 before any implementation began, preventing a costly rework cycle.

3. **Code review caught 3 Critical + 9 Major findings** — EG-10 phantom check_names, overbroad allowlist regexes, and concurrency contradictions were all caught in Round 1. The implementer fix batch resolved all Critical issues in a single round.

4. **Zero retries across 51 dispatches** — No agent failed or required retry. All 51 pipeline steps completed on first attempt.

5. **Verification cascade was thorough** — 142 total checks across 16 tasks with per-task acceptance criteria matching. Only 2 cosmetic failures (both on TASK-004 line count).

6. **Evaluation scores solid** — Mean usefulness 7.87/10, mean clarity 8.35/10 across 23 evaluations. No artifact scored below 6.

---

## What Failed / Needed Correction

1. **EG-10 phantom check_names (Critical)** — sql-templates.md referenced `baseline-verified` and `integration-tests-passed` as check_names in evidence gate queries, but no agent was designed to produce these names. This made all 3 lane variants of EG-10 unsatisfiable. Root cause: design-output.yaml did not include a producer/consumer mapping table for check_names.

2. **Overbroad command allowlist (Critical)** — `^node .+` allowed arbitrary code execution, `^curl -s .+` allowed arbitrary HTTP exfiltration. Root cause: allowlist patterns were written for convenience rather than security-first. The tightened patterns restrict to specific safe subcommands.

3. **E2E concurrency cap contradiction (Major)** — Orchestrator specified 2-4 concurrent E2E tasks, dispatch-patterns specified 1, e2e-integration.md specified 1. Root cause: no single source of truth was designated during design. Resolved by aligning to cap=1 everywhere.

4. **Implementer line count self-reporting (Minor)** — TASK-004 reported 546 lines for e2e-integration.md; actual was 697 lines (28% discrepancy). Root cause: implementer may count sections written rather than total file lines, or used an inaccurate measurement method.

---

## Root Causes

| Finding                   | Root Cause                                                       | Prevention                                                           |
| ------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------- |
| EG-10 phantom check_names | No producer/consumer cross-reference in design                   | Add check_name producer table to design template                     |
| Overbroad allowlist       | Security patterns written for functionality, not least-privilege | Security-sentinel review of all regex patterns before implementation |
| Concurrency contradiction | Multiple documents authoring the same rule                       | Designate single authoritative source for each operational parameter |
| Line count discrepancy    | Implementer self-report methodology                              | Verifier always independently measures (already in place)            |

---

## Top 3 Bottleneck Steps (AC-6.4)

| Rank | Step                        | Duration | Agent    | Why Slow                                                        |
| ---- | --------------------------- | -------- | -------- | --------------------------------------------------------------- |
| 1    | step-3 (design)             | 420s     | designer | 18 decisions, 14 files, 26 revision items from 3 reviewers      |
| 2    | step-3-r2 (design revision) | 420s     | designer | Addressed 3C + 18M findings, added 8 new decisions              |
| 3    | step-4 (planning)           | 420s     | planner  | 16 tasks with dependency analysis, 4 waves, risk classification |

**Observation:** The design phase (step-3 + step-3-r2) consumed ~14 minutes combined, which is the largest single-step duration. This is expected for a 🔴 risk feature touching 15 files.

---

## Agent Reliability Metrics

| Agent                | Dispatches | Success | NEEDS_REVISION | ERROR | Retries |
| -------------------- | ---------- | ------- | -------------- | ----- | ------- |
| researcher           | 4          | 4       | 0              | 0     | 0       |
| spec                 | 2          | 2       | 0              | 0     | 0       |
| designer             | 2          | 2       | 0              | 0     | 0       |
| planner              | 2          | 2       | 0              | 0     | 0       |
| implementer          | 17         | 17      | 0              | 0     | 0       |
| verifier             | 16         | 15      | 1              | 0     | 0       |
| adversarial-reviewer | 9          | 9       | 0              | 0     | 0       |
| **TOTAL**            | **51**     | **51**  | **1**          | **0** | **0**   |

**Notes:**

- Verifier NEEDS_REVISION on TASK-004 was for cosmetic line count overage — the task was accepted as complete by the pipeline.
- Spec agent dispatched twice (user clarification revision).
- Designer dispatched twice (design review revision).
- Planner dispatched twice (Playwright CLI integration revision).

---

## Evaluation Quality Analysis

- **23 evaluations** across design artifacts, implementation reports, and codebase reviews
- **Mean usefulness:** 7.87/10 — consistently useful artifacts with minor gaps
- **Mean clarity:** 8.35/10 — well-structured with minor formatting inconsistencies
- **Lowest score:** TASK-004 implementation report clarity (6/10) due to self-reported line count discrepancy
- **Common gap:** Producer/consumer traceability for SQL check_names — this should be a required design template field

---

## Improvement Suggestions

1. **Add check_name producer/consumer table to design template** — Require the designer to explicitly map every check_name referenced in evidence gate queries to the agent + phase that produces it. This would have prevented the EG-10 Critical finding.

2. **Security-first allowlist review** — All command allowlist regex patterns should undergo security-sentinel review BEFORE implementation, not after. Consider adding a security pre-review gate for allowlist changes.

3. **Single-source-of-truth enforcement** — When a parameter (like concurrency cap) appears in multiple files, the design must designate exactly one file as authoritative and require others to reference it. The designer template should include a "parameter authority" section.

4. **Independent line count measurement** — Verifier already does this, but the implementer should be instructed to use a consistent measurement command (`Get-Content | Measure-Object` or `wc -l`) rather than estimating.

5. **Design-level E2E tool evaluation** — When the design specifies external tools (like Playwright CLI), require the designer or researcher to verify the tool's actual capabilities match the proposed usage model. The architecture-guardian caught that Playwright CLI couldn't do what the design assumed.

---

_Generated by knowledge-agent, pipeline run 2026-03-04T00:00:00Z_
