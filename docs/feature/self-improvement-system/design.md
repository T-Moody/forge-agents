# Design: Self-Improvement System for Forge (Revision 1)

**Summary:** Additive cross-agent evaluation, orchestrator telemetry, and post-mortem analysis system for the Forge 8-step pipeline. Adds evaluation workflow steps to 14 consuming agents (referencing a shared evaluation schema), telemetry passed via orchestrator context to a new PostMortem agent (Step 8), and 3 new per-feature storage directories. Orchestrator write-tool restriction (separate implementation task) prevents direct file creation/modification while retaining read tools for cluster decisions.

**Revision:** Addresses CT cluster findings (1 Critical, 6 High). Key changes: (1) orchestrator keeps read tools, restricts only write tools; (2) tool restriction separated as independent implementation task; (3) telemetry moved out of memory.md — orchestrator passes via context to PostMortem; (4) evaluation schema centralized in shared reference document; (5) PostMortem uses memory-first reading pattern; (6) improvement_recommendations removed from PostMortem output; (7) memory tool scope dependency eliminated for telemetry.

---

## CT Revision Summary

| #   | CT Source                                | Severity | Finding                                                                                        | Resolution                                                                            | Design Section                       |
| --- | ---------------------------------------- | -------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------ |
| 1   | ct-strategy                              | Critical | Tool restriction removes read_file/grep_search/semantic_search — breaks cluster decision flows | Keep read-only tools; restrict ONLY write tools                                       | §Orchestrator Write-Tool Restriction |
| 2   | ct-strategy                              | High     | Bundling restrictive + additive changes compounds risk                                         | Tool restriction designated as independent implementation task (Track B)              | §Implementation Checklist            |
| 3   | ct-security, ct-scalability, ct-strategy | High     | memory.md telemetry "never pruned" → unbounded growth visible to all agents                    | Telemetry NOT in memory.md; orchestrator accumulates in context, passes to PostMortem | §Telemetry Accumulation              |
| 4   | ct-maintainability                       | High     | Evaluation schema duplicated across 14 agents with no shared template                          | Schema defined ONCE in `.github/agents/evaluation-schema.md`; agents reference it     | §Shared Evaluation Schema            |
| 5   | ct-scalability                           | High     | PostMortem ingests 40-50+ files with no pagination                                             | PostMortem uses memory-first reading pattern with targeted reads                      | §PostMortem Agent Definition         |
| 6   | ct-strategy                              | High     | improvement_recommendations overlaps R-Knowledge scope                                         | Field removed; PostMortem produces quantitative output only                           | §Post-Mortem Report Schema           |
| 7   | ct-security, ct-maintainability          | High     | memory tool scope unverified — telemetry depends on it                                         | Telemetry no longer depends on memory tool; passed via orchestrator context           | §Telemetry Accumulation              |

---

## Context & Inputs

| Source                                                             | Key Content Used                                                                                                         |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| [initial-request.md](initial-request.md)                           | 6-part feature request + orchestrator tool restriction                                                                   |
| [feature.md](feature.md)                                           | FR-1–FR-8, NFR-1–NFR-7, AC-1–AC-14, EC-1–EC-11, TS-1–TS-15                                                               |
| [research/architecture.md](research/architecture.md)               | §Repository Structure, §Agent Definition Format, §Pipeline Architecture, §Memory System, §Tool Usage Patterns            |
| [research/impact.md](research/impact.md)                           | §Artifact Consumption Map, §Existing Agent Files Requiring Modification, §Orchestrator Changes                           |
| [research/dependencies.md](research/dependencies.md)               | §Artifact Consumption Chain, §Orchestrator Dispatch Patterns, §Orchestrator Tool Restriction                             |
| [research/patterns.md](research/patterns.md)                       | §Agent File Format Pattern, §Non-Blocking Agent Pattern, §YAML/Structured Output Patterns, §Completion Contract Patterns |
| [ct-review/ct-security.md](ct-review/ct-security.md)               | §Findings — tool restriction breaking change, memory tool scope, telemetry accumulation                                  |
| [ct-review/ct-scalability.md](ct-review/ct-scalability.md)         | §Findings — telemetry growth, PostMortem context saturation, orchestrator definition size                                |
| [ct-review/ct-maintainability.md](ct-review/ct-maintainability.md) | §Findings — schema duplication, orchestrator complexity, telemetry DSL                                                   |
| [ct-review/ct-strategy.md](ct-review/ct-strategy.md)               | §Findings — tool restriction breaks pipeline, bundling risk, telemetry bloat, PostMortem/R-Knowledge overlap             |
| [memory/spec.mem.md](memory/spec.mem.md)                           | Evaluation storage decisions, PostMortem two-state contract, per-feature scoping                                         |
| [memory/researcher-\*.mem.md](memory/)                             | Architecture, Impact, Dependencies, Patterns key findings                                                                |
| [memory/ct-\*.mem.md](memory/)                                     | CT cluster key findings and artifact indexes                                                                             |

---

## High-level Architecture

### System Overview

Three subsystems are added to the existing pipeline, plus one cross-cutting change implemented as a separate task:

| Subsystem                                                 | Description                                                                                    | Components Affected                                     |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| **Artifact Evaluation**                                   | 14 agents gain an additive evaluation step referencing a shared schema document                | 14 `.agent.md` files (additive only) + 1 new schema doc |
| **Orchestrator Telemetry**                                | Orchestrator accumulates dispatch telemetry in context; passes to PostMortem at Step 8         | `orchestrator.agent.md` (modified)                      |
| **PostMortem Analysis**                                   | New agent runs as Step 8; receives telemetry + reads evaluations; produces quantitative report | `post-mortem.agent.md` (new)                            |
| **Orchestrator Write-Tool Restriction** _(separate task)_ | Remove write/execute tools from orchestrator; retain read tools for cluster decisions          | `orchestrator.agent.md` (modified independently)        |
| **Storage Architecture**                                  | 3 new per-feature directories for evaluations, metrics, and reports                            | Orchestrator docs structure (updated)                   |

> **Separation principle (CT-2):** The orchestrator write-tool restriction is designed as an **independent implementation task** that can be implemented and validated separately from the evaluation/telemetry/PostMortem changes. Neither depends on the other at runtime.

### Data Flow

```
Pipeline Steps 1–7 (existing behavior preserved):
  Each of 14 consuming agents:
    [Primary Output] ──────────────── → existing artifact path
    [Evaluation Output] ────────────── → artifact-evaluations/<agent>.md
    [Isolated Memory] ──────────────── → memory/<agent>.mem.md (existing)

  Orchestrator after each dispatch:
    [Observes completion contract + reads memory file]
    [Accumulates telemetry data in working context]

Step 8 (new — PostMortem):
  Orchestrator passes accumulated telemetry as dispatch context ──→ PostMortem
  PostMortem Agent reads:
    ← memory/*.mem.md (memory files first — orientation)
    ← artifact-evaluations/*.md (targeted reads based on memory findings)
  PostMortem Agent writes:
    → agent-metrics/<run-date>-run-log.md (structured run log from telemetry)
    → post-mortems/<run-date>-post-mortem.md (quantitative analysis report)
    → memory/post-mortem.mem.md (isolated memory)
```

> **Key change from v0 (CT-3, CT-7):** Telemetry data does NOT live in `memory.md`. The orchestrator accumulates telemetry in its own context window and passes it directly to PostMortem in the Step 8 dispatch prompt. This eliminates unbounded growth in the shared memory file and removes the dependency on the `memory` tool for telemetry writes.

### Component Responsibilities

| Component                | Responsibility                                                                                       | Reads                                                                                      | Writes                                            |
| ------------------------ | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| Shared Evaluation Schema | Single source of truth for evaluation YAML format                                                    | (reference document)                                                                       | N/A                                               |
| 14 Evaluating Agents     | Produce structured evaluations as secondary output after primary work                                | Upstream artifacts (existing) + evaluation-schema.md (reference)                           | Primary output (existing) + evaluation file (new) |
| Orchestrator             | Accumulate telemetry in context; dispatch PostMortem at Step 8 with telemetry data                   | Subagent completion contracts + isolated memories (existing, via read tools)               | memory.md (existing, via memory tool)             |
| PostMortem Agent         | Write run-log from orchestrator-provided telemetry; analyze evaluations; produce quantitative report | evaluation files, memory files (memory-first pattern), telemetry from orchestrator context | run-log, post-mortem report, isolated memory      |

