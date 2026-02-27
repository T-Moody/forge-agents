# Adversarial Design Review — Round 2, Reviewer 1 (gpt-5.3-codex perspective)

**Focus Lens:** Security
**Design Version:** v4
**Round 1 Findings:** 12 (2 Critical, 5 High, 4 Medium, 1 Low)

## Overall Verdict: APPROVE

All 12 Round 1 findings from Reviewer 1 are **RESOLVED**. The v4 revisions are thorough and well-integrated. 2 new findings identified (1 Medium, 1 Low) — neither blocking.

---

## Round 1 Finding Verification

### [Finding 1] (Critical): `anvil_checks` schema missing `verdict`/`severity` columns → **RESOLVED**

**Evidence:** §Decision 6, SQL Schema block (lines ~690–710). The `anvil_checks` table now includes:

- `run_id TEXT NOT NULL`
- `verdict TEXT CHECK(verdict IN ('approve', 'needs_revision', 'blocker'))`
- `severity TEXT CHECK(severity IN ('Blocker', 'Critical', 'Major', 'Minor'))`
- `round INTEGER NOT NULL DEFAULT 1`

CHECK constraints enforce allowed values. Column semantics for review vs. verification records are explicitly documented. Security blocker detection is now a SQL-level operation: `WHERE verdict='blocker'`.

### [Finding 2] (Critical): Evidence gate queries accumulate stale records across rounds → **RESOLVED**

**Evidence:** §Decision 6, Evidence Gating Mechanism. All review gate queries now include `AND round = {current_round}` filter. The `round` column has `NOT NULL DEFAULT 1` constraint. Round incrementing is specified in pipeline flow (Steps 3b, 7). Stale records from prior rounds are excluded from gate counts without losing audit trail.

### [Finding 3] (High): Concurrent SQLite writes cause SQLITE_BUSY → **RESOLVED**

**Evidence:** §Decision 6, SQL Schema init block and v4 concurrency fix note. `PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;` are set in centralized Step 0 initialization. The design explicitly states: _"These are hard requirements, not optional."_ WAL mode enables concurrent reads + single-writer queuing. busy_timeout=5000 prevents immediate SQLITE_BUSY failures.

### [Finding 4] (High): No `pipeline_run_id` for distinguishing runs → **RESOLVED**

**Evidence:** §Decision 6, SQL Schema. `run_id TEXT NOT NULL` is present in both `anvil_checks` and `pipeline_telemetry`. Generated at Step 0 as ISO8601 timestamp. All evidence gate queries filter on `run_id='{run_id}'`. Cross-run contamination is prevented.

### [Finding 5] (High): `task_id` semantics undefined for feature-level reviews → **RESOLVED**

**Evidence:** §Decision 6, `task_id` convention note (v4 — H6). Convention documented: per-task operations use Planner-assigned IDs (e.g., `task-03`); feature-level operations use `{feature_slug}-design-review` (Step 3b) or `{feature_slug}-code-review` (Step 7). Documented requirement to include in `schemas.md`.

### [Finding 6] (High): Missing index on `task_id` → **RESOLVED**

**Evidence:** §Decision 6, SQL Schema init block. Three indexes are now created:

- `idx_anvil_task_phase ON anvil_checks(task_id, phase)`
- `idx_anvil_run_round ON anvil_checks(run_id, round)`
- `idx_telemetry_step ON pipeline_telemetry(step)`

These cover the primary query patterns for evidence gates.

### [Finding 7] (High): Adversarial Reviewer has no YAML output → **RESOLVED**

**Evidence:** §Decision 2, Agent Detail #8 (Adversarial Reviewer). The reviewer now produces a typed YAML verdict summary at `review-verdicts/<scope>.yaml` with fields: `reviewer_model`, `review_focus`, `scope`, `verdict`, `findings_count`, `summary`. This restores the "typed YAML at every boundary" contract. Pipeline resume can now recover review verdicts from structured files.

### [Finding 8] (Medium): `CREATE TABLE IF NOT EXISTS` race condition → **RESOLVED**

**Evidence:** §Decision 3, Step 0 expansion. SQLite initialization (both tables + indexes + PRAGMAs) is centralized in Step 0, executed by the orchestrator via `run_in_terminal` before any agent dispatch. Agent-level `CREATE TABLE IF NOT EXISTS` is retained as a safety net only, not the primary initialization path. Race condition eliminated.

### [Finding 9] (Medium): Review evidence gate doesn't filter on `passed`/verdict → **RESOLVED**

**Evidence:** §Decision 6, Evidence Gating Mechanism. Review approval gates now filter `AND verdict = 'approve'`. Security blocker gates check `AND verdict = 'blocker'`. Completion gates check `AND verdict IS NOT NULL`. The gate no longer silently passes when all reviewers find issues — it explicitly checks for approval verdicts.

### [Finding 10] (Medium): Design review gate query missing `check_name` filter → **RESOLVED**

