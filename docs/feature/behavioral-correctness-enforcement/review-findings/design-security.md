# Adversarial Review: design — security

## Review Metadata

- **Review Focus:** security
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-01T12:00:00Z

## Overall Verdict

**Verdict:** approve
**Confidence:** High

## Findings

No security findings identified. The design introduces no new attack surface, no authentication/authorization changes, no secrets handling, and no data exposure risks. Analysis of each security concern area follows.

### Analysis 1: SQL Injection Risk in EG-7 Query

- **Severity:** N/A (no issue found)
- **Category:** security
- **Description:** The proposed EG-7 evidence gate query (`SELECT task_id, COUNT(*) ... WHERE run_id = '{run_id}' AND check_name = 'behavioral-coverage' ...`) uses the same `{run_id}` parameterization pattern as existing EG-1 through EG-6 queries. The `run_id` is an orchestrator-generated ISO 8601 timestamp, not user-supplied input. The query is a read-only SELECT with no DML. The design explicitly states EG-7 follows existing `sql_escape()` and stdin piping patterns (SEC-2).
- **Affected artifacts:** [design.md](../design.md#L163-L170) (EG-7 query), [sql-templates.md](../../../NewAgents/.github/agents/sql-templates.md) §0 (sanitization rules)
- **Evidence:** EG-7 query structure is identical to existing evidence gate queries. No new interpolation patterns introduced. Existing sql_escape() mandate in sql-templates.md §0 covers all string interpolation.

### Analysis 2: New Schema Fields — Injection Surface

- **Severity:** N/A (no issue found)
- **Category:** security
- **Description:** New fields introduced: `behavioral_coverage` (list of objects with `ac_id`, `test_file`, `test_name`, `status`, `justification`), `tdd_red_green` (object with `tests_written_first`, `initial_run_failures`, `initial_run_exit_code`), and structured AC objects (`id`, `text`, `test_method`). All are YAML fields in implementation reports and task schemas — they do not flow directly into SQL. When the verifier records check results into `anvil_checks`, any derived text (e.g., check summary in `output_snippet`) is governed by the existing sql_escape() mandate and the 500-character CHECK constraint. No new SQL insertion patterns are introduced.
- **Affected artifacts:** [design.md](../design.md#L100-L140) (Data Models & DTOs section)
- **Evidence:** Implementation report Schema 7 fields are consumed by verifier via `read_file` (YAML parsing), not SQL interpolation. The `output_snippet` path is already covered by sql-templates.md §0 line 27: "Every free-text field — `output_snippet`, `notes`... — MUST be processed through `sql_escape()` before insertion."

### Analysis 3: Data Exposure Through New Evidence Fields

- **Severity:** N/A (no issue found)
- **Category:** security
- **Description:** New `check_name` values (`behavioral-coverage`, `runtime-wiring`) and their associated `output_snippet` content contain only: test file paths, AC IDs, pass/fail counts, and justification text. None of these carry PII, credentials, secrets, or security-sensitive data. The `justification` free-text field in `behavioral_coverage` could theoretically contain arbitrary agent-generated text, but this is no different from existing `output_snippet` content (e.g., build output, test output) that is already truncated to 500 chars and escaped.
- **Affected artifacts:** [design.md](../design.md#L150-L158) (New check_name Values table)
- **Evidence:** Existing `output_snippet` values in schemas.md examples include build output (`"Build completed successfully in 3.2s"`), test output (`"52 tests passed, 0 failed"`), and lint output. New behavioral check outputs are comparable in sensitivity.

### Analysis 4: Auth/Access Control Bypass Through Schema Changes

- **Severity:** N/A (no issue found)
- **Category:** security
- **Description:** All changes are to agent instruction text (`.agent.md` files) and shared reference documents (`schemas.md`, `sql-templates.md`, `severity-taxonomy.md`, `review-perspectives.md`). No runtime code is modified. No authentication mechanisms are changed. No authorization checks are added, removed, or bypassed. The verifier continues to use its existing read-only tools (`grep_search`, `read_file`) with no new tool grants. The Security Blocker Policy and escalation logic are explicitly preserved (design Security Considerations section).
- **Affected artifacts:** [design.md](../design.md#L205-L215) (Security Considerations section)
- **Evidence:** Design explicitly states: "No new tool access grants (verifier already has grep_search and read_file)" and "No changes to the Security Blocker Policy or escalation logic."

### Analysis 5: EG-7 Informational-First Approach — Security Gap Assessment

- **Severity:** N/A (no issue found)
- **Category:** security
- **Description:** EG-7 is informational (logged, not blocking). This means behavioral coverage gaps will not halt the pipeline. From a security perspective, this creates zero additional risk because: (a) behavioral correctness enforcement addresses test quality, not security vulnerabilities — security Blockers are handled by the adversarial reviewer's existing Security Blocker Policy which is unchanged, (b) the existing EG-4 blocker check query remains in place and is unmodified, (c) the informational approach specifically addresses EC-4 (pipeline resume compatibility) which is a correctness/availability concern, not a security concern.
- **Affected artifacts:** [design.md](../design.md#L163-L170) (EG-7 section), [design-output.yaml](../design-output.yaml) D-8
- **Evidence:** FR-5.1 from spec explicitly mandates "informational initially." Security Blocker escalation path (EG-4 query, verdict='blocker' logic) is entirely separate from EG-7 and unmodified by this design.

### Analysis 6: DDL Deduplication — CHECK Constraint Visibility

- **Severity:** N/A (no issue found)
- **Category:** security
- **Description:** D-5 proposes removing ~286 lines of illustrative DDL from schemas.md and replacing with cross-references to sql-templates.md §1. The DDL includes security-relevant CHECK constraints (e.g., `severity CHECK (severity IN ('Blocker','Critical','Major','Minor'))`, `output_snippet CHECK (LENGTH(output_snippet) <= 500)`). These constraints remain in their canonical location (sql-templates.md §1). The design preserves the check_name Naming Patterns table and phase semantics documentation in schemas.md (per FR-7.2). Agents already reference sql-templates.md for canonical DDL per existing convention.
- **Affected artifacts:** [design-output.yaml](../design-output.yaml) D-5, [schemas.md](../../../NewAgents/.github/agents/schemas.md#L961-L1246) (SQLite Schemas section)
- **Evidence:** The existing inline DDL is already marked "Illustrative reference only. Execute sql-templates.md §1 for the canonical DDL." (schemas.md line 986). Removing it does not change the authoritative source.

## Summary

The design introduces no new security vulnerabilities, attack surfaces, or data exposure risks. All proposed SQL follows existing safe patterns (sql_escape() sanitization, stdin piping, CHECK constraints). New schema fields are YAML-only and do not create new SQL injection vectors. No authentication, authorization, or secrets handling is affected. The informational-first EG-7 approach does not weaken the security posture because behavioral correctness enforcement is orthogonal to the Security Blocker Policy, which remains unchanged. The DDL deduplication preserves all CHECK constraints in their canonical location. Verdict: approve with high confidence.
