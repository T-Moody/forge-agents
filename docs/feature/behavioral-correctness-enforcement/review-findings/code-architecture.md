# Adversarial Review: code — architecture

## Review Metadata

- **Review Focus:** architecture
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-01T12:00:00Z
- **Reviewer:** claude-opus-4.6

## Overall Verdict

**Verdict:** needs_revision
**Confidence:** High

Three Major findings and two Minor findings identified. The Major issues involve a cross-table contract violation in a newly added SQL query, an undocumented out-of-scope file modification that bypasses the pipeline's own verification mechanisms, and a schema evolution framework gap. None are security blockers, but they represent genuine architectural defects that should be addressed.

---

## Findings

### Finding A-1: EG-7 Gap-Detection Query References Non-Existent `pipeline_telemetry.task_id` Column

- **Severity:** Major
- **Category:** architecture
- **Description:** The EG-7 gap-detection companion query added by task-03 to `sql-templates.md` §6 joins `pipeline_telemetry t` with `anvil_checks a` using `t.task_id`. However, the `pipeline_telemetry` table DDL (sql-templates.md §1, lines 124–137) has no `task_id` column. Its columns are: `id`, `run_id`, `step`, `agent`, `instance`, `started_at`, `completed_at`, `status`, `dispatch_count`, `retry_count`, `notes`, `ts`. Executing this query against the actual schema would produce a SQLite error: `no such column: t.task_id`. The `instance` column contains task-scoped identifiers (e.g., `implementer-task-03`), but there is no direct `task_id` column for join operations.
- **Affected artifacts:** `NewAgents/.github/agents/sql-templates.md` lines 531–541 (EG-7 gap-detection query)
- **Recommendation:** Either (a) add a `task_id TEXT` column to `pipeline_telemetry` DDL with a corresponding schema migration, or (b) rewrite the gap-detection query to extract task ID from the `instance` column using `SUBSTR()` + `INSTR()`, or (c) use `anvil_checks` self-join (checking for tasks with `phase='after'` but no `check_name='behavioral-coverage'`) which avoids the cross-table dependency entirely. Option (c) is preferred architecturally as it keeps the query within a single table contract.
- **Evidence:** `pipeline_telemetry` DDL at sql-templates.md lines 124–137 contains no `task_id` column. EG-7 gap-detection query at lines 531–541 references `t.task_id` three times. Additionally, `schemas.md` line 1044 (`task_id` Naming Convention) claims "`task_id` appears in `anvil_checks`, `pipeline_telemetry`..." which is also incorrect per the DDL — this is pre-existing but the new EG-7 query operationalizes this documentation error into a runtime failure.

---

### Finding A-2: Undocumented orchestrator.agent.md Modification Outside Task Scope

- **Severity:** Major
- **Category:** architecture
- **Description:** The staged diff includes a 2-line addition to `NewAgents/.github/agents/orchestrator.agent.md` (adding a "Parallel dispatch (CRITICAL)" note about `runSubagent` call batching). This change is NOT listed in any of the 8 task definitions, NOT claimed by any implementation report, NOT verified by any verification report, and NOT captured in `anvil_checks` evidence. It bypasses the pipeline's own task-scoped enforcement mechanisms (baseline capture, verification cascade, evidence gating). This is an agent boundary violation — the behavioral-correctness-enforcement feature's implementation scope was explicitly 8 files, and orchestrator.agent.md was not among them.
- **Affected artifacts:** `NewAgents/.github/agents/orchestrator.agent.md` (staged diff shows +2 lines at line 485)
- **Recommendation:** Either (a) revert the orchestrator change from the staged area (`git checkout HEAD -- NewAgents/.github/agents/orchestrator.agent.md`) since it was never planned, implemented, or verified through the pipeline, or (b) create a separate task covering this change with proper baseline capture and verification. Option (a) is preferred for this pipeline run — the change can be contributed through a proper task in a future iteration.
- **Evidence:** `git --no-pager diff --staged --stat` shows `orchestrator.agent.md | 2 +`. No entry in `implementation-reports/task-01.yaml` through `task-08.yaml` claims changes to orchestrator.agent.md. No entry in `verification-reports/task-01.yaml` through `task-08.yaml` verifies orchestrator.agent.md. The plan-output.yaml lists 8 tasks modifying 8 files — orchestrator.agent.md is not among them.

---

### Finding A-3: Schema Evolution Framework Gap — Per-Schema Versioning vs Common Header

