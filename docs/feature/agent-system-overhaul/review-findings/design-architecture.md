# Adversarial Review: design — claude-opus-4.6

## Review Metadata

- **Review Focus:** architecture
- **Review Perspective:** Architecture Purist
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-02-27T10:00:00Z
- **Task ID:** agent-system-overhaul-design-review

## Overall Verdict

**Verdict:** needs_revision
**Confidence:** High

The design is well-structured and the 3-tier content architecture is sound. However, three major issues must be addressed: the Knowledge Agent has accumulated too many responsibilities (approaching a god-object pattern), there is no documented evolution strategy for the 4 SQLite table schemas, and the orchestrator's target line count reduction contains a 124-line unsubstantiated gap that puts NFR-1 compliance at risk.

---

## Findings

### Finding 1: Knowledge Agent Responsibility Bloat

- **Severity:** Major
- **Category:** architecture
- **Description:** The Knowledge Agent is accumulating an excessive number of distinct responsibilities, violating the single-responsibility principle. Currently it handles: (1) knowledge capture via store_memory, (2) architectural decision log maintenance (decisions.yaml), (3) evidence bundle assembly (evidence-bundle.md), (4) cross-session knowledge persistence. The design adds: (5) SQL-based pipeline telemetry queries and summarization, (6) artifact evaluation consumption and quality analysis, (7) instruction_updates INSERT for governed update tracking, (8) enhanced post-mortem analysis with SQL aggregation, (9) instruction file updates via governed mechanism. That is 9 distinct functional areas in a single agent. The Knowledge Agent's target of 345 lines also makes it the 2nd largest agent (after orchestrator at 480), suggesting the content density is high for an agent that should be focused.
- **Affected artifacts:** `docs/feature/agent-system-overhaul/design.md` §6.9, `docs/feature/agent-system-overhaul/design-output.yaml` → `agent_designs[knowledge-agent]`
- **Recommendation:** Split the Knowledge Agent's responsibilities into two clear phases within its workflow, or explicitly document the cohesion argument: all responsibilities serve the common purpose of "pipeline learning and observability." At minimum, add a responsibility inventory table to the Knowledge Agent design section that maps each responsibility to its justification and lists which could be deferred to Phase 2. Consider whether instruction update tracking (responsibility 7/9) should be a separate concern from post-mortem analysis (responsibility 8) — these serve different stakeholders (future pipeline runs vs. current run audit).
- **Evidence:** The `agent_designs` entry for knowledge-agent lists 6 items under `new_content` and 4 under `extracted_content`. The `key_changes` section lists 5 distinct capability enhancements. No other agent receives more than 3 new capabilities.

---

### Finding 2: SQLite Schema Evolution Strategy Absent

- **Severity:** Major
- **Category:** architecture
- **Description:** The design documents a clear schema evolution strategy for YAML schemas in schemas.md (additive changes = minor bump, breaking changes = major bump with coordinated consumer updates, documented in the Schema Evolution Strategy section at line ~1247 of schemas.md). However, no equivalent evolution strategy exists for the 4 SQLite table schemas (anvil_checks, pipeline_telemetry, artifact_evaluations, instruction_updates). These tables have CHECK constraints on enum columns (e.g., `phase IN ('baseline','after','review')`, `status IN ('DONE','NEEDS_REVISION','ERROR','TIMEOUT')`). Adding a new phase value, a new status value, or a new column requires ALTER TABLE operations. The design does not document: (a) how to add columns to existing tables, (b) how to extend CHECK constraint enums, (c) whether DDL changes require version bumping, (d) which agent is responsible for schema migrations. SQLite's ALTER TABLE is limited — you can add columns but cannot modify constraints without recreating the table. The design's `CREATE TABLE IF NOT EXISTS` pattern means once a table exists, it is never updated by re-running init.
- **Affected artifacts:** `docs/feature/agent-system-overhaul/design.md` §8 (SQLite Schema Design), `docs/feature/agent-system-overhaul/design-output.yaml` → `sqlite_schemas`
- **Recommendation:** Add a "SQLite Schema Evolution Strategy" section to the design (parallel to the YAML schema evolution strategy in schemas.md). Document: (1) Additive changes: `ALTER TABLE ADD COLUMN` is safe and backward-compatible — existing queries ignore unknown columns. (2) Enum expansion: CHECK constraints must be dropped/re-added (requires table recreation in SQLite) — document the migration DDL pattern. (3) Version tracking: consider a `schema_meta` table with `(table_name, version, migrated_at)` to track which DDL version each table is at. (4) Migration ownership: the orchestrator's Step 0 init should check schema versions and apply migrations. This section should be included in sql-templates.md as a new §9.
- **Evidence:** schemas.md lines 1247–1262 document YAML evolution but no SQLite DDL has equivalent coverage. The `CREATE TABLE IF NOT EXISTS` pattern in sql-templates.md §1 is creation-only — no migration path exists. The `artifact_evaluations` table is brand new and will inevitably need schema changes as evaluation criteria evolve.

---

### Finding 3: Orchestrator Line Count Target Not Substantiated

