# Adversarial Review: design — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-08T12:00:00Z

---

## Security Analysis

**Category Verdict:** approve

### Assessment

Through the architecture lens, the trust boundary model is sound. The hub-and-spoke pattern (D-2, FR-2.5) ensures no agent-to-agent communication bypasses the orchestrator. This creates a single chokepoint for routing validation and prevents unauthorized cross-agent data leakage.

The migration from instruction-enforced tool restrictions to VS Code native `tools` YAML frontmatter (D-3, CR-4) is a clear architectural improvement — platform-enforced boundaries replace advisory ones. Two residual instruction-level constraints remain (Implementer `run_in_terminal` scope, Knowledge `create_file` scope), documented with a mitigation path via mandatory code review (Step 7). This is an acceptable residual risk given that both restrictions are defense-in-depth behind the platform-level tool restriction.

The `agents: []` field on non-orchestrator agents (FR-12.3) prevents unauthorized subagent dispatch at the platform level, which is architecturally correct. The web research toggle via instruction-level parameter (D-10) is instruction-enforced only, but the secondary control layer (VS Code per-invocation approval in interactive mode) adequately compensates.

No security findings.

---

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: Orchestrator 150-line budget is high-risk for DAG + routing + 8-step coordination

- **Severity:** Major
- **Description:** The orchestrator has the most responsibilities of any agent: 8-step pipeline coordination, DAG topological-sort dispatch, file ownership conflict detection, completion-contract routing, approval gates, risk classification, git operations, and feedback loop management. The line budget is 150 lines (D-8). The design itself acknowledges this is "tight" for the DAG+routing logic. The body sections include "Pipeline Steps (8 steps with routing logic)" and "DAG Dispatch Algorithm" — fitting both a non-trivial scheduling algorithm and 8-step routing tables into ~120 lines of body (after frontmatter) is a significant implementation risk.
- **Affected artifacts:** design-output.yaml D-8 alt-1 cons, component_design orchestrator, design.md Agent Inventory table
- **Recommendation:** Either (a) extract the DAG dispatch algorithm into a second shared reference document (budget allows: only 1 of 2 permitted shared docs is used, and ~400 lines of line budget remain), or (b) add an explicit fallback plan: if the orchestrator exceeds 150 lines, what gets extracted first? The current D-7 has an explicit fallback (split Tester into 2 agents), but D-8 lacks an analogous fallback for the orchestrator.
- **Evidence:** D-8 alt-1 cons: "Orchestrator at 150 lines for full DAG+routing logic is tight." The component_design lists 7 distinct responsibilities + 5 body sections. The prior system's orchestrator was 539 lines (repo memory) — a 72% reduction to 150 lines for the most complex agent is ambitious.

### Finding A-2: Tester dual-mode design creates a cohesion tension within a single agent

- **Severity:** Major
- **Description:** D-7 merges two fundamentally different execution models into one agent file: (1) static evidence evaluation (read-only, parallelizable, side-effect-free) and (2) dynamic test execution (terminal access, application lifecycle management, singleton constraint). These modes have different concurrency models, different tool requirements, different failure modes, and different concerns. The Tester's body must contain two distinct workflow sections ("Static Evaluation Workflow" + "Dynamic Testing Workflow"), which is unique among all agents — every other agent has a single workflow section. The design acknowledges "Medium" confidence (the only non-High confidence decision) and already has a fallback plan (split into 9 agents), but doesn't define a triggering condition.
- **Affected artifacts:** design-output.yaml D-7, component_design tester (body_sections has 7 sections vs the standard 6)
- **Recommendation:** Define a crisp trigger for the D-7 fallback: "If the combined Tester file exceeds 140 lines OR the static/dynamic workflows cannot be clearly separated within the file, split into Tester (dynamic) + Verifier (static) at 9 agents total." The current "if it doesn't fit, splitting is the fallback" is insufficiently specific for an implementer to act on.
- **Evidence:** D-7 confidence: "Medium" — the only non-High decision. Body sections: 7 (vs 6 standard). Alt-2 explicitly noted as fallback. The 140-line budget for 7 body sections averages 20 lines per section — tight for two complete workflows.

### Finding A-3: Missing explicit contract between Planner file-ownership and Orchestrator DAG dispatch

