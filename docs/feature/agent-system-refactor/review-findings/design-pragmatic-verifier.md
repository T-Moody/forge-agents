# Adversarial Review: design — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-08T12:00:00Z

## Security Analysis

**Category Verdict:** approve

### Finding S-1: Web research toggle is instruction-enforced, not platform-enforced

- **Severity:** Minor
- **Description:** The `fetch_webpage` tool is present in the `tools` YAML frontmatter for Researcher, Architect, and Implementer agents regardless of the `web_research_enabled` parameter value. In autonomous mode, the only enforcement preventing web requests is the instruction text "Use fetch_webpage when web_research is enabled." An agent ignoring this instruction retains platform-level access to `fetch_webpage`. D-10 rates this as "Medium" confidence and acknowledges the instruction-level toggle.
- **Affected artifacts:** design-output.yaml D-10, design.md "Web Research Security" section
- **Recommendation:** Document explicitly in the design whether VS Code's per-invocation user approval for `fetch_webpage` applies in autonomous mode dispatch. If it does, the platform provides a secondary control and this is fully mitigated. If it does not, note the residual risk in the Security Considerations section.
- **Evidence:** D-10 alt-1 cons: "Agents could use fetch_webpage even when disabled (instruction-enforced only)." Design Security section: "fetch_webpage requires user approval per invocation in interactive mode" — autonomous mode behavior unspecified.

No other security findings. The VS Code `tools` YAML frontmatter is a genuine improvement over instruction-only enforcement. The remaining instruction-level restrictions (Implementer `run_in_terminal` scope, Knowledge `create_file` scope) are appropriately mitigated by mandatory Step 7 Code Review.

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Quick-fix prompt pipeline is listed but undefined

- **Severity:** Minor
- **Description:** `quick-fix.prompt.md` (~25 lines) appears in the file inventory (design.md Implementation Checklist, design-output.yaml file_inventory) but no corresponding pipeline step sequence, skip rules, or reduced step set is defined anywhere in the design. The full pipeline design (D-5) specifies the 8-step `feature-workflow.prompt.md` pipeline in detail, but the quick-fix variant has zero specification. An implementer would have to invent the reduced pipeline behavior.
- **Affected artifacts:** design.md "Implementation Checklist & Deliverables" § file #11, design-output.yaml file_inventory new_files
- **Recommendation:** Either (a) add a brief specification for the quick-fix pipeline defining which steps are included (at minimum: Setup, Implementation, Testing, Code Review, Completion — matching the immutable minimum step set pattern), or (b) remove it from the file inventory and add it in a future iteration.
- **Evidence:** design-output.yaml lists `quick-fix.prompt.md` with `line_estimate: 25` and `description: "Simplified pipeline"` — no further definition. The feature.md "User Stories" section includes "Story 2: Simple Bug Fix (🟢)" which describes a reduced flow but doesn't reference the quick-fix prompt.

### Finding A-2: Tester mode parameter dispatch mechanism unspecified

- **Severity:** Minor
- **Description:** D-7 establishes the dual-mode Tester pattern where the orchestrator passes a `mode` parameter (dynamic/static). However, the dispatch mechanism for this parameter is not defined. VS Code's `runSubagent` API uses natural language task descriptions as the dispatch message — there is no structured parameter passing. The design should clarify whether mode is communicated via structured YAML dispatch parameters or natural language instructions in the dispatch message.
- **Affected artifacts:** design-output.yaml D-7, component_design tester section
- **Recommendation:** Add a brief note to the Tester's component_design specifying the dispatch format, e.g., "Orchestrator dispatch message includes: 'Mode: static' or 'Mode: dynamic' as the first line of the task description."
- **Evidence:** D-7 alt-1 says "Orchestrator passes mode (dynamic/static) as dispatch parameter" but the dispatch mechanism (natural language vs. structured) is unspecified. The pipeline_design Step 6 says `dispatches: "1-2 (static + dynamic)"` confirming two separate dispatches but not the communication format.

No other architecture findings. The 8-agent inventory (D-2), DAG dispatch (D-6), and hub-and-spoke orchestration (D-11) are architecturally sound. The 150-line orchestrator budget is tight but the section-by-section math (D-8: ~115-135 lines estimated) confirms feasibility. The D-7 Tester dual-mode carries acknowledged medium-confidence risk with a viable 9-agent fallback. The staging directory strategy (D-1) is clean.

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Architect has no conditional handling for missing research on 🟢 features

- **Severity:** Major
- **Description:** 🟢 (simple) features skip Research (Step 2) entirely — no `research/*.yaml` files are produced. However, the Architect agent's component design (design-output.yaml) lists its workflow as consuming "research outputs + initial-request.md" without any conditional for when research is absent. EC-8 in the spec states "Architect works from initial-request.md only" for 🟢 features, and design.md Step 3 routing says "🟢 → skip to Step 3." But the Architect's own component_design body_sections and responsibilities don't reflect this conditional input pattern. If the Architect instructions say "Read all research outputs" without a guard for missing files, the agent will either fail (attempting to read non-existent files), hallucinate research content, or produce a confusing error on 🟢 features.
- **Affected artifacts:** design-output.yaml component_design architect section (responsibilities, body_sections), design.md "Agent Inventory" table (Architect responsibilities), spec-output.yaml EC-8
- **Recommendation:** Add an explicit conditional to the Architect's component_design: "If no research/\*.yaml files exist (🟢 simple features), derive requirements and design solely from initial-request.md. Do not fail or hallucinate missing research." This should appear in the body_sections Workflow or Inputs section specification.
- **Evidence:** design-output.yaml architect responsibilities: "Read research outputs and initial request" (no conditional). Spec EC-8: "🟢 simple feature — Architect works from initial-request.md only." design-output.yaml pipeline_design Step 2 risk_scaling green: "SKIP." These three statements are inconsistent without explicit conditional handling in the Architect's own specification.

