# Adversarial Review: design — claude-opus-4.6

## Review Metadata

- **Review Focus:** correctness
- **Review Perspective:** Pedantic Correctness Checker
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-02-27T10:00:00Z
- **Task ID:** agent-system-overhaul-design-review

## Overall Verdict

**Verdict:** needs_revision
**Confidence:** High

## Findings

### Finding 1: Evidence Gate Queries Cannot Distinguish Design Review (Step 3b) from Code Review (Step 7)

- **ID:** CORR-1
- **Severity:** Critical
- **Category:** correctness
- **Description:** The six evidence gate queries in `sql-templates.md §6` (EG-1 through EG-6) filter only by `run_id`, `phase='review'`, and `round`. They do not filter by `task_id`, pipeline step, or review scope (`design` vs `code`). In a full pipeline run, both Step 3b (design review) and Step 7 (code review) produce `anvil_checks` records with `phase='review'` and `round=1`. The reviewer instances share the same names across both steps (e.g., `adversarial-reviewer-security-sentinel`). When the orchestrator runs the evidence gate after Step 7, the queries would see records from **both** Step 3b and Step 7, producing incorrect counts and potentially passing the gate based on stale Step 3b data even if Step 7 reviewers failed or haven't completed.

  The check_name values used in EG-4 are scope-agnostic: `'review-security'`, `'review-architecture'`, `'review-correctness'`. These would match records from any review step. The current adversarial-reviewer convention (visible in this very review task) uses scope-qualified check_names like `'review-design-correctness'`—the design's new convention drops the scope qualifier, creating a regression in specificity.

- **Affected artifacts:**
  - `design.md` Section 5.3 (sql-templates.md §6 Evidence Gate SELECT Queries)
  - `design-output.yaml` → `review_design.evidence_gate_queries`
  - `design-output.yaml` → `sqlite_schemas.tables[0].key_queries`
- **Recommendation:** Either (a) include scope in check_names (`review-design-security`, `review-code-security`, etc.) and update EG-4's IN clause per step, or (b) add a `task_id` filter to all evidence gate queries (`AND task_id LIKE '%design-review'` or `AND task_id LIKE '%code-review'`), or (c) add a `scope` or `step` field to the `anvil_checks` INSERT convention for review records and filter on it. Option (a) is simplest and aligns with the existing convention observed in real pipeline output.
- **Evidence:** EG-3 query: `SELECT COUNT(DISTINCT instance) FROM anvil_checks WHERE run_id='{run_id}' AND phase='review' AND round={round};` — no step or scope filter. Design review round 1 and code review round 1 would both match with identical `round=1`. The design's 3 reviewer perspective instances (`security-sentinel`, `architecture-guardian`, `pragmatic-verifier`) appear at both steps, so `DISTINCT instance` conflates them.

---

### Finding 2: EG-6 Majority Approval Query is Incompatible with 3-INSERT-per-Reviewer Design

- **ID:** CORR-2
- **Severity:** Critical
- **Category:** correctness
- **Description:** Decision D-8 changes from 1 SQL INSERT per reviewer to 3 INSERTs per reviewer (one per category: `review-security`, `review-architecture`, `review-correctness`). However, EG-6 (majority approval) was not updated to account for this change:

  ```sql
  SELECT COUNT(DISTINCT instance) FROM anvil_checks
    WHERE run_id='{run_id}' AND phase='review' AND round={round}
    AND verdict='approve';
  ```

  This query counts reviewer instances that have **any** record with `verdict='approve'`. A reviewer with `security: approve`, `architecture: needs_revision`, `correctness: approve` has an overall verdict of `needs_revision`, but would be counted as an approving reviewer by EG-6 because two of its three records match `verdict='approve'`.

  The design's verdict YAML structure (Section 6.8) correctly derives `overall_verdict` as the worst of category verdicts. But the SQL evidence gate has no concept of "overall verdict"—no 4th INSERT exists for the overall verdict, and EG-6 doesn't aggregate per-reviewer category verdicts correctly.

- **Affected artifacts:**
  - `design.md` Section 5.3 (sql-templates.md §6 EG-6)
  - `design.md` Section 7.3 (Updated Evidence Gate Queries, point 4)
  - `design-output.yaml` → `review_design.evidence_gate_queries`
  - `design-output.yaml` → `decisions[7]` (D-8 rationale)
- **Recommendation:** Replace EG-6 with a query that identifies reviewers whose **all** category verdicts are `approve`:
  ```sql
  -- EG-6 (corrected): Majority fully-approving reviewers
  SELECT COUNT(*) FROM (
    SELECT instance FROM anvil_checks
    WHERE run_id='{run_id}' AND phase='review' AND round={round}
      AND check_name IN ('review-security','review-architecture','review-correctness')
    GROUP BY instance
    HAVING COUNT(CASE WHEN verdict != 'approve' THEN 1 END) = 0
  );
  ```
  Expected: ≥2 of 3. Alternatively, add a 4th INSERT per reviewer for the overall verdict with a distinct `check_name` like `'review-overall'`.
