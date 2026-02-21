# Self-Improvement System — Summary of Changes

> **Feature:** self-improvement-system
> **Date:** 2026-02-20
> **Satisfies:** FR-8.1, AC-14

This document summarizes all changes made for the self-improvement system feature: a structured cross-agent evaluation and post-mortem learning system for the Forge pipeline. It covers what was changed, where evaluation data is stored, how agents generate evaluations, and how to trigger the PostMortem agent.

---

## 1. Summary of All Changes

The feature was implemented across **2 new files** and **16 modified files**, organized into two independent tracks:

- **Track A (Self-Improvement):** Evaluation schema, PostMortem agent, 14 agent evaluation steps, orchestrator Step 8 + telemetry, feature-workflow prompt updates
- **Track B (Tool Restriction):** Orchestrator write-tool restriction (independent of Track A)

### New Files Created

| File                                  | Purpose                                                                                                                                                                                                                           |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.github/agents/evaluation-schema.md` | Shared reference document defining the artifact evaluation YAML schema (v1). Used by all 14 evaluating agents and the PostMortem agent. Centralizes the schema to eliminate duplication.                                          |
| `.github/agents/post-mortem.agent.md` | New non-blocking agent definition (~250 lines). Runs at Step 8 to produce quantitative cross-agent metrics from evaluations and telemetry. Uses memory-first reading pattern and two-state completion contract (DONE/ERROR only). |

### Modified Files (16 total)

#### Orchestrator (2 tasks touched this file)

| File                                   | Task         | Changes                                                                                                                                                                                                                                                                                                                                                 |
| -------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.github/agents/orchestrator.agent.md` | 03 (Track B) | Write-tool restriction: Operating Rule 5 declares allowed tools; `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal` prohibited; Global Rule 1 updated to require delegation; Step 0 rewritten to delegate file creation                                                                                         |
| `.github/agents/orchestrator.agent.md` | 10 (Track A) | Step 8 (Post-Mortem) added after Step 7; Global Rule 13 (Telemetry Context Tracking) added; Documentation Structure updated with 3 new directories; Orchestrator Expectations table updated with PostMortem row; Parallel Execution Summary updated; Memory Lifecycle updated with post-mortem merge; Completion Contract and Anti-Drift Anchor updated |

#### Feature-Workflow Prompt

| File                                         | Task | Changes                                                                                                                                                                                                |
| -------------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `.github/prompts/feature-workflow.prompt.md` | 04   | Added 2 new Rules (PostMortem non-blocking dispatch, artifact evaluations with schema reference); added 3 new Key Artifacts table entries (`artifact-evaluations/`, `agent-metrics/`, `post-mortems/`) |

#### 14 Evaluating Agents (Tasks 05–09)

Each agent received two additive changes: (1) a new entry in its Outputs section for the evaluation file, and (2) a new "Evaluate Upstream Artifacts" workflow step referencing `.github/agents/evaluation-schema.md`.

| File                                           | Task | Evaluation Targets                                                                                   |
| ---------------------------------------------- | ---- | ---------------------------------------------------------------------------------------------------- |
| `.github/agents/spec.agent.md`                 | 05   | `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md` |
| `.github/agents/designer.agent.md`             | 05   | `feature.md`                                                                                         |
| `.github/agents/planner.agent.md`              | 05   | `design.md`, `feature.md`                                                                            |
| `.github/agents/ct-security.agent.md`          | 06   | `design.md`, `feature.md`                                                                            |
| `.github/agents/ct-scalability.agent.md`       | 06   | `design.md`, `feature.md`                                                                            |
| `.github/agents/ct-maintainability.agent.md`   | 06   | `design.md`, `feature.md`                                                                            |
| `.github/agents/ct-strategy.agent.md`          | 07   | `design.md`, `feature.md`                                                                            |
| `.github/agents/implementer.agent.md`          | 07   | `tasks/<task>.md`, `design.md`, `feature.md`                                                         |
| `.github/agents/documentation-writer.agent.md` | 07   | `tasks/<task>.md`, `design.md`, `feature.md`                                                         |
| `.github/agents/v-tests.agent.md`              | 08   | `verification/v-build.md`                                                                            |
| `.github/agents/v-tasks.agent.md`              | 08   | `verification/v-build.md`, `plan.md`, `tasks/*.md`                                                   |
| `.github/agents/v-feature.agent.md`            | 08   | `verification/v-build.md`, `feature.md`                                                              |
| `.github/agents/r-quality.agent.md`            | 09   | `design.md`                                                                                          |
| `.github/agents/r-testing.agent.md`            | 09   | `feature.md`                                                                                         |

