# Adversarial Review: design — pragmatic-verifier (Round 2)

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🟡
- **Round:** 2
- **Run ID:** 2026-03-08T12:00:00Z

---

## Round 1 Finding Resolution Assessment

| Finding                                                                     | Severity | Status       | Assessment                                                                                                                                                                                         |
| --------------------------------------------------------------------------- | -------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PV-C-1: Architect missing research conditional for 🟢                       | Major    | **Resolved** | Architect `conditional_inputs` added in component_design. Step 3 `conditional_research` specifies "use initial-request.md + codebase analysis when research/\*.yaml absent." Clear, implementable. |
| PV-C-2: Evidence bundle only on success — failed pipelines lose audit trail | Major    | **Resolved** | D-15 adds `pipeline-log.yaml` maintained incrementally by orchestrator after each dispatch. Two-layer audit design (incremental + enriched) is sound. Schema provided.                             |
| PV-S-1: fetch_webpage autonomous toggle                                     | Minor    | **Resolved** | Addressed via D-10 `autonomous_mode_residual_risk`. Agents log URLs, Reviewer audits trail. Accepted residual risk documented.                                                                     |
| PV-A-1: Quick-fix pipeline undefined                                        | Minor    | **Resolved** | D-13 defines quick-fix step set {1,5,6,7,8}. Code Review (Step 7) preserved per immutable minimum.                                                                                                 |
| PV-A-2: Tester dispatch format unspecified                                  | Minor    | **Resolved** | `dispatch_format` added to Tester component_design: "First line MUST be 'Mode: static' or 'Mode: dynamic'."                                                                                        |
| PV-C-3: Naming convention break                                             | Minor    | **Resolved** | `naming_convention` specified in Step 3 and state_management. `architecture-output.yaml` chosen with rationale.                                                                                    |
| PV-C-4: Feedback loop counter interaction                                   | Minor    | **Resolved** | `counter_reset` field added to both feedback loops: "Cycle counter resets to 0 at start of each Code Review round."                                                                                |

**All 7 Round 1 findings (2 Major + 5 Minor) fully resolved.** The R2 revisions are thorough and implementable.

---

## Security Analysis

**Category Verdict:** approve

The pragmatic-verifier perspective examines security through the lens of "can an implementer actually enforce these controls, and are there gaps that would surface during real execution?"

### Assessment: Security Controls Are Implementable

**D-14 Trust Boundary Model:** The 3-tier model (Full/Standard/Read-Only) maps cleanly to VS Code's `tools` YAML frontmatter. Tier boundaries are enforceable: the platform prevents agents without `run_in_terminal` in their tools list from using it. The instruction-level constraints (Implementer command scope, Knowledge output scope) have concrete audit mechanisms:

- Implementer: `commands_executed[]` logged in implementation report → Tester audits against allowlist → Reviewer double-checks. Two-layer audit.
- Knowledge: pre-commit git diff validation against output_path_allowlist. Concrete, automatable check.
- fetch_webpage: URL logging + Reviewer audit. Residual risk in autonomous mode explicitly accepted with documented mitigations.

**Evidence:** design-output.yaml `trust_boundary_model` section specifies all 3 tiers with capabilities, boundary notes, and data flow transitions. `accepted_residual_risks` list 3 explicit risks with mitigations. The model is specific enough for an implementer to translate into agent instructions without guesswork.

No security findings.

---

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Orchestrator pipeline-log.yaml append mechanism needs implementation clarity

- **Severity:** Minor
- **Description:** D-15 specifies the orchestrator appends to `pipeline-log.yaml` after each dispatch. The orchestrator's tool list includes `replace_string_in_file` (for edits) and `create_file` (for new files), but YAML append-to-list operations require either: (a) `replace_string_in_file` targeting the last line and appending, or (b) reading the file, reconstructing, and rewriting. Neither is explicitly specified. The `pipeline-log.yaml` schema shows a `dispatches:` list that grows over time — the implementer needs to know which tool operation to use for appending entries to an existing YAML list.
- **Affected artifacts:** design-output.yaml D-15, component_design.orchestrator (tools), state_management.audit_trail
- **Recommendation:** Add a brief implementation note to D-15 or the orchestrator component_design: "Orchestrator appends dispatch entries to pipeline-log.yaml using replace_string_in_file to insert new list items at the end of the dispatches: array." This is a minor implementability clarification — the implementer can figure this out, but explicit guidance reduces ambiguity.
- **Evidence:** Orchestrator tools include `replace_string_in_file` but the append pattern for growing a YAML list isn't specified. The pipeline-log schema (state_management.audit_trail.pipeline_log_schema) shows the target structure but not the mutation mechanism.

