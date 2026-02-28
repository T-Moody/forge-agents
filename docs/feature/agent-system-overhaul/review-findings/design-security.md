# Adversarial Review: design — claude-opus-4.6

## Review Metadata

- **Review Focus:** security
- **Review Perspective:** Adversarial Security Auditor
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-02-27T10:00:00Z
- **Task ID:** agent-system-overhaul-design-review

## Overall Verdict

**Verdict:** needs_revision
**Confidence:** High

## Findings

### Finding 1: SQL Injection via Unparameterized String-Interpolated Templates

- **ID:** SEC-1
- **Severity:** Critical
- **Category:** security
- **Description:** All SQL INSERT and SELECT templates in the design use direct string interpolation (`'{output_snippet}'`, `'{notes}'`, `'{missing_information}'`, `'{inaccuracies}'`, `'{impact_on_work}'`, `'{change_summary}'`) rather than parameterized queries. The design explicitly acknowledges this in Section 11.2 of design.md: "All SQL queries use string formatting with run_id (not parameterized — sqlite3 CLI limitation)." While the design recommends "All text fields should sanitize single quotes (replace `'` with `''`)", this is stated as a guideline, not enforced in any template. The templates in sql-templates.md §2–§5 show raw `'{field}'` placeholders with no sanitization wrapper. Free-text fields in `artifact_evaluations` (`missing_information`, `inaccuracies`, `impact_on_work`) and `pipeline_telemetry` (`notes`) accept arbitrary agent-generated content. An adversarial artifact under review (e.g., code containing `'); DROP TABLE anvil_checks; --` in a filename or comment) could propagate through agent output into these fields, causing SQL injection when another agent or the orchestrator executes the INSERT via `run_in_terminal`.
- **Affected artifacts:**
  - `design.md` Section 5.3 (sql-templates.md §2–§5 template definitions)
  - `design.md` Section 11.2 (SQL Injection Prevention)
  - `design-output.yaml` lines 469–490 (template content outlines)
- **Recommendation:** (1) Define a mandatory `sql_escape(value)` function in sql-templates.md that replaces `'` with `''` and strips null bytes. (2) Update ALL INSERT templates to wrap every text field with this function — not as a suggestion, but as the canonical template form: e.g., `sql_escape('{output_snippet}')`. (3) Add a self-verification checklist item to every agent that writes SQL: "Verify all text values are single-quote-escaped before INSERT." (4) Consider using sqlite3's `.import` or heredoc-based approaches to avoid shell-level string interpolation entirely.
- **Evidence:** design.md line 1184: "All SQL queries use string formatting with run_id (not parameterized — sqlite3 CLI limitation)". design.md line 1187: "All text fields should sanitize single quotes (replace `'` with `''`)" — advisory only, not enforced in templates. sql-templates.md §4 template: `VALUES ('{run_id}', '{evaluator_agent}', '{evaluator_instance}', '{artifact_path}', {usefulness_score}, {clarity_score}, '{missing_information}', '{inaccuracies}', '{impact_on_work}')` — raw interpolation of 5 text fields.

---

### Finding 2: Command Injection via Shell-Interpreted SQL Commands in run_in_terminal

