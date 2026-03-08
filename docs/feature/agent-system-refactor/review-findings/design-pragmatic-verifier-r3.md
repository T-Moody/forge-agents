# Adversarial Review: design — pragmatic-verifier (Round 3)

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🟡
- **Round:** 3 (final review pass)
- **Run ID:** 2026-03-08T12:00:00Z

---

## Round 2 Finding Resolution Assessment

| Finding                                    | Severity | Status         | Assessment                                                                                                                                                                                                          |
| ------------------------------------------ | -------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PV-A-1: pipeline-log.yaml append mechanism | Minor    | **Unresolved** | D-15 and orchestrator component_design still do not specify which tool operation (replace_string_in_file targeting YAML list end) implements the append. Non-blocking — implementer can infer from available tools. |
| PV-C-1: Gate vocabulary disambiguation     | Minor    | **Unresolved** | Gate description "≥2 approve AND 0 blocker findings" still doesn't disambiguate verdict field vs finding severity. Non-blocking — the two-field model (verdict + findings[].severity) is clear from the schemas.    |
| PV-C-2: Step 4 Planning failure path       | Minor    | **Unresolved** | Step 4 routing still only shows "DONE → Step 5" with no NEEDS_REVISION or ERROR path. Non-blocking — global retry policy covers transient failures.                                                                 |

**Assessment:** All 3 R2 findings remain Minor documentation clarity items. None were addressed in revision, but none block implementation. An implementer reading the full design-output.yaml can infer correct behavior for all three from surrounding context and global rules.

### Cross-Reviewer R2 Finding Status

AG-R2 C-1 (Major: pre-commit validation scope contradicts pipeline function) remains in the design text. This is assessed below under Correctness Analysis as C-1.

---

## Security Analysis

**Category Verdict:** approve

The R2 trust boundary model (D-14) remains pragmatically implementable. Through the pragmatic-verifier lens — "can an implementer enforce these controls without guesswork?" — the security design holds:

1. **Platform-enforced tool restrictions:** VS Code `tools` YAML frontmatter is ground truth. If an agent's frontmatter omits `run_in_terminal`, the platform blocks it. Zero instruction-reliance for tool access.
2. **Command audit trail:** Implementer logs `commands_executed[]` → Tester audits → Reviewer double-checks. Two independent audit layers with concrete audit points.
3. **Knowledge output gate:** Pre-commit git diff validation is automatable (git diff --staged + path filtering). Implementation is straightforward regardless of the scope ambiguity (see C-1 below — the correct implementation is derivable).
4. **D-14 Tier 2 fetch_webpage omission (SS-R2 S-1, AG-R2 A-1):** The tool access table grants Implementer fetch_webpage but D-14 Tier 2 doesn't list it. This inconsistency persists as a documentation issue, but the Implementer's component_design.frontmatter.tools is the canonical tool list for implementation — frontmatter is what VS Code enforces. No security gap in practice.

No security findings.

---

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: pipeline-log.yaml append mechanism still unspecified (carried from R2)

- **Severity:** Minor
- **Description:** D-15 specifies orchestrator appends to pipeline-log.yaml after each dispatch, but the YAML list mutation mechanism is unspecified. The orchestrator has `replace_string_in_file` and `create_file`. Appending to a YAML `dispatches:` list requires `replace_string_in_file` targeting the end of the list. This is a standard pattern that an implementer with VS Code tool familiarity will solve in one line of instruction, but the design leaves it implicit.
- **Affected artifacts:** design-output.yaml D-15, component_design.orchestrator, state_management.audit_trail
- **Recommendation:** Implementation task for the orchestrator should include a note: "Use replace_string_in_file to append new dispatch entries at the end of the dispatches: array in pipeline-log.yaml." This can be handled at planning time rather than requiring a design revision.
- **Evidence:** D-15 says "appends to pipeline-log.yaml" without specifying the tool call pattern. Orchestrator tools include replace_string_in_file.

No other architecture findings. The architectural decisions remain well-structured:

- D-6 DAG algorithm is concrete and implementable as orchestrator reasoning steps.
- D-8 fallback (extract dag-dispatch.md) provides a clear escape valve if 150 lines is tight.
- D-9 single shared doc eliminates cross-file fragility.
- File inventory totals (~1,155 lines) leave ~345-line margin — sufficient buffer even if estimates are slightly off.

---

## Correctness Analysis

**Category Verdict:** approve

### Finding C-1: Pre-commit validation scope inconsistency across design documents (unresolved from AG-R2)

- **Severity:** Major
- **Description:** The pre-commit validation is described with two conflicting scopes across the design documents:

  **Scope A (overly broad — would break the pipeline):**
  - design.md Security Considerations: "Only evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md, and files in docs/feature/\<slug\>/ are permitted."
  - design-output.yaml Step 8 description: "validates git diff only contains files in Knowledge output_path_allowlist + feature directory."
  - design-output.yaml knowledge.output_path_allowlist: "Orchestrator validates git diff at pre-commit: any file outside this allowlist triggers a commit rejection."

  **Scope B (correct — Knowledge-output-only validation):**
  - design-output.yaml trust_boundary_model: "Pre-commit gate: orchestrator validates Knowledge output paths against allowlist before git commit."

  If an implementer follows Scope A, the orchestrator would reject every commit containing implementation source code (e.g., Controllers/HealthController.cs) because those files are outside both the Knowledge allowlist and docs/feature/\<slug\>/. The pipeline would fail on its first successful implementation.

  **Pragmatic assessment:** The correct implementation (Scope B) IS present in the trust_boundary_model — the design's most authoritative security section. An implementer who reads the trust model will implement correctly. However, 3 of 4 references describe Scope A, making it the "majority voice" in the design. An implementer reading design.md would implement the broken version.