### Files NOT Modified (Safety — AC-3)

These 5 agent files were intentionally excluded because they do not consume upstream pipeline-produced artifacts:

| File                                       | Reason                                                               |
| ------------------------------------------ | -------------------------------------------------------------------- |
| `.github/agents/researcher.agent.md`       | Consumes only `initial-request.md` (user input) and codebase         |
| `.github/agents/v-build.agent.md`          | Evaluates codebase only                                              |
| `.github/agents/r-security.agent.md`       | Consumes only git diff and codebase                                  |
| `.github/agents/r-knowledge.agent.md`      | Excluded per FR-1.8 — self-referential consumption of `decisions.md` |
| `.github/agents/critical-thinker.agent.md` | Deprecated agent — not touched                                       |

---

## 2. Where Evaluation Data Is Stored

### Storage Directories

Three new per-feature directories store all self-improvement system data under `docs/feature/<feature-slug>/`:

| Directory               | Contents                                          | Written By           |
| ----------------------- | ------------------------------------------------- | -------------------- |
| `artifact-evaluations/` | Structured evaluation files from consuming agents | 14 evaluating agents |
| `agent-metrics/`        | Pipeline execution telemetry run logs             | PostMortem agent     |
| `post-mortems/`         | Quantitative post-mortem analysis reports         | PostMortem agent     |

These directories are defined in the orchestrator's Documentation Structure section and are created lazily by the first agent that writes to them.

### Evaluation File Format

Each evaluating agent writes a Markdown file to `artifact-evaluations/` containing one or more YAML code blocks — one per consumed upstream artifact.

**File naming conventions:**

| Agent Type             | Filename Pattern            | Example                                           |
| ---------------------- | --------------------------- | ------------------------------------------------- |
| Single-instance agents | `<agent-name>.md`           | `spec.md`, `designer.md`, `ct-security.md`        |
| Multi-instance agents  | `<agent-name>-<task-id>.md` | `implementer-01.md`, `documentation-writer-11.md` |

**Collision avoidance:** If a file already exists (e.g., from a NEEDS_REVISION re-execution), the agent appends a sequence suffix: `<agent-name>-2.md`, `<agent-name>-3.md`. No files are ever overwritten.

### Evaluation YAML Schema

The schema is defined centrally in `.github/agents/evaluation-schema.md` (v1). Each `artifact_evaluation` block contains:

```yaml
artifact_evaluation:
  evaluator: "<agent-name>" # Must match the evaluating agent's name
  source_artifact: "<relative path>" # Path relative to feature directory
  usefulness_score: <integer 1-10> # How useful was this artifact
  clarity_score: <integer 1-10> # How clear and well-structured
  useful_elements:
    - "<specific element>" # ≥1 entry; "N/A — none identified" if empty
  missing_information:
    - "<missing detail>" # ≥1 entry; "N/A — none identified" if empty
  information_not_used:
    - "<info not used>" # ≥1 entry; "N/A — none identified" if empty
  inaccuracies:
    - "<error found>" # ≥1 entry; "N/A — none identified" if empty
  impact_on_work:
    - "<rework needed>" # ≥1 entry; "N/A — none identified" if empty
```

If evaluation generation fails, agents write an `evaluation_error` fallback block instead:

```yaml
evaluation_error:
  evaluator: "<agent-name>"
  source_artifact: "<path>"
  error: "<description of what failed>"
```

### Run-Log and Post-Mortem Storage

| File                                     | Naming Pattern       | Contents                                                                                                     |
| ---------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------ |
| `agent-metrics/<run-date>-run-log.md`    | Date-stamped per run | Consolidated YAML telemetry: per-agent dispatch entries + cluster summaries                                  |
| `post-mortems/<run-date>-post-mortem.md` | Date-stamped per run | Quantitative analysis: recurring issues, bottlenecks, most common missing information, agent accuracy scores |

Same-day collisions use sequence suffixes: `2026-02-20-2-run-log.md`.

---

## 3. How Agents Generate Evaluations

### Overview

14 agents that consume upstream pipeline-produced artifacts now include an **"Evaluate Upstream Artifacts"** workflow step. This step runs **after** the agent's primary work is complete and **before** self-verification or isolated memory writing.

### The Evaluation Workflow Step

Each agent's evaluation step follows a consistent template:

1. **After completing primary work**, the agent evaluates each upstream pipeline-produced artifact it consumed during the task.
2. For each source artifact, the agent produces one `artifact_evaluation` YAML block following the schema in `.github/agents/evaluation-schema.md`.
3. All evaluation blocks are written to a single file: `docs/feature/<feature-slug>/artifact-evaluations/<agent-name>.md`.

**Example** (from `spec.agent.md`):

> **Evaluate Upstream Artifacts:** After completing your primary work, evaluate each upstream pipeline-produced artifact you consumed.
>
> For each source artifact listed below, produce one `artifact_evaluation` YAML block following the schema defined in `.github/agents/evaluation-schema.md`. Write all blocks to: `docs/feature/<feature-slug>/artifact-evaluations/spec.md`.
>
> Source artifacts to evaluate:
>
> - research/architecture.md
> - research/impact.md
> - research/dependencies.md
> - research/patterns.md

### Non-Blocking Behavior

Evaluation is **always secondary** to the agent's primary output. Three rules ensure evaluations never disrupt the pipeline:

1. **Evaluation failure must never cause the agent's completion status to be ERROR.** If evaluation cannot be completed, the agent writes an `evaluation_error` block and proceeds.
2. **Evaluation is secondary to primary output.** The agent's task deliverables always take priority.
3. **The agent returns DONE with its primary output intact** even if evaluation generation fails entirely.

### Shared Schema Reference

All 14 agents reference the same centralized schema document (`.github/agents/evaluation-schema.md`) rather than having the schema inlined in each agent file. This means:

- Schema changes require updating only one file
- All agents are guaranteed to use the same schema version
- The schema document includes field-level constraints, the error fallback format, collision avoidance rules, and an output file format example

---

## 4. How to Trigger the PostMortem Agent

### Pipeline Step 8 (Automatic)

The PostMortem agent is dispatched automatically as **Step 8** of the orchestrator pipeline, immediately after the R cluster completes at Step 7. No manual action is required.

```
Step 7:  R Cluster (r-quality, r-security, r-testing, r-knowledge)
           ↓ (pipeline success/failure determined here)
Step 8:  PostMortem dispatch (non-blocking)
           ↓
         Pipeline complete
```

### Dispatch Details

The orchestrator dispatches the PostMortem agent with the following context:

| Dispatch Parameter    | Contents                                              |
| --------------------- | ----------------------------------------------------- |
| Feature slug          | The current feature's slug identifier                 |
| Run date              | Current date for file naming                          |
| Telemetry dataset     | Full accumulated telemetry from Steps 1–7 (see below) |
| Evaluation files path | `artifact-evaluations/`                               |
| Memory files path     | `memory/`                                             |

### Telemetry Context Tracking

Throughout Steps 1–7, the orchestrator accumulates execution telemetry **in its working context** (not in any file). After each `runSubagent` return, it records:

- Agent name, pipeline step, dispatch pattern
- Completion status (DONE/NEEDS_REVISION/ERROR)
- Retry count, failure reason (if any)
- Iteration number (for Pattern C loops)
- Human intervention flag
- Best-effort timestamp

At Step 8, the orchestrator includes this data as a structured Markdown table in the PostMortem dispatch prompt:

```markdown
## Dispatch Telemetry

### Agent Dispatches

| Agent                   | Step | Pattern    | Status | Retries | Failure | Iteration | Human | Timestamp  |
| ----------------------- | ---- | ---------- | ------ | ------- | ------- | --------- | ----- | ---------- |
| researcher-architecture | 1.1  | A          | DONE   | 0       | —       | 1         | no    | 2026-02-20 |
| spec                    | 2    | sequential | DONE   | 0       | —       | 1         | no    | 2026-02-20 |
| ...                     | ...  | ...        | ...    | ...     | ...     | ...       | ...   | ...        |

### Cluster Summaries

| Step | Cluster  | Dispatched | Errors | Outcome |
| ---- | -------- | ---------- | ------ | ------- |
| 1.1  | research | 4          | 0      | DONE    |
| 3b   | ct       | 4          | 0      | DONE    |
| ...  | ...      | ...        | ...    | ...     |
```

### Non-Blocking Behavior

**PostMortem ERROR does NOT change the pipeline outcome.** Pipeline success or failure is determined at Step 7 (R cluster evaluation). Step 8 runs regardless of the pipeline's final state.

If PostMortem returns ERROR:

- The orchestrator logs a warning in `memory.md` Recent Updates
- Memory merge (Step 8.2) is skipped
- The pipeline result remains unchanged

### PostMortem Outputs