No other architecture findings. The architectural decisions are well-structured and implementable:

- D-6 DAG dispatch algorithm description is concrete and translatable to orchestrator reasoning steps.
- D-7 Tester split trigger (>140 lines OR >20 lines duplicated) is measurable and actionable.
- D-8 orchestrator fallback (extract to dag-dispatch.md) is a viable escape valve.
- D-9 single shared doc strategy eliminates cross-file fragility.
- D-13 quick-fix step set is enumerable and enforces immutable minimum.
- The file inventory totals (~1,155 lines) leave ~345-line margin — adequate buffer.

---

## Correctness Analysis

**Category Verdict:** approve

### Finding C-1: Embedded design review gate criteria uses different verdict vocabulary than Code Review

- **Severity:** Minor
- **Description:** The Code Review gate (Step 7) uses verdict values `approve` / `request-changes` (design-output.yaml schemas.review_verdict). The embedded design review for 🔴 features (Step 3) uses `request-changes` as the failure trigger (pipeline_design Step 3 `failure_routing`). However, the failure routing text says "returns request-changes" while the design.md Step 3 says "Review gate: ≥2 approve + 0 blockers." These are consistent, but the Reviewer agent output schema has `verdict: "approve" | "request-changes"` — no `blocker` verdict value. Blocker findings appear under `findings[].severity: "blocker"`, not as the verdict itself. The gate logic "≥2 approve AND 0 blocker findings" is correct — it checks findings severity, not verdict value. An implementer reading only the gate description might conflate "0 blockers" (finding severity) with the verdict field. This is a documentation clarity issue, not a logic error.
- **Affected artifacts:** design-output.yaml pipeline_design Step 7 gate, schemas.review_verdict, design.md Step 7
- **Recommendation:** Consider adding a parenthetical to the gate description: "≥2 of N approve (verdict field) AND 0 findings with severity: blocker." This disambiguates verdict values from finding severity values for implementers.
- **Evidence:** design-output.yaml Step 7 gate: `"≥2 of N approve AND 0 blocker findings"`. schemas.review_verdict: `verdict: "approve" | "request-changes"`, `findings[].severity: "blocker" | "critical" | "major" | "minor"`. The two-field model (verdict + finding severity) is correct but the gate description could be clearer about which field it references.

### Finding C-2: Step 4 Planning has no failure/retry path specified

- **Severity:** Minor
- **Description:** Every other step in the pipeline specifies failure handling: Research has "halt with ERROR if <2 DONE" (EC-1), Architecture has R2 failure routing (EC-7), Implementation has max 3 cycles (EC-2), Testing routes to Step 5, Code Review has max 2 rounds (EC-3). However, Step 4 (Planning) has no `failure_routing` or edge case for Planner returning `NEEDS_REVISION` or `ERROR`. The completion contract supports all 3 status values for all agents. If the Planner returns `NEEDS_REVISION` (e.g., architecture output is insufficiently detailed for task decomposition), or `ERROR` (e.g., architecture output is malformed), the orchestrator's behavior is undefined. The global retry policy (max 2 retries for transient errors, never retry deterministic) applies, but explicit routing for Planner failure would be clearer.
- **Affected artifacts:** design-output.yaml pipeline_design Step 4, edge_cases section
- **Recommendation:** Add a brief failure note to Step 4: "Planner NEEDS_REVISION → re-dispatch with context (max 1 retry). Planner ERROR → pipeline ERROR." Or add EC-9 covering Planner failure.
- **Evidence:** Step 4 has `routing: "DONE → Step 5"` with no NEEDS_REVISION or ERROR path specified. Steps 2, 3, 5, 6, 7 all have explicit failure handling. The gap is minor because the global retry policy covers transient failures, but explicit specification matches the pattern established for all other steps.

No other correctness findings. The R2 revisions comprehensively address the R1 findings:

- EC-6 (malformed YAML status → ERROR) closes the undefined-behavior gap.
- EC-7 (design review max 1 round → ERROR) closes the routing gap.
- EC-8 (Knowledge outside allowlist → commit rejected) closes the pre-commit gap.
- Feedback loop counter reset is explicit.
- Conditional inputs for Architect on 🟢 are specified.
- Pipeline-log.yaml schema is complete and incremental.
- All 15 decisions are internally consistent and traceable to requirements.

---

## Summary

The R2 design is thorough, implementable, and internally consistent. All 13 Major findings from 3 reviewers have been addressed with concrete design additions — not vague acknowledgments. The 3 new decisions (D-13 quick-fix steps, D-14 trust model, D-15 incremental logging) are well-reasoned with alternatives analysis. The remaining 3 Minor findings are documentation clarity improvements that do not block implementation. The design is ready for planning and implementation.
