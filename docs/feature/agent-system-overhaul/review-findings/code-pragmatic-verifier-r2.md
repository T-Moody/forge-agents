# Adversarial Review: code — pragmatic-verifier (Round 2)

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🔴
- **Round:** 2
- **Run ID:** 2026-02-27T10:00:00Z

---

## Round 1 Finding Resolution Status

### C-1 (Critical): schemas.md Schema 9 Uses Outdated Field Names — **RESOLVED** ✅

Schema 9 YAML Verdict Summary Fields table now uses `reviewer_perspective` (line 780) with correct allowed values (`security-sentinel | architecture-guardian | pragmatic-verifier`). The old `reviewer_model` and `review_focus` fields are gone. The single `verdict` field is replaced with a `verdicts` object containing per-category sub-verdicts (`security`, `architecture`, `correctness`) plus an `overall` aggregate verdict (lines 783–786). The example YAML block (lines 827–841) matches the new structure. This is fully consistent with adversarial-reviewer.agent.md's YAML Verdict Summary Structure.

### C-2 (Major): Perspective Names Don't Match — **RESOLVED** ✅

The Verdict File Naming Convention (lines 771–774) now uses the correct perspective IDs: `security-sentinel`, `architecture-guardian`, `pragmatic-verifier`. Example paths are `review-verdicts/design-security-sentinel.yaml` and `review-verdicts/code-pragmatic-verifier.yaml` — matching review-perspectives.md §3–4 and all agent files.

### C-3 (Major): SQL check_name Convention Contradicts Evidence Gate Queries — **RESOLVED** ✅

The SQL INSERT Convention table (lines 808–822) now specifies `check_name = 'review-{scope}-{category}'` where category is `security | architecture | correctness`. The `instance` column is documented for storing the reviewer perspective ID. A note (line 805) explains the 3-INSERT-per-reviewer pattern and cross-references sql-templates.md §6 (EG-5, EG-6). The check_name Naming Patterns table (line 1284) is updated with the consolidated `review-{scope}-{category}` pattern. All references are now consistent with sql-templates.md §6 EG-5 query which uses `check_name IN ('review-{scope}-security', 'review-{scope}-architecture', 'review-{scope}-correctness')`.

### C-4 (Major): Designer NEEDS_REVISION Routing Matrix — **RESOLVED** ✅

The Completion Contract Routing Matrix (line 1315) Designer row now shows `N/A (not returned)` in the `NEEDS_REVISION` column, consistent with designer.agent.md's Completion Contract ("The Designer agent NEVER returns `NEEDS_REVISION`") and the matrix's own note that "Only Adversarial Reviewer and Verifier support `NEEDS_REVISION`."

### C-5 (Minor): pipeline-conventions.md Abbreviated Perspective Names — **RESOLVED** ✅

pipeline-conventions.md lines 19–20 now show full perspective IDs in examples: `design-security-sentinel.yaml`, `code-architecture-guardian.yaml`, `code-pragmatic-verifier.md`. Consistent with adversarial-reviewer.agent.md and schemas.md.

### A-1 (Minor): Orchestrator §5–§6 Reference — **RESOLVED** ✅

All orchestrator.agent.md references to sql-templates.md now correctly cite §6 only (5 occurrences verified: lines 24, 175, 242, 282, 362). The stale §5 reference is gone.

### C-6 (Minor): copilot-instructions.md echo vs printf — **DEFERRED** (Accepted)

Deferred with valid rationale: copilot-instructions.md is a global config file outside the review scope, and `echo` is safe on the target Windows/PowerShell platform.

---

## Security Analysis

**Category Verdict:** approve

No new security findings. The deferred SEC-1 (sql_escape enforcement) and SEC-2 (shell injection patterns) from other reviewers remain documented in the fix report with reasonable rationale. No regression from the fixes applied.

---

## Architecture Analysis

**Category Verdict:** approve

No new architecture findings. The schemas.md DDL is now marked "Illustrative reference only" with a pointer to sql-templates.md §1 as canonical source (line 988), properly addressing the ARCH-1 dual-source-of-truth concern. Evidence gate queries in schemas.md replaced with sql-templates.md §6 pointers (ARCH-3). No new coupling or consistency issues introduced by the fixes.

---

## Correctness Analysis

**Category Verdict:** approve

### Finding R2-1: Live DB Schema Lacks `instance` Column Defined in Canonical DDL

- **Severity:** Minor
- **Category:** correctness
- **Description:** The live `verification-ledger.db` in this feature directory was created before the Round 1 fixes and uses the old DDL without the `instance` column. The canonical DDL in sql-templates.md §1 (line 113) includes `instance TEXT`, and schemas.md's illustrative DDL (line 1001) also shows it. Any agent attempting to INSERT with the `instance` column against this DB will get "no such column: instance". This is an operational issue with the pre-existing DB, not a code defect — a fresh pipeline run using the fixed sql-templates.md §1 DDL would create the column correctly.
- **Affected artifacts:** [verification-ledger.db](verification-ledger.db) (pre-existing DB created before fixes)
- **Recommendation:** No code change needed. For the current run, agents should omit the `instance` column from INSERTs against this DB (as Round 2 records 73–76 already do). For future runs, the fixed DDL in sql-templates.md §1 will create the column automatically.
- **Evidence:** `echo ".schema anvil_checks" | sqlite3 verification-ledger.db` shows no `instance` column. sql-templates.md §1 line 113 defines `instance TEXT`. Round 2 DB records (IDs 73–76) were successfully inserted without the `instance` column.

---

## Summary

All 4 Critical and Major findings from Round 1 (C-1, C-2, C-3, C-4) have been **fully resolved** in schemas.md. The Schema 9 YAML Verdict Summary now correctly uses `reviewer_perspective` with per-category `verdicts`, the perspective names match the canonical IDs from review-perspectives.md, the SQL INSERT convention uses `review-{scope}-{category}` check_names consistent with sql-templates.md §6, and the Designer routing matrix entry is corrected. Both Minor findings within fix scope (C-5, A-1) are also resolved. The one new Minor observation (R2-1) is an operational artifact of the pre-existing DB, not a code defect.
