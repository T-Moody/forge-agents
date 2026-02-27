# Adversarial Design Review — Round 2, Reviewer 2

**Model Persona:** gemini-3-pro-preview  
**Focus Lens:** Architecture / Correctness  
**Design Version:** v4 (post-adversarial revision)  
**Round 1 Reference:** [adversarial-review-2.md](adversarial-review-2.md) (15 findings: 4C, 6H, 5M)

---

## Overall Verdict: APPROVE

The v4 design addresses the majority of Round 1 findings substantively. All 4 Critical findings are fully resolved. Of the 6 High findings, 5 are fully resolved and 1 is partially resolved (session recall acknowledged as intentional omission). Of the 5 Medium findings, 3 are resolved, and 2 remain unresolved but are non-blocking. However, v4 introduces **1 new High-severity inconsistency** (duplicate schema definitions with conflicting constraints) and **3 new Medium issues** that should be corrected before implementation but do not block planning.

---

## Round 1 Finding Verification

### Finding 1 (Critical): FR-1.6 / CR-4 / FR-5.3 — No Deviation Records

**Status: RESOLVED**

**Evidence:** §Deviation Records (v4 — C3) adds two formal deviation records:

- **DR-1:** Documents orchestrator `run_in_terminal` usage with constrained scope (SQLite queries, DDL, git read/commit), rationale, and spec reconciliation guidance for FR-1.6.
- **DR-2:** Documents always-3 reviewers with rationale (simplicity, consistent quality, low marginal cost) and spec reconciliation guidance for FR-5.3 and CR-4.

Both records follow the pattern requested: rationale, tradeoff, and pointers to which spec items need updating. The constraint on `run_in_terminal` usage (no builds, tests, code execution, file modification) is explicitly documented in Orchestrator agent detail and DR-1.

---

### Finding 2 (Critical): SQLite Concurrent Write Hazard

**Status: RESOLVED**

**Evidence:** §Decision 6 SQL Schema now includes `PRAGMA journal_mode=WAL;` and `PRAGMA busy_timeout=5000;` as the first lines of the Step 0 initialization block. The SQLite Operational Notes explicitly state: "WAL mode is set in Step 0 initialization; all agents inherit the journal mode from the existing database file." The §v4 Revision Log C2 entry marks this as "MANDATORY."

---

### Finding 3 (Critical): Stale v2 Sequence — Read plan-output.yaml Before It Exists

**Status: RESOLVED**

**Evidence:** The Standard Feature Flow (§Sequence / Interaction Notes, ~line 1700+) no longer contains the stale line `→ Orchestrator: Read plan-output.yaml overall_risk_summary` between Step 3 (Designer) and Step 3b (Design Review). The flow now proceeds:

```
→ Designer: Produce design-output.yaml + design.md
→ Orchestrator: Read completion contract
→ Orchestrator: Dispatch Adversarial Reviewer ×3 (Step 3b)
```

The orchestrator reads `overall_risk_summary` from `plan-output.yaml` only after Step 4 (Planner), which is the correct sequence position.

---

### Finding 4 (Critical): Baseline Cross-Check Physically Impossible

**Status: RESOLVED**

**Evidence:** §Decision 6, Baseline Capture Integration (v4 — C5) replaces the v3 approach entirely:

1. Orchestrator creates `git tag pipeline-baseline-{run_id}` before dispatching implementers (Step 5).
2. Verifier uses `git show pipeline-baseline-{run_id}:<filepath>` to inspect pre-implementation state.
3. Discrepancies between Verifier's independent reading and Implementer's self-report are flagged as `check_name='baseline-discrepancy'`.

This is a physically possible approach that provides genuine independent baseline verification. §Decision 2 (Verifier detail) also references this mechanism explicitly.

---

### Finding 5 (High): Design Review `task_id` Undefined

**Status: RESOLVED**

**Evidence:** §Decision 6, `task_id` convention (v4 — H6) states: "Per-task operations use Planner-assigned IDs (e.g., `task-03`). Feature-level operations use `{feature_slug}-design-review` (Step 3b) or `{feature_slug}-code-review` (Step 7). This convention MUST be documented in `schemas.md` and used by all agents consistently."

This provides the explicit convention needed for feature-scoped evidence gate queries.

---

### Finding 6 (High): Code Review Context Bomb — Full `git diff --staged`

**Status: UNRESOLVED**

**Evidence:** The v4 design still dispatches code reviewers at Step 7 with `git diff --staged` as primary input (§Decision 3, Step 7: "Input: git diff --staged, verification evidence, implementation reports"). No partitioning strategy is introduced. For a 20-task / 50+ file feature, the staged diff could be thousands of lines per reviewer.