### Boundary: PostMortem vs. R-Knowledge (Revised — CT-6)

| Aspect               | PostMortem Agent                                                            | R-Knowledge Agent                                                               |
| -------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Focus**            | Quantitative cross-agent metrics — scores, retry rates, frequencies, counts | Qualitative pattern analysis — knowledge evolution, forward-looking suggestions |
| **Data Sources**     | Structured evaluations (YAML), orchestrator-provided telemetry              | Pipeline artifacts, codebase, git diff                                          |
| **Outputs**          | Post-mortem report (metrics only), run-log                                  | knowledge-suggestions.md, decisions.md, r-knowledge.md                          |
| **Timing**           | Step 8 (after R cluster)                                                    | Step 7 (R cluster member)                                                       |
| **Blocking**         | Non-blocking (ERROR does not affect pipeline)                               | Non-blocking (existing behavior)                                                |
| **Does NOT produce** | improvement_recommendations, knowledge-suggestions.md, decisions.md         | post-mortem reports, run-logs                                                   |

> **Key change (CT-6):** PostMortem no longer produces `improvement_recommendations`. It identifies WHAT is happening (patterns, frequencies, scores) but does NOT prescribe WHAT TO DO. R-Knowledge remains the sole source of qualitative improvement suggestions. This eliminates the boundary blur identified by CT-strategy and CT-maintainability.

---

## Data Models & DTOs

### Artifact Evaluation Schema (Shared Reference — CT-4)

The evaluation schema is defined **once** in `.github/agents/evaluation-schema.md` — a shared reference document. All 14 evaluating agents and the PostMortem agent reference this document. Schema changes require updating only this single file, not 14+ agent definitions.

**Schema contents (defined in evaluation-schema.md):**

```yaml
artifact_evaluation:
  evaluator: "<agent-name>" # string, must match the evaluating agent's name
  source_artifact: "<relative path>" # string, path relative to feature directory
  usefulness_score: <integer 1-10> # how useful was this artifact for completing the task
  clarity_score: <integer 1-10> # how clear and well-structured was the artifact
  useful_elements:
    - "<specific section or element>" # ≥1 entry; use "N/A — none identified" if empty
  missing_information:
    - "<concrete missing detail>" # ≥1 entry; use "N/A — none identified" if empty
  information_not_used:
    - "<info read but not used>" # ≥1 entry; use "N/A — none identified" if empty
  inaccuracies:
    - "<incorrect assumption or error>" # ≥1 entry; use "N/A — none identified" if empty
  impact_on_work:
    - "<rework or clarification needed>" # ≥1 entry; use "N/A — none identified" if empty
```

**Evaluation Error Fallback (also in evaluation-schema.md):**

```yaml
evaluation_error:
  evaluator: "<agent-name>"
  source_artifact: "<path>"
  error: "<description of what failed>"
```

**Rules (defined in reference document):**

- One YAML block per source artifact, each in its own fenced code block
- All list fields MUST have ≥1 entry. Use `"N/A — none identified"` if nothing applies
- Scores MUST be integers 1–10
- Evaluation failure MUST NOT cause agent's completion status to be ERROR
- Evaluation is secondary to primary output

> **Rationale (CT-4):** Centralizing the schema in a single reference document eliminates the 14-file duplication identified by CT-maintainability. Schema evolution requires updating one file. Version information can be added to the reference document to support future schema migration.

### Telemetry Data Structure (Revised — CT-3, CT-7)

Telemetry is **not stored in any intermediate file**. The orchestrator accumulates dispatch telemetry in its working context throughout the pipeline run. At Step 8, it includes the telemetry data in the PostMortem dispatch prompt.

**Telemetry fields per agent dispatch (per FR-2.2):**

| Field                       | Type           | Description                                         |
| --------------------------- | -------------- | --------------------------------------------------- |
| agent_name                  | string         | Dispatched agent name                               |
| pipeline_step               | string         | Step identifier (1.1, 2, 3, 3b, 4, 5, 6.1, 6.2, 7)  |
| dispatch_pattern            | string         | A, B, C, or sequential                              |
| start_timestamp             | string         | Best-effort ISO 8601 or date only                   |
| end_timestamp               | string         | Best-effort; same as start if precision unavailable |
| retry_count                 | integer        | 0 if no retries                                     |
| completion_status           | string         | DONE, NEEDS_REVISION, or ERROR                      |
| failure_reason              | string or null | Reason if ERROR, else null                          |
| iteration_number            | integer        | For Pattern C loops                                 |
| human_intervention_required | boolean        | Whether human was needed                            |

**Cluster summary fields:**

| Field            | Type    | Description                 |
| ---------------- | ------- | --------------------------- |
| pipeline_step    | string  | Step identifier             |
| cluster_name     | string  | research, ct, v, r          |
| total_dispatched | integer | Agents dispatched           |
| total_errors     | integer | Agents returning ERROR      |
| cluster_outcome  | string  | DONE, NEEDS_REVISION, ERROR |

The orchestrator includes this data in a structured format within the PostMortem dispatch prompt. PostMortem expands it into the formal run-log YAML.

> **Key change (CT-3, CT-7):** No compact telemetry DSL. No `## Telemetry` section in memory.md. No "never prune" policy. Telemetry flows directly from orchestrator context to PostMortem, eliminating the implicit parsing contract between the two agents (CT-maintainability F4) and the unbounded memory.md growth (CT-scalability F1).

### Run-Log File Format (YAML — written by PostMortem)

PostMortem writes the run-log from the orchestrator-provided telemetry data:

````markdown
# Pipeline Run Log

> Timestamp precision limited by runtime. Timestamps reflect best-effort date context.

**Feature:** <feature-slug>
**Run Date:** <run-date>

## Agent Telemetry

```yaml
agent_telemetry:
  agent_name: "researcher-architecture"
  pipeline_step: "1.1"
  dispatch_pattern: "A"
  start_timestamp: "2026-02-20"
  end_timestamp: "2026-02-20"
  retry_count: 0
  completion_status: "DONE"
  failure_reason: null
  iteration_number: 1
  human_intervention_required: false
```

```yaml
agent_telemetry:
  agent_name: "spec"
  pipeline_step: "2"
  ...
```

## Cluster Summaries

```yaml
cluster_summary:
  pipeline_step: "1.1"
  cluster_name: "research"
  total_dispatched: 4
  total_errors: 0
  cluster_outcome: "DONE"
```
````

### Post-Mortem Report Schema (Revised — CT-6)

```yaml
post_mortem_report:
  run_date: "<YYYY-MM-DD>"
  feature_slug: "<slug>"
  summary: "<one-line natural-language summary>"

  recurring_issues:
    - issue: "<description>"
      affected_agents: ["<agent1>", "<agent2>"]
      frequency: <integer>

  bottlenecks:
    - agent: "<agent-name>"
      reason: "<why bottleneck>"
      retry_count: <integer>

  most_common_missing_information:
    - item: "<missing info>"
      reported_by: ["<agent1>", "<agent2>"]
      source_artifact: "<artifact path>"

  agent_accuracy_scores:
    - agent: "<producer-agent-name>"
      average_usefulness: <float 1.0-10.0>
      average_clarity: <float 1.0-10.0>
      total_inaccuracies_reported: <integer>
```

> **Key change (CT-6):** `improvement_recommendations` field **removed**. PostMortem produces only quantitative metrics and pattern identification. No qualitative "what to do" suggestions — that is R-Knowledge's exclusive scope.

### Evaluation File Format

Each file in `artifact-evaluations/` is a Markdown file containing one or more YAML fenced code blocks:

````markdown
# Artifact Evaluations by <Agent Name>

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "spec"
  source_artifact: "research/architecture.md"
  usefulness_score: 8
  ...
```

```yaml
artifact_evaluation:
  evaluator: "spec"
  source_artifact: "research/impact.md"
  usefulness_score: 7
  ...