- **Evidence:** D-8 states "Enables deterministic evidence gate: COUNT(DISTINCT check_name) = 3 per reviewer." D-8's rationale focuses on EG-4 (coverage), not EG-6 (approval). EG-6 appears to be a carryover from the old 1-INSERT-per-reviewer pattern that was not updated for D-8's 3-INSERT pattern. The adversarial reviewer's verdict YAML example in Section 6.8 shows `overall_verdict: "needs_revision"` with mixed category verdicts—exactly the scenario where EG-6 produces a false positive.

---

### Finding 3: SQL INSERT Template Has Inconsistent Quoting for Nullable String Fields

- **ID:** CORR-3
- **Severity:** Major
- **Category:** correctness
- **Description:** The `anvil_checks` INSERT template in `design.md` Section 5.3 (sql-templates.md §2) uses mixed quoting conventions:

  ```sql
  VALUES ('{run_id}', '{task_id}', '{phase}', '{check_name}', '{tool}',
    '{command}', {exit_code}, '{output_snippet}', {passed}, {verdict},
    {severity}, {round}, '{instance}');
  ```

  String fields `{verdict}` and `{severity}` are **unquoted**, while other string fields (`{run_id}`, `{tool}`, etc.) are single-quoted. If `verdict` is `'approve'`, the unquoted substitution produces `..., approve, ...` which is invalid SQL (SQLite interprets `approve` as a column reference, not a string literal). If `verdict` is `NULL`, the unquoted form is correct.

  The template is ambiguous about how to handle the string-or-NULL case. Agents implementing this template will need to add conditional logic not documented in the design.

- **Affected artifacts:**
  - `design.md` Section 5.3 (sql-templates.md §2 anvil_checks INSERT Template)
  - `design-output.yaml` → `shared_documents[2].content_outline` ("§2 anvil_checks INSERT Template: 14-column INSERT with CHECK constraints")
