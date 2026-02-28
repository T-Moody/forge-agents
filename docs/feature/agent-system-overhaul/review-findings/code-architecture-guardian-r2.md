# Adversarial Review: code — architecture-guardian (Round 2)

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🔴
- **Round:** 2
- **Run ID:** 2026-02-27T10:00:00Z
- **Task ID:** agent-system-overhaul-code-review
- **Scope:** code

---

## R1 Findings Resolution

### ARCH-1: Dual-Source-of-Truth DDL — RESOLVED ✓

The duplicate "Full Step 0 Initialization Script" (formerly schemas.md lines 1359–1416) has been removed and replaced with a cross-reference: "See sql-templates.md §1 for the canonical DDL. Do not duplicate DDL here to prevent drift." All remaining inline DDL blocks in schemas.md now carry "Illustrative reference only. Execute sql-templates.md §1 for the canonical DDL." warnings (schemas.md lines 986, 1052). The single-source-of-truth for DDL is now unambiguously sql-templates.md §1.

**Evidence:** schemas.md line 1186: "See sql-templates.md §1 for the canonical DDL." schemas.md lines 986, 1052: "Illustrative reference only" warning on inline DDL. No competing initialization script section exists.

### ARCH-2: schemas.md Scope Creep — INTENTIONALLY DEFERRED (acknowledged)

schemas.md reduced from 1528 → 1422 lines (~100 lines removed via DDL extraction). The most architecturally dangerous non-schema content (duplicate DDL, LIKE-based evidence gate queries) was extracted. Remaining non-schema governance content persists: `check_name` Naming Patterns (lines 1266–1303), Completion Contract Routing Matrix (lines 1310–1365), Output Format Classification (lines 1367–1420). The fix report explicitly defers this with rationale: "schemas.md scope reduction … requires broader refactoring. Critical content drift fixed; full extraction deferred."

**Assessment:** The deferred scope is reasonable — the residual content is lower-risk (conventions and routing tables) compared to the extracted content (executable DDL, approval-counting queries). Re-classified as Minor residual for R2.

### ARCH-3: LIKE-Based Evidence Gate Queries — RESOLVED ✓

The LIKE-based counting queries from the "Key Evidence Gate Queries" section have been removed and replaced with an explicit pointer: "See sql-templates.md §6 for all canonical evidence gate queries (EG-1 through EG-6). Do not duplicate queries here to prevent drift. Evidence gate logic uses instance-based grouping and per-category check_name matching — do not use LIKE-based counting for review approval checks." (schemas.md line 1044).

**Note:** The `check_name` Naming Patterns section (line 1287) retains `LIKE`-based filter examples for general record listing (`WHERE check_name LIKE 'review-design-%'`). These are convenience filters for record inspection, NOT approval-counting logic. They serve a different architectural purpose than the removed evidence gate queries and are appropriate in context. No action required.

**Evidence:** schemas.md line 1044 — pointer with explicit anti-LIKE warning. sql-templates.md §6 lines 422–480 — canonical EG-5/EG-6 using `IN()` clauses and `instance`-based grouping.

### CORR-1: Column-Level DDL Conflicts — RESOLVED ✓

Inline DDL in schemas.md now matches canonical sql-templates.md §1: `instance TEXT` column present (schemas.md line ~1000), `task_id TEXT` nullable, `tool TEXT` nullable, `round INTEGER DEFAULT 1` nullable, `ts TEXT NOT NULL DEFAULT (datetime('now'))` correct type. All 5 column-level conflicts identified in R1 are resolved in documentation.

**Evidence:** schemas.md lines 986–1005 vs sql-templates.md lines 99–118 — column definitions match on nullability, type, and constraint for all 15 columns including `instance`.

### CORR-2: Cross-Reference Drift — RESOLVED ✓

orchestrator.agent.md now correctly references "sql-templates.md §6" for evidence gate queries (line 370: "All evidence gates use SQL queries from sql-templates.md §6"). The incorrect "§5–§6" reference (where §5 is `instruction_updates` templates, not evidence gates) has been corrected.

**Evidence:** orchestrator.agent.md line 370: "sql-templates.md §6". Line 28: "sql-templates.md §6" in evidence gate verification responsibility.

---

## Security Analysis

**Category Verdict:** approve

No new security-relevant architecture findings. The R1 SEC-1 observation (instruction-based tool scope enforcement as accepted risk) remains architecturally appropriate with its documented 4-layer defense-in-depth.

---

## Architecture Analysis

**Category Verdict:** approve

### Finding ARCH-2-R2: schemas.md Non-Schema Governance Content Persists (Deferred Tech Debt)

- **Severity:** Minor
- **Category:** architecture
- **Description:** schemas.md remains at 1422 lines with 3 non-schema governance sections: (1) `check_name` Naming Patterns (lines 1266–1303 — pipeline convention), (2) Completion Contract Routing Matrix (lines 1310–1365 — pipeline governance), (3) Output Format Classification (lines 1367–1420 — pipeline convention). The most dangerous content was extracted in R1 fixes (duplicate DDL → pointer to sql-templates.md §1, LIKE evidence gates → pointer to sql-templates.md §6). The residual content is lower-risk: naming patterns and routing tables do not create conflicting executable logic. This is documented tech debt with reasonable deferral justification.
- **Affected artifacts:** [schemas.md lines 1266–1422](NewAgents/.github/agents/schemas.md#L1266-L1422)
- **Recommendation:** In a future iteration, extract: naming patterns → `pipeline-conventions.md`, routing matrix → `global-operating-rules.md` §5, output format classification → `pipeline-conventions.md`. Target: schemas.md ≤ 900 lines. Not blocking — deferred content poses no correctness or consistency risk.
- **Evidence:** schemas.md line 4 declares scope: "Single-source-of-truth for all typed YAML schema definitions." Lines 1266–1422 (156 lines) contain pipeline conventions and governance content outside this declared scope. File size: 1422 lines vs R1 recommendation of ≤900 lines. Deferred in fix report: REVIEW-FIXES-R1.yaml `findings_deferred[id='ARCH-2']`.

---

## Correctness Analysis

**Category Verdict:** approve

No remaining correctness findings. CORR-1 (column-level DDL conflicts) and CORR-2 (cross-reference drift) both verified as fully resolved. The inline DDL, column reference tables, and cross-document pointers are now consistent with their canonical sources.

---

## Overall Verdict

**Verdict:** approve
**Confidence:** High

## Summary

All 4 Major R1 findings (ARCH-1, ARCH-3, CORR-1, CORR-2) are fully resolved. Duplicate DDL replaced with sql-templates.md §1 pointer. LIKE-based evidence gate counting removed and replaced with sql-templates.md §6 pointer. Inline DDL synced to canonical. Orchestrator cross-references corrected. ARCH-2 (schemas.md scope reduction from 1422 to ≤900 lines) was intentionally deferred with reasonable justification — dangerous content extracted, lower-risk governance content remains as documented tech debt. 1 Minor residual observation. The single-source-of-truth architecture for DDL and evidence gate logic is now correctly established.
