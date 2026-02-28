# Code Review: Security Sentinel (Round 1)

## Review Metadata

- **Perspective:** security-sentinel
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-02-27T10:00:00Z
- **Task ID:** agent-system-overhaul-code-review
- **Scope:** code
- **Verification Evidence:** 48 checks passed, 0 failed

---

## Security Analysis

**Category Verdict:** needs_revision

### Finding SEC-1: `sql_escape()` is unenforced pseudocode — relies on LLM consistent application

- **Severity:** Major
- **Description:** The `sql_escape()` function defined in `sql-templates.md` §0 is a conceptual specification (pseudocode), not an enforced function or macro. It describes a 3-step process (double single quotes, strip null bytes, truncate) that agents must manually implement within each SQL INSERT. Since agents are LLMs, there is no guarantee they will consistently apply `sql_escape()` every time they interpolate a free-text value. The pipeline has 5 SQL-writing agents (Orchestrator, Implementer, Verifier, Adversarial Reviewer, Knowledge Agent) each constructing SQL strings independently. A single omission exposes `anvil_checks`, `pipeline_telemetry`, `artifact_evaluations`, or `instruction_updates` to SQL injection via unescaped single quotes in values like `output_snippet`, `notes`, or `change_summary`.
- **Affected artifacts:** [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L15-L27) (§0 definition), all agent files referencing sql_escape
- **Recommendation:** (1) Add explicit pre-INSERT checklist items to each SQL-writing agent's self-verification section that require logging proof of escaping. (2) Consider wrapping the INSERT logic in a shell function or heredoc pattern that structurally prevents unescaped interpolation. (3) Add a verifier check that audits generated SQL for unescaped quotes in output_snippet fields.
- **Evidence:** `sql_escape()` defined at [sql-templates.md lines 15-27](NewAgents/.github/agents/sql-templates.md#L15-L27) as text instructions only. The self-verification checklist at [lines 67-73](NewAgents/.github/agents/sql-templates.md#L67-L73) includes "All free-text SQL values have single quotes escaped" but this is a checklist item, not enforcement. Each agent independently implements escaping per its instruction-following capability.

### Finding SEC-2: Double-quoted shell patterns don't prevent shell metacharacter expansion in piped SQL

- **Severity:** Major
- **Description:** `sql-templates.md` §0 claims stdin piping "bypasses shell interpretation of the SQL content." This is incorrect for the recommended patterns. Both `printf '%s' "INSERT INTO ... '${value}' ..."` and `echo "INSERT INTO ... '${value}' ..."` use double-quoted strings, which undergo shell expansion (variable substitution, command substitution via `$(...)` and backticks) BEFORE the content reaches sqlite3. Only the heredoc pattern with a quoted delimiter (`<<'EOSQL'`) truly bypasses shell interpretation. If an `output_snippet` value contains shell metacharacters (e.g., from captured build output like `error: $(function) undefined`), these could be interpreted as commands during the `echo/printf` shell execution. The current Windows/PowerShell environment partially mitigates this since PowerShell's `echo` (Write-Output) doesn't expand `$()`, but the documentation targets cross-platform use with explicit bash patterns.
- **Affected artifacts:** [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L30-L53) (§0 Shell Injection Prevention section), [line 36](NewAgents/.github/agents/sql-templates.md#L36) (printf example), [line 50](NewAgents/.github/agents/sql-templates.md#L50) (echo example)
- **Recommendation:** (1) Change the preferred printf pattern to use single quotes: `printf '%s' 'INSERT INTO table VALUES (''escaped_value'');'`. (2) Promote the heredoc pattern (`<<'EOSQL'`) as the primary recommended pattern for bash environments. (3) Correct the misleading claim that stdin piping bypasses shell interpretation — clarify that only single-quoted strings or quoted heredocs achieve this. (4) For PowerShell environments, document that `echo` is safe because PowerShell's Write-Output doesn't perform subshell expansion.
- **Evidence:** The printf pipe example at [line 36](NewAgents/.github/agents/sql-templates.md#L36) uses `printf '%s' "INSERT INTO table VALUES ('escaped_value');"` — the `%s` format specifier prevents printf-specific format interpretation but the double-quoted argument still undergoes shell variable/command expansion. The explanatory note at [lines 55-56](NewAgents/.github/agents/sql-templates.md#L55-L56) states "Stdin piping bypasses shell interpretation of the SQL content" which is inaccurate for double-quoted patterns.

### Finding SEC-3: `artifact_path` CHECK constraint allows embedded path traversal sequences

- **Severity:** Minor
- **Description:** The `artifact_path` CHECK constraint in `artifact_evaluations` (`artifact_path NOT LIKE '../%' AND artifact_path NOT LIKE '/%'`) only rejects paths that START with `../` or `/`. Paths containing embedded traversal sequences like `foo/../../sensitive/file` pass the CHECK. While this field stores opaque strings in SQLite (no filesystem access from the DB layer), the stored path could be consumed by downstream agents that use it for file reads.
- **Affected artifacts:** [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L150) (DDL definition), [schemas.md](NewAgents/.github/agents/schemas.md#L1155) (schema definition)
- **Recommendation:** Strengthen the CHECK to also reject embedded traversal: `artifact_path NOT LIKE '%../%'` (or `artifact_path NOT GLOB '*../*'`). This blocks `../` anywhere in the path, not just as a prefix.
- **Evidence:** DDL at [sql-templates.md line 150](NewAgents/.github/agents/sql-templates.md#L150): `artifact_path TEXT NOT NULL CHECK (artifact_path NOT LIKE '../%' AND artifact_path NOT LIKE '/%')`. The Knowledge Agent reads `artifact_path` values via SQL SELECT ([sql-templates.md §7 "Worst-Rated Artifacts"](NewAgents/.github/agents/sql-templates.md#L541)) and could use them in `read_file` calls.

### Finding SEC-4: Implementer revert command doesn't sanitize `file_path` against shell metacharacters

- **Severity:** Minor
- **Description:** The Implementer's revert mode constructs `git checkout <baseline_tag> -- <file_path>` commands directly from orchestrator-provided parameters without sanitization. While these values originate from pipeline-internal sources (Planner-generated task definitions), a maliciously crafted `file_path` value (e.g., `; rm -rf /` or `$(malicious)`) could be interpreted by the shell. The risk is low because the attack requires a compromised upstream agent, but defense-in-depth is missing.
- **Affected artifacts:** [implementer.agent.md](NewAgents/.github/agents/implementer.agent.md#L198) (revert mode step 1)
- **Recommendation:** (1) Quote file paths in the git checkout command: `git checkout <baseline_tag> -- "<file_path>"`. (2) Add a validation step to confirm `file_path` matches expected patterns (no shell metacharacters, matches workspace-relative path format).
- **Evidence:** [implementer.agent.md line 198](NewAgents/.github/agents/implementer.agent.md#L198): `git checkout <baseline_tag> -- <file_path>` — no quoting or validation of the `file_path` parameter.

---

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding ARCH-1: All tool access restrictions rely on instruction-based enforcement only — no programmatic guardrails

- **Severity:** Major
- **Description:** `tool-access-matrix.md` §11 explicitly documents that all scope restrictions are enforced via agent instructions (LLM prompt adherence) with no programmatic enforcement from VS Code's agent runtime. This means every security boundary between agents — including `run_in_terminal` scope restrictions, `create_file` path constraints, and `replace_string_in_file` prohibitions — can be violated if an agent's instruction-following fails. The four enforcement layers (prompt instructions, self-verification checklists, regex validation patterns, verifier as secondary enforcement) all depend on LLM behavior. A single agent misbehavior could bypass any tool restriction. While this is documented as "accepted risk" with "residual threat level: Medium," from a security sentinel perspective, the lack of any hard enforcement layer represents a systemic trust boundary weakness.
- **Affected artifacts:** [tool-access-matrix.md](NewAgents/.github/agents/tool-access-matrix.md#L79-L86) (§11 Enforcement Note)
- **Recommendation:** (1) Document the accepted risk more prominently — add a "Security Assumptions" section to `global-operating-rules.md` listing this as an explicit assumption. (2) Add post-pipeline audit checks (e.g., Knowledge Agent verifying that no agent created files outside its declared scope by comparing git diff against expected output paths per agent). (3) Consider adding the regex validation patterns from §2–§10 as a verifier check that audits `run_in_terminal` command history for each agent.
- **Evidence:** [tool-access-matrix.md §11](NewAgents/.github/agents/tool-access-matrix.md#L79-L86): "Scope restrictions are enforced via agent instructions. VS Code's agent runtime does not support parameterized tool access policies. Compliance depends on LLM instruction-following... This is an accepted risk (residual threat level: Medium)."

### Finding ARCH-2: Knowledge Agent has `replace_string_in_file` access which could target files beyond governed scope

- **Severity:** Minor
- **Description:** The Knowledge Agent is granted `replace_string_in_file` in tool-access-matrix.md §10. Its file boundary instructions restrict writes to specific output files, and the Safety Constraint Filter (global-operating-rules.md §9) provides defense-in-depth for instruction files. However, `replace_string_in_file` operates on any file in the workspace — if instruction-following fails, the Knowledge Agent could modify source code, agent definitions, or other files. The mitigation layers (file boundary instructions + Safety Constraint Filter + evaluator separation) are all instruction-based. The `replace_string_in_file` grant is justified for `decisions.yaml` append operations, but the tool's unrestricted scope creates a broader attack surface than necessary.
- **Affected artifacts:** [tool-access-matrix.md §10](NewAgents/.github/agents/tool-access-matrix.md#L66-L76), [knowledge-agent.agent.md](NewAgents/.github/agents/knowledge-agent.agent.md#L267) (file boundaries), [knowledge-agent.agent.md](NewAgents/.github/agents/knowledge-agent.agent.md#L309) (tool list)
- **Recommendation:** (1) Document the `replace_string_in_file` justification explicitly (used for `decisions.yaml` append-only updates). (2) Add a self-verification check to the Knowledge Agent: "Before each `replace_string_in_file` call, verify the target file is in the declared file boundary list."
- **Evidence:** [tool-access-matrix.md line 10](NewAgents/.github/agents/tool-access-matrix.md#L10): `replace_string_in_file` column shows ✅ for Know(ledge Agent). [knowledge-agent.agent.md line 267](NewAgents/.github/agents/knowledge-agent.agent.md#L267): "Write ONLY to: knowledge-output.yaml, decisions.yaml (append-only), evidence-bundle.md, agent-metrics/{date}-run-log.md, post-mortems/{date}-{slug}.md."

---

## Correctness Analysis

**Category Verdict:** approve

### Finding CORR-1: Orchestrator references sql-templates.md "§5–§6" but §5 is Knowledge Agent's instruction_updates template

- **Severity:** Minor
- **Description:** The orchestrator agent references "sql-templates.md §5–§6" for evidence gate templates in two locations. However, §5 is the `instruction_updates INSERT Template` which is only used by the Knowledge Agent. The evidence gate queries are in §6 only. This cross-reference error could cause confusion when the orchestrator agent follows links to find evidence gate SQL templates, though the orchestrator's instructions are detailed enough that it would likely navigate to the correct section regardless.
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md#L24) (line 24), [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md#L362) (line 362)
- **Recommendation:** Change "§5–§6" to "§6" in both locations. The evidence gate queries are exclusively in §6.
- **Evidence:** [orchestrator.agent.md line 24](NewAgents/.github/agents/orchestrator.agent.md#L24): "using templates from sql-templates.md §5–§6". [orchestrator.agent.md line 362](NewAgents/.github/agents/orchestrator.agent.md#L362): "All evidence gates use SQL queries from sql-templates.md §5–§6." Meanwhile, [sql-templates.md §5](NewAgents/.github/agents/sql-templates.md#L326) is "instruction_updates INSERT Template — Used by: Knowledge Agent only" and [§6](NewAgents/.github/agents/sql-templates.md#L361) is "Evidence Gate Queries." The verification evidence also flagged this: verifier import-check noted "lines 24,353 ref sql-templates.md S5-S6 for evidence gates but S5 is instruction_updates."

---

## Summary

The agent system overhaul demonstrates strong security design: centralized SQL templates with sanitization requirements, stdin piping for shell injection prevention, CHECK constraints for path traversal prevention, Safety Constraint Filter for governed updates, and documented tool access restrictions. However, two Major security findings remain: (1) `sql_escape()` exists only as pseudocode that LLMs must voluntarily apply, creating a gap between specification and enforcement, and (2) the recommended double-quoted shell patterns for SQL piping don't actually prevent shell metacharacter expansion as documented, making the shell injection documentation misleading. One Major architecture finding highlights the systemic reliance on instruction-based enforcement for all agent trust boundaries. Verification evidence is comprehensive (48/48 passed) indicating solid implementation quality on the documented requirements.
