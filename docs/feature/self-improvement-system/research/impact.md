# Impact Analysis: Self-Improvement System

## Focus Area

**impact** — Affected files, modules, and components; what needs to change and where; existing code that will be modified or extended.

## Summary

The self-improvement system requires modifications to 14 existing agent files (adding artifact evaluation sections), significant changes to the orchestrator (telemetry + tool restriction + new pipeline step), creation of 1 new agent file, 3 new storage directories, and updates to the documentation structure and feature-workflow prompt.

---

## Findings

### 1. Artifact Consumption Map (Which Agents Consume Upstream Artifacts)

The following table maps every agent to the pipeline-produced artifacts it consumes. Only agents consuming artifacts produced by OTHER agents are candidates for artifact evaluation. Agents that only consume codebase/git-diff or user-provided input (initial-request.md) are excluded.

| Consumer Agent           | Upstream Artifact(s) Consumed                                                                | Artifact Producer(s)          |
| ------------------------ | -------------------------------------------------------------------------------------------- | ----------------------------- |
| **spec**                 | research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md | researcher ×4                 |
| **designer**             | feature.md, research/\*.md                                                                   | spec, researcher ×4           |
| **ct-security**          | design.md, feature.md                                                                        | designer, spec                |
| **ct-scalability**       | design.md, feature.md                                                                        | designer, spec                |
| **ct-maintainability**   | design.md, feature.md                                                                        | designer, spec                |
| **ct-strategy**          | design.md, feature.md                                                                        | designer, spec                |
| **planner**              | design.md, feature.md, research/\*.md                                                        | designer, spec, researcher ×4 |
| **implementer**          | tasks/\*.md, feature.md, design.md                                                           | planner, spec, designer       |
| **documentation-writer** | tasks/\*.md, feature.md, design.md                                                           | planner, spec, designer       |
| **v-tests**              | verification/v-build.md                                                                      | v-build                       |
| **v-tasks**              | verification/v-build.md, plan.md, tasks/\*.md                                                | v-build, planner              |
| **v-feature**            | verification/v-build.md, feature.md                                                          | v-build, spec                 |
| **r-quality**            | design.md (via designer.mem.md)                                                              | designer                      |
| **r-testing**            | feature.md                                                                                   | spec                          |

#### Agents Excluded from Artifact Evaluation

| Agent               | Reason for Exclusion                                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **researcher** (×4) | Consumes only initial-request.md (user input) and existing codebase — no upstream agent-produced artifacts                |
| **v-build**         | Consumes only the codebase and planner memory for orientation — primary input is the build system, not pipeline artifacts |
| **r-security**      | Consumes only git diff and codebase — no structured pipeline artifact input                                               |
| **r-knowledge**     | Consumes decisions.md which is its own output from prior runs — self-referential, not cross-agent evaluation              |

### 2. Existing Agent Files Requiring Modification (PART 1 — Artifact Evaluation)

Each of these 14 agent files needs an artifact evaluation section added to their workflow and outputs:

#### 2a. `.github/agents/spec.agent.md` (150 lines)

- **Current Outputs section** (line ~35): `feature.md` + `memory/spec.mem.md`
- **Change needed:** Add artifact evaluation output for each research/\*.md file consumed
- **Evaluation targets:** research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md
- **Output addition:** Store evaluations in `artifact-evaluations/` directory or append to agent output
- **Workflow change:** Add evaluation step between reading research artifacts and producing feature.md

#### 2b. `.github/agents/designer.agent.md` (118 lines)

- **Current Outputs section** (line ~45): `design.md` + `memory/designer.mem.md`
- **Change needed:** Add artifact evaluation for feature.md (primary upstream artifact)
- **Evaluation target:** feature.md (produced by spec agent)
- **Workflow change:** Add evaluation step in workflow after reading feature.md

#### 2c. `.github/agents/ct-security.agent.md` (150 lines)

- **Current Outputs section** (line ~38): `ct-review/ct-security.md` + `memory/ct-security.mem.md`
- **Change needed:** Add artifact evaluation for design.md and feature.md
- **Evaluation targets:** design.md (designer), feature.md (spec)