```
````

---

## APIs & Interfaces

### Shared Evaluation Schema Document (New — CT-4)

**File:** `.github/agents/evaluation-schema.md`

This is a reference document (not an agent definition) containing:

1. The `artifact_evaluation` YAML schema with field descriptions and constraints
2. The `evaluation_error` fallback schema
3. Rules for list fields, score ranges, and non-blocking behavior
4. A version identifier for future schema evolution

All 14 evaluating agents reference this document in their evaluation workflow step. PostMortem references it for parsing expectations. This eliminates the 14-file schema duplication identified by CT-maintainability F1.

### Evaluating Agent Changes (14 Agents — Revised)

Each of the 14 evaluating agents requires three additive changes:

#### 1. Outputs Section Addition

Add to each agent's `## Outputs` section:

```markdown
- docs/feature/<feature-slug>/artifact-evaluations/<agent-name>.md (artifact evaluation — secondary, non-blocking)
```

For task-ID agents:

- `artifact-evaluations/implementer-<task-id>.md`
- `artifact-evaluations/documentation-writer-<task-id>.md`

#### 2. Workflow Step Addition (Revised — CT-4)

Insert a new step between primary work completion and self-verification (or memory writing if no self-verification exists). The step template references the shared schema instead of inlining it:

```markdown
### N. Evaluate Upstream Artifacts

After completing your primary work, evaluate each upstream pipeline-produced artifact you consumed.

For each source artifact, produce one `artifact_evaluation` YAML block following the schema defined in `.github/agents/evaluation-schema.md`. Write all blocks to: `docs/feature/<feature-slug>/artifact-evaluations/<agent-name>.md`.

**Rules:**

- Follow all rules specified in the evaluation schema reference document
- If evaluation generation fails, write an `evaluation_error` block instead (see `.github/agents/evaluation-schema.md` Rule 4) and proceed — evaluation failure MUST NOT cause your completion status to be ERROR
- Evaluation is secondary to your primary output
```

> **Key change (CT-4):** Agents reference `.github/agents/evaluation-schema.md` instead of each inlining the full schema. Schema changes propagate from one document.

#### 3. Per-Agent Evaluation Targets

| Evaluating Agent     | Source Artifacts to Evaluate                                                                 | Evaluation File                                          |
| -------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| spec                 | research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md | `artifact-evaluations/spec.md`                           |
| designer             | feature.md                                                                                   | `artifact-evaluations/designer.md`                       |
| ct-security          | design.md, feature.md                                                                        | `artifact-evaluations/ct-security.md`                    |
| ct-scalability       | design.md, feature.md                                                                        | `artifact-evaluations/ct-scalability.md`                 |
| ct-maintainability   | design.md, feature.md                                                                        | `artifact-evaluations/ct-maintainability.md`             |
| ct-strategy          | design.md, feature.md                                                                        | `artifact-evaluations/ct-strategy.md`                    |
| planner              | design.md, feature.md                                                                        | `artifact-evaluations/planner.md`                        |
| implementer          | tasks/\<task\>.md, design.md, feature.md                                                     | `artifact-evaluations/implementer-<task-id>.md`          |
| documentation-writer | tasks/\<task\>.md, design.md, feature.md                                                     | `artifact-evaluations/documentation-writer-<task-id>.md` |
| v-tests              | verification/v-build.md                                                                      | `artifact-evaluations/v-tests.md`                        |
| v-tasks              | verification/v-build.md, plan.md, tasks/\*.md                                                | `artifact-evaluations/v-tasks.md`                        |
| v-feature            | verification/v-build.md, feature.md                                                          | `artifact-evaluations/v-feature.md`                      |
| r-quality            | design.md                                                                                    | `artifact-evaluations/r-quality.md`                      |
| r-testing            | feature.md                                                                                   | `artifact-evaluations/r-testing.md`                      |

#### Agents Excluded (NO Modifications)

| Agent           | Reason                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------- |
| researcher (×4) | Consumes only initial-request.md (user input) and codebase — no upstream agent-produced artifacts |
| v-build         | Consumes only codebase — no pipeline artifact evaluation applicable                               |
| r-security      | Consumes only git diff and codebase                                                               |
| r-knowledge     | Excluded per FR-1.8 / AC-3                                                                        |

### Orchestrator Changes

#### Write-Tool Restriction (FR-6 — Revised per CT-1, CT-2)

> **IMPORTANT: This is a separate implementation task (Track B)** that can be implemented and validated independently of the evaluation/telemetry/PostMortem changes (Track A).

**Revised tool policy:** The orchestrator retains read-only tools and loses write/execute tools:

| Retained Tools               | Purpose                                                                   |
| ---------------------------- | ------------------------------------------------------------------------- |
| `agent`, `agent/runSubagent` | Dispatch subagents (core orchestration)                                   |
| `memory`                     | Memory operations (memory-scoped files)                                   |
| `read_file`                  | Read agent memory files for cluster decisions, read artifacts for routing |
| `grep_search`                | Search for specific content in files                                      |
| `semantic_search`            | Semantic code/doc search                                                  |
| `file_search`                | Discover files by glob pattern                                            |
| `list_dir`                   | List directory contents                                                   |
| `get_errors`                 | Check for compile/lint errors                                             |

| Removed Tools                  | Reason                |
| ------------------------------ | --------------------- |
| `create_file`                  | Delegate to subagents |
| `replace_string_in_file`       | Delegate to subagents |
| `multi_replace_string_in_file` | Delegate to subagents |
| `run_in_terminal`              | Delegate to subagents |

**Rationale (CT-1):** The orchestrator's cluster decision flows (CT at Step 3b, V at Step 6, R at Step 7) depend on reading `memory/<agent>.mem.md` files to extract severity values and determine outcomes. Removing `read_file` would break these core coordination flows. The goal of the restriction — "the orchestrator only orchestrates" — is achieved by removing write/execute tools while retaining reads needed for orchestration decisions.

**FR-6 deviation:** FR-6.1 specifies `[agent, agent/runSubagent, memory]` only. This design deviates by retaining read tools, based on CT-strategy's Critical finding that removing reads breaks the pipeline. The spirit of FR-6 (orchestrator delegates all file modifications) is preserved.

**Operating Rules update:**

```markdown
5. **Tool access:** The orchestrator's write/execute tools are restricted. Allowed tools: `[agent, agent/runSubagent, memory, read_file, grep_search, semantic_search, file_search, list_dir, get_errors]`. The orchestrator MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, or `run_in_terminal`. All file creation and modification MUST be delegated to subagents via `runSubagent`.
```

**Global Rule 1 update:**

```markdown
1. Never modify code, documentation, or any file directly — delegate all file creation and modification to subagents via `runSubagent`. Use read tools (`read_file`, `grep_search`, `semantic_search`, `file_search`, `list_dir`) freely for orchestration decisions.
```

#### Step 0 Modifications

With write-tool restriction (Track B), Step 0 file creation is delegated:

```markdown
### 0. Setup

1. **Initialize memory:** Use the `memory` tool to create `docs/feature/<feature-slug>/memory.md` with the empty template. If the `memory` tool cannot create this file, delegate to a setup subagent via `runSubagent`.
2. **Create initial-request.md:** Delegate to a setup subagent, or pass the user's feature request text as context to each subagent invocation — the first writing agent creates `initial-request.md`.
3. **Directory creation (lazy):** The 3 new directories (`agent-metrics/`, `artifact-evaluations/`, `post-mortems/`) are created lazily by the first agent that writes to them. No explicit directory creation in Step 0.
4. If memory.md cannot be created, log warning and proceed — memory is beneficial but not required.
```

> **Note:** Step 0 does NOT initialize a Telemetry section in memory.md. Telemetry is accumulated in the orchestrator's working context, not stored in memory.md.

#### Telemetry Accumulation (Revised — CT-3, CT-7)

**Replaces the previous "telemetry recording in memory.md" design.**

The orchestrator accumulates telemetry data in its working context as the pipeline executes. After each `runSubagent` return:

1. The orchestrator observes the completion contract (DONE/NEEDS_REVISION/ERROR)
2. The orchestrator reads the agent's isolated memory file (using `read_file`)
3. The orchestrator notes the dispatch metadata (agent name, step, pattern, status, retries, failure reason, iteration number, human intervention, timestamp) in its working context

No file writes are performed for telemetry during Steps 1–7. The telemetry data exists only in the orchestrator's context window.

At Step 8, the orchestrator includes the full telemetry dataset in the PostMortem dispatch prompt. PostMortem writes the structured run-log. This satisfies FR-2.7 (telemetry writing delegated to a subagent — PostMortem is the writing subagent).

