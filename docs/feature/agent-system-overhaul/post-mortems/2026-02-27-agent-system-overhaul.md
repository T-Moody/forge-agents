# Post-Mortem — Agent System Overhaul

> **Pipeline Run:** 2026-02-27T10:00:00Z
> **Feature:** agent-system-overhaul
> **Outcome:** SUCCESS (High Confidence)
> **Date:** 2026-02-27

---

## Summary

The agent-system-overhaul pipeline completed successfully with high confidence. 18 tasks across 5 waves implemented a comprehensive restructuring of the 9-agent pipeline system: extracting 7 shared reference documents, adding instruction file infrastructure, restructuring all 9 agent definitions to be under 350 lines (orchestrator ≤550), and transitioning from multi-model review to prompt-diversity-based adversarial review.

---

## What Worked Well

### 1. Shared Document Extraction Strategy
Extracting repeated content (SQL templates, tool access, error handling, review perspectives) into 7 shared reference docs was highly effective. The orchestrator alone achieved a 41% line reduction (797→471 lines). Section-number references (e.g., "sql-templates.md §6") provide precise cross-linking without duplication.

### 2. Wave-Based Execution
The 5-wave plan correctly sequenced dependencies: foundation docs (Wave 1) → templates and schemas (Wave 2) → simple agent mods (Wave 3) → core agent mods (Wave 4) → final agents (Wave 5). No dependency violations occurred and all waves completed without blocking.

### 3. Comprehensive Verification
52 verification checks across 18 tasks with 0 failures and 0 regressions. The verifier effectively used baseline cross-checks (`git show pipeline-baseline-*`) to validate implementer claims. Tier 4 checks on high-risk tasks (TASK-010, TASK-012) provided additional operational readiness confidence.

### 4. Effective Review-Fix Cycle
R1 code review identified 10 actionable findings concentrated in `schemas.md`. The implementer fixed all of them (including 1 Critical Schema 9 update, DDL dual-source elimination, evidence gate query correction). R2 achieved unanimous approval (3/3) with only 2 Minor residual observations. The 2-round maximum was used efficiently.

### 5. Perspective-Based Review Design
The three reviewer personas (security-sentinel, architecture-guardian, pragmatic-verifier) produced genuinely different findings in R1. Security-sentinel focused on SQL/shell injection risks, architecture-guardian on structural concerns and DDL conflicts, pragmatic-verifier on correctness of schema definitions and routing. This validates the prompt-diversity approach.

---

## What Could Improve

### 1. schemas.md Scope Management
`schemas.md` grew to 1528 lines during implementation (TASK-007) by accumulating SQLite DDL, routing matrix, evidence gates, and naming conventions alongside YAML schemas. This caused the DDL dual-source-of-truth conflict caught in code review. **Recommendation:** Future schema updates should be accompanied by a scope check — if adding non-YAML-schema content, it belongs in the appropriate reference doc instead.

### 2. Operational Database Schema Migration
The `CREATE TABLE IF NOT EXISTS` pattern doesn't handle schema evolution. When the `instance` column was added to canonical DDL, existing `verification-ledger.db` files didn't get updated. **Recommendation:** Implement the schema migration pattern documented in sql-templates.md §9: version tracking via `schema_meta` table, `ALTER TABLE ADD COLUMN` for additive changes, Step 0 migration check.

### 3. Design Review Findings Visibility
Design review (Step 3b) produced valuable findings (SEC-1/SEC-2 injection risks, ARCH-1 Knowledge Agent bloat, CORR-1/CORR-2 evidence gate conflicts) that shaped the implementation plan. However, the flow from design review findings → plan constraints was manual. **Recommendation:** Formalize the design-review-to-plan constraint propagation so planners can trace each constraint to its originating review finding.

### 4. Implementer Self-Check Line Count Accuracy
Multiple verification reports noted minor discrepancies between implementer-reported line counts and actual measured counts (e.g., 798 vs 797, 325 vs 334). While not functionally significant, these inaccuracies reduce trust in self-check data. **Recommendation:** Implementer should use `Measure-Object -Line` (PowerShell) or `wc -l` for authoritative line counts rather than estimating.

### 5. Pre-Existing IDE Errors Noise
Multiple files had pre-existing broken markdown anchor link errors (schemas.md, spec.agent.md, researcher.agent.md, knowledge-agent.agent.md). These are typically false positives from VS Code's markdown link checker unable to resolve heading fragments. The verifier correctly classified these as pre-existing, but they add noise to every verification report. **Recommendation:** Establish a baseline error registry to filter known false positives.

---

## Agent Performance Observations

### Spec Agent (Step 2)
Produced comprehensive specification with 16 FRs, 3 directions (A/B/C), and common requirements. Direction C (Full Architectural Overhaul) was selected. Strong FR-to-acceptance-criteria traceability.

### Design Agent (Step 3)
Delivered detailed design with 20-file inventory, SQLite schema definitions, and data flow diagrams. Overcomplicated the orchestrator line budget (initially 480, later revised to 550). Design review caught several critical issues that improved implementation quality.

### Plan Agent (Step 4)
Effective wave decomposition (5 waves, 18 tasks). Correctly captured all design review constraints (SEC-1/SEC-2, CORR-1/CORR-2, ARCH-1/ARCH-2/ARCH-3) as planning constraints. Risk classification aligned well with actual implementation difficulty.

### Implementer (Step 5)
Completed all 18 tasks plus R1 review fixes. Consistent output format across all implementation reports. Minor line-count self-reporting inaccuracies noted. Effectively handled the schemas.md R1 fixes (10 findings addressed in single batch).

### Verifier (Step 6)
Rigorous baseline cross-checking using `git show pipeline-baseline-*` tags. Appropriately used batch verification for low-risk tasks and individual verification for high-risk tasks. Tier 4 operational readiness checks on TASK-010 and TASK-012 added valuable assurance.

### Adversarial Reviewer (Step 7)
All three perspectives produced substantively different findings. Security-sentinel identified the deepest injection risks (SEC-1/SEC-2). Architecture-guardian found the DDL dual-source conflict. Pragmatic-verifier caught the critical Schema 9 staleness. R1→R2 cycle worked as designed — fix critical/major, approve with minor residuals.

---

## Pipeline Bottlenecks

1. **Wave 4 (Core Agent Modifications):** Highest risk wave with 4 concurrent tasks including the two 🔴-risk items (orchestrator, knowledge-agent). This is the critical path for the entire pipeline.

2. **Review Round 1 → Fix → Round 2 Cycle:** The R1 findings were concentrated in `schemas.md`, making the fix scope narrow but the review-then-fix round adds latency. In this case, the concentration was beneficial — a single-file fix batch addressed all major issues.

3. **Verification Report Volume:** 8 verification report files for 18 tasks (mix of batch and individual). The verifier appropriately batched low-risk tasks but this creates a non-uniform report structure. Consider standardizing batch sizes.

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Tasks completed | 18/18 |
| Verification checks | 52 passed, 0 failed |
| Code review rounds | 2 (R1: needs_revision, R2: approve) |
| R1 findings fixed | 10 (1C, 6Maj, 3Min) |
| R1 findings deferred | 6 (non-blocking) |
| R2 verdict | Unanimous approve (3/3) |
| Files created | 9 new shared/instruction docs |
| Files modified | 11 existing agent/schema files |
| Overall risk | 🔴 (2 red, 8 yellow, 8 green tasks) |
| Overall confidence | High |
| Known issues | 2 Minor |
| Regressions | 0 |