- **Severity:** Major
- **Category:** architecture
- **Description:** The design calculates the orchestrator's size reduction as: 800 − (25+15+70+50+20+20+20+20) + (10+5+10+5+4+10) = 800 − 240 + 44 = **604 lines**. It then states "further compaction of verbose step descriptions → ~480 lines" without itemizing what those 124 lines of compaction consist of. This is a 20% gap between the calculated result and the claimed target. If the compaction fails to achieve 480, the orchestrator remains at 600+ lines, violating AC-10.1 ("Orchestrator does not exceed 500 lines") and NFR-1. The entire 3-tier extraction architecture was designed to reduce agent file sizes, so the largest agent failing its target undermines the primary design goal. The current orchestrator is 797 lines (verified via `wc -l`), and the design already claims 800 — a minor discrepancy that compounds the uncertainty.
- **Affected artifacts:** `docs/feature/agent-system-overhaul/design.md` §6.1 (Orchestrator design), `docs/feature/agent-system-overhaul/design-output.yaml` → `agent_designs[orchestrator]`
- **Recommendation:** Itemize the 124-line "compaction" with specific sections and expected savings. For example: "Steps 1–9 descriptions: currently ~25 lines each = 225 lines → compact to ~12 lines each = 108 lines, saving 117 lines." If the savings cannot be itemized to reach ≤500 lines, either (a) raise the orchestrator target to 550–600 with an explicit NFR-1 exception justified by coordination complexity, or (b) identify additional content for extraction to Tier 2. An honest 600-line orchestrator with documented justification is architecturally preferable to an aspirational 480-line target that may not be achieved.
- **Evidence:** Design.md §6.1 shows the arithmetic explicitly: "800 − 240 + 44 = **604** → further compaction of verbose step descriptions → **~480 lines**". The 124-line gap is 20% of the target (480) and 15% of the calculated result (604).

---

### Finding 4: Dual Evidence Representation for Review Verdicts

- **Severity:** Minor
- **Category:** architecture
- **Description:** Each adversarial reviewer produces review evidence in two separate stores with potential for drift: (1) 3 SQL INSERTs into anvil_checks (one per category: review-security, review-architecture, review-correctness), each with its own `verdict` and `severity`; (2) 1 YAML verdict file with `category_verdicts` containing per-category `verdict` and `severity`. These two representations of the same data are written independently by the reviewer agent with no reconciliation mechanism. The orchestrator reads SQL for evidence gating, while the Designer reads YAML for revision context. If a reviewer writes "approve" in the security SQL INSERT but "needs_revision" in the YAML's `category_verdicts.security.verdict`, consumers see contradictory signals. The design does not define a source-of-truth hierarchy between the two representations.
- **Affected artifacts:** `docs/feature/agent-system-overhaul/design.md` §6.8 (Adversarial Reviewer), §7.2 (All-Category Coverage), `docs/feature/agent-system-overhaul/design-output.yaml` → `agent_designs[adversarial-reviewer]`
- **Recommendation:** Define an explicit source-of-truth hierarchy: "SQL INSERTs into anvil_checks are the authoritative evidence for pipeline routing decisions. YAML category_verdicts are derived summaries for human consumption and revision context. If discrepancy exists, SQL prevails." Add this statement to the adversarial reviewer's self-verification checklist: "Verify that YAML category_verdicts match the corresponding SQL INSERTs before returning." This is low-cost and prevents silent drift.
- **Evidence:** Design.md §6.8 shows the updated verdict YAML structure with `category_verdicts.security.verdict`, `category_verdicts.architecture.verdict`, `category_verdicts.correctness.verdict`. Separately, §7.2 shows SQL evidence gate EG-4 checking `check_name IN ('review-security','review-architecture','review-correctness')`. These are two independent write paths with no consistency check.

---

### Finding 5: Mutable Tier 1 Instruction Files Create Hidden Global Coupling