- **Severity:** Major
- **Category:** architecture
- **Description:** Schema 6 in `schemas.md` is labeled "Version 2.0 (breaking change)" with a migration note stating "Existing string-format ACs are not forward-compatible." However, the common header's `schema_version: "1.0"` remains unchanged, and the Schema Evolution Strategy (sql-templates.md §9) covers only SQLite table evolution — no framework exists for YAML schema versioning. This creates an ambiguous state: agents emit `schema_version: "1.0"` in their outputs while producing Schema 6 at v2.0 format. Future consumers cannot distinguish a Schema 6 v1.0 output (string ACs) from a v2.0 output (object ACs) based on the common header alone. Additionally, `design-output.yaml` decision D-1 described this change as "Schema version bumps from 1.0 to 1.1 (additive)" — contradicting the implementation's honest "Version 2.0 (breaking change)" label. No deviation record was filed for this CR-1/D-1 discrepancy.
- **Affected artifacts:** `NewAgents/.github/agents/schemas.md` lines 458–459 (Version 2.0 label), line 14 (schema_version: "1.0" requirement), `NewAgents/.github/agents/sql-templates.md` §9 (Schema Evolution Strategy — SQLite only)
- **Recommendation:** (a) Document a YAML schema versioning convention alongside the SQLite evolution strategy — minimally, define how `schema_version` in the common header relates to per-schema version labels and when it should be bumped. (b) Either bump the common header to `schema_version: "2.0"` (since a breaking change occurred) or add a per-schema `format_version` field to Schema 6 outputs that consumers can check. (c) File a deviation record for the D-1/CR-1 discrepancy in the feature's `decisions.yaml`.
- **Evidence:** schemas.md line 14: `schema_version: "1.0"` is required for all outputs. schemas.md line 458: "Version 2.0 (breaking change)". design-output.yaml D-1 rationale: "Schema version bumps from 1.0 to 1.1 (additive)". sql-templates.md §9 (lines 642–736) covers only SQLite evolution — no YAML schema versioning framework.

---

### Finding A-4: Stale Evidence Gate Cross-Reference in schemas.md

- **Severity:** Minor
- **Category:** architecture
- **Description:** The Evidence Gate Queries cross-reference in schemas.md (line 1034) states "See sql-templates.md §6 for all canonical evidence gate queries (EG-1 through EG-6)." However, task-03 added EG-7 to sql-templates.md §6. The cross-reference is now stale and incomplete. This is a concrete instance of the cross-reference coupling risk inherent in the DDL deduplication strategy.
- **Affected artifacts:** `NewAgents/.github/agents/schemas.md` line 1034
- **Recommendation:** Update the reference to "EG-1 through EG-7" or remove the range qualifier entirely (e.g., "all canonical evidence gate queries") to avoid future staleness when additional gates are added.
- **Evidence:** schemas.md line 1034: "EG-1 through EG-6". sql-templates.md lines 497–543: EG-7 section exists under §6.

---

### Finding A-5: Cross-Reference Coupling Fragility Between schemas.md and sql-templates.md

- **Severity:** Minor
- **Category:** architecture
- **Description:** The DDL deduplication (task-01) replaced ~260 lines of inline DDL in schemas.md with cross-references to sql-templates.md using section-number notation (§1, §6). This creates a maintenance coupling: if sql-templates.md sections are reordered, renumbered, or split, schemas.md references silently break with no automated detection mechanism. Finding A-4 demonstrates this coupling risk materializing within the same pipeline run — EG-7 was added to §6 but the schemas.md cross-reference wasn't updated to reflect it. While deduplication was the correct architectural decision (consistent with CR-4 and FR-7), the coupling warrants awareness.
- **Affected artifacts:** `NewAgents/.github/agents/schemas.md` lines 1015, 1034, 1038 (cross-references to sql-templates.md)
- **Recommendation:** Consider adding a verifier self-check or reviewer checklist item that cross-references between schemas.md and sql-templates.md are consistent after any sql-templates.md modification. This would provide structural enforcement rather than relying on implementer diligence.
- **Evidence:** schemas.md cross-references: "sql-templates.md §1" (line 1015), "sql-templates.md §6" (line 1034), "sql-templates.md §1" (line 1038). Finding A-4 demonstrates a cross-reference becoming stale within the same run. No automated mechanism exists to detect such drift.

---

## Architectural Strengths Noted

1. **Line budget compliance:** All four agent files are within the 400-line NFR-2 budget — planner (343), implementer (377), verifier (330), adversarial-reviewer (345). schemas.md is 1219 lines, well under the 1300-line target.

2. **Structured AC scalability:** The `{id, text, test_method}` approach creates a clean linear dependency chain (spec → planner → implementer → verifier → reviewer) with clear ownership at each stage. The producer/consumer contracts are well-defined in schemas.md with field tables and examples.

3. **Tier 2 placement of behavioral checks:** The decision to place behavioral-coverage and runtime-wiring as unconditional Tier 2 checks (using always-available workspace tools) is architecturally sound. The explicit "Always-Run Behavioral Checks" sub-heading clearly distinguishes them from tool-gated Tier 2 checks while keeping them within the existing tier framework.

4. **Consistent check_name patterns:** The `behavioral-coverage` and `runtime-wiring` entries are consistently registered in schemas.md check_name Naming Patterns table, verifier.agent.md Quick Reference table, and the EG-7 query — the primary query (non-gap-detection) correctly targets the `anvil_checks` table.

5. **Edge case handling:** EC-1 (inspection-only), EC-2 (TDD fallback), and EC-3 (modification-only) are consistently documented across implementer, verifier, and schemas.md with aligned behavior.

## Summary

The behavioral-correctness-enforcement implementation is architecturally well-structured overall — the distributed enforcement pattern (planner→implementer→verifier→reviewer) with consistent schema contracts is sound. However, three Major issues require attention: (1) the EG-7 gap-detection query would fail at runtime due to a non-existent column reference, (2) an undocumented orchestrator.agent.md change bypasses the pipeline's own verification mechanisms, and (3) the YAML schema versioning framework has a gap where per-schema versions (v2.0) don't align with the common header (schema_version: "1.0"). Two Minor cross-reference issues (stale EG range, section-number coupling fragility) are low-impact but illustrate maintenance risks. Recommend fixing the three Major issues before merge.
