# Review: Testing

## Review Tier

**Lightweight** — 3 files modified, all prompt/markdown text. No compiled code, no automated test framework. Static text verifications performed by v-tests, v-tasks, and v-feature.

## Findings

### [Severity: Minor] Test ID Mapping Discrepancy Between Design and V-Tests for T-N1/T-N2

- **What:** The design §12.2 specifies T-N1 = "Memory Lifecycle Actions table still exists" and T-N2 = "Merge steps still exist" as regression tests. The v-tests report remapped T-N1 to "'Use read tools' does NOT appear" and T-N2 to "'read tools freely' does NOT appear." These are useful additional checks, but they test different things than the design specifies and reuse the same IDs.
- **Where:** [verification/v-tests.md](../verification/v-tests.md) — User-Specified Tests table, rows T-N1 and T-N2
- **Why:** Test ID reuse creates traceability confusion — the design's regression tests appear covered when they were actually performed by a different verifier (v-feature).
- **Suggested Fix:** In future test reports, differentiate novel tests from design-specified tests. Use separate IDs (e.g., T-A1, T-A2) for additional tests not in the design.
- **Affects Tasks:** N/A (reporting issue, not implementation)

### [Severity: Minor] Design T-6 (memory.md Preservation) Not Explicitly in V-Tests

- **What:** Design §12.1 specifies T-6: "memory.md references are preserved (not removed) throughout orchestrator prompt." This regression test was not executed by v-tests but was covered by v-feature's regression checks (confirming Global Rule 6, Memory Lifecycle Table, cluster decision logging all preserved).
- **Where:** [verification/v-tests.md](../verification/v-tests.md) — test table, T-6 absent; [verification/v-feature.md](../verification/v-feature.md) — §Regression Check Results covers this
- **Why:** For a Phase 1 change that explicitly preserves memory.md while adding disambiguation language, verifying preservation is important. The gap is compensated by v-feature, but v-tests should own design-specified tests.
- **Suggested Fix:** Include all design §12 tests in v-tests, even those that overlap with v-feature regression checks. Duplication across verifiers is acceptable for critical preservation checks.
- **Affects Tasks:** N/A

### [Severity: Minor] Design T-12 Search Pattern Is Case/Form Sensitive

- **What:** V-tests Design T-12 searched for "dispatches a subagent to merge" (lowercase, with "es" suffix) and found 7 matches. However, 3 merge steps (3b.2 at L287, 6.3 at L367, 7.3 at L400) use "Dispatch a subagent to merge" (uppercase "D", no "es"). A case-insensitive regex `[Dd]ispatch.* a subagent to merge` returns 10 matches (7 lowercase + 3 uppercase). The 7-match count in v-tests is technically correct for the exact search string used, but the test missed 3 merge steps.
- **Where:** [verification/v-tests.md](../verification/v-tests.md) — Design T-12; orchestrator.agent.md lines 287, 367, 400
- **Why:** Merge step wording was a core Task 02 deliverable across 8 locations. A search that catches 7/10 relevant matches (5/8 merge steps) has incomplete coverage. The gap is compensated by v-tasks, which individually verified all 8 merge steps with exact text matching.
- **Suggested Fix:** Use case-insensitive regex patterns for text verification searches, especially when the same concept may appear in different grammatical forms (imperative "Dispatch" vs. indicative "dispatches").
- **Affects Tasks:** Task 02

### [Severity: Minor] Design T-N3/T-N4 Regression Tests Not in V-Tests

- **What:** Design §12.2 specifies T-N3 ("All 20 subagent files are unmodified") and T-N4 ("dispatch-patterns.md is unmodified") as regression tests. Neither appears in v-tests. Both were confirmed by v-feature's regression checks.
- **Where:** [verification/v-tests.md](../verification/v-tests.md) — missing; [verification/v-feature.md](../verification/v-feature.md) — §Regression Check Results items 4 and 5
- **Why:** These are low-risk regression checks for a Phase 1 change that explicitly does not touch subagent files. Cross-verifier coverage is adequate.
- **Suggested Fix:** For completeness, v-tests should include all design-specified negative/regression tests, even if they are expected to be trivially passing.
- **Affects Tasks:** N/A

### [Severity: Minor] No Explicit Cross-File Prohibited Tool List Consistency Test

- **What:** Operating Rule 5 (L93) and Anti-Drift Anchor (L540) both list 8 prohibited tools. No test explicitly verified these two lists are identical. V-feature's Cross-File Consistency table checked allowed tools consistency (YAML ↔ OR5 ↔ Anti-Drift) but focused on the 5 allowed tools, not the 8 prohibited tools.
- **Where:** All verification files; orchestrator.agent.md lines 93 and 540
- **Why:** If the prohibited tool lists diverge, the orchestrator receives conflicting restriction signals. Since both lists are on single lines, a future edit could add/remove a tool from one but not the other.
- **Suggested Fix:** Add a test verifying the prohibited tool lists in Operating Rule 5 and Anti-Drift Anchor are identical (same 8 tools in same order).
- **Affects Tasks:** Task 01

## Coverage Assessment

- **Changed files with tests:** 3 of 3 — all modified files have corresponding verifications
- **Coverage gaps:**
  - Design T-6 (memory.md preservation) — covered by v-feature, not v-tests
  - Design T-N3 (subagent files unmodified) — covered by v-feature, not v-tests
  - Design T-N4 (dispatch-patterns.md unmodified) — covered by v-feature, not v-tests
  - Prohibited tool list cross-file consistency — partially covered but not explicit
- **Edge cases missing:**
  - No test verifies that the `memory` tool does NOT appear in the YAML `tools:` field's *prohibited* context (it correctly appears in the *allowed* list — this is expected)
  - No test verifies the YAML `tools:` field is syntactically valid YAML (v-build confirmed this separately)

## Test Quality Assessment

- **Behavioral tests:** 16 (all test observable text presence/absence — behavioral for a prompt engineering change)
- **Implementation-detail tests:** 0
- **Test data quality:** Good — all searches use exact strings from the design document; pragmatic AC-3 interpretation is well-documented

## Missing Test Scenarios

Prioritized by importance:

1. **(Low priority)** Prohibited tool list identity check between Operating Rule 5 and Anti-Drift Anchor
2. **(Low priority)** Explicit memory.md preservation check (design T-6) in v-tests
3. **(Low priority)** Case-insensitive variant of merge step wording search to catch all 10 matches
4. **(Informational)** Regression tests T-N3/T-N4 in v-tests for traceability

All missing scenarios are compensated by cross-verification between v-tests, v-tasks, and v-feature. None represent uncovered behavioral gaps.

## Cross-Cutting Observations

- The three-verifier architecture (v-tests, v-tasks, v-feature) provides robust compensating coverage — where one verifier has a gap, another catches it. This is a strength of the design.
- The v-tests T-12 discrepancy (user-specified vs. design-specified) was thoroughly documented with root cause analysis. This is exemplary test reporting for ambiguous requirements.
- For future prompt-engineering features, consider standardizing test search patterns to use case-insensitive regex by default, since prompt text often uses varied capitalization across sections.

## Summary

- **Blocker:** 0
- **Major:** 0
- **Minor:** 5
- **Total:** 5 issues, 0 blocking

All 16 static verifications pass. Coverage is adequate for a lightweight review of a 3-file prompt-only change. The 5 minor findings relate to test traceability and search precision, not to uncovered defects. Cross-verification between v-tests, v-tasks, and v-feature compensates for all identified gaps. No behavioral aspects of the changes are untested.
