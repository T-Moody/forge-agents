# Adversarial Review: design — correctness

## Review Metadata

- **Review Focus:** correctness
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-01T12:00:00Z

## Overall Verdict

**Verdict:** needs_revision
**Confidence:** High

## Findings

### Finding 1: Tier 2 framing mismatch — behavioral checks are unconditional but placed under conditional tier

- **Severity:** Major
- **Category:** correctness
- **Description:** The design places behavioral-coverage (Step 3e) and runtime-wiring (Step 3f) in Tier 2 of the verifier cascade. The existing Tier 2 heading in `verifier.agent.md` reads "**Run If Tooling Exists**" with the instruction: "Run **only if** corresponding tooling exists. Detect dynamically from config files." The new behavioral checks use `read_file` and `grep_search` — always-available workspace tools, NOT external tooling detected from config files. However, an implementer modifying the verifier would naturally place the new steps under the existing Tier 2 conditional framing, potentially gating them behind tooling detection logic. Since behavioral checks are the core enforcement mechanism of this feature, conditional execution would defeat the purpose.
- **Affected artifacts:** D-3 rationale in `design-output.yaml`, Verifier Behavioral Checks section in `design.md`, `verifier.agent.md` Tier 2 heading (lines 171–175)
- **Recommendation:** The design must explicitly specify that Steps 3e and 3f are **unconditional for code tasks** — they do not require external tooling detection. Either: (a) add an explicit note in the implementation checklist that these checks always run regardless of config-file detection, or (b) specify that the Tier 2 heading be amended to distinguish tooling-dependent checks (3a–3d) from always-run checks (3e–3f), e.g., by adding a sub-heading "Always-Run Checks (Behavioral)" after the tooling-dependent block.
- **Evidence:** `verifier.agent.md` line 171 states "### 3. Tier 2 — Run If Tooling Exists"; line 173 states "Run **only if** corresponding tooling exists. Detect dynamically from config files." D-3 rationale says "Tier 2 checks are unconditional when tooling exists" — but this mischaracterizes the current Tier 2 framing, which is conditional on tooling _existence_, not unconditional when it exists. The behavioral checks use `grep_search`/`read_file` (always available per tool-access-matrix.md), not external build/test tools.

### Finding 2: behavioral_coverage status enum incomplete — no accommodation for TDD Fallback (EC-2)

- **Severity:** Major
- **Category:** correctness
- **Description:** The `behavioral_coverage` field in Schema 7 defines status values as `covered | not_applicable`. The design example shows `not_applicable` used exclusively for `test_method=inspection` criteria (visual-only). However, EC-2 (TDD Fallback) describes scenarios where `test_method='test'` criteria legitimately have no automated tests (no test framework exists, task creates the framework, etc.). In these cases, the implementer needs to express "this AC has test_method='test' but tests could not be written for a valid reason." The current status enum doesn't accommodate this: `covered` implies a test exists, and `not_applicable` is semantically wrong (the criterion IS applicable for testing, the runtime conditions prevent it). The spec (EC-2) says "The check fails only if no justification is provided AND no tests exist" but the designed data model has no field to carry TDD Fallback justification for test_method='test' ACs.
- **Affected artifacts:** Schema 7 `behavioral_coverage` field definition in `design.md` (Data Models & DTOs section), EC-2 handling in implementation checklist item 6
- **Recommendation:** Either: (a) add a third status value `tdd_fallback` with a required `justification` string, or (b) explicitly document in the schema definition that `not_applicable` with a justification covers TDD Fallback scenarios for test_method='test' ACs (not just inspection criteria), and update the design example to show this case. Also ensure the verifier behavioral-coverage check logic knows to accept `not_applicable` + justification for test_method='test' ACs when a TDD Fallback is documented.
- **Evidence:** `design.md` Data Models section shows `status: "not_applicable"` example only for `test_method=inspection`. EC-2 in `spec-output.yaml` requires handling TDD Fallback with justification. The `tdd_red_green` field at the task level captures whether tests were written, but the per-AC `behavioral_coverage` mapping has no per-AC fallback status. FR-2.4 says the implementer "MUST either: (a) write tests, or (b) explicitly document WHY tests are not possible with TDD Fallback justification" — but the behavioral_coverage schema doesn't carry this per-AC justification for test_method='test' ACs.

