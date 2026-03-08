# Adversarial Review: design — architecture-guardian (Round 2)

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🟡
- **Round:** 2
- **Run ID:** 2026-03-08T12:00:00Z

---

## Round 1 Finding Resolution Assessment

All 5 Round 1 findings from architecture-guardian were addressed in the R2 revision:

| R1 Finding                                       | Severity | Resolution                                                                                      | Status            |
| ------------------------------------------------ | -------- | ----------------------------------------------------------------------------------------------- | ----------------- |
| A-1: Orchestrator 150-line budget lacks fallback | Major    | D-8 amended: extract to dag-dispatch.md (~50 lines) if over 150. Uses 2nd shared doc allowance. | ✅ Fully resolved |
| A-2: Tester dual-mode cohesion tension           | Major    | D-7 amended: split if >140 lines OR >20 duplicated lines. Crisp, measurable trigger.            | ✅ Fully resolved |
| A-3: File-ownership preventive enforcement       | Major    | Implementer MUST NOT modify undeclared files. Returns NEEDS_REVISION with files_requested[].    | ✅ Fully resolved |
| C-1: Naming inconsistency                        | Major    | Standardized on architecture-output.yaml. Documented in state_management naming_convention.     | ✅ Fully resolved |
| C-2: Design review failure routing               | Major    | Step 3 failure_routing: max 1 revision round for 🔴, then ERROR. EC-7 edge case added.          | ✅ Fully resolved |

Minor findings A-4 (schema evolution), A-5 (prompt files), C-3 (research gate) also addressed: D-9 acknowledges tradeoff, D-13 defines quick-fix steps, gate_policy documents ≥2 as intentional.

---

## Security Analysis

**Category Verdict:** approve

The R2 trust boundary model (D-14) is architecturally sound. The 3-tier model cleanly separates Full Trust (orchestrator), Standard Trust (implementer, tester), and Read-Only+Create (all others). Data flow transitions between tiers are explicitly documented with trust boundary annotations at each transition.

Key architectural strengths:

- Hub-and-spoke pattern (D-2, FR-2.5) prevents lateral agent-to-agent communication
- Platform-enforced tool restrictions via YAML frontmatter `tools` field upgrade enforcement from instruction-based (Medium risk) to platform-based (Low risk)
- Pre-commit validation at Step 8 gates Knowledge agent output before git commit
- Command allowlist for Implementer terminal commands with reviewer audit trail

No security findings.

---

## Architecture Analysis

**Category Verdict:** approve

All 3 Major and 2 Minor R1 architecture findings are fully resolved. The R2 design is structurally improved with concrete fallback plans, measurable split triggers, and preventive enforcement mechanisms. Two new Minor observations noted below — neither warrants revision.

### Finding A-1: D-14 trust boundary model omits Implementer fetch_webpage capability

- **Severity:** Minor
- **Description:** The trust boundary model (D-14, tier_2_standard_trust) lists Implementer capabilities as `run_in_terminal` (scoped) + file creation/editing + `get_errors`. However, the Per-Agent Tool Access table and the Implementer's component_design frontmatter both grant `fetch_webpage` to the Implementer. D-14's tier_3_read_only_create section says `fetch_webpage (Researcher, Architect only, when enabled)` — which contradicts the tool matrix. The trust model should reflect the Implementer's actual tool set for accurate security analysis.
- **Affected artifacts:** design-output.yaml trust_boundary_model.tier_2_standard_trust, design.md Per-Agent Tool Access table, design-output.yaml component_design.implementer.frontmatter.tools
- **Recommendation:** Add `fetch_webpage (when web_research enabled)` to D-14 Tier 2 capabilities, and update the tier_3 note to say "Researcher, Architect, Implementer" instead of "Researcher, Architect only." Alternatively, remove fetch_webpage from implementer's tools list if the intent was Tier 3 only.
- **Evidence:** Tool access table shows Impl ✅ for fetch_webpage. component_design.implementer.frontmatter.tools includes fetch_webpage. D-14 tier_2 does not list it. D-14 tier_3 says "Researcher, Architect only."

### Finding A-2: Orchestrator body sections deviate from D-3 universal 6-section standard