- **Affected artifacts:** design.md "R2: Pre-Commit Output Validation" section, design-output.yaml pipeline_design Step 8, design-output.yaml component_design.knowledge.output_path_allowlist, design-output.yaml trust_boundary_model
- **Recommendation:** The Planner should document the correct scope in the implementation task for the orchestrator: "Pre-commit validation checks that Knowledge agent's declared output_paths are within its allowlist (evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md in docs/feature/\<slug\>/). Implementation source code, test reports, review verdicts, and other pipeline artifacts are expected in the git diff and are NOT subject to this check." If a design revision is feasible, update design.md and Step 8 description to match the trust_boundary_model wording.
- **Evidence:** design.md says "Only [...] are permitted" (Scope A). trust_boundary_model says "validates Knowledge output paths against allowlist" (Scope B). Implementation source files (Controllers/_.cs, Tests/_.cs) are outside both the allowlist and docs/feature/\<slug\>/.

### Finding C-2: Step 4 Planning has no failure routing (carried from R2)

- **Severity:** Minor
- **Description:** Step 4 routing shows "DONE → Step 5" with no NEEDS_REVISION or ERROR handling. All other steps (2, 3, 5, 6, 7) have explicit failure paths. The Planner CAN return NEEDS_REVISION (e.g., architecture output insufficient for task decomposition) or ERROR (e.g., malformed input). The global retry policy applies, but the gap breaks the otherwise comprehensive routing specification.
- **Affected artifacts:** design-output.yaml pipeline_design Step 4
- **Recommendation:** The Planner task should add: "Planner NEEDS_REVISION → re-dispatch once with context. Planner ERROR → pipeline ERROR." This can be handled at planning time.
- **Evidence:** Step 4 has `routing: "DONE → Step 5"` only. Steps 2, 3, 5, 6, 7 all have explicit failure routing.

---

## Implementation Readiness Assessment (R3 Focus)

This is the pragmatic-verifier's primary R3 contribution — assessing whether an implementer can successfully build the system from these design documents.

**Verdict: Ready for implementation with one caveat.**

| Agent           | Implementability | Notes                                                                                                          |
| --------------- | ---------------- | -------------------------------------------------------------------------------------------------------------- |
| Orchestrator    | ✅ High          | Pipeline steps, DAG algorithm, routing logic all concrete. Pre-commit scope needs Planner clarification (C-1). |
| Researcher      | ✅ High          | Simple role, clear outputs, web research toggle straightforward.                                               |
| Architect       | ✅ High          | Conditional inputs for 🟢 specified. Dual output (YAML + MD) clear.                                            |
| Planner         | ✅ High          | DAG schema with depends_on + files is concrete and testable.                                                   |
| Implementer     | ✅ High          | TDD workflow, command allowlist, file ownership all specified.                                                 |
| Tester          | ✅ High          | Dual-mode with dispatch_format "Mode: static/dynamic" is clear. Split trigger (D-7) provides escape valve.     |
| Reviewer        | ✅ High          | Perspective assignment, findings schema, verdict model all clear.                                              |
| Knowledge       | ✅ High          | Reads pipeline-log.yaml, output allowlist, recommendation file — all specified.                                |
| global-rules.md | ✅ High          | 6 cross-cutting concerns identified. ~100 lines is achievable.                                                 |
| Prompt files    | ✅ High          | D-13 defines quick-fix steps. Feature workflow is the full 8-step pipeline.                                    |

**Key strengths for implementers:**

1. Co-located output schemas eliminate cross-file lookup during implementation.
2. Per-agent line budgets provide clear targets.
3. The 6-section body structure (D-3) gives a consistent template.
4. All 15 decisions include alternatives analysis — implementers understand WHY.
5. Edge cases (EC-1 through EC-8) cover the most likely failure modes.

**One required caveat:** The pre-commit validation scope (C-1) MUST be clarified in the Planner's task definitions before implementation begins. The trust_boundary_model has the correct scope; design.md and Step 8 description have the wrong scope.

---

## Summary

Round 3 final review from pragmatic-verifier perspective. The R2 design is implementation-ready with one material inconsistency: pre-commit validation scope is described with two conflicting scopes across the design documents (3 references say "validate entire git diff against allowlist" vs 1 reference says "validate Knowledge output paths against allowlist"). The correct implementation (Knowledge-only scope) is present in the trust boundary model. 3 Minor items from R2 remain unresolved but are non-blocking documentation clarity issues. The design is architecturally sound, internally consistent on all other points, and provides sufficient detail for all 11 deliverable files across 8 agents + 1 shared doc + 2 prompts.