#### 2d. `.github/agents/ct-scalability.agent.md` (151 lines)

- **Current Outputs section** (line ~38): `ct-review/ct-scalability.md` + `memory/ct-scalability.mem.md`
- **Change needed:** Add artifact evaluation for design.md and feature.md

#### 2e. `.github/agents/ct-maintainability.agent.md` (148 lines)

- **Current Outputs section** (line ~36): `ct-review/ct-maintainability.md` + `memory/ct-maintainability.mem.md`
- **Change needed:** Add artifact evaluation for design.md and feature.md

#### 2f. `.github/agents/ct-strategy.agent.md` (159 lines)

- **Current Outputs section** (line ~38): `ct-review/ct-strategy.md` + `memory/ct-strategy.mem.md`
- **Change needed:** Add artifact evaluation for design.md and feature.md

#### 2g. `.github/agents/planner.agent.md` (260 lines)

- **Current Outputs section** (line ~55): `plan.md`, `tasks/*.md`, `memory/planner.mem.md`
- **Change needed:** Add artifact evaluation for design.md and feature.md
- **Evaluation targets:** design.md (designer), feature.md (spec)

#### 2h. `.github/agents/implementer.agent.md` (206 lines)

- **Current Outputs section** (line ~42): Code files, test files, updated task file, `memory/implementer-<task-id>.mem.md`
- **Change needed:** Add artifact evaluation for its assigned task file, and relevant sections of feature.md and design.md
- **Evaluation targets:** tasks/<task>.md (planner), design.md (designer), feature.md (spec)

#### 2i. `.github/agents/documentation-writer.agent.md` (91 lines)

- **Current Outputs section** (line ~34): Documentation files, `memory/documentation-writer-<task-id>.mem.md`, updated task file
- **Change needed:** Add artifact evaluation for its assigned task file and relevant feature.md/design.md sections
- **Evaluation targets:** tasks/<task>.md (planner), design.md (designer), feature.md (spec)

#### 2j. `.github/agents/v-tests.agent.md` (180 lines)

- **Current Outputs section** (line ~30): `verification/v-tests.md` + `memory/v-tests.mem.md`
- **Change needed:** Add artifact evaluation for v-build.md
- **Evaluation target:** verification/v-build.md (v-build)

#### 2k. `.github/agents/v-tasks.agent.md` (169 lines)

- **Current Outputs section** (line ~31): `verification/v-tasks.md` + `memory/v-tasks.mem.md`
- **Change needed:** Add artifact evaluation for v-build.md, plan.md, and tasks/\*.md
- **Evaluation targets:** verification/v-build.md (v-build), plan.md (planner), tasks/\*.md (planner)

#### 2l. `.github/agents/v-feature.agent.md` (183 lines)

- **Current Outputs section** (line ~31): `verification/v-feature.md` + `memory/v-feature.mem.md`
- **Change needed:** Add artifact evaluation for v-build.md and feature.md
- **Evaluation targets:** verification/v-build.md (v-build), feature.md (spec)

#### 2m. `.github/agents/r-quality.agent.md` (211 lines)

- **Current Outputs section** (line ~36): `review/r-quality.md` + `memory/r-quality.mem.md`
- **Change needed:** Add artifact evaluation for design.md
- **Evaluation target:** design.md (designer)

#### 2n. `.github/agents/r-testing.agent.md` (215 lines)

- **Current Outputs section** (line ~35): `review/r-testing.md` + `memory/r-testing.mem.md`
- **Change needed:** Add artifact evaluation for feature.md
- **Evaluation target:** feature.md (spec)

### 3. Orchestrator Changes (PART 2 — Telemetry + Tool Restriction)

**File:** `.github/agents/orchestrator.agent.md` (486 lines)

#### 3a. Telemetry Tracking (PART 2)

The following sections need modification to add per-agent telemetry capture:

- **Outputs section** (lines 20–26): Add `docs/feature/<feature-slug>/agent-metrics/` to output list
- **Documentation Structure** (lines 46–69): Add `agent-metrics/` directory to the structure listing
- **Memory Lifecycle Actions** (lines 443–453): Add telemetry capture action
- **Steps 1–7** (lines 200–430): Each step that dispatches a subagent must capture:
  - `agent_name`, `execution_time_ms`, `retry_count`, `failure_reason`, `number_of_iterations`, `human_intervention_required`
- **Between-step merge operations** (1.1m, 2m, 3m, 4m, etc.): Must include telemetry log write alongside memory merge
- **Orchestrator Expectations Per Agent table** (lines 406–432): May need a column or note about telemetry capture

Estimated scope: Significant scattered changes across the 486-line file — every dispatch point needs telemetry wrapping.

#### 3b. Tool Restriction (Additional Requirement)

The orchestrator currently performs direct file operations in several places:

- **Step 0 (line 178–179):** Creates `initial-request.md` and `memory.md` directly
- **Step 0 (line 200):** Creates `memory/` directory directly
- **Memory merges (1.1m, 2m, 3m, 4m):** Orchestrator writes to `memory.md` directly
- **Memory pruning, invalidation, cleaning:** Orchestrator modifies `memory.md` directly

The tool restriction request (`[agent, agent/runSubagent, memory]`) means:

1. The orchestrator can NO LONGER create files directly — must delegate `initial-request.md` creation and `memory.md` initialization to a subagent
2. All memory merge/prune/invalidate operations that write to `memory.md` must be delegated to subagents OR use only the `memory` tool
3. **Global Rule 1** (line 30): "Never modify code or documentation directly" — already aligned, but the file creation in Step 0 contradicts this
4. **Operating Rules section** (line 73–80): Must be updated to reference allowed tools only
5. **All instructions referencing direct file writes** must be reworded to delegate to subagents

Impact: Fundamental change to how the orchestrator handles setup and memory operations.

#### 3c. New Pipeline Step — Post-Mortem (PART 3)

- **Workflow Steps section:** Add Step 8 (Post-Mortem) after Step 7 (R Cluster)
- **Parallel Execution Summary** (lines 457–468): Add Step 8
- **Completion Contract** (lines 470–476): May need updating — currently workflow completes when R cluster determines DONE; now it should also run post-mortem (but post-mortem should be non-blocking)
- **Orchestrator Expectations Per Agent table** (lines 406–432): Add row for post-mortem agent

### 4. New Agent File (PART 3 — Post-Mortem Agent)

**New file:** `.github/agents/post-mortem.agent.md`

Must follow the existing agent file conventions:

- YAML frontmatter with `name` and `description`
- Standard Operating Rules section (matching patterns from other agents)
- Inputs section listing: all artifact-evaluation files, all agent-metrics files, memory.md
- Outputs section: `docs/feature/<feature-slug>/post-mortems/<timestamp>-report.md` + `memory/post-mortem.mem.md`
- Completion Contract: `DONE:` / `ERROR:`
- Anti-Drift Anchor
- Read-only enforcement (does not modify source code or existing artifacts)

### 5. New Storage Directories (PART 4)

Three new directories under `docs/feature/<feature-slug>/`:

| Directory               | Purpose                                            | Contents                                                               |
| ----------------------- | -------------------------------------------------- | ---------------------------------------------------------------------- |
| `agent-metrics/`        | Orchestrator telemetry logs                        | Per-run structured YAML/JSON with agent execution metadata             |
| `artifact-evaluations/` | Structured evaluation output from consuming agents | Per-agent evaluation YAML files (e.g., `spec-evaluates-research.yaml`) |
| `post-mortems/`         | Post-mortem agent output                           | Timestamped structured reports                                         |

These directories must be:

- Added to the orchestrator's **Documentation Structure** section (lines 46–69)
- Created during **Step 0 Setup** or on first write
- Listed in the orchestrator's **Outputs** section

### 6. Impact on Documentation Structure

**File:** `.github/agents/orchestrator.agent.md`, **Documentation Structure section** (lines 46–69)

Current structure listing needs additions:

```
- agent-metrics/          # Per-run execution telemetry (orchestrator writes)
- artifact-evaluations/   # Structured evaluations from consuming agents
- post-mortems/           # Post-mortem analysis reports
```