- **Severity:** Minor
- **Description:** D-3 states "Every agent body follows this consistent structure" with 6 sections: Role, Inputs, Workflow, Output Schema, Constraints, Anti-Drift Anchor. The orchestrator's component_design lists 5 sections: Role, Pipeline Steps, DAG Dispatch Algorithm, Constraints, Anti-Drift Anchor — omitting Inputs and Output Schema, and splitting Workflow into two domain-specific sections. While the orchestrator is inherently different from worker agents (it's a coordinator, not a producer), the design claims universal consistency that the orchestrator doesn't follow.
- **Affected artifacts:** design-output.yaml component_design.orchestrator.body_sections, design.md Agent File Format §Body Structure
- **Recommendation:** Add a note to D-3 or the Body Structure section: "The orchestrator is an exception — its body replaces the standard Inputs/Workflow/Output sections with Pipeline Steps and DAG Dispatch Algorithm, reflecting its coordinator role." This acknowledges the deviation rather than contradicting the "every agent" claim.
- **Evidence:** D-3 alt-1: "Each agent body has exactly 6 sections." Orchestrator component_design has 5 non-standard sections.

---

## Correctness Analysis

**Category Verdict:** needs_revision

Both R1 correctness findings (C-1 naming, C-2 failure routing) are fully resolved. However, the R2 pre-commit validation control has a scope inconsistency that could make the pipeline non-functional if implemented as design.md specifies.

### Finding C-1: Pre-commit validation scope contradicts pipeline function — would reject all implementation changes

- **Severity:** Major
- **Description:** The design.md Security Considerations section states: "At Step 8, orchestrator validates git diff against Knowledge output allowlist before committing. Only evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md, and files in docs/feature/\<slug\>/ are permitted. Unexpected files trigger commit rejection and ERROR status." However, the pipeline's core function is that the Implementer modifies source code files (e.g., Controllers/HealthController.cs, Tests/HealthControllerTests.cs) which are OUTSIDE both the Knowledge output allowlist AND the feature directory (docs/feature/\<slug\>/). When the orchestrator runs `git add .` + `git commit` at Step 8, the git diff includes all implementation changes from Step 5. Validating the diff against "Knowledge allowlist + feature directory" would reject every commit that contains implementation work.

  By contrast, design-output.yaml's trust_boundary_model says "Pre-commit gate: orchestrator validates Knowledge output paths against allowlist before git commit" — suggesting the check targets only Knowledge's outputs, not the entire diff. This contradicts design.md's broader scope.

- **Affected artifacts:** design.md "R2: Pre-Commit Output Validation (SS-S-3)" section, design-output.yaml trust_boundary_model (Knowledge → Orchestrator transition), design-output.yaml component_design.knowledge.output_path_allowlist, EC-8
- **Recommendation:** Clarify the pre-commit validation scope. The correct intent (from the trust model and SS-S-3 context) is: "Orchestrator validates that Knowledge agent did not create or modify files outside its allowlist." The implementation should compare Knowledge's declared `output_paths` against its allowlist, NOT validate the entire git diff against the allowlist. Update design.md's Security Considerations to match the trust_boundary_model wording: "Orchestrator validates Knowledge output_paths against allowlist before committing. Knowledge output must be limited to evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md within docs/feature/\<slug\>/. Files from other pipeline steps (implementation source code, test reports, review verdicts) are expected in the diff and not subject to this check."
- **Evidence:** design.md: "Only evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md, and files in docs/feature/\<slug\>/ are permitted." Step 5 produces source code files outside docs/feature/\<slug\>/. `git add .` at Step 8 stages everything. The validation as written would reject all implementation changes.

---

## Summary

The R2 design revision successfully addresses all 13 Major and 11 Minor findings from Round 1 across 3 reviewers. The 3 new decisions (D-13 quick-fix steps, D-14 trust boundary model, D-15 incremental logging) are well-structured. One Major correctness finding remains: the pre-commit validation scope in design.md is inconsistent with design-output.yaml's trust model and would reject legitimate implementation changes if implemented literally. This is a documentation fix — the trust model has the correct intent, design.md's wording needs to narrow the scope to Knowledge-specific output validation. Two Minor architecture notes (D-14 fetch_webpage omission, orchestrator body section deviation) are documentation quality items that don't affect structural soundness.