### Finding C-2: Evidence bundle produced only on success — failed pipelines lose audit trail

- **Severity:** Major
- **Description:** The `evidence-bundle.yaml` audit trail is produced by the Knowledge agent at Step 8 (Completion) — the terminal pipeline step. If the pipeline fails at any earlier step (Step 5 implementation failure after max retries, Step 6 persistent test failure, Step 7 code review halt after 2 rounds), the Knowledge agent is never dispatched and no evidence bundle is created. The design claims this replaces the current SQLite audit trail (design-output.yaml state_management what_this_eliminates includes "verification-ledger.db"), but the current system records pipeline_telemetry incrementally via SQL INSERT at each step completion. The new design breaks this incremental audit capability — failed pipeline runs produce only scattered YAML files with no consolidated dispatch log, timing summary, or outcomes analysis.
- **Affected artifacts:** design-output.yaml pipeline_design Step 8, state_management audit_trail section; design.md "State Management (D-4)" section; feature.md "Risks" table ("Loss of audit trail without SQLite: Low/Low")
- **Recommendation:** Either (a) have the orchestrator produce a minimal `pipeline-log.yaml` incrementally at each step (append dispatch event, timing, status after each agent returns), or (b) have the orchestrator dispatch the Knowledge agent on pipeline ERROR as a cleanup step before halting, or (c) add an explicit note in the design that failed pipeline audit trail is limited to individual step YAML files + git log and acknowledge this as a deliberate trade-off vs. the current system. Option (a) is simplest and preserves the audit trail without requiring SQLite.
- **Evidence:** design-output.yaml Step 8 description: "Knowledge agent: produces evidence-bundle.yaml (dispatch log, timing, findings)." design-output.yaml feedback_loops: Implementation-Testing failure and Code Review failure both halt with ERROR — neither routes to Step 8. Feature.md Risks table rates "Loss of audit trail without SQLite" as "Low likelihood / Low impact" but doesn't address partial-failure specifically. Spec FR-11.4: "Pipeline telemetry SHOULD be recorded in an evidence-bundle.yaml file produced by the Knowledge agent at completion" — "SHOULD" priority, not "MUST", but the design presents it as the primary audit mechanism.

### Finding C-3: Naming convention break with existing feature directory patterns

- **Severity:** Minor
- **Description:** The new system uses `architecture-output.yaml` and `architecture.md` as the Architect's output filenames (design.md Step 3 outputs). Existing features in `docs/feature/` use `spec-output.yaml` + `design-output.yaml` + `design.md` as separate files. The naming divergence means navigating between old and new feature runs requires knowing which naming convention applies to each run, reducing discoverability.
- **Affected artifacts:** design-output.yaml pipeline_design Step 3 outputs, design.md Step 3
- **Recommendation:** Either (a) use `spec-output.yaml` + `design-output.yaml` as the combined output filenames for backward compatibility (even though one agent now produces both), or (b) explicitly document the naming change in global-rules.md or the orchestrator so future readers know the convention changed.
- **Evidence:** design-output.yaml Step 3 outputs: `"architecture-output.yaml"`, `"architecture.md"`. Existing features: `docs/feature/agent-system-refactor/` contains `spec-output.yaml`, `design-output.yaml`, `design.md`. `docs/feature/agent-pipeline-improvements/` also uses `spec-output.yaml` + `design-output.yaml`.

### Finding C-4: Overlapping feedback loop iteration counters lack explicit relationship

- **Severity:** Minor
- **Description:** The design defines two independent feedback loops: Implementation-Testing (max 3 cycles) and Code Review Fix (max 2 rounds). The Code Review Fix loop routes "Step 7 → Step 5 → Step 6 → Step 7", which means the Implementation-Testing loop runs WITHIN the Code Review Fix loop. The design doesn't specify whether the Implementation-Testing cycle counter resets between review rounds or persists across them. If persistent, a pipeline that uses 2 implementation-testing cycles in Round 1, then gets code review rejection, would have only 1 remaining cycle for Round 2 — potentially insufficient. If reset, the total possible implementation dispatches increases to 2 rounds × 3 cycles = 6 dispatches per task.
- **Affected artifacts:** design-output.yaml pipeline_design feedback_loops section, design.md "Failure & Recovery" table
- **Recommendation:** Add an explicit note: "Implementation-Testing cycle counter resets to 0 at the start of each Code Review round" or "Implementation-Testing cycle counter persists across Code Review rounds — total budget is 3 implementation-testing cycles across all review activity."
- **Evidence:** design-output.yaml feedback_loops defines Implementation-Testing (max 3) and Code Review Fix (max 2) as separate entries with no cross-reference. Design.md "Feedback Loop Limits" lists them independently without specifying interaction.

## Summary

The design is ambitious, well-researched, and internally consistent across its 12 architectural decisions. The spec-to-design traceability is strong — all 22 acceptance criteria and 15 functional requirement groups have concrete design solutions. The industry traceability (12 decisions mapped to MetaGPT, SWE-Agent, AutoGen, Anthropic, etc.) is thorough. The line budget math is realistic (~1,100 target against 1,500 ceiling). However, two Major correctness findings require design-level attention: the Architect agent lacks conditional handling for 🟢 features where research is skipped, and the evidence-bundle audit trail is only produced on pipeline success, leaving failed runs without consolidated observability. Both are addressable with small design additions.