**Assessment:** This is a scalability concern (NFR-3 requires handling 10+ tasks / 50+ files) but not a correctness bug. For the initial implementation targeting 6-task features (~26 dispatches), this is unlikely to cause problems. It becomes a risk at the upper bounds of the design's stated scalability targets. **Severity remains High** for the design's full scope, but is non-blocking for initial implementation planning.

---

### Finding 7 (High): No Git Staging Step

**Status: RESOLVED**

**Evidence:** §Decision 3, Step 5 now includes: "After self-fix: `git add -A` to stage changes (v4 — H8)." The Implementer's workflow explicitly includes git staging as the final step before returning. The §v4 Revision Log H8 entry confirms this addition.

---

### Finding 8 (High): Operational Readiness Checks Dropped

**Status: RESOLVED**

**Evidence:** §Decision 2 (Verifier detail) adds "Tier 4 (v4 — Large tasks only): Operational Readiness — observability hooks present, graceful degradation paths tested, no hardcoded secrets (`check_name='readiness-{type}'`)." §Decision 3, Step 6 includes: "Tier 4 (v4 — H11): Operational Readiness (Large tasks only) — observability, degradation handling, secrets scan."

This directly ports Anvil's Step 5d operational readiness checks into the Verifier's cascade, scoped to Large tasks only as in Anvil.

---

### Finding 9 (High): Session Recall Dropped from Anvil

**Status: PARTIALLY_RESOLVED**

**Evidence:** §v4 Revision Log H10 states: "session recall intentionally omitted (VS Code `store_memory` provides broad patterns but not per-file regression history — accepted as known capability regression)." The design acknowledges the gap but does not restore the capability.

**Assessment:** This is an honest characterization of a deliberate scope reduction. The design correctly identifies that `store_memory` is not a substitute for `session_store` SQL (different query model, different granularity). The gap is documented rather than hidden. Acceptable as a v1 limitation since the Knowledge Agent's `store_memory` calls provide a weaker but functional cross-session learning path.

---

### Finding 10 (High): Reversion Responsibility After 2 Failed Fix Attempts

**Status: RESOLVED**

**Evidence:** §Decision 6, Verification-Replan Loop (v4 — H9/FR-4.7) explicitly assigns revert to the Implementer in "revert mode": `{mode: 'revert', files_to_revert: [...], baseline_tag: 'pipeline-baseline-{run_id}'}`. The Implementer executes `git checkout pipeline-baseline-{run_id} -- {files}` and records the revert in `anvil_checks` with `check_name='revert-{task_id}'`.

The rationale for choosing Implementer (only agent with both `run_in_terminal` and file-write permissions) is sound.

---

### Finding 11 (Medium): Orchestrator Context Window at 20+ Tasks

**Status: UNRESOLVED**

**Evidence:** No context management strategy has been added for the orchestrator. The Pipeline State Model still tracks all state in-context. For a 20-task feature with 54+ dispatches, the orchestrator accumulates routing decisions, error logs, task statuses, and SQL output parsing results without any pruning or offloading mechanism.

**Assessment:** This remains a scalability concern for complex features. The `pipeline_telemetry` SQLite table exists and could serve as context offloading, but the design doesn't describe the orchestrator writing to it or reading from it as a context management strategy. Severity remains Medium — the design works for typical 6-task features and the concern is at the stated upper bounds.

---

### Finding 12 (Medium): Mermaid Diagram Stale "init manifest" Reference

**Status: RESOLVED**

**Evidence:** §Decision 10, Mermaid diagram now reads:

```
S0["Step 0: Setup<br/>Init SQLite + WAL, git hygiene, check prerequisites"]
```

The stale "init manifest" reference has been replaced with the actual v4 Step 0 activities.

---

### Finding 13 (Medium): Design Review Evidence Gate Query Unfiltered

**Status: RESOLVED**

**Evidence:** §Decision 6, Evidence Gating Mechanism now includes `check_name LIKE 'review-design-%'` filter on the Step 3b design review gate query and `check_name LIKE 'review-code-%'` on the Step 7 code review gate query. Both gate types are symmetric and future-proof.

---

### Finding 14 (Medium): `anvil_checks` Columns Meaningless for Review Verdicts

**Status: RESOLVED**

**Evidence:** §Decision 6, SQL Schema includes "Column semantics for review records (v4)": "When `phase='review'`: `command='adversarial-review'`, `exit_code=NULL`, `output_snippet` = first 500 chars of summary, `verdict` = overall verdict, `severity` = highest finding severity." This explicitly documents how each column is used for review vs. verification records.

---