**Telemetry dispatch prompt format:** The orchestrator includes a structured telemetry summary when dispatching PostMortem:

```markdown
## Dispatch Telemetry

**Feature:** self-improvement-system
**Run Date:** 2026-02-20

### Agent Dispatches

| Agent                   | Step | Pattern    | Status | Retries | Failure | Iteration | Human | Timestamp  |
| ----------------------- | ---- | ---------- | ------ | ------- | ------- | --------- | ----- | ---------- |
| researcher-architecture | 1.1  | A          | DONE   | 0       | —       | 1         | no    | 2026-02-20 |
| researcher-impact       | 1.1  | A          | DONE   | 0       | —       | 1         | no    | 2026-02-20 |
| spec                    | 2    | sequential | DONE   | 0       | —       | 1         | no    | 2026-02-20 |
| designer                | 3    | sequential | DONE   | 0       | —       | 1         | no    | 2026-02-20 |
| ...                     | ...  | ...        | ...    | ...     | ...     | ...       | ...   | ...        |

### Cluster Summaries

| Step | Cluster  | Dispatched | Errors | Outcome |
| ---- | -------- | ---------- | ------ | ------- |
| 1.1  | research | 4          | 0      | DONE    |
| 3b   | ct       | 4          | 0      | DONE    |
| ...  | ...      | ...        | ...    | ...     |
```

> **Rationale (CT-3):** Moving telemetry out of memory.md eliminates unbounded growth in shared memory visible to all 19 agents (~3.5–7KB per run, never pruned). Telemetry data is now ephemeral in the orchestrator's context until written by PostMortem.
>
> **Rationale (CT-7):** This removes the dependency on the `memory` tool for telemetry operations entirely. The `memory` tool is used only for its existing purposes (memory.md lifecycle).
>
> **Resilience trade-off:** If the orchestrator crashes before Step 8 or PostMortem fails, structured telemetry is lost for that run. However: (a) individual agent completion statuses persist in `memory/*.mem.md` files, (b) orchestrator crash terminates the pipeline anyway, (c) PostMortem failure is non-blocking and telemetry is diagnostic. For additional resilience, the orchestrator may optionally delegate a periodic telemetry snapshot write to a subagent — this is NOT required for primary functionality.

#### Step 8 Addition (Revised)

```markdown
### 8. Post-Mortem (Non-Blocking)

#### 8.1 Dispatch PostMortem Agent

- **Invoke subagent:** `post-mortem`
- **Context to include in dispatch prompt:**
  - Feature slug and run date
  - Full telemetry dataset (accumulated during Steps 1–7, using the format above)
  - Paths to evaluation files directory: `artifact-evaluations/`
  - Paths to memory files directory: `memory/`
- **Outputs expected:** `post-mortems/<run-date>-post-mortem.md`, `agent-metrics/<run-date>-run-log.md`, `memory/post-mortem.mem.md`
- Wait for `DONE:` or `ERROR:`.
- **Non-blocking:** PostMortem ERROR does NOT change the pipeline outcome. Pipeline success/failure is determined at Step 7 (R cluster evaluation).

#### 8.2 Merge PostMortem Memory

Orchestrator reads `memory/post-mortem.mem.md` (using `read_file`) and merges Key Findings, Decisions, and Artifact Index into `memory.md` (using `memory` tool or delegating to subagent). If PostMortem returned ERROR, log warning and skip merge.
```

#### Documentation Structure Update

Add to the orchestrator's `## Documentation Structure` section:

```markdown
- agent-metrics/ # Pipeline telemetry — run logs
  - <run-date>-run-log.md # Structured YAML telemetry per pipeline run
- artifact-evaluations/ # Structured evaluations from consuming agents
  - <agent-name>.md # One file per evaluating agent per run
- post-mortems/ # PostMortem agent analysis reports
  - <run-date>-post-mortem.md
```

#### Orchestrator Expectations Table Update

Add row:

```markdown
| Post-Mortem | Orchestrator reads memory and merges | N/A (DONE/ERROR only) | Non-blocking; orchestrator proceeds without |
```

#### Parallel Execution Summary Update

```
Step 8: Post-Mortem (sequential, non-blocking) → orchestrator merges memory
```

#### Memory Lifecycle Update

Add merge action:

| Action              | When         | What                                                                                         |
| ------------------- | ------------ | -------------------------------------------------------------------------------------------- |
| Merge (post-mortem) | After Step 8 | Read `memory/post-mortem.mem.md`, merge into `memory.md`. Skip if PostMortem returned ERROR. |

> **Removed (CT-3):** No `## Telemetry` section in memory.md. No "never prune" exception. No prune-exclusion complexity.

#### Completion Contract Update

```markdown
Workflow completes only when the R cluster review determines `DONE:` (Step 7). Step 8 (Post-Mortem) runs afterward but is non-blocking — PostMortem ERROR does not change the workflow outcome.
```

#### Anti-Drift Anchor Update

```markdown
You dispatch the PostMortem agent at Step 8 (non-blocking) with accumulated telemetry data. You delegate all file creation and modification to subagents — you never create or edit files directly. You use read tools freely for cluster decisions and routing.
```

### PostMortem Agent Definition (Revised — CT-5, CT-6)

Full specification for `.github/agents/post-mortem.agent.md`:

#### Frontmatter

```yaml
---
name: post-mortem
description: "Post-mortem analysis agent: collects artifact evaluations and execution telemetry to produce quantitative cross-agent metrics. Non-blocking — ERROR does not block the pipeline."
---
```

#### Title & Role

```markdown
# PostMortem Agent Workflow

You are the **PostMortem Agent**.

You analyze structured artifact evaluations and execution telemetry from a completed pipeline run to produce a **quantitative** cross-agent report. You consolidate telemetry data (provided by the orchestrator in your dispatch context) into a structured run log and identify recurring issues, bottleneck agents, and accuracy trends — grounded in specific evaluation data.

You focus on **quantitative metrics only**: scores, retry rates, missing information frequency, and accuracy counts. You do NOT produce qualitative improvement suggestions or recommendations (that is R-Knowledge's scope). You do NOT produce knowledge-suggestions.md, decisions.md, or improvement_recommendations.

You NEVER modify source code, agent definitions, existing pipeline artifacts, or any file outside your declared Outputs. You write only to your isolated memory file (`memory/post-mortem.mem.md`), never to shared `memory.md`.

Post-Mortem is **non-blocking**: your ERROR does not block the pipeline.
```

#### Inputs

```markdown
## Inputs

- Orchestrator dispatch context (read first — contains accumulated telemetry data for the entire run)
- docs/feature/<feature-slug>/memory/\*.mem.md (all isolated agent memories — read for orientation FIRST)
- docs/feature/<feature-slug>/artifact-evaluations/\*.md (evaluation files — read targeted sections after memory orientation)
- docs/feature/<feature-slug>/memory.md (shared memory — for artifact index and context)
- .github/agents/evaluation-schema.md (evaluation schema reference — for validation expectations)
```

#### Outputs

```markdown
## Outputs

- docs/feature/<feature-slug>/agent-metrics/<run-date>-run-log.md (consolidated telemetry run log)
- docs/feature/<feature-slug>/post-mortems/<run-date>-post-mortem.md (quantitative analysis report)
- docs/feature/<feature-slug>/memory/post-mortem.mem.md (isolated memory)
```

Where `<run-date>` is derived from the current date context (e.g., `2026-02-20`). If a file with the same name exists, append a sequence suffix: `-2`, `-3`, etc. (append-only constraint).

#### Operating Rules

Standard 6 rules with these specifics:

```markdown
1. Context-efficient reading (standard)
2. Error handling (standard, with: corrupted YAML in evaluation files → skip and note, do not retry)
3. Output discipline (standard)
4. File boundaries (standard — only write to files listed in Outputs)
5. Tool preferences: Use `file_search` to discover evaluation files. Use `grep_search` and `read_file` for targeted examination. Never use tools that modify source code or agent definitions.
6. Memory-first reading: Read memory files BEFORE full artifact reads. Use memory file findings to prioritize which evaluation files need detailed examination.
```

#### Memory-First Reading Pattern (CT-5)

