# Adversarial Review: code — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-02-27T10:00:00Z
- **Task ID:** agent-system-overhaul-code-review
- **Scope:** code
- **Verification Evidence:** All checks passed across all 18 tasks (48+ verification records, 0 failures)

---

## Security Analysis

**Category Verdict:** approve

### Finding SEC-1: Tool scope enforcement is instruction-based — accepted risk is architecturally appropriate

- **Severity:** Minor
- **Category:** security
- **Description:** All tool access restrictions (run_in_terminal scope, create_file path constraints, replace_string_in_file prohibitions) are enforced via LLM instruction-following with no programmatic guardrails from the VS Code agent runtime. The system mitigates this with a 4-layer defense-in-depth architecture: (1) per-agent prompt instructions with explicit allow/deny lists, (2) self-verification checklists requiring agents to confirm compliance before returning, (3) regex validation patterns specified per-agent in tool-access-matrix.md §2–§10, and (4) the verifier acting as secondary enforcement auditing other agents' outputs. This layered approach is the strongest available defense given VS Code's runSubagent API constraints. The residual threat level (Medium) is appropriately assessed and documented.
- **Affected artifacts:** [tool-access-matrix.md §11](NewAgents/.github/agents/tool-access-matrix.md#L79-L86)
- **Recommendation:** No action required. From an architecture perspective, the defense layers are properly separated (each layer operates independently of the others), and the accepted risk is documented with correct residual threat assessment. Future mitigation: if VS Code adds parameterized tool policies, adopt them as a 5th enforcement layer.
- **Evidence:** tool-access-matrix.md §11: "Scope restrictions are enforced via agent instructions. VS Code's agent runtime does not support parameterized tool access policies. Compliance depends on LLM instruction-following... This is an accepted risk (residual threat level: Medium)." All 9 agent files reference their tool-access-matrix section. Verifier agent explicitly includes "verify no tool scope violations" in its verification workflow.

---

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding ARCH-1: Dual-Source-of-Truth for DDL — schemas.md and sql-templates.md Both Claim Ownership

- **Severity:** Major
- **Category:** architecture
- **Description:** Two shared reference documents both contain the full DDL for all 4 SQLite tables (`anvil_checks`, `pipeline_telemetry`, `artifact_evaluations`, `instruction_updates`), and their definitions conflict. `sql-templates.md` §1 states it is "the single source of truth for all database operations" and the orchestrator's Step 0 instructions say to execute DDL from sql-templates.md §1. Meanwhile, `schemas.md` contains a "Full Step 0 Initialization Script" (lines 1359–1416) with a competing DDL definition that diverges in multiple column constraints. This dual-ownership pattern violates the fundamental architecture principle that each concern should have exactly one authoritative source. It creates confusion for any agent or developer asked to initialize the database: which DDL should be used?
- **Affected artifacts:** [schemas.md lines 1359–1416](NewAgents/.github/agents/schemas.md#L1359-L1416) ("Full Step 0 Initialization Script"), [sql-templates.md §1](NewAgents/.github/agents/sql-templates.md#L75-L135) (canonical DDL)
- **Recommendation:** Remove the DDL from schemas.md entirely. schemas.md should remain the "single source of truth for all typed YAML schema definitions" — SQLite DDL is not a YAML schema. Replace the "Full Step 0 Initialization Script" section with a reference: "See sql-templates.md §1 for all SQLite DDL. Do not duplicate DDL here to prevent drift." The column reference tables in schemas.md (human-readable documentation) can remain as they serve a different purpose than executable DDL.
- **Evidence:** sql-templates.md §1 header: "All SQL DDL and DML templates for the pipeline. This document is the **single source of truth** for all database operations." schemas.md header: "Single-source-of-truth for all typed YAML schema definitions." Conflict examples: `anvil_checks.task_id` is `TEXT NOT NULL` in schemas.md vs `TEXT` (nullable) in sql-templates.md; `anvil_checks.instance` column exists in sql-templates.md but is absent from schemas.md DDL; `pipeline_telemetry.status` CHECK includes `'running'` in schemas.md but not in sql-templates.md.

### Finding ARCH-2: schemas.md Has Exceeded Its Architectural Scope — 1528 Lines Spanning 6+ Concern Domains

- **Severity:** Major
- **Category:** architecture
- **Description:** schemas.md was designed as the "single source of truth for all typed YAML schema definitions." It has grown to 1528 lines — 4.8× the average agent file size — and now contains 6+ distinct concern domains beyond YAML schemas: (1) SQLite DDL and column reference tables, (2) evidence gate queries, (3) `check_name` naming conventions with SQL LIKE filter patterns, (4) a Completion Contract Routing Matrix, (5) an Output Format Classification table, and (6) a Schema Evolution Strategy. Items 3–5 are pipeline-governance concerns, not schema definitions. This scope creep creates a maintenance burden: any change to DDL, naming conventions, routing rules, or output format policies requires editing this single monolithic document. It also blurs the boundary between schemas.md (YAML schemas) and the appropriate owners of each concern (sql-templates.md for DDL/queries, global-operating-rules.md for routing, pipeline-conventions.md for naming).
- **Affected artifacts:** [schemas.md](NewAgents/.github/agents/schemas.md) (entire file, 1528 lines)
- **Recommendation:** Extract non-schema content into its existing or natural owner documents: (a) Move SQLite DDL/queries → already in sql-templates.md (remove the duplicate per ARCH-1). (b) Move `check_name` Naming Patterns → pipeline-conventions.md (naming is a convention, not a schema). (c) Move Completion Contract Routing Matrix → global-operating-rules.md §5 already references this; embed it there. (d) Move Output Format Classification → pipeline-conventions.md (format classification is a convention). (e) Keep: YAML schema definitions (Schemas 1–9), Producer/Consumer Dependency Table, Schema Evolution Strategy (tied to schema management). Target: schemas.md ≤ 900 lines after extraction.
- **Evidence:** Current sections in schemas.md: "Schemas 1–9" (core purpose), "SQLite Schemas" (4×DDL + column refs + queries — duplicate of sql-templates.md), "task_id Naming Convention" (pipeline convention), "check_name Naming Patterns" (pipeline convention + SQL LIKE patterns), "Completion Contract Routing Matrix" (pipeline governance), "Output Format Classification" (pipeline governance), "Schema Evolution Strategy" (schema management). Of these, only Schemas 1–9, the dependency table, and schema evolution are within the declared scope.

### Finding ARCH-3: Evidence Gate Queries in schemas.md Use Incompatible Logic With the 3-INSERT-per-Reviewer Pattern

- **Severity:** Major
- **Category:** architecture
- **Description:** schemas.md's "Key Evidence Gate Queries" section uses a `LIKE`-based counting approach: `SELECT COUNT(*) FROM anvil_checks WHERE check_name LIKE 'review-design-%' AND verdict = 'approve'` expecting result ≥ 2 for majority approval. Meanwhile, sql-templates.md §6 EG-5/EG-6 uses specific `check_name IN ('review-{scope}-security', 'review-{scope}-architecture', 'review-{scope}-correctness')` filtering with `instance`-based grouping. Under the implemented 3-INSERT-per-reviewer pattern (each of 3 reviewers inserts 3 category rows = 9 total rows), the schemas.md `LIKE`-based count of `verdict='approve'` would reach ≥2 even if only **one** reviewer approves all 3 categories (3 approve rows ≥ 2 threshold). This means the schemas.md queries produce false-positive gate approvals — they cannot correctly determine reviewer-level majority approval. This isn't just a naming issue (covered by the pragmatic-verifier's C-3) — it's a fundamental logical incompatibility between the documented evidence gate architecture and the actual verification model.
- **Affected artifacts:** [schemas.md](NewAgents/.github/agents/schemas.md#L1040-L1060) ("Key Evidence Gate Queries" section), [sql-templates.md §6](NewAgents/.github/agents/sql-templates.md#L410-L470) (EG-5/EG-6 canonical evidence gates)
- **Recommendation:** Remove the evidence gate queries from schemas.md entirely (per ARCH-2 recommendation to extract non-schema content). If they must remain for reference, replace them with a pointer: "See sql-templates.md §6 (EG-5, EG-6) for all evidence gate queries. Evidence gate logic uses instance-based grouping and per-category check_name matching — do not use LIKE-based counting." At minimum, update the queries to match the sql-templates.md §6 logic to prevent incorrect usage.
- **Evidence:** schemas.md evidence gate query: `WHERE check_name LIKE 'review-design-%' AND verdict = 'approve'` with threshold `≥ 2`. sql-templates.md §6 EG-5: `WHERE check_name IN ('review-{scope}-security', 'review-{scope}-architecture', 'review-{scope}-correctness') AND round = {round}` with `GROUP BY instance HAVING COUNT(CASE WHEN verdict='approve' THEN 1 END) = 3` (all 3 categories must approve per reviewer, then count approving reviewers ≥ 2). These are fundamentally different verification models — count-of-approvals vs. reviewer-level-unanimity checking.

---

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding CORR-1: schemas.md DDL Has Column-Level Conflicts That Would Cause Runtime Failures

- **Severity:** Major
- **Category:** correctness
- **Description:** The schemas.md "Full Step 0 Initialization Script" DDL has 5 specific column-level conflicts with the canonical sql-templates.md §1 DDL. If any agent or manual process executes the schemas.md DDL instead of sql-templates.md §1, the resulting database schema would be incompatible with agent INSERT statements. Specifically: (1) The `instance` column is missing from schemas.md's `anvil_checks` DDL — any INSERT setting `instance` (required by adversarial-reviewer per review-perspectives.md §8) would fail with "table anvil_checks has no column named instance." (2) `task_id TEXT NOT NULL` in schemas.md vs `TEXT` (nullable) in sql-templates.md — orchestrator-level INSERTs that leave task_id NULL would violate the NOT NULL constraint. (3) `ts DATETIME DEFAULT CURRENT_TIMESTAMP` vs `ts TEXT NOT NULL DEFAULT (datetime('now'))` — different type affinity and nullability. (4) `pipeline_telemetry.status` CHECK includes `'running'` in schemas.md but not in sql-templates.md — a `status='running'` INSERT would succeed against schemas.md DDL but fail against sql-templates.md DDL. (5) `round INTEGER NOT NULL DEFAULT 1` vs `round INTEGER DEFAULT 1` — nullability difference.
- **Affected artifacts:** [schemas.md lines 1362–1399](NewAgents/.github/agents/schemas.md#L1362-L1399) (competing DDL), [sql-templates.md §1](NewAgents/.github/agents/sql-templates.md#L75-L135) (canonical DDL)
- **Recommendation:** The immediate fix is the same as ARCH-1: remove the DDL from schemas.md. As a correctness concern, the priority is preventing agents from executing the wrong DDL. If removal is deferred, at minimum add a prominent warning: "⚠️ This DDL is an illustrative reference only. Execute sql-templates.md §1 for the canonical schema."
- **Evidence:** Missing `instance` column: sql-templates.md §1 line ~90: `instance TEXT,` — present. schemas.md line ~1367: no `instance` column in CREATE TABLE. Nullability conflict: sql-templates.md: `task_id TEXT,` (nullable) vs schemas.md: `task_id TEXT NOT NULL,`. Status enum: sql-templates.md: `CHECK (status IN ('DONE','NEEDS_REVISION','ERROR','TIMEOUT'))` vs schemas.md: `CHECK(status IN ('DONE', 'NEEDS_REVISION', 'ERROR', 'TIMEOUT', 'running'))`. The `instance` column omission is the most critical — it would cause all review INSERT statements to fail.

### Finding CORR-2: Systemic Cross-Reference Drift Pattern — 3 Separate Document References Disagree on Evidence Gates

- **Severity:** Minor
- **Category:** correctness
- **Description:** Three independent documents reference the same concern (evidence gate queries for review verdicts) and all three disagree: (1) The orchestrator's purpose section references "sql-templates.md §5–§6" but §5 is instruction_updates (not evidence gates). (2) schemas.md contains LIKE-based evidence gate queries with a count-threshold model. (3) sql-templates.md §6 contains IN-based evidence gate queries with instance-grouping. This pattern of triple-reference drift suggests the cross-reference architecture lacks a maintenance mechanism — when the canonical source (sql-templates.md §6) is updated, the secondary references (schemas.md, orchestrator) are not systematically updated. While each individual drift was independently confirmed by other reviewers (pragmatic-verifier A-1, security-sentinel CORR-1), the systemic pattern is the architectural concern.
- **Affected artifacts:** [orchestrator.agent.md line 28](NewAgents/.github/agents/orchestrator.agent.md#L28) (§5–§6 reference), [schemas.md](NewAgents/.github/agents/schemas.md#L1040-L1060) (LIKE-based queries), [sql-templates.md §6](NewAgents/.github/agents/sql-templates.md#L410-L470) (canonical queries)
- **Recommendation:** Adopt a "single canonical reference" discipline: secondary documents should contain only pointers (e.g., "See sql-templates.md §6") rather than duplicating or paraphrasing queries. Add a "Referenced By" comment header to sql-templates.md §6 listing all documents that reference it, to facilitate coordinated updates. This would prevent the triple-drift pattern from recurring.
- **Evidence:** Orchestrator line 28: "sql-templates.md §5–§6" (§5 is wrong target). schemas.md evidence gates: `WHERE check_name LIKE 'review-design-%' AND verdict = 'approve'` (wrong logic). sql-templates.md §6 EG-5: `WHERE check_name IN ('review-{scope}-security', ...) ... GROUP BY instance HAVING COUNT(...) = 3` (correct logic). Three documents, three different answers for the same concern.

---

## Overall Verdict

**Verdict:** needs_revision
**Confidence:** High

## Summary

The 3-tier content architecture (Tier 1 auto-loaded, Tier 2 shared docs, Tier 3 agent-inline) is well-designed and the DRY extraction across 9 agents is effective. All agents are within line budgets (orchestrator: 316/550, all others: 145–253/350). However, schemas.md has grown beyond its declared scope into a 1528-line monolithic document containing DDL that conflicts with the canonical sql-templates.md §1, evidence gate queries using logic incompatible with the 3-INSERT-per-reviewer pattern, and pipeline-governance content that belongs in other reference documents. The DDL conflicts include a missing `instance` column that would break all review INSERT statements if the wrong DDL is executed. Recommend extracting non-schema content from schemas.md and removing duplicate DDL to restore single-source-of-truth discipline.
