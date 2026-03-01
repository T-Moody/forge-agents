# Adversarial Review: code — security

## Review Metadata

- **Review Focus:** security
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-01T12:00:00Z
- **Reviewer Model:** claude-opus-4.6
- **Scope:** code (behavioral-correctness-enforcement)

## Overall Verdict

**Verdict:** approve
**Confidence:** High

No security blockers, critical, or major findings. The changes maintain and extend existing security patterns (stdin piping for SQL, sql_escape() for free-text, CHECK constraints) without introducing new attack surface. All new content is Markdown documentation defining agent instructions, schema definitions, and SQL query templates — no executable code, secrets, or credential material.

## Findings

### Finding 1: EG-7 gap-detection query references non-existent `pipeline_telemetry.task_id` column

- **Severity:** Minor
- **Category:** security (observability)
- **Description:** The EG-7 gap-detection companion query in `sql-templates.md` §6 references `t.task_id` from the `pipeline_telemetry` table, but this column does not exist in the DDL (§1). The `pipeline_telemetry` table has columns: `id`, `run_id`, `step`, `agent`, `instance`, `started_at`, `completed_at`, `status`, `dispatch_count`, `retry_count`, `notes`, `ts` — no `task_id`. Any agent executing this query would receive a `no such column: t.task_id` SQLite error.
- **Affected artifacts:** `NewAgents/.github/agents/sql-templates.md` lines 500–512 (EG-7 gap-detection query)
- **Recommendation:** Replace `t.task_id` references with a valid join strategy. Two options: (a) join via `t.instance` which contains task identifiers (e.g., `implementer-task-01`), using pattern matching like `t.instance LIKE 'implementer-%'`; or (b) add a `task_id` column to `pipeline_telemetry` via the §9 schema evolution process. Option (a) is non-breaking.
- **Evidence:** `PRAGMA table_info(pipeline_telemetry)` on the live `verification-ledger.db` confirms no `task_id` column. The DDL in `sql-templates.md` §1 (lines 96–112) also lacks `task_id`. The gap-detection query (lines 500–512) uses `t.task_id` in the SELECT, JOIN, and GROUP BY clauses.
- **Security relevance:** While primarily a correctness defect, broken audit queries weaken security observability — the orchestrator cannot programmatically detect which code tasks lack behavioral-coverage evidence. EG-7 is INFORMATIONAL (non-blocking) so this does not create a deployment risk, but it degrades the defense-in-depth audit layer. Severity remains Minor because the primary EG-7 query (per-task check) is valid and functional.

### Finding 2: New free-text fields not explicitly enumerated in §0 sql_escape() field list

- **Severity:** Minor
- **Category:** security (injection surface documentation)
- **Description:** The `sql_escape()` field enumeration in `sql-templates.md` §0 lists specific fields: `output_snippet`, `notes`, `missing_information`, `inaccuracies`, `impact_on_work`, `change_summary`, `command`, `tool`. The new `behavioral_coverage[].justification` field (Schema 7 in `schemas.md`) accepts arbitrary free text from the implementer. While this field is written to YAML (not directly to SQL), its content may flow into `output_snippet` when the verifier summarizes behavioral-coverage check results in SQL INSERTs. The existing sql_escape() + 500-char truncation on `output_snippet` covers this indirectly, but `justification` is not called out in §0's explicit field list.
- **Affected artifacts:** `NewAgents/.github/agents/sql-templates.md` §0 (field enumeration, line 18); `NewAgents/.github/agents/schemas.md` Schema 7 (behavioral_coverage.justification field, lines 560–577)
- **Recommendation:** Either (a) add a note in §0 that the enumerated list is non-exhaustive ("including but not limited to"), or (b) add `justification` and other new free-text fields to the enumerated list. Option (a) is simpler and future-proof.
- **Evidence:** §0 line 18 states "Every free-text field — `output_snippet`, `notes`, `missing_information`, `inaccuracies`, `impact_on_work`, `change_summary`, `command`, `tool` — MUST be processed through `sql_escape()` before insertion." The `justification` field does not appear in this list. Schema 7 field table (line ~565) defines `behavioral_coverage[].justification` as type `string`, Conditional, with free-text content.
- **Security relevance:** The indirect protection via `output_snippet` truncation and sql_escape() means this is not an exploitable gap — all SQL-written fields already go through sanitization. The concern is documentation completeness: a future agent developer reading §0 might not realize that new YAML fields contributing to SQL values also need escaping. Low exploitation probability; documentation improvement only.

## Positive Security Observations

The following aspects of the implementation are commendable from a security perspective:

1. **DDL deduplication (Task 01):** Replacing ~260 lines of duplicated DDL in `schemas.md` with cross-references to the canonical source in `sql-templates.md` §1 eliminates schema drift risk — a common source of security-relevant inconsistencies.

2. **EG-7 is INFORMATIONAL-only:** The new behavioral coverage gate is correctly marked as non-blocking with documented promotion criteria. This conservative rollout prevents false-positive pipeline halts while building confidence in the new signal.

3. **No new SQL INSERT patterns:** All new verifier checks (`behavioral-coverage`, `runtime-wiring`) use the existing `anvil_checks` INSERT template from §2, inheriting all existing safety mechanisms (stdin piping, sql_escape(), CHECK constraints, output_snippet truncation).

4. **No secrets or credentials:** All 8 modified files contain only agent instructions, schema definitions, and SQL query templates. No API keys, tokens, passwords, connection strings, or PII were found in any changed content.

5. **CHECK constraints maintained:** The new `check_name` values (`behavioral-coverage`, `runtime-wiring`) pass through the existing `anvil_checks` schema without requiring enum expansion — the `check_name` column has no CHECK constraint restricting values, which is appropriate for extensibility.

6. **Structured identifiers for `task_id` and `run_id`:** These fields use kebab-case and ISO 8601 formats respectively, constrained by the naming convention documentation. The injection risk from these structured fields is negligible.

## Summary

The behavioral-correctness-enforcement feature's code changes are security-clean. All 8 tasks modify Markdown documentation files (agent definitions, schema references, SQL templates) — no executable source code was changed. The changes maintain established security patterns: stdin piping for SQL execution, sql_escape() for free-text fields, CHECK constraints for enum validation, and output_snippet truncation. Two Minor findings were identified: (1) a non-functional gap-detection query referencing a non-existent column (weakens audit observability but is non-blocking), and (2) incomplete sql_escape() field enumeration in §0 (documentation gap, not an exploitable vector). Neither finding represents an active security risk. Verdict: approve.