**Evidence:** §Decision 6, Evidence Gating Mechanism. Design review gate now includes `AND check_name LIKE 'review-design-%'`. Code review gate includes `AND check_name LIKE 'review-code-%'`. Symmetric and consistent. Cross-scope record collision is prevented.

### [Finding 11] (Medium): Contradiction with `orchestrator-tool-restriction` feature → **RESOLVED**

**Evidence:** §Deviation Records, DR-1. The design now explicitly documents the deviation from FR-1.6 with rationale, constraint scope (SQLite queries, SQLite DDL, Git read/commit only), and spec reconciliation guidance. The contradiction is acknowledged, justified, and bounded — not silently inherited.

### [Finding 12] (Low): No `output_snippet` length constraint → **RESOLVED**

**Evidence:** §Decision 6, SQL Schema. `output_snippet TEXT CHECK(LENGTH(output_snippet) <= 500)` is now in the schema, matching Anvil's 500-char truncation rule.

---

## New Findings (Security Lens)

### [NEW-1]: `git add -A` stages all files indiscriminately — potential sensitive file exposure

- **Severity:** Medium
- **Area:** §Decision 3 (Step 5, v4 — H8), §Decision 3 (Step 9)
- **Issue:** The v4 fix for H8 adds `git add -A` as the Implementer's final step after self-fix, and Step 9 uses `git add -A && git commit` for auto-commit. `git add -A` stages ALL files in the working directory, including untracked files. If the working directory contains sensitive files (`.env`, API key files, temporary debug files with credentials, IDE workspace settings with secrets), these will be staged and committed. The design includes a Tier 4 secrets scan (`check_name='readiness-{type}'`), but this runs only for Large tasks and occurs at Step 6 — after `git add -A` has already staged files at Step 5. By the time the Verifier checks for secrets, the sensitive files are already in the staging area.
- **Likelihood:** Medium — depends on project hygiene. Well-maintained projects have comprehensive `.gitignore` files. Greenfield projects or projects with temporary files are at risk.
- **Impact:** Medium — accidental commit of secrets or sensitive configuration. The auto-commit at Step 9 would permanently record these in git history.
- **Assumption at risk:** That the project's `.gitignore` is comprehensive enough to exclude all sensitive files from `git add -A`.
- **Mitigation note:** Git does respect `.gitignore` during `git add -A`. The risk is limited to files not covered by `.gitignore`. The Step 0 git hygiene check (`git status --porcelain`) could be extended to warn about untracked files before implementation begins.

### [NEW-2]: `verdict` and `severity` columns are nullable for review records — no composite enforcement

- **Severity:** Low
- **Area:** §Decision 6, SQL Schema
- **Issue:** The `verdict` and `severity` columns lack `NOT NULL` constraints. The design documents that for `phase='review'` records, `verdict` should always be populated, and for `phase='baseline'`/`phase='after'`, they should be NULL. However, there is no composite CHECK constraint enforcing this relationship (e.g., `CHECK(phase != 'review' OR verdict IS NOT NULL)`). A review INSERT that accidentally omits `verdict` would succeed, but the evidence gate query `WHERE verdict IS NOT NULL >= 3` would not count it — creating a silent gate failure that's hard to debug. The agent would appear to have completed (record exists) but the gate would fail (verdict missing).
- **Likelihood:** Low — agents are instructed to include verdict, and self-validation should catch omissions.
- **Impact:** Low — gate would fail conservatively (block rather than pass), which is the safe direction. But debugging why the gate fails when all 3 records exist would require examining individual record fields.
- **Assumption at risk:** That LLM-generated SQL INSERT statements will always include all semantically-required fields even when the schema permits NULL.

---

## Summary

| Category                  | Count  | Status                                     |
| ------------------------- | ------ | ------------------------------------------ |
| Round 1 Critical findings | 2      | 2 RESOLVED                                 |
| Round 1 High findings     | 5      | 5 RESOLVED                                 |
| Round 1 Medium findings   | 4      | 4 RESOLVED                                 |
| Round 1 Low findings      | 1      | 1 RESOLVED                                 |
| **Total Round 1**         | **12** | **12 RESOLVED**                            |
| New findings (Medium)     | 1      | NEW-1: `git add -A` sensitive file risk    |
| New findings (Low)        | 1      | NEW-2: nullable verdict for review records |

**Assessment:** The v4 design comprehensively addresses all 12 Round 1 findings from this reviewer. The fixes are well-integrated into the document — not bolted on as patches but woven into the relevant sections with cross-references. The SQL schema is now robust for multi-agent, multi-round, multi-run scenarios. Evidence gate queries are properly filtered. The 2 new findings are non-blocking: NEW-1 is a reasonable observation about `git add -A` hygiene (mitigated by `.gitignore`), and NEW-2 is a minor schema strictness gap that fails conservatively.

**Final Verdict: APPROVE**