- **ID:** SEC-2
- **Severity:** Critical
- **Category:** security
- **Description:** SQL commands are executed via `run_in_terminal` which passes strings to the system shell (PowerShell or bash) before they reach sqlite3. This creates a dual injection surface: shell metacharacters in agent-generated values (e.g., `` ` ``, `$()`, `|`, `;`) are interpreted by the shell _before_ sqlite3 sees the SQL. For example, if `output_snippet` contains `$(malicious_command)` on bash or `` `malicious_command` `` on PowerShell, the shell will execute the embedded command. The design templates show patterns like: `sqlite3 "verification-ledger.db" "INSERT INTO ... VALUES ('...{output_snippet}...')"` — the entire SQL statement is passed as a shell argument with agent-controlled content embedded. The design documents no shell escaping requirements; Section 11.2 only addresses SQL-level single-quote escaping, not shell-level metacharacter escaping.
- **Affected artifacts:**
  - `design.md` Section 5.3 — all SQL command templates executed via run_in_terminal
  - `design.md` Section 11.2 — SQL Injection Prevention (missing shell injection coverage)
  - `design-output.yaml` shared_documents sql-templates.md content outline
- **Recommendation:** (1) Add a "Shell Injection Prevention" subsection to Section 11 specifying that all values interpolated into `run_in_terminal` SQL commands MUST be shell-escaped in addition to SQL-escaped. (2) Define the escaping order: first SQL-escape (replace `'` → `''`), then shell-escape (platform-specific: PowerShell uses `'...' -replace` or .NET escape; bash uses `printf '%q'`). (3) As a stronger alternative, design SQL commands to use sqlite3's `.parameter set` or heredoc/stdin piping to avoid shell interpretation of values entirely — e.g., `echo "INSERT INTO ..." | sqlite3 db.db` with properly escaped stdin. (4) Add this as a global operating rule in copilot-instructions.md.
- **Evidence:** design.md Section 5.3 pipeline_telemetry INSERT template shows direct shell execution: `INSERT INTO pipeline_telemetry ... VALUES ('{run_id}', '{step}', '{agent}', '{instance}', '{started_at}', '{completed_at}', '{status}', {dispatch_count}, {retry_count}, '{notes}');` — the `{notes}` field is free text with no shell escaping. design.md Section 11.2 mentions only SQL-level sanitization, not shell-level.

---

### Finding 3: Knowledge Agent Scope Contradiction — SELECT-Only Declared but INSERT Required

- **ID:** SEC-3
- **Severity:** Major
- **Category:** security
- **Description:** The Knowledge Agent's `run_in_terminal` scope is declared as "SELECT-only on verification-ledger.db" in 14 locations across the design artifacts (design.md lines 146, 828, 1170, 1308, 1309, 1389; design-output.yaml lines 138, 250, 422, 435, 528, 749, 869). However, the design simultaneously requires the Knowledge Agent to execute `INSERT INTO instruction_updates` (design.md lines 408, 411, 869; design-output.yaml lines 139, 426, 475). This is a direct contradiction: the agent needs INSERT capability for its core function (governed instruction update tracking), but the security model claims it has SELECT-only access. The practical consequence is that the "SELECT-only" restriction must be silently broken during implementation, which undermines the entire tool scope restriction model. If the Knowledge Agent can INSERT, there is no technical barrier preventing it from executing UPDATE, DELETE, or DROP statements — the "scope restriction" is purely instructional and the contradiction erodes trust in it.
- **Affected artifacts:**
  - `design.md` line 146 (tool access changes summary)
  - `design.md` line 828 (Knowledge Agent scope definition)
  - `design.md` lines 869–872 (instruction_updates INSERT template)
  - `design-output.yaml` lines 422, 435 (knowledge-agent tool access changes)
  - `design-output.yaml` tool-access-matrix.md §10 outline
- **Recommendation:** (1) Correct the scope restriction to accurately state "SELECT + INSERT on verification-ledger.db" — specifically: SELECT on all tables, INSERT on `instruction_updates` only. (2) Update all 14 instances of "SELECT-only" to the corrected scope. (3) Add an explicit prohibition list: "MUST NOT execute: UPDATE, DELETE, DROP, ALTER, CREATE, PRAGMA (except PRAGMA busy_timeout), ATTACH, or any non-SQL command." (4) Consider adding self-verification: Knowledge Agent must confirm its run_in_terminal commands match the allowed pattern before execution.
- **Evidence:** design.md line 828: `run_in_terminal: ALLOWED — scope: SELECT-only queries on verification-ledger.db` directly contradicts design.md line 869: `INSERT INTO instruction_updates (run_id, agent, file_path, change_type, change_summary, applied)`.

---

### Finding 4: Tool Scope Restrictions Are Instructional-Only with No Enforcement Mechanism

- **ID:** SEC-4
- **Severity:** Major
- **Category:** security
- **Description:** All tool scope restrictions in the design are enforced entirely through LLM instruction-following — there is no technical enforcement layer. Specifically: (1) Researcher's `create_file` is "scoped to research/_.yaml" — but the create_file tool accepts any path; (2) Verifier's `create_file` is "scoped to verification-reports/_.yaml" — same lack of enforcement; (3) Knowledge Agent's `run_in_terminal` is "scoped to SELECT-only" — the terminal accepts any command; (4) Adversarial Reviewer's `run_in_terminal` is "scoped to git diff + SQL INSERT only" — the terminal accepts any command. The design does not specify any validation layer between agent intent and tool execution. LLMs can deviate from instructions, especially under adversarial prompting (e.g., a malicious code artifact containing prompt injection patterns). The self-verification checklists check output format but not tool usage patterns.
- **Affected artifacts:**
  - `design.md` Section 10 (Tool Access Matrix) — footnotes 2, 3, 4, 5, 6
  - `design.md` Section 11.3 (Tool Scope Restrictions)
  - `design-output.yaml` tool-access-matrix.md content outline (§§2–10)
- **Recommendation:** (1) Acknowledge in Section 11 that scope restrictions are advisory constraints, not enforceable guardrails, and document this as an accepted risk with residual threat level. (2) Add a self-verification checklist item for every scoped tool: "Before calling run_in_terminal, verify the command matches allowed patterns: [specific regex]." (3) Design tool-access-matrix.md to include regex patterns for allowed commands per agent (e.g., Knowledge Agent `run_in_terminal` must match `^sqlite3\s+.*verification-ledger\.db.*\s+(SELECT|\.mode|\.headers)`). (4) Long-term: propose a VS Code extension hook or middleware that validates tool calls against the access matrix before execution.
- **Evidence:** design.md Section 11.3 states restrictions as declarative rules but describes no mechanism for enforcement. The VS Code `run_in_terminal` tool accepts arbitrary commands. The `create_file` tool accepts arbitrary paths.

---

### Finding 5: Underspecified Safety Constraint Filter for Knowledge Agent Instruction Updates

- **ID:** SEC-5
- **Severity:** Major
- **Category:** security
- **Description:** The Knowledge Agent can modify `.github/copilot-instructions.md` (the global rules file affecting ALL agents) and files in `.github/instructions/`. The design references a "Safety Constraint Filter" (design.md Section 11.1, design-output.yaml global-operating-rules.md §9, spec-output.yaml EC-8) that "rejects changes that weaken safety rules." However, the filter is never formally specified: (1) What constitutes "weakening safety rules" is undefined — no enumeration of protected rules or invariants; (2) No algorithm or matching logic is described; (3) No examples of what would be rejected vs. accepted; (4) The filter is to be implemented by the Knowledge Agent itself (an LLM), meaning the entity being restricted is also the entity evaluating the restriction — a classic confused deputy problem. If the Knowledge Agent's LLM generates a plausible justification for removing a safety rule, it may self-approve the change.
- **Affected artifacts:**
  - `design.md` Section 11.1 (Agent Boundary Integrity — "Safety constraint filter")
  - `design-output.yaml` shared_documents global-operating-rules.md §9 ("Safety filter rejects weakening")
  - `spec-output.yaml` EC-8 (Instruction file update safety)
- **Recommendation:** (1) Define the Safety Constraint Filter formally in global-operating-rules.md §9 with: (a) An enumerated list of immutable rules that cannot be weakened (e.g., bounded loop limits, no-file-redirect, terminal-only testing, agent boundary integrity); (b) Pattern-matching criteria: any update that removes or modifies text matching these rules is REJECTED; (c) A "protected phrases" list that cannot be deleted from instruction files. (2) Separate the evaluator from the actor: the orchestrator (not the Knowledge Agent) should validate proposed instruction changes against the Safety Constraint Filter. (3) In autonomous mode, make the default "log but DO NOT apply" — ensure the design explicitly prohibits auto-application without human approval for any change touching copilot-instructions.md.
- **Evidence:** EC-8 in spec-output.yaml: "Safety constraint filter (already in Knowledge Agent) rejects changes that weaken safety rules." This is the only specification — no formal definition exists anywhere in the design artifacts.

---

### Finding 6: File Path Fields in SQL Tables Lack Validation or Sandboxing

- **ID:** SEC-6
- **Severity:** Major
- **Category:** security
- **Description:** Multiple SQL table columns store file paths as free text with no validation: (1) `artifact_evaluations.artifact_path` — stores paths like "tasks/TASK-001.yaml" but could contain "../../../etc/passwd" or absolute paths; (2) `instruction_updates.file_path` — directly used by the Knowledge Agent to target file modifications; (3) Task YAML `relevant_context` fields reference file paths consumed by implementers. The `instruction_updates.file_path` is particularly dangerous because it determines which file the Knowledge Agent will modify. If the Knowledge Agent (or an adversarial influence on its input) sets `file_path` to an agent definition file (e.g., `NewAgents/.github/agents/orchestrator.agent.md`), the governed update mechanism could be used to modify agent definitions — violating CR-8 ("No agent MUST modify another agent's definition files at runtime"). The design states this is prevented by targeting "only .github/instructions/ and .github/copilot-instructions.md" but this is an instructional restriction with no path validation.
- **Affected artifacts:**
  - `design.md` Section 8.2 (artifact_evaluations table — `artifact_path` column)
  - `design.md` Section 8.3 (instruction_updates table — `file_path` column)
  - `design-output.yaml` instruction_updates DDL
- **Recommendation:** (1) Add a CHECK constraint to `instruction_updates.file_path`: `CHECK (file_path LIKE '.github/instructions/%' OR file_path = '.github/copilot-instructions.md')` — this provides database-level enforcement that the Knowledge Agent can only target approved paths. (2) Add a CHECK constraint to `artifact_evaluations.artifact_path`: `CHECK (artifact_path NOT LIKE '../%' AND artifact_path NOT LIKE '/%')` to prevent path traversal. (3) Document these constraints in the DDL in sql-templates.md §1. (4) Add to the Knowledge Agent's self-verification: "Verify file_path starts with `.github/instructions/` or equals `.github/copilot-instructions.md` before INSERT."
- **Evidence:** instruction_updates DDL (design.md Section 8.3) has `file_path TEXT NOT NULL` with no CHECK constraint on path patterns. CR-8 in spec-output.yaml mandates "No agent MUST modify another agent's definition files at runtime" but no technical enforcement exists.

---

### Finding 7: No Run ID Uniqueness Enforcement or Data Retention Policy

- **ID:** SEC-7
- **Severity:** Minor
- **Category:** security
- **Description:** The `run_id` field serves as the universal cross-run namespace filter (CR-6, design.md Section 11.4), but: (1) No UNIQUE constraint or index prevents duplicate `run_id` values from different runs — if a run_id collides (e.g., rapid manual re-runs with same timestamp), queries will return mixed data from both runs, potentially causing incorrect evidence gate decisions; (2) There is no data retention or cleanup policy — all historical run data accumulates indefinitely in the database, growing the query search space and potentially leaking information about previous pipeline runs; (3) The run_id is an ISO 8601 timestamp which is predictable — an agent aware of the naming scheme could craft queries targeting other runs' data. The practical risk is low for normal operation but increases in adversarial scenarios.
- **Affected artifacts:**
  - `design.md` Section 11.4 (Evidence Integrity — run_id filtering)
  - `design-output.yaml` sqlite_schemas database_strategy
  - `spec-output.yaml` CR-6 (run_id universal namespace)
- **Recommendation:** (1) Add a `runs` table with `run_id TEXT PRIMARY KEY` and a creation timestamp, ensuring run_id uniqueness. (2) Add foreign key references from all 4 tables to `runs(run_id)` or at minimum document the collision risk. (3) Add a data retention recommendation: "At pipeline completion, the orchestrator MAY delete records where run_id != current_run_id to prevent cross-run data accumulation." (4) Consider adding a random suffix to run_id (e.g., `2026-02-27T10:00:00Z-a7f3`) to prevent collision and predictability.
- **Evidence:** All 4 table DDLs (design.md Sections 5.3, 8.2, 8.3) show `run_id TEXT NOT NULL` with no uniqueness constraint. No cleanup or retention policy is documented anywhere.

---

### Finding 8: Unbounded Free-Text SQL Fields Enable Resource Exhaustion

- **ID:** SEC-8
- **Severity:** Minor
- **Category:** security
- **Description:** The `output_snippet` field in `anvil_checks` correctly has a `CHECK (LENGTH(output_snippet) <= 500)` constraint. However, all other free-text fields across the 4 tables have no size limits: `pipeline_telemetry.notes`, `artifact_evaluations.missing_information`, `artifact_evaluations.inaccuracies`, `artifact_evaluations.impact_on_work`, `instruction_updates.change_summary`, and `anvil_checks.command`. An agent generating excessive text in these fields (intentionally or due to LLM verbosity) could cause database bloat, slow queries, and increased disk usage over time. While SQLite has a default max string length of ~1 billion bytes, even a single 10MB text field in a frequently-written table could degrade query performance.
- **Affected artifacts:**
  - `design.md` Section 8.2 (artifact_evaluations — 3 unbounded text fields)
  - `design.md` Section 8.3 (instruction_updates — change_summary unbounded)
  - `design-output.yaml` pipeline_telemetry DDL (notes unbounded)
- **Recommendation:** (1) Add `CHECK (LENGTH(field) <= N)` constraints to all free-text fields: `notes` ≤ 1000, `missing_information` ≤ 2000, `inaccuracies` ≤ 2000, `impact_on_work` ≤ 2000, `change_summary` ≤ 1000, `command` ≤ 500. (2) Document these limits in sql-templates.md alongside the DDL. (3) Add truncation guidance for agents: "If text exceeds the field limit, truncate to limit and append '... [truncated]'."
- **Evidence:** anvil_checks DDL has `output_snippet TEXT CHECK (LENGTH(output_snippet) <= 500)` — demonstrating the design team knows about length constraints — but all other text fields omit this protection.

## Summary

The design has 2 Critical and 4 Major security findings, primarily around SQL/command injection via unparameterized string-interpolated templates passed through shell execution, a scope contradiction in the Knowledge Agent's tool access, instructional-only enforcement of tool restrictions, an underspecified safety filter for instruction updates, and unvalidated file path fields. No Blockers were identified — the threat model is LLM-mediated (not direct external attacker), which reduces exploitability. The 2 Critical findings (SEC-1, SEC-2) require addressing before implementation to prevent injection-class vulnerabilities from being baked into the SQL template infrastructure.