- **Severity:** Major
- **Description:** The DAG execution model (D-6) requires the orchestrator to read the planner's task graph, compute topological levels, AND perform file-ownership overlap detection. The planner declares file ownership per task (FR-6.1, FR-6.2), but the design does not specify what happens when the planner's declared file ownership is incomplete or wrong — a task modifies a file it didn't declare. EC-4 acknowledges this: "Two tasks inadvertently modify the same file not declared in ownership." The mitigation is "Tester/Reviewer catches the conflict." However, this means the DAG dispatch's correctness guarantee (no parallel file conflicts) is only as strong as the planner's declarations, and violations are caught post-hoc rather than prevented.
- **Affected artifacts:** design-output.yaml D-6, EC-4, design.md Step 5 dispatch algorithm, spec-output.yaml FR-6.2
- **Recommendation:** Add an explicit constraint in the Implementer agent: "Implementer MUST NOT modify files not declared in the task's `files` list. If additional files need modification, return NEEDS_REVISION with the expanded file list." This converts a post-hoc detection problem into a preventive contract.
- **Evidence:** EC-4 states the conflict is caught by "Tester/Reviewer" — this is steps 6-7, potentially after both conflicting tasks have already written to the same file, creating merge conflicts that are expensive to resolve. The current system avoids this by using planner-assigned waves, which is a weaker but more deterministic model.

### Finding A-4: No versioning or schema evolution strategy for completion contracts

- **Severity:** Minor
- **Description:** The completion contract schema is defined once in global-rules.md and co-located in each agent's output section. If the schema evolves (e.g., adding a `confidence` field or changing `output_paths` to `outputs`), all 8 agent files plus global-rules.md must be updated simultaneously. The current system's schemas.md had this same problem at a larger scale (1,422 lines). The new design correctly reduces the surface but doesn't address the evolutionary coupling.
- **Affected artifacts:** design-output.yaml D-9, D-12, design.md Data Models & Schemas section
- **Recommendation:** Acknowledge this tradeoff explicitly in D-9 rationale: "Completion contract evolution requires updating global-rules.md + all consuming agents. This is acceptable because: (a) the contract is deliberately minimal (3 fields), (b) changes are rare (0 changes across 5 pipeline runs), (c) the cost of updating 9 files is acceptable for a 3-field schema." No structural change needed — just make the tradeoff explicit.
- **Evidence:** The completion contract schema (status, summary, output_paths) appears in design.md Data Models section, design-output.yaml schemas section, and will be replicated in each agent's Output Schema body section. The agent_output wrapper adds 3 more fields (agent, started_at, completed_at) that must also be consistent.

### Finding A-5: Prompt files (feature-workflow.prompt.md, quick-fix.prompt.md) lack definition

- **Severity:** Minor
- **Description:** The file inventory includes 2 prompt files (feature-workflow.prompt.md at ~30 lines, quick-fix.prompt.md at ~25 lines) but neither the design.md nor design-output.yaml defines their content, behavior, or contract. The implementation order lists them as step 7 of 8, but the implementer will have no design specification to work from. What does "quick-fix" allow vs the full pipeline? Which steps does it skip? Is it analogous to the current plan-and-implement.prompt.md (fast-track pipeline)?
- **Affected artifacts:** design-output.yaml file_inventory (prompt files), design.md Implementation Checklist
- **Recommendation:** Add a "Prompt Design" section or at minimum a component_design entry for each prompt file, specifying: which pipeline steps they invoke, what parameters they pass, and how quick-fix differs from feature-workflow. The current immutable minimum step set {Step 0, Step 7} from prior pipeline experience (repo memory) should be referenced if applicable.
- **Evidence:** design-output.yaml file_inventory lists both prompts with line estimates but no description beyond "Full pipeline entry point" and "Simplified pipeline." No component_design entry exists for either prompt. The immutable minimum step set {0, 7, 9} (repo memory) established precedent that reduced pipelines must never skip code review.

---

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Design uses "architecture-output.yaml" + "architecture.md" but spec and feature directory use "design-output.yaml" + "design.md" naming

