# Adversarial Review: design — architecture-guardian (Round 3)

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🟡
- **Round:** 3
- **Run ID:** 2026-03-08T12:00:00Z

---

## Round 2 Finding Resolution Assessment

Round 2 had 1 Major and 2 Minor findings. None were addressed in the current design revision — the text is identical to R2.

| R2 Finding                                                          | Severity | Status        | Notes                                                                                                           |
| ------------------------------------------------------------------- | -------- | ------------- | --------------------------------------------------------------------------------------------------------------- |
| C-1: Pre-commit validation scope contradicts pipeline function      | Major    | ❌ Unresolved | design.md lines 341, 534-535 unchanged. Still describes entire git diff validation against Knowledge allowlist. |
| A-1: D-14 omits Implementer fetch_webpage                           | Minor    | ❌ Unresolved | D-14 tier_2 still omits fetch_webpage. tier_3 still says "Researcher, Architect only."                          |
| A-2: Orchestrator body sections deviate from D-3 universal standard | Minor    | ❌ Unresolved | D-3 "every agent" claim vs orchestrator 5-section deviation unchanged.                                          |

---

## Security Analysis

**Category Verdict:** approve

No new security findings. The D-14 trust boundary model remains architecturally sound. The 3-tier structure (Full Trust → Standard Trust → Read-Only+Create) with explicit data flow transitions provides clear security boundaries. Platform-enforced YAML frontmatter `tools` field, hub-and-spoke dispatch pattern, and pre-commit validation (despite the scoping issue noted under Correctness) collectively form a reasonable defense-in-depth posture.

No security findings.

---

## Architecture Analysis

**Category Verdict:** approve

The core architecture — 8 agents, 8 steps, hub-and-spoke orchestration, DAG-based dispatch, YAML-only state, completion-contract routing — is coherent and well-designed. All R1 architecture findings remain fully resolved. Two R2 Minor findings are unchanged but do not affect structural soundness.

### Finding A-1: D-14 trust boundary model omits Implementer fetch_webpage capability (carried from R2)

- **Severity:** Minor
- **Description:** D-14 tier_2_standard_trust does not list `fetch_webpage` in Implementer capabilities. D-14 tier_3_read_only_create says "fetch_webpage (Researcher, Architect only, when enabled)" — contradicting the Per-Agent Tool Access table and component_design.implementer.frontmatter.tools, both of which grant fetch_webpage to the Implementer.
- **Affected artifacts:** design-output.yaml trust_boundary_model.tier_2_standard_trust, tier_3_read_only_create
- **Recommendation:** Add fetch_webpage to D-14 Tier 2 capabilities and update Tier 3 note to include Implementer. Alternatively, remove fetch_webpage from implementer's tool list.
- **Evidence:** Tool access table: Impl ✅ for fetch_webpage. component_design.implementer.frontmatter.tools includes fetch_webpage. D-14 tier_2 omits it. D-14 tier_3 says "Researcher, Architect only."
- **Round history:** First reported R2 A-1. Unresolved in R3.

### Finding A-2: Orchestrator body sections deviate from D-3 universal standard (carried from R2)

- **Severity:** Minor
- **Description:** D-3 states "Each agent body has exactly 6 sections" (Role, Inputs, Workflow, Output Schema, Constraints, Anti-Drift Anchor). The orchestrator's component_design has 5 different sections (Role, Pipeline Steps, DAG Dispatch Algorithm, Constraints, Anti-Drift Anchor). The design claims universality that the orchestrator doesn't follow.
- **Affected artifacts:** design-output.yaml component_design.orchestrator.body_sections, design.md Agent File Format §Body Structure
- **Recommendation:** Add exception note to D-3 acknowledging the orchestrator's coordinator role requires different sections.
- **Evidence:** D-3 alt-1: "Each agent body has exactly 6 sections." Orchestrator component_design has 5 non-standard sections.
- **Round history:** First reported R2 A-2. Unresolved in R3.

---

## Correctness Analysis

**Category Verdict:** needs_revision

The R2 Major finding (C-1) on pre-commit validation scope remains unresolved. The design.md text is byte-identical to R2.

### Finding C-1: Pre-commit validation scope contradicts pipeline function — would reject all implementation changes (carried from R2)

- **Severity:** Major
- **Description:** Three locations in design.md describe the pre-commit validation as checking the **entire** git diff against the Knowledge output allowlist:
  1. **Line 341** (Step 8): "R2: Pre-commit validation — verifies git diff only contains files in Knowledge output allowlist + feature directory"
  2. **Lines 534-535** (Security Considerations): "At Step 8, orchestrator validates git diff against Knowledge output allowlist before committing. Only evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md, and files in docs/feature/\<slug\>/ are permitted. Unexpected files trigger commit rejection and ERROR status."
  3. **EC-8** (Edge Cases): "Orchestrator pre-commit validation detects unexpected files in git diff."

  The pipeline's Step 5 (Implementation) produces source code files (e.g., Controllers/HealthController.cs, Tests/HealthControllerTests.cs) that are **outside** both the Knowledge allowlist and docs/feature/\<slug\>/. When Step 8 runs `git add .` + `git commit`, the diff includes all implementation changes. Validating the entire diff against "Knowledge allowlist + feature directory" would reject every commit containing implementation work.

  By contrast, design-output.yaml's trust_boundary_model data_flow_transitions (Knowledge → Orchestrator) correctly states: "Pre-commit gate: orchestrator validates Knowledge output paths against allowlist before git commit." This targets Knowledge's declared `output_paths`, not the entire diff.

- **Affected artifacts:** design.md lines 341, 534-535, 565; design-output.yaml EC-8 (line 1881); design-output.yaml trust_boundary_model lines 1043-1044 (ambiguous), line 1100 (correct)
- **Recommendation:** Update design.md to match the trust model's correct intent: "Orchestrator validates that Knowledge agent's declared output_paths are within its allowlist (evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md in docs/feature/\<slug\>/). Files from other pipeline steps (implementation source code, test reports, review verdicts) are expected in the diff and not subject to this check." Also update EC-8 to say "unexpected files in Knowledge output_paths" rather than "unexpected files in git diff."
- **Evidence:** design.md line 534: "Only evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md, and files in docs/feature/\<slug\>/ are permitted." Step 5 produces code outside docs/. `git add .` stages everything. Literal implementation would reject all implementation commits. design-output.yaml line 1100 has the correct scoping.
- **Round history:** First reported R2 C-1. Text unchanged in R3. **Known issue — logged for implementer awareness.**

---

## Summary

Round 3 final review of the agent-system-refactor design. All 13 R1 Major findings remain fully resolved. The R2 Major finding (pre-commit validation scope contradiction between design.md and design-output.yaml trust model) remains unresolved — design.md text is identical to R2. Two R2 Minor findings (D-14 fetch_webpage omission, D-3 orchestrator exception) also unchanged. The core architecture is sound: 8 agents, 8 steps, YAML-only state, completion-contract routing, DAG dispatch, and 3-tier trust boundaries are coherent and well-traced to industry practices. The Major finding is a documentation wording issue — the trust model has the correct intent — but an implementer reading only design.md could produce a non-functional pre-commit check. Recommend addressing C-1 before implementation begins, or explicitly briefing the implementer that design-output.yaml line 1100 is authoritative.