### Finding 3: D-1 overclaims closing the AC narrowing gap (F-3) — planner translation completeness remains unverified

- **Severity:** Major
- **Category:** correctness
- **Description:** Architecture research F-3 identified that "Acceptance criteria flow has a structural narrowing gap from spec to implementer." The gap has two dimensions: (1) format loss (structured objects degrade to strings) and (2) completeness loss (planner may omit or weaken spec ACs when decomposing into tasks). D-1 addresses dimension (1) by changing Schema 6 to structured objects. D-6 addresses dimension (2) with a planner self-verification item. However, the self-verification item is the same enforcement mechanism that F-3 itself identified as insufficient: "self-check, not an external audit." No new external verification is added to confirm the planner includes ALL relevant spec ACs. If the planner omits an AC from a task, all downstream checks (implementer behavioral_coverage, verifier behavioral-coverage, reviewer correctness) operate only on the task's AC list — the omitted spec AC is invisible to the entire enforcement chain. D-1 rationale states it "clos[es] the narrowing gap identified in architecture F-3" — this overclaim could lead to false confidence that the gap is fully addressed.
- **Affected artifacts:** D-1 rationale in `design-output.yaml`, D-6 rationale in `design-output.yaml`, Acceptance Criteria Traceability section in `design.md`
- **Recommendation:** Amend the D-1 rationale to state it "partially addresses" or "narrows" the F-3 gap rather than "closing" it. Add a note in the design's Failure & Recovery table acknowledging that planner completeness remains a self-attested property. Optionally, consider whether the verifier could cross-reference the task's ACs against the spec ACs listed in the task's `relevant_context.spec_requirements` pointer to detect significant omissions — this would add external verification at the verifier layer without requiring a new agent.
- **Evidence:** `research/architecture.yaml` F-3: "Neither translation step has structural verification. The Planner's self-verification includes 'Every task must trace to at least one requirement' but this is a self-check, not an external audit." D-6 adds another self-check ("Spec AC IDs and test_method propagated to task-level criteria") — same mechanism F-3 identified as insufficient. All behavioral checks (FR-3.1, FR-3.2) operate on `task.acceptance_criteria`, not on the spec-level AC list.

### Finding 4: EG-7 query cannot identify tasks MISSING behavioral coverage — limited diagnostic value

- **Severity:** Minor
- **Category:** correctness
- **Description:** The EG-7 query `SELECT task_id, COUNT(*) ... WHERE check_name='behavioral-coverage' AND passed=1 GROUP BY task_id` identifies tasks that HAVE behavioral-coverage records. However, it cannot identify code tasks that are MISSING behavioral-coverage records — they simply don't appear in the result set. For an informational gate, the orchestrator would need to independently enumerate code tasks and compare against the EG-7 results to find gaps. This limits the diagnostic value of the query, particularly for the Knowledge Agent's post-mortem analysis.
- **Affected artifacts:** EG-7 query in `design.md` (APIs & Interfaces section), D-8 in `design-output.yaml`
- **Recommendation:** Consider enhancing the EG-7 query to use a subquery or LEFT JOIN pattern that identifies code tasks WITHOUT behavioral-coverage records, e.g., selecting tasks from `anvil_checks WHERE phase='after'` that lack a `behavioral-coverage` check_name. Alternatively, add a complementary "gap detection" query alongside EG-7. This enhancement would improve diagnostic value even in informational mode and would be required when EG-7 is promoted to a required gate.
- **Evidence:** `design.md` EG-7 SQL: `SELECT task_id, COUNT(*) AS behavioral_signals FROM anvil_checks WHERE ... GROUP BY task_id` — returns only rows for tasks with matching records. Compare with existing evidence gates that check for minimum thresholds — EG-7 only counts successes without establishing the denominator.