- **Severity:** Major
- **Description:** The design introduces a new naming convention: the Architect agent produces `architecture-output.yaml` and `architecture.md` (design.md Step 3, design-output.yaml pipeline step 3). However, the existing feature directory pattern uses `design-output.yaml` and `design.md` (as seen in the current agent-system-refactor feature). The spec-output.yaml doesn't explicitly specify the output artifact names. This naming inconsistency will cause confusion: does the orchestrator look for `architecture-output.yaml` or `design-output.yaml`? The evidence check table says "Architecture: architecture-output.yaml exists" — is this the sole source of truth?
- **Affected artifacts:** design-output.yaml pipeline step 3 outputs, design.md Step 3, design.md State Management evidence checks
- **Recommendation:** Settle on one naming convention and apply it consistently. Since the agent is called "Architect" and the step is "Architecture," `architecture-output.yaml` is the logical choice. But the feature directory template needs to be updated to match. Document this naming migration explicitly so implementers and future pipeline users know the canonical name.
- **Evidence:** design-output.yaml step 3 outputs: ["docs/feature/<slug>/architecture-output.yaml", "docs/feature/<slug>/architecture.md"]. Current feature directory for agent-system-refactor contains: design-output.yaml, design.md (old naming). The spec doesn't use either name explicitly.

### Finding C-2: Review gate semantics differ between design.md and spec

- **Severity:** Major
- **Description:** The spec states (FR-9.3): "≥2 of 3 reviewers approve AND zero blocker findings. If the gate fails, the Orchestrator re-dispatches the **Implementer** with review findings (max 2 review rounds)." However, the design's Step 7 says: "Failure routes back to **Step 5** with findings." Step 5 is Implementation. This means review failure always goes back to implementation, but for design review (Step 3 embedded review for 🔴), review failure should route back to the **Architect** (Step 3), not the Implementer (Step 5). The embedded design review routing is unspecified.
- **Affected artifacts:** design-output.yaml pipeline step 7, step 3 (🔴 embedded review), design.md Steps 3 and 7
- **Recommendation:** Explicitly document the routing for embedded design review failure: "If embedded design review at Step 3 fails, route back to Architect with findings (max 1 round, per bounded feedback loops)." The current Step 7 failure routing is correct for code review but doesn't cover the design review case.
- **Evidence:** design.md Step 3: "[🔴 complex: embedded design review sub-phase with 2-3 Reviewers]" — no failure routing specified. design.md Step 7: "Failure routes back to Step 5 with findings. Max 2 review rounds." — only covers code review. EC-5 only mentions "Code review fails after 2 rounds."

### Finding C-3: Research gate says "≥2 researchers" but 🟡 dispatches "2-3 researchers" — edge case when exactly 2 dispatched and 1 fails

- **Severity:** Minor
- **Description:** FR-4.3 requires "≥2 of dispatched researchers to complete successfully." For 🟡 risk with 2 researchers dispatched, if 1 fails, the gate fails (1 < 2). This means for 🟡 with minimum 2 researchers, there is zero tolerance for researcher failure — both must succeed. For 🔴 with 4 researchers, up to 2 can fail. The gate is asymmetric by risk level, which may be intentional but is not documented as such.
- **Affected artifacts:** design-output.yaml pipeline step 2 gate, spec FR-4.3, design.md Risk-Based Scaling table
- **Recommendation:** Clarify whether the gate is intentionally "≥2 absolute" or "≥N/2 proportional." If absolute, document that 🟡 has no failure tolerance at minimum dispatch. If proportional, change the gate to "≥ceil(N/2) of N researchers." Either interpretation is valid — the ambiguity should be resolved.
- **Evidence:** Pipeline step 2 gate: "≥2 of N researchers return status: DONE." Risk scaling: 🟡 = 2-3 researchers, 🔴 = 4 researchers.

---

## Summary

The design is architecturally sound overall — the reduction from 6,416 to ~1,100 lines is achieved through well-justified merges, co-location of schemas, elimination of proven-unused infrastructure (SQLite evidence gates), and adoption of platform-enforced tool restrictions. The 12 architectural decisions are well-structured with clear alternatives analysis and industry traceability.

Three Major findings require attention: (1) the orchestrator's 150-line budget has no explicit fallback plan despite being the most complex agent, (2) the Tester dual-mode design needs a crisp split trigger, and (3) the file-ownership contract between Planner and Implementer needs a preventive enforcement mechanism rather than post-hoc detection. Two additional Major findings address correctness: artifact naming inconsistency and missing embedded design review failure routing. None are blocking — all are addressable through design clarifications without structural changes.