- **Severity:** Minor
- **Category:** architecture
- **Description:** The 3-tier content architecture defines Tier 1 as globally auto-loaded by VS Code (`.github/copilot-instructions.md` and `.github/instructions/pipeline-conventions.md`). The design also designates `.github/instructions/pipeline-conventions.md` as a "Knowledge Agent governed update target" (file inventory item #2, shared_documents → pipeline-conventions.md). This means the Knowledge Agent can modify content that is automatically loaded into every agent's context in future pipeline runs. While the governed update mechanism includes safety filters (§9 of global-operating-rules.md: "Safety filter rejects weakening"), the architectural boundary between "immutable global rules" and "agent-updatable content" is blurred. A Knowledge Agent update to pipeline-conventions.md could silently change behavior for all 9 agents in the next pipeline run.
- **Affected artifacts:** `docs/feature/agent-system-overhaul/design.md` §5.8 (pipeline-conventions.md), §5.1 (copilot-instructions.md), `docs/feature/agent-system-overhaul/design-output.yaml` → `shared_documents[pipeline-conventions.md]`
- **Recommendation:** Establish a clear mutability classification for Tier 1 files: (a) `.github/copilot-instructions.md` is **immutable** — only modifiable via feature implementation, never by any agent at runtime. (b) `.github/instructions/pipeline-conventions.md` is **governed-mutable** — updatable by Knowledge Agent with safety filters, changes logged to instruction_updates table. Document this classification in the Tier 1 section of the design. Ensure the Knowledge Agent's governed update mechanism explicitly excludes `copilot-instructions.md` from its update targets.
- **Evidence:** Design.md §5.1 describes copilot-instructions.md as "Tier 1, auto-loaded." Design.md §5.8 describes pipeline-conventions.md as "initial content created at implementation, updated by Knowledge Agent's governed mechanism." The file inventory confirms pipeline-conventions.md at `.github/instructions/pipeline-conventions.md` with `addresses_frs: ["FR-15"]`. Global-operating-rules.md §9 describes the governed update mechanism but doesn't list which Tier 1 files are off-limits.

---

### Finding 6: Single Database File as Single Point of Failure

- **Severity:** Minor
- **Category:** architecture
- **Description:** Decision D-1 consolidates all 4 tables into a single `verification-ledger.db`. This creates a single point of failure where DB corruption (interrupted WAL checkpoint, disk full, file system error) causes total evidence loss across all domains: verification evidence (anvil_checks), execution telemetry (pipeline_telemetry), quality feedback (artifact_evaluations), and audit trail (instruction_updates). The previous architecture at least isolated verification evidence from telemetry in separate DB files. Not all tables have equal criticality: anvil_checks is mission-critical (pipeline gates depend on it), while artifact_evaluations is informational (post-mortem degrades gracefully without it).
- **Affected artifacts:** `docs/feature/agent-system-overhaul/design.md` §8.1 (Database Consolidation), `docs/feature/agent-system-overhaul/design-output.yaml` → `decisions[D-1]`
- **Recommendation:** Document data criticality tiers within the consolidated DB: (1) **Mission-critical**: anvil_checks — pipeline cannot proceed without valid evidence records. (2) **Important**: pipeline_telemetry — post-mortem analysis degrades without timing data. (3) **Informational**: artifact_evaluations, instruction_updates — quality analysis degrades but pipeline function is unaffected. Add a note in the failure/recovery section that DB corruption recovery prioritizes anvil_checks reconstruction (from agent output files) over telemetry or evaluation data. Also consider adding a periodic `PRAGMA integrity_check` in the orchestrator's Step 0 init for existing DB files.
- **Evidence:** Design.md §8.1 states "Single PRAGMA configuration, single Step 0 initialization, single DB file for Knowledge Agent queries." The consolidation rationale is convenience-focused. D-1 risk is rated 🟡 with mitigation "WAL + busy_timeout," which addresses concurrency but not corruption/data-loss scenarios.

---

### Finding 7: Tool Scope Restrictions Are Prompt-Only Enforcement

- **Severity:** Minor
- **Category:** architecture
- **Description:** The tool-access-matrix.md defines precise scope restrictions: verifier's create_file is scoped to `verification-reports/*.yaml`, Knowledge Agent's run_in_terminal is scoped to SELECT-only, Researcher's create_file is scoped to `research/*.yaml`. These restrictions exist solely as text instructions within agent prompts. There is no runtime enforcement mechanism — VS Code's tool system does not support parameterized tool access. An LLM agent instructed to use `create_file` only for `verification-reports/*.yaml` could still call `create_file` for any path if it hallucinates or is confused by complex context. This is a known platform limitation, not a design flaw, but the design should acknowledge it explicitly and document the risk profile.
- **Affected artifacts:** `docs/feature/agent-system-overhaul/design.md` §10 (Tool Access Matrix), `docs/feature/agent-system-overhaul/design-output.yaml` → `architecture.tool_access_matrix`
- **Recommendation:** Add a paragraph to the tool-access-matrix.md design acknowledging: "All scope restrictions are enforced via prompt instructions. VS Code's agent runtime does not support parameterized tool access policies. Compliance depends on LLM instruction-following capability. The self-verification checklist in each agent should include a check that tool usage remained within scope. The verifier agent (which reviews other agents' outputs) serves as a secondary enforcement point for file path violations." This honest acknowledgment helps implementers understand the security model's limitations.
- **Evidence:** The tool access matrix shows 6 scoped entries (footnotes ²–⁶ in design.md §10). None of these scopes can be enforced by the VS Code runtime. The design's §11 (Security Considerations) discusses tool scope restrictions at §11.3 but does not mention the prompt-only enforcement limitation.

---

## Summary

The design is architecturally sound with a well-defined 3-tier content strategy, clean data flow graph (acyclic), bounded feedback loops, and a reasonable decomposition of responsibilities across 9 agents. No blockers were found — the design is implementable. Three major issues require attention: the Knowledge Agent has accumulated 9 distinct responsibilities (trending toward god-object), the 4-table SQLite schema has no documented evolution strategy (unlike the YAML schemas which have explicit version management), and the orchestrator's claimed ~480-line target has a 124-line unsubstantiated gap in its reduction arithmetic. Four minor issues provide additional architectural hygiene recommendations. All findings have specific, actionable recommendations.
