# Adversarial Review: code — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-02-27T10:00:00Z

---

## Security Analysis

**Category Verdict:** approve

### Finding S-1: Tool Access Restrictions Are Honor-System With No Runtime Enforcement

- **Severity:** Minor
- **Description:** All tool scope restrictions (e.g., Verifier's `create_file` limited to `verification-reports/*.yaml`, Knowledge Agent's `run_in_terminal` limited to SELECT + instruction_updates INSERT) depend entirely on LLM instruction-following. VS Code's agent runtime does not support parameterized tool access policies. An LLM that ignores its instructions could call any tool with any arguments.
- **Affected artifacts:** [tool-access-matrix.md](../../NewAgents/.github/agents/tool-access-matrix.md#L85) §11 (Enforcement Note)
- **Recommendation:** No action required — this is an acknowledged and accepted risk (residual: Medium). The multi-layer mitigations (agent prompts, self-verification checklists, regex patterns, verifier secondary enforcement) are the best available approach given platform constraints. Document this explicitly in the evidence bundle as a known limitation.
- **Evidence:** tool-access-matrix.md §11 states: "Scope restrictions are enforced via agent instructions. VS Code's agent runtime does not support parameterized tool access policies. Compliance depends on LLM instruction-following." The verification ledger confirms `readiness-secrets` checks pass for all agents.

---

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Orchestrator References sql-templates.md "§5–§6" for Evidence Gates But §5 Is Wrong Target

- **Severity:** Minor
- **Description:** The orchestrator.agent.md top-level responsibility list says "evidence gate verification — independently verify evidence via `run_in_terminal` SQL queries on `verification-ledger.db` using templates from sql-templates.md §5–§6". However, sql-templates.md §5 is the `instruction_updates INSERT Template` (Knowledge Agent only), not an evidence gate query. The actual evidence gate queries are in §6 only. All **specific** gate references later in the orchestrator (Steps 3b, 7, Evidence Gate Logic section) correctly cite "§6" alone.
- **Affected artifacts:** [orchestrator.agent.md](../../NewAgents/.github/agents/orchestrator.agent.md#L28) line 28 (responsibility #4)
- **Recommendation:** Change "sql-templates.md §5–§6" to "sql-templates.md §6" in the orchestrator's responsibility summary. Low impact since all specific references are correct.
- **Evidence:** sql-templates.md §5 heading is "instruction_updates INSERT Template — Used by: Knowledge Agent only." Evidence gates are exclusively in §6 ("Evidence Gate Queries"). The orchestrator's Step 3b and Step 7 sections both correctly reference "sql-templates.md §6".

### Finding A-2: No Graceful Degradation for Missing Tier 2 Shared Reference Docs

- **Severity:** Minor
- **Description:** All 9 agents reference Tier 2 shared docs (global-operating-rules.md, sql-templates.md, tool-access-matrix.md, etc.) via markdown links. But no agent specifies fallback behavior if a referenced shared doc is missing or unreadable. Edge case EC-9 in spec-output.yaml covers verdict file path mismatches, but there's no equivalent edge case for missing shared reference documents. If any Tier 2 doc is accidentally deleted, agents would fail with no clear recovery path.
- **Affected artifacts:** All 9 agent files; global-operating-rules.md (no missing-doc recovery section)
- **Recommendation:** Add a brief note to global-operating-rules.md §2 (Error Categories) classifying "referenced shared doc not found" as a deterministic error that agents should report (not retry). Alternatively, add a bullet to §6 (Self-Verification): "Verify all referenced shared docs exist before proceeding." Low priority since shared docs are committed alongside agent files and should always be present.
- **Evidence:** No agent file contains fallback text like "if global-operating-rules.md is not found..." or similar. global-operating-rules.md §2 Error Categories table lists "Missing required input file" as Deterministic but doesn't explicitly mention shared reference docs vs. pipeline data files.

---

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: schemas.md Schema 9 Uses Outdated Field Names Incompatible With Implemented Agents

- **Severity:** Critical
- **Description:** schemas.md Schema 9 (`review-findings`) still defines the **old** single-focus, single-model schema with fields `reviewer_model` (string, e.g., `"gpt-5.3-codex"`) and `review_focus` (string, e.g., `"security"`), plus a single `verdict` field. However, the **implemented** adversarial-reviewer.agent.md uses a fundamentally different schema: `reviewer_perspective` (e.g., `"security-sentinel"`), per-category `verdicts` object (`security`, `architecture`, `correctness`), and an `overall` verdict field. Any downstream agent or tool that reads Schema 9 as defined in schemas.md would produce or expect outputs incompatible with what the adversarial reviewer actually generates. This is the canonical schema definition — it must match the implementation.
- **Affected artifacts:** [schemas.md](../../NewAgents/.github/agents/schemas.md#L777-L830) lines 777–830 (Schema 9 Fields table and Example)
- **Recommendation:** Update Schema 9 fields table to match adversarial-reviewer.agent.md's YAML Verdict Summary Structure: replace `reviewer_model` with `reviewer_perspective`, replace `review_focus` with removal (no longer a field), replace single `verdict` with `verdicts` object containing `security`/`architecture`/`correctness` sub-verdicts plus `overall`. Update the example YAML block at line 820 to show the new structure.
- **Evidence:** schemas.md line 779: `| reviewer_model | string | Yes | Model identifier (e.g., gpt-5.3-codex...) |`. adversarial-reviewer.agent.md YAML Verdict Summary Structure shows: `reviewer_perspective: "<perspective>"`, `verdicts: {security: ..., architecture: ..., correctness: ...}`, `overall: "<verdict>"`. These schemas are incompatible — different field names, different structure, different semantics.

### Finding C-2: schemas.md Schema 9 Perspective Names Don't Match Actual Perspective IDs

- **Severity:** Major
- **Description:** schemas.md Schema 9 Verdict File Naming Convention (line 772) uses perspective names `architecture-purist` and `correctness-checker`, with example `review-verdicts/code-correctness-checker.yaml`. But the actual perspective IDs defined in review-perspectives.md and used throughout all agent files are `architecture-guardian` and `pragmatic-verifier`. This would cause file path mismatches — a consumer looking for `code-correctness-checker.yaml` would not find `code-pragmatic-verifier.yaml`.
- **Affected artifacts:** [schemas.md](../../NewAgents/.github/agents/schemas.md#L771-L772) lines 771–772 (Schema 9, Verdict File Naming Convention)
- **Recommendation:** Replace `architecture-purist` with `architecture-guardian` and `correctness-checker` with `pragmatic-verifier` in the naming convention and examples. Update the example path from `review-verdicts/code-correctness-checker.yaml` to `review-verdicts/code-pragmatic-verifier.yaml`.
- **Evidence:** review-perspectives.md §3 defines ID as `architecture-guardian`, §4 as `pragmatic-verifier`. orchestrator.agent.md Step 3b table lists `architecture-guardian` and `pragmatic-verifier`. dispatch-patterns.md confirms same names. schemas.md line 772 says `"architecture-purist"` and `"correctness-checker"` — these IDs exist nowhere else in the codebase.

### Finding C-3: schemas.md Schema 9 SQL check_name Convention Contradicts Evidence Gate Queries

- **Severity:** Major
- **Description:** schemas.md Schema 9 SQL INSERT Convention defines `check_name = 'review-{scope}-{perspective}'` (e.g., `'review-design-security-sentinel'`), where each reviewer produces **one** record with their perspective as the check_name suffix. However, the actual pattern implemented in sql-templates.md §6 (EG-5), review-perspectives.md §8, and adversarial-reviewer.agent.md is: each reviewer produces **3** records with `check_name = 'review-{scope}-{category}'` (e.g., `review-code-security`, `review-code-architecture`, `review-code-correctness`) and stores the perspective in the `instance` field. The EG-5 query explicitly filters on `check_name IN ('review-{scope}-security', 'review-{scope}-architecture', 'review-{scope}-correctness')`. If a reviewer followed the schemas.md convention, no EG-5 results would match.
- **Affected artifacts:** [schemas.md](../../NewAgents/.github/agents/schemas.md#L808) line 808 (Schema 9, SQL INSERT Convention, check_name row)
- **Recommendation:** Update the `check_name` row in Schema 9's SQL INSERT Convention table to `'review-{scope}-{category}'` where category is `security|architecture|correctness`. Add a note explaining that each reviewer INSERTs 3 rows (one per category) with `instance` set to the perspective ID. Update the example from `'review-design-security-sentinel'` to the 3-row pattern shown in review-perspectives.md §8.
- **Evidence:** sql-templates.md §6 EG-5 query uses `check_name IN ('review-{scope}-security', 'review-{scope}-architecture', 'review-{scope}-correctness')`. review-perspectives.md §8 shows 3 INSERT statements per reviewer with check_names `review-{scope}-security`, `review-{scope}-architecture`, `review-{scope}-correctness` and `instance='{perspective_id}'`. adversarial-reviewer.agent.md SQL Output section lists these same 3 check_names. schemas.md line 808 says `'review-{scope}-{perspective}'` — incompatible with all three sources.

### Finding C-4: Routing Matrix Shows Designer NEEDS_REVISION Support But Designer Never Returns It

- **Severity:** Major
- **Description:** The Completion Contract Routing Matrix in schemas.md shows Designer with an entry under the `NEEDS_REVISION` column: "→ Revision (from review feedback)". This implies the Designer returns `NEEDS_REVISION` as a completion status. However, the designer.agent.md's Completion Contract explicitly states: "The Designer agent NEVER returns `NEEDS_REVISION` — that determination is made by the Adversarial Reviewers in Step 3b and routed by the Orchestrator." The routing matrix's own note confirms: "Only Adversarial Reviewer and Verifier support `NEEDS_REVISION`." The Designer is re-dispatched by the orchestrator in revision mode when reviewers find issues — but the Designer itself never returns NEEDS_REVISION.
- **Affected artifacts:** [schemas.md](../../NewAgents/.github/agents/schemas.md#L1420) line ~1420 (Routing Matrix, Designer row, NEEDS_REVISION column)
- **Recommendation:** Change the Designer's `NEEDS_REVISION` cell to `N/A (not returned)` consistent with Researcher, Spec, Planner, Implementer, and Knowledge Agent rows. The orchestrator's revision routing for the Designer is triggered by the Adversarial Reviewer's verdict, not by the Designer's own completion status.
- **Evidence:** designer.agent.md Completion Contract: "The Designer agent NEVER returns `NEEDS_REVISION`". schemas.md routing matrix note: "Only Adversarial Reviewer and Verifier support `NEEDS_REVISION`". But the table row for Designer has a non-N/A entry: "→ Revision (from review feedback)".

### Finding C-5: pipeline-conventions.md Uses Abbreviated Perspective Names in Examples

- **Severity:** Minor
- **Description:** The pipeline-conventions.md file naming section shows examples `design-security.yaml` and `code-architecture.yaml` for review verdicts and findings. But the actual file naming convention uses full perspective IDs: `design-security-sentinel.yaml`, `code-architecture-guardian.yaml`. The abbreviated examples could mislead consumers into looking for files at wrong paths.
- **Affected artifacts:** [pipeline-conventions.md](../../.github/instructions/pipeline-conventions.md#L14-L15) lines 14–15
- **Recommendation:** Update the example file names to use full perspective IDs: `design-security-sentinel.yaml`, `code-architecture-guardian.md`, etc. Align with the convention documented in adversarial-reviewer.agent.md and schemas.md.
- **Evidence:** pipeline-conventions.md line 14: "Review verdicts: `<scope>-<perspective>.yaml` (e.g., `design-security.yaml`, `code-architecture.yaml`)". adversarial-reviewer.agent.md output files: `review-verdicts/<scope>-<perspective>.yaml` where perspective is `security-sentinel`, `architecture-guardian`, or `pragmatic-verifier` — not `security` or `architecture` alone.

### Finding C-6: copilot-instructions.md SQL Safety Example Uses echo Instead of printf

- **Severity:** Minor
- **Description:** The copilot-instructions.md § SQL Safety section shows `echo "INSERT INTO table ..." | sqlite3 database.db` as the recommended pattern. However, sql-templates.md §0 (the authoritative SQL safety reference) documents `printf '%s' "..." | sqlite3` as the **preferred** pattern because it avoids shell-specific echo behavior differences. The copilot-instructions.md is auto-loaded by VS Code for all agents, so this simpler-but-less-safe pattern has wider reach.
- **Affected artifacts:** [copilot-instructions.md](../../.github/copilot-instructions.md#L30-L32) lines ~30–32 (SQL Safety section)
- **Recommendation:** Either update copilot-instructions.md to use `printf '%s'` (matching sql-templates.md §0) or add a note referencing sql-templates.md §0 for the detailed pattern. Since copilot-instructions.md is Tier 1 (auto-loaded), it should at minimum not contradict the authoritative Tier 2 reference. Low impact since the PowerShell `echo` is safe on the target Windows platform.
- **Evidence:** copilot-instructions.md line ~31: `echo "INSERT INTO table ..." | sqlite3 database.db`. sql-templates.md §0 "Preferred pattern — printf pipe": `printf '%s' "INSERT INTO table VALUES ('escaped_value');" | sqlite3 verification-ledger.db`. The sql-templates.md explicitly explains: "Why stdin piping? When SQL is passed as a shell argument... shell metacharacters... are interpreted by the shell."

---

## Summary

The implementation is **substantially correct and well-structured** across 20 files (8 new reference docs + 12 modified agent definitions). The 3-tier content architecture is practical, DRY extraction is well-executed (9 agents reduced from ~3,930 to ~2,775 lines), and the perspective-based review system with all-category coverage addresses the core FR-2/FR-3 requirements. However, **schemas.md Schema 9 was not updated to reflect the new perspective-based review architecture** — it still contains the old `reviewer_model`/`review_focus` field definitions, incorrect perspective names, and an incompatible SQL `check_name` convention. These inconsistencies between the canonical schema reference and the actual implementation could cause interoperability failures if any agent or consumer relies on schemas.md as the source of truth. The routing matrix for the Designer is also inconsistent with the Designer's own completion contract. All 4 findings (1 Critical + 3 Major) are in schemas.md, suggesting a single-file update would resolve all correctness issues.