The PostMortem agent produces three files:

| Output             | Path                                     | Contents                                                                                                             |
| ------------------ | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Run log            | `agent-metrics/<run-date>-run-log.md`    | Expanded YAML telemetry: per-agent `agent_telemetry` entries and `cluster_summary` entries                           |
| Post-mortem report | `post-mortems/<run-date>-post-mortem.md` | Quantitative analysis: `recurring_issues`, `bottlenecks`, `most_common_missing_information`, `agent_accuracy_scores` |
| Isolated memory    | `memory/post-mortem.mem.md`              | Standard memory format with key findings                                                                             |

After PostMortem completes (DONE), the orchestrator reads `memory/post-mortem.mem.md` and merges Key Findings, Decisions, and Artifact Index into shared `memory.md`.

### PostMortem / R-Knowledge Boundary

The PostMortem agent focuses on **quantitative metrics only**: scores, retry rates, frequencies, accuracy counts. It does NOT produce improvement recommendations, knowledge-suggestions, or decisions.md entries. R-Knowledge (Step 7, R cluster) remains the sole source of qualitative improvement suggestions.

| Aspect       | PostMortem (Step 8)                        | R-Knowledge (Step 7)                           |
| ------------ | ------------------------------------------ | ---------------------------------------------- |
| Focus        | Quantitative — scores, frequencies, counts | Qualitative — knowledge evolution, suggestions |
| Outputs      | Run log + post-mortem report               | knowledge-suggestions.md + decisions.md        |
| Blocking     | Non-blocking                               | Non-blocking                                   |
| Data sources | Evaluations + telemetry                    | Pipeline artifacts + codebase + git diff       |

---

## Appendix: Orchestrator Tool Restriction (Track B)

As part of this feature, the orchestrator's tool access was restricted independently (Track B):

- **Allowed tools:** `agent`, `agent/runSubagent`, `memory`, `read_file`, `grep_search`, `semantic_search`, `file_search`, `list_dir`, `get_errors`
- **Prohibited tools:** `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`
- **Impact:** All file creation and modification is now delegated to subagents via `runSubagent`. Read tools are retained for orchestration decisions (cluster decision flows).
- **Step 0:** Rewritten to delegate `initial-request.md` creation and `memory.md` initialization to subagents or the `memory` tool.

This restriction was implemented as an independent task (Task 03) and has no runtime dependency on the evaluation/telemetry/PostMortem features (Track A).

---

## File Reference Index

| Path                                           | Type     | Role in Self-Improvement System                    |
| ---------------------------------------------- | -------- | -------------------------------------------------- |
| `.github/agents/evaluation-schema.md`          | New      | Shared evaluation YAML schema (v1)                 |
| `.github/agents/post-mortem.agent.md`          | New      | PostMortem agent definition                        |
| `.github/agents/orchestrator.agent.md`         | Modified | Step 8, telemetry, tool restriction, doc structure |
| `.github/prompts/feature-workflow.prompt.md`   | Modified | New rules and artifact references                  |
| `.github/agents/spec.agent.md`                 | Modified | Evaluation step (4 research files)                 |
| `.github/agents/designer.agent.md`             | Modified | Evaluation step (feature.md)                       |
| `.github/agents/planner.agent.md`              | Modified | Evaluation step (design.md, feature.md)            |
| `.github/agents/ct-security.agent.md`          | Modified | Evaluation step (design.md, feature.md)            |
| `.github/agents/ct-scalability.agent.md`       | Modified | Evaluation step (design.md, feature.md)            |
| `.github/agents/ct-maintainability.agent.md`   | Modified | Evaluation step (design.md, feature.md)            |
| `.github/agents/ct-strategy.agent.md`          | Modified | Evaluation step (design.md, feature.md)            |
| `.github/agents/implementer.agent.md`          | Modified | Evaluation step (task file, design.md, feature.md) |
| `.github/agents/documentation-writer.agent.md` | Modified | Evaluation step (task file, design.md, feature.md) |
| `.github/agents/v-tests.agent.md`              | Modified | Evaluation step (v-build.md)                       |
| `.github/agents/v-tasks.agent.md`              | Modified | Evaluation step (v-build.md, plan.md, tasks)       |
| `.github/agents/v-feature.agent.md`            | Modified | Evaluation step (v-build.md, feature.md)           |
| `.github/agents/r-quality.agent.md`            | Modified | Evaluation step (design.md)                        |
| `.github/agents/r-testing.agent.md`            | Modified | Evaluation step (feature.md)                       |