### 7. Impact on the Memory System

- **New memory file:** `memory/post-mortem.mem.md` — needs to be recognized by the orchestrator
- **Memory merge for post-mortem:** The orchestrator's merge step after post-mortem completes must be defined
- **Memory Lifecycle Actions table** (lines 443–453): Add post-mortem merge action
- **Telemetry data** in agent-metrics/ is separate from the memory system — no direct memory impact, but orchestrator may reference it during merge logging
- **Artifact evaluations** stored in `artifact-evaluations/` are separate from memory files but must be referenced in the Artifact Index of each evaluating agent's isolated memory

### 8. Impact on Existing Completion Contracts

**No breaking changes to completion contracts.** The existing DONE:/NEEDS_REVISION:/ERROR: three-state contract remains unchanged for all agents. Artifact evaluations are ADDITIONAL output sections, not replacements. Key observations:

- All 21 existing agent files use `DONE:` / `ERROR:` (some also support `NEEDS_REVISION:` via orchestrator routing)
- The post-mortem agent will follow the same contract pattern
- The orchestrator's completion contract (line 470) currently ties to R cluster DONE — this must be extended to include post-mortem (recommended: post-mortem is non-blocking, so R cluster DONE still triggers workflow completion, post-mortem runs as a final optional step)

### 9. Impact on Feature-Workflow Prompt

**File:** `.github/prompts/feature-workflow.prompt.md`

- May need updating to mention the post-mortem step in the Rules section
- The Key Artifacts table should include `agent-metrics/`, `artifact-evaluations/`, `post-mortems/`

### 10. Files That Will NOT Be Modified (Safety Constraint)

Per PART 5 of the initial request, these files must NOT be changed:

| File                                  | Reason for No Change                                                                |
| ------------------------------------- | ----------------------------------------------------------------------------------- |
| `.github/agents/researcher.agent.md`  | Only consumes initial-request.md and codebase — no cross-agent artifact consumption |
| `.github/agents/v-build.agent.md`     | Only consumes codebase — no pipeline artifact evaluation applicable                 |
| `.github/agents/r-security.agent.md`  | Only consumes git diff and codebase — no structured artifact consumption            |
| `.github/agents/dispatch-patterns.md` | Reference document for dispatch patterns — not an agent, no evaluation logic        |
| `LICENSE`                             | License file — not part of the agent system                                         |
| `README.md`                           | Project README — not part of the agent pipeline                                     |
| All existing artifact formats         | Per safety requirement: "Do not modify artifact formats"                            |

**Note on r-knowledge.agent.md:** It consumes `decisions.md` which is its own output from prior runs (self-referential). Including it in artifact evaluation is debatable. The initial request specifies agents that consume artifacts from OTHER agents. R-Knowledge is borderline — it could be included to evaluate the quality of prior decisions.md entries, but this is a design decision, not an impact finding.

---

## File References