### Finding 5: D-1 backward-compatibility claim is semantically misleading

- **Severity:** Minor
- **Category:** correctness
- **Description:** D-1 rationale states: "Made backward-compatible by defaulting test_method to 'test' if omitted, so existing string-only consumers degrade gracefully." This conflates two different compatibility concerns. The `test_method` default provides compatibility for object-format ACs with a missing optional field. It does NOT provide compatibility for string-format consumers — changing the field type from `list of strings` to `list of objects` is a breaking change for any consumer parsing strings. The actual backward-compatibility mechanism is the synchronized update of all producers and consumers (acknowledged separately in the Migration section). The default protects against incomplete objects, not against the format change itself.
- **Affected artifacts:** D-1 rationale in `design-output.yaml`, Migration & Backwards Compatibility section in `design.md`
- **Recommendation:** Revise D-1 rationale to state: "Schema 6 field type changes from string list to object list (breaking change for unupdated consumers). Backward compatibility is ensured by synchronous update of all producers (Planner) and consumers (Implementer, Verifier). The test_method default of 'test' provides forward compatibility for objects with omitted optional fields." This accurately separates the compatibility mechanisms.
- **Evidence:** `design.md` Migration section already acknowledges: "This is technically a breaking change for the field type. Mitigation: Planner (sole producer) and Implementer/Verifier (consumers) are all updated synchronously." This contradicts D-1's claim of graceful degradation for "existing string-only consumers."

### Finding 6: tdd_red_green field is self-attested with no structural verification

- **Severity:** Minor
- **Category:** correctness
- **Description:** FR-2.5 requires structural verification of the TDD red-green cycle. The design adds a `tdd_red_green` object with `tests_written_first: boolean`, `initial_run_failures: integer`, and `initial_run_exit_code: integer`. However, all three values are self-reported by the implementer. An agent could write production code first, then tests, set `tests_written_first: true`, and report `initial_run_failures: 0` (tests pass against existing production code). Neither the verifier behavioral-coverage check nor the reviewer correctness guidance explicitly validates that `tests_written_first: true` AND `initial_run_failures: 0` is a suspicious combination indicating tests were NOT written first. The field provides useful telemetry but doesn't achieve "structural verification" as FR-2.5 requires.
- **Affected artifacts:** Schema 7 `tdd_red_green` field in `design.md`, D-2 rationale regarding TDD structural recording, FR-2.5 in `spec-output.yaml`
- **Recommendation:** Add to the reviewer correctness guidance (D-2 or FR-10.1 scope): flag `tdd_red_green.tests_written_first: true` combined with `initial_run_failures: 0` as a suspicious pattern warranting deeper test code review. This would provide at least one external verification layer for the self-reported TDD ordering claim.
- **Evidence:** `design.md` shows `tdd_red_green` as a self-reported field in Schema 7. Neither the verifier check specification (D-3, Steps 3e/3f) nor the reviewer guidance expansion (D-2 implementation checklist item 8) mentions validating `tdd_red_green` consistency. FR-2.5 requires "the TDD red-green cycle MUST be structurally verified" but the design only achieves structural recording, not structural verification.

## Summary

The design is well-structured with thorough coverage of the 10 FR groups and 18 ACs. The layered enforcement approach (implementer self-check → verifier cascade → reviewer severity) is sound. However, three Major correctness issues require attention: (1) the Tier 2 conditional framing creates risk that behavioral checks are incorrectly implemented as tooling-dependent, (2) the behavioral_coverage status enum doesn't accommodate TDD Fallback scenarios for test_method='test' ACs, and (3) D-1 overclaims closing the AC narrowing gap (F-3) when planner translation completeness remains unverified. Three Minor findings address EG-7 query limitations, misleading backward-compatibility language, and unverified TDD ordering claims. No Blockers or Critical issues found — the design is fundamentally sound but needs targeted revisions for implementation correctness.
