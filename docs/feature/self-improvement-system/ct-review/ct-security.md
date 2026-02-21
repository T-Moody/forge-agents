# Critical Review: Security & Backwards Compatibility (Revision 1 Re-Review)

## Previous Finding Resolution Summary

| #   | Previous Finding                                        | Previous Severity | Status                          | Notes                                                                                                              |
| --- | ------------------------------------------------------- | ----------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| F1  | Orchestrator Tool Restriction Removes Read Capabilities | High              | **RESOLVED**                    | Design retains read-only tools; only write/execute tools removed (§Write-Tool Restriction)                         |
| F2  | `memory` Tool Scope Unverified for Telemetry            | High              | **RESOLVED**                    | Telemetry no longer depends on memory tool; context-passing eliminates dependency (§Telemetry Accumulation)        |
| F3  | Indirect Prompt Injection via Evaluation Content        | Medium            | **Remains — downgraded to Low** | Free-text fields persist but PostMortem no longer produces recommendations, reducing exploitation surface          |
| F4  | Telemetry "Never Prune" in memory.md                    | Medium            | **RESOLVED**                    | Telemetry moved to orchestrator context; no memory.md telemetry section (§CT Revision Summary #3)                  |
| F5  | No YAML Schema Enforcement Mechanism                    | Medium            | **Remains — unchanged**         | Still LLM self-compliance only; shared schema doc improves consistency but not enforcement                         |
| F6  | Evaluation File Collision Avoidance                     | Medium            | **Partially resolved**          | Consistent sequence suffixes address naming inconsistency; cross-run data contamination remains (see new F3 below) |
| F7  | PostMortem Broadest Read Scope                          | Medium            | **Remains — unchanged**         | Memory-first reading reduces context pressure but doesn't reduce read scope                                        |
| F8  | Compact Telemetry DSL Coupling                          | Low               | **RESOLVED**                    | No compact DSL; telemetry passed as structured Markdown tables in dispatch prompt                                  |
| F9  | Evaluation Step Ordering Bias                           | Low               | **Remains — unchanged**         | No change to step placement                                                                                        |

---

## Findings

### [Severity: Medium] F1 — Spec-Design Divergence on Tool Restriction Creates Conflicting Implementation Targets

- **What:** The feature spec (FR-6.1, FR-6.4, AC-5, TS-5) explicitly mandates the orchestrator's allowed tools as `[agent, agent/runSubagent, memory]` ONLY, and AC-5 Pass criteria explicitly prohibit references to `read_file`, `grep_search`, `semantic_search`, `file_search`. The revised design intentionally deviates from this, retaining all read tools. The design documents the deviation with CT-1 rationale (§Write-Tool Restriction → "FR-6 deviation"), and this deviation is correct for pipeline safety. However, the feature spec (feature.md) was NOT updated to reflect this. Three risks: (1) An implementer working from feature.md alone would implement the strict restriction, breaking the pipeline. (2) AC-5 and TS-5 as written will FAIL against the revised design's implementation — the implementation would be correct yet fail acceptance testing. (3) If a future validation agent enforces AC-5 literally, it will flag the correct implementation as non-compliant.
- **Where:** feature.md FR-6.1 (line 167), FR-6.4 (line 173), AC-5 (lines 258–260), TS-5 (line 401) vs. design.md §Write-Tool Restriction
- **Likelihood:** High — the spec is a primary input to implementers and validators
- **Impact:** Medium — incorrect implementation would break cluster decisions; acceptance tests would false-fail against correct implementation
- **Assumption at risk:** That implementers and validators will read the design's deviation rationale rather than implementing the spec literally

### [Severity: Medium] F2 — No YAML Schema Enforcement Mechanism (Unchanged from v0)

- **What:** The design introduces structured YAML for evaluations, run-logs, and post-mortem reports — the first structured YAML in this codebase. The shared evaluation schema (§Shared Evaluation Schema) improves consistency by centralizing the definition, but provides no enforcement mechanism. All 14 evaluating agents and the PostMortem agent must self-enforce schema compliance via prompt instructions. LLMs are documented to produce malformed YAML (inconsistent indentation, missing quotes, invalid types). The mitigation remains "PostMortem skips corrupted files" (EC-5) — this means data loss is the expected degradation path with no retry or correction mechanism. This is unchanged from v0 and remains a medium risk because a high proportion of corrupted evaluations would render the post-mortem report empty or misleading.
- **Where:** design.md §Data Models (all YAML schemas); design.md §Failure & Recovery; `.github/agents/evaluation-schema.md` (new — defines schema but cannot enforce it)
- **Likelihood:** High — LLM YAML generation errors are well-documented and this is the first YAML in the codebase
- **Impact:** Medium — silent data loss; potentially empty post-mortem reports that appear complete
- **Assumption at risk:** That LLM-generated YAML will be consistently valid across 14 different agents producing it independently

### [Severity: Medium] F3 — Cross-Run Evaluation Data Contamination in PostMortem Analysis

- **What:** Evaluation files use bare agent names (`spec.md`, `designer.md`) per FR-1.4. On re-runs, sequence suffixes are appended (`spec-2.md`, `designer-2.md`). PostMortem discovers evaluations via `file_search` for `artifact-evaluations/*.md` (§PostMortem Workflow step 4). This glob matches ALL evaluation files in the directory — including files from prior pipeline runs. The design provides no mechanism for PostMortem to distinguish current-run evaluations from prior-run evaluations. PostMortem would compute aggregate scores mixing data from different runs (different pipeline contexts, different design versions), producing misleading metrics. The revised design's consistent sequence suffixes (§Append-Only Semantics) address naming but not the filtering problem. Neither the evaluation files nor the PostMortem workflow include metadata linking an evaluation to a specific pipeline run.
- **Where:** design.md §Storage Architecture → Append-Only Semantics (line 756); design.md §PostMortem Workflow steps 4–6; design.md §PostMortem Inputs
- **Likelihood:** Medium — requires re-runs on the same feature, which CT/V replan loops and designer revision cycles make routine
- **Impact:** Medium — PostMortem report metrics contaminated with stale evaluation data from prior runs; misleading accuracy scores and recurring issue counts
- **Assumption at risk:** That each pipeline run's evaluations are isolated, and PostMortem analyzes only current-run data

### [Severity: Medium] F4 — Spec-Design Divergence on `improvement_recommendations` Creates Requirement Gap

- **What:** The feature spec (FR-3.3) explicitly requires the PostMortem report to include an `improvement_recommendations` field with `recommendation`, `target_agent`, `priority`, and `evidence` sub-fields. User Story US-3 (feature.md line 382) describes a developer using `improvement_recommendations` to decide which agent definitions to update. The revised design removes this field entirely (§Post-Mortem Report Schema, CT-6 rationale). The removal is well-reasoned (clean boundary with R-Knowledge), but feature.md was not updated. This creates: (1) AC-8 Pass criteria will expect `improvement_recommendations` in the report — the correct implementation will fail acceptance testing. (2) US-3 describes a user flow that is impossible with the revised design. (3) A future spec-driven revision could re-introduce the field, undoing the CT-6 resolution.
- **Where:** feature.md FR-3.3 (lines 121–126), US-3 (line 382), AC-8 vs. design.md §Post-Mortem Report Schema ("Key change CT-6")
- **Likelihood:** High — FR-3.3 is an explicit requirement with a defined schema
- **Impact:** Medium — AC-8 false-fails against correct implementation; US-3 user flow broken; potential re-introduction of boundary blur
- **Assumption at risk:** That all stakeholders understand FR-3.3 has been superseded by the CT-6 design revision

### [Severity: Low] F5 — Indirect Prompt Injection via Evaluation Free-Text Fields (Reduced from v0)

- **What:** The evaluation schema retains five free-text list fields (`useful_elements`, `missing_information`, `information_not_used`, `inaccuracies`, `impact_on_work`). These fields are written by 14 agents and read by PostMortem. An agent that incorporates user-controlled content (e.g., evaluating `initial-request.md` content that contains adversarial instructions) could pass instruction-like text through evaluation fields into PostMortem's context. The revised design reduces the exploitation surface: PostMortem now produces quantitative metrics only (no recommendations), constraining what a compromised PostMortem could be directed to do. PostMortem's File Boundaries and Read-Only Enforcement (§PostMortem Agent Definition) further limit the blast radius. Downgraded from Medium to Low due to these structural defenses.
- **Where:** design.md §Data Models → Artifact Evaluation Schema (free-text fields); design.md §PostMortem Agent Definition → File Boundaries, Read-Only Enforcement
- **Likelihood:** Low — all agents are prompt-controlled; no direct user input in evaluation fields currently
- **Impact:** Low — PostMortem's quantitative-only output and strict File Boundaries limit exploitation
- **Assumption at risk:** That no evaluating agent will ever incorporate user-controlled content in evaluation text fields

### [Severity: Low] F6 — PostMortem Agent Retains Broadest Read Scope in Pipeline (Unchanged from v0)

- **What:** PostMortem reads orchestrator-provided telemetry, all `memory/*.mem.md` files, all `artifact-evaluations/*.md` files, `memory.md`, and the evaluation schema reference. This is the broadest read scope of any single agent. The revised design's memory-first reading pattern (§Reading Strategy) manages context pressure but does not reduce the scope of accessible data. The design's threat model asserts evaluation data contains "no secrets, credentials, or PII" (§Security → Data Protection), but this is a current-state assertion, not a structural guarantee. If the pipeline ever processes a feature involving credentials or internal security configurations, agent memory files or evaluations could reference these, and PostMortem would aggregate derived insights into `post-mortems/` — a less-protected, append-only report location.
- **Where:** design.md §PostMortem Agent Definition → Inputs; design.md §Security → Data Protection
- **Likelihood:** Low — currently no secrets in pipeline; requires future changes
- **Impact:** Low — aggregated sensitive data in report location, mitigated by repository-level access controls
- **Assumption at risk:** That no agent will ever produce memory or evaluation content referencing sensitive data

### [Severity: Low] F7 — Orchestrator Context Window Pressure from Telemetry Accumulation

- **What:** The orchestrator now accumulates all dispatch telemetry in its working context window throughout the pipeline run (§Telemetry Accumulation). For a standard run, this is 14+ agent entries plus cluster summaries. For Pattern C replan loops (max 3 iterations across Steps 5–6), this could reach 30+ entries. Each entry includes agent name, step, pattern, status, retries, failure reason, iteration number, timestamp — roughly 50–100 tokens per entry. At 30+ entries, telemetry consumes ~1.5–3K tokens in orchestrator context. While this is modest compared to the orchestrator's total context budget, it is context that persists across ALL remaining pipeline steps. There is no mechanism to signal if telemetry accumulation approaches a problematic threshold. The design acknowledges "ephemeral until PostMortem writes run-log" (§Telemetry Accumulation) but does not address context window degradation for orchestrator routing decisions at later pipeline steps.
- **Where:** design.md §Telemetry Accumulation; design.md §Sequence / Interaction Notes
- **Likelihood:** Low — 1.5–3K tokens is modest; would only matter in unusually long pipelines
- **Impact:** Low — degraded orchestrator decision quality in late pipeline steps if context is heavily loaded
- **Assumption at risk:** That telemetry accumulation in orchestrator context will not degrade cluster decision quality at Steps 6–7

### [Severity: Low] F8 — Evaluation Step Ordering May Leak Bias Into Self-Verification (Unchanged from v0)

- **What:** The evaluation step runs "between primary work completion and self-verification" (§Workflow Step Addition). An agent evaluating upstream artifacts generates scores and narrative assessments (e.g., "source artifact was unclear, required significant rework") that remain in context during its subsequent self-verification step. This could subtly anchor self-verification: an agent that just articulated upstream deficiencies might verify its own output more leniently. The revised design does not change this ordering. The risk remains low because self-verification is prompt-driven and agents have explicit self-verification checklists.
- **Where:** design.md §Evaluating Agent Changes → Workflow Step Addition
- **Likelihood:** Low — subtle LLM context bias
- **Impact:** Low — marginally degraded self-verification
- **Assumption at risk:** That evaluation generation context has no influence on self-verification

## Cross-Cutting Observations

- **Strategy (ct-strategy scope):** The spec-design divergences on FR-6.1/AC-5 (tool restriction scope) and FR-3.3 (improvement_recommendations) represent a process gap: the CT revision cycle updated the design but not the spec. If the spec serves as the contract for acceptance testing, these divergences will cause false failures. This is a workflow process concern beyond security scope.

- **Maintainability (ct-maintainability scope):** The shared evaluation schema (`.github/agents/evaluation-schema.md`) is a significant improvement over 14-file duplication. However, the schema document is a runtime dependency for 14 agents — its absence or corruption would degrade evaluation quality across the entire pipeline. A versioning strategy for the schema document (mentioned but not specified in §Shared Evaluation Schema) would benefit long-term maintainability.

- **Scalability (ct-scalability scope):** The move from memory.md telemetry to orchestrator context is a net positive for scalability (eliminates downstream context pollution for 19 agents). The new orchestrator context pressure from telemetry accumulation (F7) is a much smaller concern than the original memory.md bloat.

## Requirement Coverage

| Requirement                   | Coverage Status                                                       | Notes                                                        |
| ----------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------ |
| FR-1 (Artifact Evaluation)    | Covered — additive, non-blocking                                      | YAML integrity risk (F2); prompt injection risk reduced (F5) |
| FR-2 (Telemetry Tracking)     | Covered — context-based approach eliminates prior risks               | Orchestrator context pressure is minor (F7)                  |
| FR-3 (PostMortem Agent)       | **Spec divergence** — design removes improvement_recommendations (F4) | Design rationale sound; spec not updated                     |
| FR-4 (Storage Design)         | Covered — append-only with consistent collision avoidance             | Cross-run contamination remains (F3)                         |
| FR-5 (Pipeline Extension)     | Covered — Step 8 non-blocking                                         | No security concerns                                         |
| FR-6 (Tool Restriction)       | **Spec divergence** — design retains read tools (F1)                  | Design rationale sound; spec not updated                     |
| FR-7 (Safety Requirements)    | Covered — additive changes; write-tool restriction in separate track  | Track B separation eliminates bundling risk                  |
| FR-8 (Deliverables)           | Covered                                                               | No security concerns                                         |
| NFR-1 (Additive-Only)         | Covered — Track B separation addresses prior violation concern        | Track A is purely additive                                   |
| NFR-4 (Non-Blocking)          | Covered — both evaluation and PostMortem non-blocking                 | No concerns                                                  |
| NFR-6 (Graceful Degradation)  | Covered — 5-level hierarchy documented                                | Telemetry loss on PostMortem failure is accepted trade-off   |
| NFR-7 (Per-Feature Isolation) | **Partially covered** — cross-run evaluation contamination (F3)       | PostMortem may analyze mixed-run data                        |