```markdown
## Reading Strategy

PostMortem uses a **memory-first reading pattern** to manage context window budget:

1. **Read orchestrator-provided telemetry first** — this is in your dispatch context, no tool calls needed.
2. **Read memory files** (`memory/*.mem.md`) — these are compact and provide orientation on which agents had issues, errors, or notable findings.
3. **Discover evaluation files** via `file_search` for `artifact-evaluations/*.md`.
4. **Read evaluation files in targeted groups:**
   - First pass: Use `grep_search` on evaluation files to extract scores and identify files with low scores or reported issues.
   - Second pass: Read full content only of evaluation files that indicate problems (low scores, inaccuracies, notable missing information).
   - For aggregate metrics (averages), extract numeric scores via grep without reading full file content.
5. **Do NOT read all files fully into context at once.** Use targeted reads to stay within context window budget.
```

> **Rationale (CT-5):** The original design had PostMortem ingesting 40–50+ files wholesale, risking context window saturation. The memory-first pattern with targeted reads reduces context pressure while enabling complete quantitative analysis.

#### File Boundaries

```markdown
## File Boundaries

PostMortem writes ONLY to these files:

- `docs/feature/<feature-slug>/agent-metrics/<run-date>-run-log.md`
- `docs/feature/<feature-slug>/post-mortems/<run-date>-post-mortem.md`
- `docs/feature/<feature-slug>/memory/post-mortem.mem.md`

PostMortem MUST NOT write to any other file. PostMortem MUST NOT modify agent definitions, source code, existing pipeline artifacts, evaluation files, memory.md, or any other project file.
```

#### Read-Only Enforcement

```markdown
## Read-Only Enforcement

PostMortem is strictly read-only with respect to the codebase and all existing pipeline artifacts. It reads evaluation files and memory files for analysis but MUST NOT modify them. It reads memory.md for orientation but MUST NOT write to it.
```

#### Workflow (Revised — CT-5, CT-6)

```markdown
## Workflow

1. **Extract telemetry** from the orchestrator-provided dispatch context. The telemetry data includes per-agent dispatch metadata and cluster summaries.
2. **Write run-log:** Expand the orchestrator-provided telemetry into structured YAML and write to `agent-metrics/<run-date>-run-log.md` (see run-log format in Data Models).
3. **Read memory files** (`memory/*.mem.md`) for orientation. Note which agents reported errors, issues, or interesting findings. This informs targeted reading of evaluation files.
4. **Discover evaluation files:** Use `file_search` to find all files in `artifact-evaluations/`.
5. **Extract evaluation metrics (targeted):** Use `grep_search` to extract `usefulness_score` and `clarity_score` values from evaluation files. Use `grep_search` to find `inaccuracies` and `missing_information` entries. Read full file content only when targeted investigation is needed.
6. **Compute agent accuracy scores:** For each producer agent, compute average usefulness and clarity scores from evaluation data. Count total inaccuracies reported.
7. **Identify recurring issues:** Find `missing_information` and `inaccuracies` entries that appear across multiple evaluators for the same or related source artifacts.
8. **Identify bottlenecks:** From telemetry data, identify agents with high retry counts, ERROR statuses, or frequent Pattern C iterations.
9. **Self-verification:** Verify:
   - All discovered evaluation files were processed (or logged as skipped)
   - All telemetry entries were accounted for in the run log
   - Report contains ONLY quantitative metrics — no qualitative recommendations
   - Report does not overlap with R-Knowledge scope (no knowledge-suggestions, no decisions.md, no improvement_recommendations)
   - Fix any gaps found before proceeding.
10. **Write post-mortem report** to `post-mortems/<run-date>-post-mortem.md`.
11. **Write isolated memory** to `memory/post-mortem.mem.md`.
```

> **Key changes (CT-5, CT-6):** (a) Memory-first reading with targeted evaluation file reads instead of wholesale ingestion. (b) No "Generate recommendations" step — PostMortem produces quantitative metrics only. (c) Self-verification explicitly checks for R-Knowledge boundary compliance.

#### Completion Contract

```markdown
## Completion Contract

Return exactly one line:

- DONE: post-mortem analysis complete — <N> evaluations processed, <M> agents scored
- ERROR: <reason>

PostMortem does NOT use NEEDS_REVISION. PostMortem ERROR is non-blocking.
```

#### Anti-Drift Anchor

```markdown
## Anti-Drift Anchor

**REMEMBER:** You are the **PostMortem Agent**. You produce **quantitative metrics only** — scores, frequencies, retry rates, accuracy counts. You do NOT produce improvement_recommendations, knowledge-suggestions.md, or decisions.md. You do NOT prescribe what agents should change — that is R-Knowledge's scope. You use a memory-first reading pattern: read memory files for orientation, then targeted evaluation reads. You NEVER modify agent definitions, source code, or existing pipeline artifacts. You are non-blocking — your failure does not block the pipeline. Stay as post-mortem.
```

### Feature Workflow Prompt Update

Add to the Key Artifacts table in `feature-workflow.prompt.md`:

```markdown
| `artifact-evaluations/` | Structured artifact evaluations from consuming agents |
| `agent-metrics/` | Pipeline execution telemetry (run logs) |
| `post-mortems/` | Post-mortem quantitative analysis reports |
```

Add to the Rules section:

```markdown
- **Post-mortem:** After the R cluster completes (Step 7), the orchestrator dispatches a PostMortem agent (Step 8) that is non-blocking.
- **Artifact evaluations:** Agents that consume upstream pipeline artifacts produce structured evaluations in `artifact-evaluations/` following the schema in `.github/agents/evaluation-schema.md`.
```

---

## Storage Architecture

### Directory Structure

All new directories are per-feature scoped under `docs/feature/<feature-slug>/`:

```
docs/feature/<feature-slug>/
├── ... (existing directories unchanged)
├── agent-metrics/                       # NEW — pipeline telemetry
│   └── <run-date>-run-log.md           # Structured YAML run log per pipeline run
├── artifact-evaluations/                # NEW — agent evaluations
│   ├── spec.md                         # Spec agent's evaluations of research/*.md
│   ├── designer.md                     # Designer's evaluation of feature.md
│   ├── ct-security.md                  # CT-Security's evaluations of design.md, feature.md
│   ├── ct-scalability.md
│   ├── ct-maintainability.md
│   ├── ct-strategy.md
│   ├── planner.md
│   ├── implementer-<task-id>.md        # One per implementer instance
│   ├── documentation-writer-<task-id>.md
│   ├── v-tests.md
│   ├── v-tasks.md
│   ├── v-feature.md
│   ├── r-quality.md
│   └── r-testing.md
└── post-mortems/                        # NEW — post-mortem reports
    └── <run-date>-post-mortem.md       # Timestamped report per pipeline run
```

### File Naming Conventions

| Directory               | Naming Pattern                                   | Example                             |
| ----------------------- | ------------------------------------------------ | ----------------------------------- |
| `agent-metrics/`        | `<YYYY-MM-DD>-run-log.md`                        | `2026-02-20-run-log.md`             |
| `artifact-evaluations/` | `<agent-name>.md` or `<agent-name>-<task-id>.md` | `spec.md`, `implementer-01-auth.md` |
| `post-mortems/`         | `<YYYY-MM-DD>-post-mortem.md`                    | `2026-02-20-post-mortem.md`         |

### Append-Only Semantics

- **Run-log and post-mortem files** use date-stamped filenames. Same-day collisions append a sequence suffix: `2026-02-20-2-run-log.md`.
- **Evaluation files** use bare agent names (per FR-1.4). For re-runs or within-run NEEDS_REVISION re-executions, if the file already exists, the agent appends a sequence suffix (`<agent>-2.md`, `<agent>-3.md`) to handle all collision scenarios consistently.
- **No files are ever overwritten.**

> **Revised (CT-strategy F6):** Collision avoidance now uses a consistent sequence suffix strategy across all file types, addressing the inconsistency between date-append (evaluations) and sequence-suffix (post-mortems/run-logs) identified in the CT review.

### Directory Creation

Directories are created lazily by the first agent that writes to them:

- `artifact-evaluations/`: Created when the first evaluating agent (typically spec at Step 2) writes its evaluation file.
- `agent-metrics/`: Created when the PostMortem agent writes the run log at Step 8.
- `post-mortems/`: Created when the PostMortem agent writes the report at Step 8.

---

## Sequence / Interaction Notes

### Standard Pipeline Run with Self-Improvement

```
Step 0:  Orchestrator → memory tool / subagent → create memory.md + initial-request.md
Step 1:  Orchestrator → dispatch researchers ×4 (existing)
         Orchestrator → reads memory files → records telemetry in context
         Orchestrator → merge memories (existing)
Step 2:  Orchestrator → dispatch spec
         Spec → reads research/*.md → produces feature.md → writes artifact-evaluations/spec.md
         Orchestrator → records telemetry in context → merge memory
Step 3:  Orchestrator → dispatch designer
         Designer → reads feature.md → produces design.md → writes artifact-evaluations/designer.md
         Orchestrator → records telemetry in context → merge memory
Step 3b: Orchestrator → dispatch CT ×4
         Each CT → reads design.md, feature.md → produces ct-review → writes artifact-evaluations/ct-*.md
         Orchestrator → records 4 agent + 1 cluster telemetry in context → CT decision → merge memories
Step 4:  Orchestrator → dispatch planner
         Planner → reads design.md, feature.md → produces plan.md, tasks → writes artifact-evaluations/planner.md
         Orchestrator → records telemetry in context → merge memory
Step 5:  Orchestrator → dispatch implementers/doc-writers per wave
         Each → reads task, design, feature → produces code/docs → writes evaluation files
         Orchestrator → records telemetry in context → merge memories between waves
Step 6:  Orchestrator → dispatch V cluster (Pattern B+C)
         V-tests, V-tasks, V-feature → produce verification → write evaluation files
         Orchestrator → records telemetry in context → V decision → merge memories
Step 7:  Orchestrator → dispatch R cluster (Pattern A)
         R-quality, R-testing → produce reviews → write evaluation files
         Orchestrator → records telemetry in context → R decision → merge memories
Step 8:  Orchestrator → dispatch PostMortem with accumulated telemetry (non-blocking)
         PostMortem → writes run-log from telemetry + reads evaluations/memories → writes report
         Orchestrator → merge post-mortem memory (if DONE)
```

### Telemetry Data Flow (Revised)

```
Steps 1-7: Orchestrator observes dispatch results → accumulates in working context (no file writes)
Step 8:    Orchestrator includes full telemetry data in PostMortem dispatch prompt
           PostMortem → writes agent-metrics/<run-date>-run-log.md (structured YAML)
```

**Fallback:** If PostMortem fails (ERROR), structured telemetry is lost for this run. Agent completion statuses remain in `memory/*.mem.md` files. The orchestrator's completion contract reports the overall pipeline outcome.

### Pattern C Replan Loop Telemetry

For V cluster replan iterations (Steps 5–6, max 3 iterations):

- Each iteration produces separate telemetry observations in the orchestrator's context with incrementing iteration numbers
- The full dataset — all iterations — is passed to PostMortem at Step 8
- PostMortem includes all iteration entries in the run log, enabling bottleneck analysis

---

## Security Considerations

### Authentication/Authorization

**N/A.** The Forge pipeline operates within a single user's GitHub Copilot session. No multi-user authentication, no API keys, no external service authentication. All data is local markdown files in a git repository.

### Data Protection

- Evaluation data, telemetry, and post-mortem reports contain no secrets, credentials, or PII
- All data is stored as Markdown files in the repository, protected by standard git access controls
- No data leaves the repository boundary

### Threat Model

| Threat                                                                 | Likelihood | Mitigation                                                                                                                        |
| ---------------------------------------------------------------------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Agent produces evaluation containing sensitive codebase details        | Low        | Evaluations are about artifact quality (scores, clarity), not about code content                                                  |
| PostMortem agent modifies existing artifacts                           | Very Low   | File Boundaries, Read-Only Enforcement, Anti-Drift Anchor constrain writes                                                        |
| Evaluation YAML injection (malformed YAML corrupts downstream parsing) | Medium     | PostMortem skips corrupted files gracefully (EC-5); each YAML block in own fenced code block                                      |
| Indirect prompt injection via evaluation free-text fields              | Low        | All agents are prompt-controlled (not user-facing); PostMortem File Boundaries constrain write scope regardless of prompt content |

### Input Validation

- PostMortem agent validates each evaluation YAML block before processing: required fields present, scores in 1–10 range, list fields non-empty
- Invalid blocks are skipped with a logged warning, not treated as errors
- Telemetry data from orchestrator context is structured Markdown tables; PostMortem validates completeness before writing run-log

### Memory Tool Scope (CT-7 — Risk Mitigation)

The revised design **eliminates the dependency on the `memory` tool for telemetry recording**. The `memory` tool is used only for:

1. Step 0: Initializing memory.md (with subagent fallback — existing behavior)
2. Memory merges after agent completions (existing behavior)

Both of these are existing uses that pre-date this feature. No new `memory` tool capabilities are assumed. If the `memory` tool cannot perform an operation, the orchestrator delegates to a subagent via `runSubagent`.

---

## Failure & Recovery

### Expected Failure Modes

| Failure Mode                                          | Probability | Impact                                   | Recovery Strategy                                                                                | Traces to   |
| ----------------------------------------------------- | ----------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------ | ----------- |
| Evaluation YAML generation error                      | Medium      | Low — no evaluation data for that agent  | Agent writes `evaluation_error` block (per schema doc); returns DONE with primary output intact  | EC-2, AC-13 |
| Agent cannot access `artifact-evaluations/` directory | Low         | Low — evaluation silently missing        | Agent creates directory on write; if creation fails, logs in isolated memory and proceeds        | EC-10       |
| PostMortem receives empty evaluation directory        | Low         | Low — report has empty analysis sections | PostMortem produces report with empty arrays and notes "no evaluation data available" in summary | EC-4        |
| PostMortem encounters corrupted evaluation YAML       | Medium      | Low — partial analysis                   | Skip corrupted file; log in report summary; process remaining files                              | EC-5        |
| PostMortem agent ERROR                                | Low         | Low — report not generated               | Non-blocking; pipeline outcome unchanged; agent memories persist                                 | AC-7        |
| Orchestrator telemetry lost (crash before Step 8)     | Very Low    | Low — no run-log produced                | Agent memories contain completion statuses; pipeline crash ends run anyway                       | —           |
| PostMortem telemetry write fails                      | Low         | Low — run-log incomplete                 | Non-blocking; agent memories persist as fallback data source                                     | —           |
| `memory` tool cannot create memory.md in Step 0       | Medium      | Medium — degraded memory state           | Delegate to subagent; if both fail, proceed without memory (existing graceful degradation)       | EC-6        |
| Same-day re-run filename collision                    | Low         | Low — data loss risk                     | Sequence suffix (`-2`, `-3`) on all file types                                                   | EC-11       |
| Concurrent evaluation writes (parallel agents)        | Expected    | None — by design                         | Each agent writes to a uniquely named file; no shared write targets                              | EC-8        |

### Graceful Degradation Hierarchy

1. **Full success:** All evaluations produced, telemetry passed to PostMortem, complete report generated
2. **Partial evaluations:** Some agents' evaluation generation fails → PostMortem analyzes available data, notes gaps
3. **No evaluations:** All evaluation generation fails → PostMortem produces telemetry-only run-log with accuracy sections empty
4. **No telemetry:** PostMortem fails or not dispatched → raw evaluation files persist for manual review; agent memories persist with completion statuses
5. **PostMortem fails:** Pipeline outcome unaffected; evaluation files + agent memories remain for manual inspection

---

## Non-functional Requirements

| NFR                           | Design Response                                                                                                                                                                                                |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| NFR-1 (Additive-Only)         | All agent changes are new workflow steps and output additions. No existing steps modified. No existing output formats changed. Excluded agents untouched. Write-tool restriction is a separate task (Track B). |
| NFR-2 (Structured Output)     | All new data uses YAML within Markdown fenced code blocks. No standalone YAML or JSON files.                                                                                                                   |
| NFR-3 (Consistency)           | PostMortem follows identical chatagent format: frontmatter, 6 Operating Rules, Inputs, Outputs, Workflow, Completion Contract, Anti-Drift Anchor.                                                              |
| NFR-4 (Non-Blocking)          | Evaluation step explicitly non-blocking (cannot cause ERROR). PostMortem non-blocking (ERROR doesn't block pipeline).                                                                                          |
| NFR-5 (Deterministic Output)  | Post-mortem YAML uses structured fields. Only `summary` allows natural language. No qualitative recommendations.                                                                                               |
| NFR-6 (Graceful Degradation)  | PostMortem handles missing/corrupted inputs. Memory-first reading manages context pressure. See Failure & Recovery.                                                                                            |
| NFR-7 (Per-Feature Isolation) | All directories under `docs/feature/<feature-slug>/`. No global or cross-feature storage.                                                                                                                      |

---

## Migration & Backwards Compatibility

### Agent File Changes

All 14 evaluating agent changes are **purely additive**:

- Outputs sections gain one new entry (evaluation file)
- Workflow gains one new step referencing shared evaluation schema document
- No existing steps are renumbered — the new step is inserted with a new number
- No existing completion contracts change
- No existing inputs change

### Orchestrator Changes — Track A (Self-Improvement)

- Step 8 added after existing Step 7 (no renumbering)
- Documentation Structure section extended (additive)
- Expectations table extended (additive)
- Completion Contract clarified (Step 8 non-blocking)
- **No new memory.md sections** — telemetry is context-based, not stored in memory.md
- Anti-Drift Anchor extended

### Orchestrator Changes — Track B (Write-Tool Restriction — Separate)

- Narrows write/execute capability (removes `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`)
- Retains all read tools (no capability loss for orchestration decisions)
- Step 0 rewritten to delegate file creation
- Global Rule 1 updated

> **Separation (CT-2):** Track A and Track B are designed to be implemented independently. Track A self-improvement features work with the orchestrator's current tool set or with the restricted set. Track B can be implemented before or after Track A without affecting its changes.

### Memory Format Impact

- **No new sections in memory.md** (telemetry removed from shared memory)
- New `memory/post-mortem.mem.md` follows existing memory file format; no impact on other agents
- No prune rule changes needed

### DB / API Migration

**N/A.** No database or external APIs exist. All changes are to Markdown files.

---

## Testing Strategy

### Unit-Level Validation (per agent file)

| Test                             | Validates | Criteria                                                                                                                                                                                                                                                 |
| -------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TS-1: Evaluation File Structure  | AC-1      | Each of 14 `.agent.md` files: has evaluation in Outputs, has evaluation workflow step referencing evaluation-schema.md                                                                                                                                   |
| TS-2: Evaluation Schema Document | AC-2      | evaluation-schema.md exists, defines all required fields, correct types, includes error fallback                                                                                                                                                         |
| TS-3: Excluded Agents            | AC-3      | researcher.agent.md, v-build.agent.md, r-security.agent.md, r-knowledge.agent.md: zero diff                                                                                                                                                              |
| TS-6: PostMortem Format          | AC-6      | post-mortem.agent.md: YAML frontmatter, Inputs, Outputs, 6 Operating Rules, Memory-First Reading Strategy, Workflow with self-verification, two-state Completion Contract, Anti-Drift Anchor (quantitative only), Read-Only Enforcement, File Boundaries |
| TS-11: No Breaking Changes       | AC-11     | All existing completion contracts unchanged; all existing Inputs sections unchanged; critical-thinker.agent.md untouched                                                                                                                                 |

### Integration-Level Validation

| Test                             | Validates      | Criteria                                                                                                                                                                                                                           |
| -------------------------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TS-4: Telemetry via Context      | AC-4           | run-log.md exists after run; contains YAML entries for all dispatched agents; cluster summaries present                                                                                                                            |
| TS-5: Tool Restriction (revised) | AC-5 (revised) | orchestrator.agent.md: no references to `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`; retains `read_file`, `grep_search`, `semantic_search`, `file_search`, `list_dir`, `get_errors` |
| TS-8: PostMortem Non-Blocking    | AC-7           | Step 8 explicitly states ERROR does not block; pipeline DONE determined at Step 7                                                                                                                                                  |
| TS-9: Storage Directories        | AC-9           | Documentation Structure includes all 3 new directories                                                                                                                                                                             |
| TS-10: Append-Only               | AC-10          | Consistent sequence suffix collision avoidance; no overwriting                                                                                                                                                                     |
| TS-12: Evaluation Non-Blocking   | AC-13          | Each evaluating agent's evaluation step says failure MUST NOT cause ERROR                                                                                                                                                          |
| TS-13: Boundary (revised)        | AC-12          | R-Knowledge unmodified; PostMortem outputs exclude knowledge-suggestions.md, decisions.md, improvement_recommendations                                                                                                             |
| TS-14: Step 8                    | AC-7           | Step 8 in workflow; post-mortem row in Expectations table; Step 8 in Parallel Execution Summary                                                                                                                                    |

### End-to-End Validation

| Test                              | Validates | Criteria                                                                                         |
| --------------------------------- | --------- | ------------------------------------------------------------------------------------------------ |
| TS-7: PostMortem Report (revised) | AC-8      | Report in post-mortems/ with YAML matching revised schema (no improvement_recommendations field) |
| TS-15: Prompt Update              | AC-14     | feature-workflow.prompt.md references new directories, PostMortem step, and evaluation schema    |

### New Tests (Revision-Specific)

| Test                                 | Validates | Criteria                                                                                                                                                       |
| ------------------------------------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TS-16: Shared Schema Document        | CT-4      | evaluation-schema.md exists at `.github/agents/evaluation-schema.md`; contains full schema with field descriptions; 14 agent workflow steps reference it       |
| TS-17: No Telemetry in memory.md     | CT-3      | memory.md initialization template has no `## Telemetry` section; no "never prune" rule in Memory Lifecycle                                                     |
| TS-18: Tool Restriction Independence | CT-2      | Self-improvement features (evaluation step, PostMortem, Step 8) work without write-tool restriction changes; no runtime dependency between Track A and Track B |
| TS-19: PostMortem Quantitative Only  | CT-6      | PostMortem report schema has no improvement_recommendations; PostMortem anti-drift anchor says "quantitative metrics only"                                     |

---

## Tradeoffs & Alternatives Considered

### 1. Telemetry: Context Passing vs. memory.md Accumulation (Revised — CT-3)

**Changed from v0: memory.md accumulation → orchestrator context passing.**

| Option                                            | Pros                                                                           | Cons                                                                                                                                            |
| ------------------------------------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| A. memory.md accumulation (v0 design)             | Persistent; survives PostMortem failure                                        | Bloats memory.md for all 19 agents (~3.5-7KB/run, never pruned); depends on unverified memory tool writes; creates compact DSL parsing contract |
| **B. Orchestrator context → PostMortem dispatch** | Zero memory.md bloat; no tool dependency; no parsing DSL; simpler architecture | Ephemeral until PostMortem writes run-log; lost if PostMortem fails                                                                             |
| C. Dedicated telemetry subagent                   | Persistent; clean separation                                                   | Adds another agent; dispatched at every step (overhead)                                                                                         |

**Rationale:** CT reviews unanimously flagged memory.md telemetry bloat as High severity. Option B eliminates this entirely. The trade-off (ephemeral until Step 8) is acceptable because telemetry is diagnostic — agent memories provide a persistent fallback for completion status data.

### 2. Evaluation Schema: Inline vs. Shared Reference (Revised — CT-4)

**Changed from v0: inline in each agent → shared reference document.**

| Option                              | Pros                                                               | Cons                                                                         |
| ----------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| A. Inline in each agent (v0 design) | Self-contained; no external reference                              | 14-file duplication; schema evolution requires 14+ coordinated updates       |
| **B. Shared reference document**    | Single source of truth; one-file schema updates; version-trackable | Agents must be able to read reference doc (they can via existing read tools) |

**Rationale:** CT-maintainability identified 14-file duplication as High severity. A shared reference document is standard practice and all agents already have file read capabilities.

### 3. PostMortem Output: Metrics + Recommendations vs. Metrics Only (Revised — CT-6)

**Changed from v0: include improvement_recommendations → quantitative metrics only.**

| Option                                 | Pros                                                                 | Cons                                                                                    |
| -------------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| A. Include recommendations (v0 design) | Comprehensive output; actionable insights in one report              | Overlaps R-Knowledge scope; blurs boundary; qualitative content in a quantitative agent |
| **B. Quantitative metrics only**       | Clean boundary with R-Knowledge; focused scope; no overlap confusion | Humans must derive action items from raw metrics                                        |

**Rationale:** CT-strategy and CT-maintainability both flagged the boundary blur. Removing recommendations enforces a clean separation: PostMortem says WHAT is happening; R-Knowledge says WHAT TO DO.

### 4. Orchestrator Tool Restriction: Full vs. Write-Only (Revised — CT-1)

**Changed from v0: remove all non-orchestration tools → remove only write/execute tools.**

| Option                                                  | Pros                                                                                                | Cons                                                                                    |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| A. Restrict to [agent, runSubagent, memory] (v0 design) | Strictest enforcement of "orchestrator only orchestrates"                                           | Breaks cluster decisions that require read_file; unverified memory tool scope for reads |
| **B. Remove write/execute tools; keep reads**           | Preserves cluster decision flows; doesn't depend on memory tool for reads; achieves delegation goal | Less strict than original literal request                                               |

**Rationale:** CT-strategy rated the v0 restriction as Critical severity — it would break the pipeline's core coordination mechanism. Keeping read tools is essential for orchestration; removing write tools achieves the stated goal ("orchestrator only orchestrates, delegates all file operations").

### 5. PostMortem Reading: Wholesale vs. Memory-First (Revised — CT-5)

**Changed from v0: read all files wholesale → memory-first with targeted reads.**

| Option                               | Pros                                                                   | Cons                                                                       |
| ------------------------------------ | ---------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| A. Read all files (v0 design)        | Complete data; simple workflow                                         | 40-50+ files risk context window saturation                                |
| **B. Memory-first + targeted reads** | Manages context budget; processes all data through targeted extraction | Slightly more complex workflow; relies on grep_search for score extraction |

**Rationale:** CT-scalability identified context window saturation as High severity. Memory-first reading is the established pattern in the pipeline — agents already read memory files for orientation before full artifacts.

### 6. Other Retained Decisions

- **Evaluation storage in separate files** (not appended to primary output) — preserved from v0, correctly addresses additive-only constraint
- **PostMortem as Step 8 (automatic, non-blocking)** — preserved, ensures data always captured regardless of pipeline outcome
- **Lazy directory creation** — preserved, simpler than explicit Step 0 directory creation
- **Bare agent names for evaluation files** (per FR-1.4) — preserved with improved collision avoidance (consistent sequence suffixes)
- **Two-state completion contract for PostMortem** (DONE/ERROR only) — preserved from v0, matches R-Knowledge non-blocking pattern

---

## Implementation Checklist & Deliverables

### Implementation Tracks (Two Independent Tracks — CT-2)

#### Track A: Self-Improvement System (evaluations + telemetry + PostMortem)

| #   | Deliverable                                       | Description                                                                             | AC                     |
| --- | ------------------------------------------------- | --------------------------------------------------------------------------------------- | ---------------------- |
| A1  | `.github/agents/evaluation-schema.md`             | Shared evaluation schema reference document                                             | AC-2, CT-4             |
| A2  | `.github/agents/post-mortem.agent.md`             | PostMortem agent definition (quantitative metrics only)                                 | AC-6, AC-7, AC-8, CT-6 |
| A3  | 14 × `.agent.md` evaluation additions             | Add evaluation output + workflow step (referencing schema doc) to each evaluating agent | AC-1, AC-13            |
| A4  | `orchestrator.agent.md` — Step 8                  | Add PostMortem dispatch, memory merge, Expectations table, Parallel Summary             | AC-7, AC-9             |
| A5  | `orchestrator.agent.md` — telemetry               | Add instructions to accumulate telemetry in context, include in PostMortem dispatch     | AC-4, CT-3             |
| A6  | `orchestrator.agent.md` — Documentation Structure | Add 3 new directories to docs structure section                                         | AC-9                   |
| A7  | `.github/prompts/feature-workflow.prompt.md`      | Add evaluation/metrics/PostMortem references                                            | AC-14                  |

#### Track B: Orchestrator Write-Tool Restriction (independent — CT-2)

| #   | Deliverable                                             | Description                                                | AC             |
| --- | ------------------------------------------------------- | ---------------------------------------------------------- | -------------- |
| B1  | `orchestrator.agent.md` — tool declaration              | Declare write-restricted tool set; retain read tools       | AC-5 (revised) |
| B2  | `orchestrator.agent.md` — Step 0 rewrite                | Delegate file creation to `memory` tool or subagents       | AC-5 (revised) |
| B3  | `orchestrator.agent.md` — reword direct file operations | Replace write-tool references with delegation instructions | AC-5 (revised) |

> **Track independence (CT-2):** Track A can be implemented and validated without Track B, and vice versa. Track A works with the orchestrator's current tool set. Track B can be implemented before or after Track A.

### New Files to Create

| File                                  | Description                             | Track | AC               |
| ------------------------------------- | --------------------------------------- | ----- | ---------------- |
| `.github/agents/evaluation-schema.md` | Shared evaluation YAML schema reference | A     | AC-2, CT-4       |
| `.github/agents/post-mortem.agent.md` | PostMortem agent definition             | A     | AC-6, AC-7, AC-8 |

### Files to Modify (Track A)

| File                                           | Changes                                                                                                                                       | AC               |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| `.github/agents/orchestrator.agent.md`         | Step 8, telemetry context instructions, Documentation Structure, Expectations table, Parallel Summary, Completion Contract, Anti-Drift Anchor | AC-4, AC-7, AC-9 |
| `.github/agents/spec.agent.md`                 | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/designer.agent.md`             | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/ct-security.agent.md`          | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/ct-scalability.agent.md`       | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/ct-maintainability.agent.md`   | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/ct-strategy.agent.md`          | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/planner.agent.md`              | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/implementer.agent.md`          | Add evaluation output + workflow step (task-ID naming)                                                                                        | AC-1             |
| `.github/agents/documentation-writer.agent.md` | Add evaluation output + workflow step (task-ID naming)                                                                                        | AC-1             |
| `.github/agents/v-tests.agent.md`              | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/v-tasks.agent.md`              | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/v-feature.agent.md`            | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/r-quality.agent.md`            | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/agents/r-testing.agent.md`            | Add evaluation output + workflow step                                                                                                         | AC-1             |
| `.github/prompts/feature-workflow.prompt.md`   | Add new artifact references                                                                                                                   | AC-14            |

### Files to Modify (Track B — separate)

| File                                   | Changes                                                                                          | AC             |
| -------------------------------------- | ------------------------------------------------------------------------------------------------ | -------------- |
| `.github/agents/orchestrator.agent.md` | Write-tool restriction declaration, Step 0 rewrite, Global Rule 1 update, reword file operations | AC-5 (revised) |

### Files NOT Modified (Safety Constraint)

| File                                       | Reason                               | AC          |
| ------------------------------------------ | ------------------------------------ | ----------- |
| `.github/agents/researcher.agent.md`       | No upstream agent artifacts consumed | AC-3        |
| `.github/agents/v-build.agent.md`          | Evaluates codebase only              | AC-3        |
| `.github/agents/r-security.agent.md`       | Consumes only git diff/codebase      | AC-3        |
| `.github/agents/r-knowledge.agent.md`      | Self-referential; excluded per spec  | AC-3, AC-12 |
| `.github/agents/critical-thinker.agent.md` | Deprecated; do not touch             | AC-11       |
| `.github/agents/dispatch-patterns.md`      | Reference doc; not an agent          | AC-11       |

### Acceptance Criteria Mapping

| AC    | Design Section                                          | Track |
| ----- | ------------------------------------------------------- | ----- |
| AC-1  | §Evaluating Agent Changes                               | A     |
| AC-2  | §Shared Evaluation Schema + §Evaluation Schema Document | A     |
| AC-3  | §Agents Excluded                                        | A     |
| AC-4  | §Telemetry Accumulation + §Run-Log Format               | A     |
| AC-5  | §Write-Tool Restriction                                 | B     |
| AC-6  | §PostMortem Agent Definition                            | A     |
| AC-7  | §Step 8 Addition + §Completion Contract                 | A     |
| AC-8  | §Post-Mortem Report Schema                              | A     |
| AC-9  | §Storage Architecture + §Documentation Structure        | A     |
| AC-10 | §Append-Only Semantics                                  | A     |
| AC-11 | §Migration & Backwards Compatibility                    | A+B   |
| AC-12 | §Boundary: PostMortem vs. R-Knowledge                   | A     |
| AC-13 | §Workflow Step Addition                                 | A     |
| AC-14 | §Feature Workflow Prompt Update + Checklist             | A     |