### Finding 15 (Medium): Knowledge Agent Safety Filter Undefined

**Status: UNRESOLVED**

**Evidence:** The Knowledge Agent detail still states: "Governed updates: Instruction file modifications require explicit approval in interactive mode; logged in autonomous mode." And "safety constraint filter prevents weakening existing security checks." No definition of the safety filter's rules, detection mechanism, or failure behavior has been added.

**Assessment:** This is a medium-severity gap retained from v3. In autonomous mode, the Knowledge Agent can modify instruction files with only logging — the "safety constraint filter" is an undefined claim. Non-blocking for planning since the Knowledge Agent itself is non-blocking (ERROR doesn't halt pipeline), but the claim of a safety filter should either be defined or removed.

---

## New Findings (Introduced in v4)

### NEW-1: Duplicate `anvil_checks` Schema with Conflicting Constraints

- **Severity:** High
- **Area:** §Decision 6 (SQL Schema, line ~684) vs. §Data Storage Strategy (SQLite Schema Details, line ~2058)
- **Issue:** The `anvil_checks` CREATE TABLE statement appears in two locations with contradictory column constraints:

  | Column           | §Decision 6 (line ~684)                                        | §Data Storage (line ~2058)                                            |
  | ---------------- | -------------------------------------------------------------- | --------------------------------------------------------------------- |
  | `output_snippet` | `CHECK(LENGTH(output_snippet) <= 500)`                         | `CHECK(length(output_snippet) <= 2000)`                               |
  | `severity`       | `CHECK(severity IN ('Blocker', 'Critical', 'Major', 'Minor'))` | `CHECK(severity IN ('blocker', 'critical', 'high', 'medium', 'low'))` |

  The Data Storage section uses a **completely different severity taxonomy** — lowercase, and substituting `high`/`medium`/`low` for `Major`/`Minor`. This directly violates **CR-13** ("Unified severity taxonomy: Blocker/Critical/Major/Minor") and makes the design internally contradictory. An implementer following §Decision 6 builds a different database than one following §Data Storage.

- **Likelihood:** High — implementers will encounter both definitions.
- **Impact:** High — one definition will be wrong; queries filtering on severity will break for records inserted using the other definition's vocabulary.
- **Recommendation:** Remove the duplicate schema from §Data Storage Strategy and reference §Decision 6 as the single authoritative source. Alternatively, keep both but make them identical, using `('Blocker', 'Critical', 'Major', 'Minor')` to comply with CR-13, and `<= 500` to match Anvil's truncation rule.

---

### NEW-2: `git tag` Fails on Replan Loop Iteration 2+

- **Severity:** Medium
- **Area:** §Decision 3 (Steps 5-6 loop), §Decision 6 (Baseline Capture Integration)
- **Issue:** The git baseline tag `pipeline-baseline-{run_id}` is created inside the Steps 5-6 implementation-verification loop (line ~1726: "[Pre-impl] Orchestrator creates git tag pipeline-baseline-{run_id}"). The `run_id` is generated once in Step 0 and is fixed for the pipeline run. On **replan loop iteration 2+**, `git tag pipeline-baseline-{run_id}` will fail because the tag already exists from iteration 1. `git tag` does not overwrite by default — it returns a fatal error.

  §Decision 6 says "The tag is created ONCE per implementation wave, not per task." But the pipeline flow diagram positions the tag creation as a sub-step of Step 5, which repeats in the replan loop.

- **Likelihood:** Medium — occurs whenever a task fails verification and triggers a replan loop.
- **Impact:** Medium — the orchestrator receives a `run_in_terminal` error for tag creation. If it treats this as a transient error and retries, it wastes a retry. If it treats it as a pipeline error, it blocks the replan loop.
- **Recommendation:** Either (a) move tag creation outside the loop to a one-time pre-Step-5 action, or (b) use `git tag -f pipeline-baseline-{run_id}` to force-update (but this loses the original baseline), or (c) add explicit "skip if tag exists" logic: `git tag pipeline-baseline-{run_id} 2>/dev/null || true`.

---

### NEW-3: `check_name` Convention for Adversarial Reviewer INSERT Undocumented

- **Severity:** Medium
- **Area:** §Decision 2 (Adversarial Reviewer detail), §Decision 6 (Evidence Gating Mechanism)
- **Issue:** Evidence gate queries filter on `check_name LIKE 'review-design-%'` and `check_name LIKE 'review-code-%'`. But the Adversarial Reviewer's agent detail and SQL INSERT specification never explicitly state what `check_name` value to write. The reviewer has a `review_focus` parameter (`security` | `architecture` | `correctness`), and the logical `check_name` would be `review-design-security`, `review-design-architecture`, etc. But this mapping is implicit, not documented.

  If an implementer chooses `check_name='adversarial-review'` (as suggested in the column semantics for `command`), the LIKE pattern won't match, and the evidence gate returns 0 — blocking the pipeline forever.

- **Likelihood:** Medium — the convention is inferable but not stated.
- **Impact:** Medium — incorrect `check_name` values cause evidence gate failure.
- **Recommendation:** Add an explicit INSERT example in the Adversarial Reviewer detail:
  ```sql
  INSERT INTO anvil_checks (run_id, task_id, phase, check_name, tool, passed, verdict, severity, round)
  VALUES ('{run_id}', '{task_id}', 'review', 'review-{scope}-{review_focus}', 'adversarial-review', 1, '{verdict}', '{severity}', {round});
  ```

---

### NEW-4: Two Separate SQLite DB Files vs. Unified Step 0 Initialization

- **Severity:** Medium
- **Area:** §Decision 10 (Pipeline Runtime File Structure), §Decision 3 (Step 0), §Decision 6 (SQL Schema)
- **Issue:** The Pipeline Runtime File Structure lists two separate database files:

  ```
  verification-ledger.db  — anvil_checks table
  pipeline-telemetry.db   — pipeline_telemetry table
  ```

  But Step 0 initialization creates both tables in a single SQL block, implying a single database connection and therefore a single `.db` file. If the tables are in separate files, Step 0 needs two separate `run_in_terminal` invocations connecting to different databases. If they're in the same file, the runtime file listing is wrong.

- **Likelihood:** Medium — implementer will need to decide which interpretation is correct.
- **Impact:** Low — easily resolved during implementation, but adds unnecessary ambiguity.
- **Recommendation:** Consolidate into a single `pipeline.db` containing both tables (simpler initialization, single WAL mode setup), or split the Step 0 initialization into two explicit blocks with the file paths specified.

---

## Requirement Coverage Gaps (Updated)

| Requirement                         | Status                         | Notes                                                                    |
| ----------------------------------- | ------------------------------ | ------------------------------------------------------------------------ |
| FR-1.6 (Orchestrator tools)         | ✅ **Formally deviated**       | DR-1 documents the deviation with constrained scope                      |
| FR-5.3 (Risk-driven reviewer count) | ✅ **Formally deviated**       | DR-2 documents always-3 with rationale                                   |
| CR-4 (Adversarial review counts)    | ✅ **Formally deviated**       | DR-2 covers this                                                         |
| FR-4.7 (Revert on failure)          | ✅ **Resolved**                | Implementer revert mode with git checkout                                |
| CR-13 (Unified severity)            | ⚠️ **Contradicted internally** | §Data Storage uses wrong taxonomy — see NEW-1                            |
| NFR-3 (Scalability 10+ tasks)       | ⚠️ **Unmitigated risk**        | Code review context bomb (Finding 6) + orchestrator context (Finding 11) |

---

## Summary Statistics

| Category                     | Count                                                                         |
| ---------------------------- | ----------------------------------------------------------------------------- |
| Round 1 — RESOLVED           | 12                                                                            |
| Round 1 — PARTIALLY_RESOLVED | 1 (Finding 9: session recall — intentional omission, documented)              |
| Round 1 — UNRESOLVED         | 2 (Finding 6: code review context bomb; Finding 11: orchestrator context)     |
| NEW — High                   | 1 (NEW-1: duplicate schema with conflicting constraints)                      |
| NEW — Medium                 | 3 (NEW-2: git tag replan; NEW-3: check_name convention; NEW-4: db file split) |
| **Total Open**               | **6** (1H, 2H-retained, 1M-retained, 3M-new)                                  |

The v4 revision resolved all 4 Critical findings and 5 of 6 High findings from Round 1. The remaining open issues are:

- **1 High (NEW-1):** Must be fixed — dual schema with conflicting severity taxonomy breaks CR-13 compliance.
- **2 High (retained):** Finding 6 (context bomb) is a scalability concern at 20+ tasks, non-blocking for typical usage. Finding 11 (orchestrator context) is similar.
- **4 Medium:** Non-blocking but should be tracked.

---

## Verdict: APPROVE

The design is architecturally sound and ready for planning with the following caveats:

1. **NEW-1 (duplicate schema) should be fixed before implementation** — a 2-minute edit to reconcile the Data Storage schema with Decision 6.
2. The two retained High findings (context bomb, orchestrator context) are scalability concerns at the design's upper bounds, not correctness bugs. They affect features with 20+ tasks / 50+ files.
3. The medium findings are implementation-level clarifications that the Planner or Implementer can resolve.