- **Recommendation:** Document the quoting convention explicitly. For nullable string fields, use `NULL` when absent or `'value'` when present (with single quotes). Update the template to either: (a) always quote with `'{verdict}'` and instruct callers to use the literal string `NULL` without quotes when the field is null, or (b) add a note: "For nullable string fields (verdict, severity, command, output_snippet), substitute `NULL` (unquoted) when the value is absent, or `'<value>'` (single-quoted) when present."
- **Evidence:** The `anvil_checks` DDL defines `verdict TEXT CHECK (verdict IN ('approve','needs_revision','blocker'))` — this is a TEXT column expecting string values. SQLite would reject `INSERT INTO ... VALUES (..., approve, ...)` unless there happens to be a column named `approve` in scope (there isn't), producing an "no such column: approve" error.

---

### Finding 4: Routing Matrix Defined in Two Locations

- **ID:** CORR-4
- **Severity:** Major
- **Category:** correctness
- **Description:** The Completion Contract Routing Matrix (which agents return which completion statuses) is defined in two locations:
  1. `global-operating-rules.md §5` — explicitly shown in `design.md` Section 5.2 with a full agent × status table
  2. `schemas.md` — `design-output.yaml` file inventory says schemas.md will include "routing matrix (agent × completion status)"

  AC-13.1 specifically requires: "schemas.md includes a routing matrix table showing which agents return which completion statuses." Having the authoritative matrix in `global-operating-rules.md` AND `schemas.md` creates a single-source-of-truth violation. If the two copies diverge during implementation or future updates, agents referencing different documents will see different routing rules.

- **Affected artifacts:**
  - `design.md` Section 5.2 (`global-operating-rules.md §5`)
  - `design-output.yaml` → `file_inventory` entry for `schemas.md` (line ~201)
  - `design-output.yaml` → `shared_documents[1]` (`global-operating-rules.md §5`)
- **Recommendation:** Designate one canonical location and have the other reference it. Since AC-13.1 requires the matrix in `schemas.md`, place the authoritative table there. `global-operating-rules.md §5` should reference `schemas.md` rather than duplicating the table: "See schemas.md §Routing Matrix for the agent × completion status table."
- **Evidence:** `design.md` Section 5.2 shows the full routing matrix table (9 agents × 3 statuses) under "§5 Completion Contract Routing Matrix." `design-output.yaml` line ~201 lists schemas.md content as including "routing matrix (agent × completion status)." Both locations define the same data independently.

---

### Finding 5: Data Flow Diagram Understates verification-ledger.db Writers

- **ID:** CORR-5
- **Severity:** Minor
- **Category:** correctness
- **Description:** The data flow description in `design-output.yaml` → `architecture.data_flow` and `design.md` Section 3.3 states:

  > "4 writers (orchestrator, implementer, verifier, adversarial-reviewer)"

  However, the Knowledge Agent also writes to `verification-ledger.db` via the `instruction_updates` table (explicitly designed in Section 6.9/8.3). The correct count is **5 writers**. The component diagram in Section 3.2 correctly shows "instruction_updates SQL INSERT" as a Knowledge Agent output, contradicting the "4 writers" claim in the same document.

- **Affected artifacts:**
  - `design.md` Section 3.3 (Data Flow diagram text)
  - `design-output.yaml` → `architecture.data_flow`
- **Recommendation:** Update the data flow text to: "5 writers (orchestrator, implementer, verifier, adversarial-reviewer, knowledge-agent)" and add knowledge-agent as a writer in the data flow diagram annotation.
- **Evidence:** `design.md` Section 3.3: "4 writers (orchestrator, implementer, verifier, adversarial-reviewer)". `design.md` Section 6.9 Knowledge Agent: "instruction_updates INSERT for tracking governed updates." `design.md` Section 8.3: full DDL for `instruction_updates` table written by Knowledge Agent.

---

### Finding 6: anvil_checks Column Count Error in Data Flow Diagram

- **ID:** CORR-6
- **Severity:** Minor
- **Category:** correctness
- **Description:** The data flow diagram in `design.md` Section 3.3 annotates `anvil_checks` as having "14 cols." Counting the columns in the DDL (Section 5.3 / 8.1): `id`, `run_id`, `task_id`, `phase`, `check_name`, `tool`, `command`, `exit_code`, `output_snippet`, `passed`, `verdict`, `severity`, `round`, `instance`, `ts` = **15 columns**.
- **Affected artifacts:**
  - `design.md` Section 3.3 (Data Flow diagram — "anvil_checks (14 cols)")
- **Recommendation:** Change "14 cols" to "15 cols" in the data flow diagram annotation.
- **Evidence:** DDL in `design.md` Section 5.3 (sql-templates.md §1) defines 15 columns. Count: id(1), run_id(2), task_id(3), phase(4), check_name(5), tool(6), command(7), exit_code(8), output_snippet(9), passed(10), verdict(11), severity(12), round(13), instance(14), ts(15).

---

### Finding 7: Researcher create_file Scope Narrowing Not Documented as Deviation

- **ID:** CORR-7
- **Severity:** Minor
- **Category:** correctness
- **Description:** Spec AC-11.2 defines the researcher's allowed output path pattern as `"research/*.yaml, research/*.md"`. The design narrows this to `research/*.yaml` only (per DP-2 rationalization). While this narrowing is the correct consequence of DP-2, it represents a deviation from the literal AC text that is not documented in the design's Deviation Records section. The existing DR-1 and DR-2 document other spec deviations, but the AC-11.2 scope narrowing is undocumented.
- **Affected artifacts:**
  - `design-output.yaml` → `deviation_records` (missing entry)
  - `design-output.yaml` → `agent_designs[researcher].tool_access_changes`
  - `design.md` Section 6.2 (Researcher changes)
- **Recommendation:** Add a deviation record (DR-3) documenting that AC-11.2's allowed scope is narrowed from `research/*.yaml, research/*.md` to `research/*.yaml` only, with rationale referencing DP-2.
- **Evidence:** Spec AC-11.2: "the allowed output path pattern (research/_.yaml, research/_.md)". Design Section 6.2: "create_file restriction clarified — allowed for research/\*.yaml only". Design deviation_records contains only DR-1 and DR-2, with no reference to this AC-11.2 narrowing.

---

### Finding 8: AC-2.4 Multi-Model Enhancement Path Not in dispatch-patterns.md Content Outline

- **ID:** CORR-8
- **Severity:** Minor
- **Category:** correctness
- **Description:** Spec AC-2.4 requires: "If a working multi-model mechanism is discovered during FR-1 research, a documented enhancement path exists describing how to layer model diversity on top of prompt diversity." The design assigns FR-1 research results to `dispatch-patterns.md`, whose content outline (`design-output.yaml` file inventory) includes "Add FR-1 research findings section documenting VS Code runSubagent concurrency behavior" but does not explicitly mention a Phase 2 enhancement path for model diversity.
- **Affected artifacts:**
  - `design-output.yaml` → `file_inventory` entry for `dispatch-patterns.md`
  - `design.md` Section 6.1 (Orchestrator, Updated Step 3b/7 Dispatch)
- **Recommendation:** Add to `dispatch-patterns.md` content outline: "§N Phase 2 Enhancement: If VS Code API research reveals model routing capability, document how to layer model diversity on top of prompt perspective diversity."
- **Evidence:** `design-output.yaml` file inventory for `dispatch-patterns.md`: "Update Pattern A for perspective-based review dispatch. Add FR-1 research findings section documenting VS Code runSubagent concurrency behavior. Update evidence gate conditions for all-category review structure." No mention of model diversity enhancement path. Spec AC-2.4 conditionally requires this documentation.

---

## Summary

The design comprehensively addresses all 16 functional requirements, 7 NFRs, 10 edge cases, and 6 decision points. Spec traceability is strong—every FR has corresponding design elements and the AC mapping table in Section 17 is thorough. However, two critical correctness issues exist in the SQL evidence gate logic: (1) the queries cannot distinguish Step 3b design review records from Step 7 code review records in the same pipeline run, and (2) the majority approval query (EG-6) produces false positives under the new 3-INSERT-per-reviewer pattern from D-8. Both issues affect the orchestrator's review gate—the primary anti-hallucination mechanism—and must be fixed before implementation. Two additional major issues (SQL template quoting ambiguity, routing matrix duplication) and four minor documentation inaccuracies round out the findings.
