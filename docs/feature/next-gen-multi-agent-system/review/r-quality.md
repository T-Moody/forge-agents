# Review: Code Quality (Re-Verification)

## Review Tier

**Full** — Re-verification of previously identified Blocker + spot-check of Major fixes across agent definitions, schemas, and README.

---

## Re-Verification Results

### [Previously Blocker — NOW RESOLVED] Baseline evidence gate has no writer — pipeline will always stall

- **Status:** ✅ **RESOLVED**
- **What was fixed:** The Implementer agent now includes explicit SQL INSERT instructions for `phase='baseline'` records in `anvil_checks`.
- **Evidence of fix (implementer.agent.md):**
  1. **Workflow Step 2 (line 138):** New mandatory step: _"SQL INSERT: INSERT each baseline check result into `anvil_checks` via `run_in_terminal` with `phase='baseline'` (see §Baseline Capture Detail — SQL INSERT for Baseline Records). This is mandatory — the orchestrator's evidence gate verifies baseline records exist in SQL."_
  2. **§SQL INSERT for Baseline Records (lines 420–444):** Full SQL INSERT template with all 12 `anvil_checks` columns: `run_id`, `task_id`, `phase`, `check_name`, `tool`, `command`, `exit_code`, `output_snippet`, `passed`, `verdict`, `severity`, `round`. Template correctly sets `phase='baseline'`, `verdict=NULL`, `severity=NULL`, `round=0`.
  3. **Self-verification checklist (line 474):** New checklist item: _"Baseline SQL INSERTs executed via `run_in_terminal` into `anvil_checks` with `phase='baseline'` and `round=0`"_
- **Schema match verified:** INSERT columns exactly match the `anvil_checks` CREATE TABLE definition in schemas.md (lines 965–980). All NOT NULL columns are populated; nullable columns (`command`, `exit_code`, `verdict`, `severity`) correctly allow NULL where appropriate.
- **Verdict:** Blocker is fully resolved. The orchestrator's evidence gate Query 1 (`WHERE phase='baseline'`) will now find records.

---

### [Previously Major — NOW RESOLVED] Three adversarial reviewers write to same verdict YAML file

- **Status:** ✅ **RESOLVED**
- **What was fixed:** Verdict files now use per-reviewer naming: `review-verdicts/<scope>-<model>.yaml`
- **Evidence of fix (adversarial-reviewer.agent.md):**
  1. **Line 6 (Outputs header):** `review-verdicts/<scope>-<model>.yaml`
  2. **Line 53 (Output table):** `review-verdicts/<scope>-<model>.yaml` with description "Machine-readable YAML verdict summary (per-reviewer)"
  3. **Line 240 (Workflow output step):** "YAML verdict summary at `review-verdicts/<scope>-<model>.yaml`"
  4. **Line 262 (Read-only enforcement):** Per-model file paths
  5. **Line 315 (Self-verification checklist):** Per-model file paths
- **Verdict:** Major is fully resolved. No more write collision between parallel reviewers.

---

### [Previously Major — NOW RESOLVED] README references nonexistent "Pattern C" and incorrect tier descriptions

- **Status:** ✅ **RESOLVED**
- **Evidence of fix (README.md):**
  1. **Line 155:** Now reads `dispatch-patterns.md  # Dispatch Patterns A/B definitions` — no Pattern C.
  2. **Line 167:** Now reads "Pattern A (parallel), B (sequential with replan loop) dispatch definitions" — no Pattern C.
  3. **Lines 100–103:** Tier descriptions now match verifier.agent.md exactly:
     - Tier 1 — IDE Diagnostics + Syntax
     - Tier 2 — Build & Test (including lint)
     - Tier 3 — Runtime Verification (import/load, smoke execution)
     - Tier 4 — Operational Readiness (Large tasks only)
- **Verdict:** Both Major issues fully resolved.

---

## Remaining Findings from Original Review (Not Re-Verified)

The following findings from the original review were **not in scope** for this re-verification pass. Their status is unknown:

| #   | Severity | Finding                                                      | Original Status |
| --- | -------- | ------------------------------------------------------------ | --------------- |
| 4   | Major    | Orchestrator creates `pipeline_telemetry` in wrong DB        | Not re-verified |
| 5   | Major    | Knowledge Agent cannot query SQLite but workflow requires it | Not re-verified |
| 6   | Major    | `relevant_context` pointers use wrong schema field path      | Not re-verified |
| 7   | Minor    | Inconsistent agent file formatting (frontmatter/code fences) | Not re-verified |
| 8   | Minor    | dispatch-patterns.md SQL queries under-filtered              | Not re-verified |
| 9   | Minor    | Orchestrator references unlisted `ask_questions` tool        | Not re-verified |

---

## Cross-Cutting Observations

None new. Original cross-cutting observations (testing gap for Markdown agents, `git add -A` secrets risk) remain applicable.

---

## Summary

**Re-verification of 3 findings: all 3 RESOLVED.**

- **Blocker (baseline evidence gate):** Fully fixed — Implementer now includes SQL INSERT instructions with correct schema, mandatory workflow step, and self-verification checklist item.
- **Major (verdict file collision):** Fully fixed — per-reviewer naming `<scope>-<model>.yaml` used consistently throughout adversarial-reviewer.agent.md.
- **Major (README Pattern C + tier descriptions):** Fully fixed — Pattern C removed, tier descriptions match verifier.agent.md.

The Blocker that would have prevented the pipeline from progressing past Step 5→6 is resolved. The two spot-checked Majors are also resolved. The remaining 3 Major + 3 Minor findings from the original review were not in scope for this re-verification.