| File/Directory                                 | Relevance                                                                                    |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `.github/agents/orchestrator.agent.md`         | Major modification — telemetry, tool restriction, new step 8, documentation structure update |
| `.github/agents/spec.agent.md`                 | Modification — add artifact evaluation for research/\*.md                                    |
| `.github/agents/designer.agent.md`             | Modification — add artifact evaluation for feature.md                                        |
| `.github/agents/ct-security.agent.md`          | Modification — add artifact evaluation for design.md, feature.md                             |
| `.github/agents/ct-scalability.agent.md`       | Modification — add artifact evaluation for design.md, feature.md                             |
| `.github/agents/ct-maintainability.agent.md`   | Modification — add artifact evaluation for design.md, feature.md                             |
| `.github/agents/ct-strategy.agent.md`          | Modification — add artifact evaluation for design.md, feature.md                             |
| `.github/agents/planner.agent.md`              | Modification — add artifact evaluation for design.md, feature.md                             |
| `.github/agents/implementer.agent.md`          | Modification — add artifact evaluation for tasks/\*.md, design.md, feature.md                |
| `.github/agents/documentation-writer.agent.md` | Modification — add artifact evaluation for tasks/\*.md, design.md, feature.md                |
| `.github/agents/v-tests.agent.md`              | Modification — add artifact evaluation for v-build.md                                        |
| `.github/agents/v-tasks.agent.md`              | Modification — add artifact evaluation for v-build.md, plan.md                               |
| `.github/agents/v-feature.agent.md`            | Modification — add artifact evaluation for v-build.md, feature.md                            |
| `.github/agents/r-quality.agent.md`            | Modification — add artifact evaluation for design.md                                         |
| `.github/agents/r-testing.agent.md`            | Modification — add artifact evaluation for feature.md                                        |
| `.github/agents/post-mortem.agent.md`          | **NEW FILE** — post-mortem agent definition                                                  |
| `.github/prompts/feature-workflow.prompt.md`   | Minor modification — add post-mortem step and new artifact references                        |
| `docs/feature/<slug>/agent-metrics/`           | **NEW DIRECTORY** — telemetry storage                                                        |
| `docs/feature/<slug>/artifact-evaluations/`    | **NEW DIRECTORY** — evaluation storage                                                       |
| `docs/feature/<slug>/post-mortems/`            | **NEW DIRECTORY** — post-mortem report storage                                               |

---

## Assumptions & Limitations

1. **Assumed evaluation scope:** Only agents consuming artifacts produced by OTHER pipeline agents need artifact evaluation sections. Agents consuming only codebase, git diff, or user input are excluded.
2. **Assumed non-blocking post-mortem:** The post-mortem agent is expected to run after the pipeline completes and should not block the workflow's completion contract.
3. **Tool restriction interpretation:** The orchestrator tool restriction (`[agent, agent/runSubagent, memory]`) is interpreted as removing the orchestrator's ability to use file creation/modification tools directly. The `memory` tool reference may mean a dedicated memory management capability rather than the current direct file writes.
4. **r-knowledge exclusion is debatable:** R-Knowledge's consumption of decisions.md is self-referential. Whether to include it as an artifact evaluation candidate is a design decision.
5. **Artifact evaluation storage location:** The initial request says evaluations should be "appended to agent output" OR "stored in a dedicated evaluation directory." Both options need consideration; the analysis assumes a dedicated `artifact-evaluations/` directory as the primary approach.
6. **No examination of runtime platform constraints:** The analysis covers .agent.md file changes only. Platform-specific tool restriction enforcement (e.g., how `agent` and `memory` tools are configured) was not examined.

---

## Open Questions

1. **Dual storage for evaluations:** Should artifact evaluations be BOTH appended to the agent's primary output AND stored in `artifact-evaluations/`? Or only one location?
2. **Telemetry granularity:** Should the orchestrator track telemetry for cluster-level operations (CT cluster, V cluster, R cluster) in addition to individual agent invocations?
3. **Post-mortem triggering:** Should the post-mortem run automatically after every pipeline completion, or only on demand? The initial request says "after the workflow completes" suggesting automatic.
4. **Tool restriction migration:** How should the orchestrator's current Step 0 file creation operations be restructured? Options: (a) new setup subagent, (b) first researcher creates files, (c) memory tool handles it.
5. **R-Knowledge inclusion:** Should r-knowledge.agent.md receive artifact evaluation for decisions.md from prior runs?
6. **Evaluation for research artifacts:** The spec agent consumes 4 research files. Should it produce 4 separate evaluations or one combined evaluation?

---

## Research Metadata

- **confidence_level:** high — all 21 agent files were examined; artifact consumption chains fully traced
- **coverage_estimate:** Complete coverage of all `.github/agents/*.agent.md` files, `.github/prompts/feature-workflow.prompt.md`, `.github/agents/dispatch-patterns.md`, and the `docs/feature/` directory structure. Every agent's Inputs and Outputs sections were read.
- **gaps:**
  - Platform/runtime tool configuration was not examined (only the `.agent.md` instruction files)
  - No `.github/instructions/` directory was found — if one exists, it would need updates
  - The `memory` tool referenced in the tool restriction is not clearly defined in the current codebase — its capabilities need clarification during design
