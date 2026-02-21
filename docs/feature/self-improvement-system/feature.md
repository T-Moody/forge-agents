# Self-Improvement System for Forge

**Summary:** A structured cross-agent evaluation and post-mortem learning system enabling Forge pipeline agents to produce quantitative artifact evaluations, the orchestrator to capture execution telemetry, and a new PostMortem agent to synthesize cross-run improvement insights — all as additive, non-breaking extensions to the existing 8-step pipeline.

---

## Background & Context

Forge is a 19-agent, 8-step deterministic pipeline orchestrated via `.agent.md` Markdown files. Agents communicate through Markdown artifacts stored under `docs/feature/<feature-slug>/` with a dual-layer memory system (isolated per-agent + shared `memory.md`). Currently, no agent evaluates upstream artifact quality, no execution telemetry is captured, and no systematic post-mortem analysis exists.

**Research references:**

- [research/architecture.md](research/architecture.md) — Repository structure, pipeline architecture, memory system, tool usage patterns
- [research/impact.md](research/impact.md) — Artifact consumption map, 14 agents requiring modification, orchestrator changes, new storage directories
- [research/dependencies.md](research/dependencies.md) — Artifact consumption chain, 9 dispatch telemetry points, tool restriction conflicts, r-knowledge/post-mortem overlap
- [research/patterns.md](research/patterns.md) — Agent file format, memory format, YAML/structured output patterns (none exist yet), non-blocking agent pattern, KE-SAFE constraints

---

## Functional Requirements

### FR-1: Artifact Evaluation Capability (PART 1)

**FR-1.1:** Each of the 14 agents that consume upstream pipeline-produced artifacts MUST produce a structured artifact evaluation as part of its workflow. The 14 agents are: spec, designer, ct-security, ct-scalability, ct-maintainability, ct-strategy, planner, implementer, documentation-writer, v-tests, v-tasks, v-feature, r-quality, r-testing.

**FR-1.2:** Each artifact evaluation MUST be a YAML block within a fenced code block inside a Markdown file, using this exact schema:

```yaml
artifact_evaluation:
  evaluator: "<agent-name>"
  source_artifact: "<relative path to evaluated artifact>"
  usefulness_score: <integer 1-10>
  clarity_score: <integer 1-10>
  useful_elements:
    - "<specific section or element that helped>"
  missing_information:
    - "<concrete missing requirement or detail>"
  information_not_used:
    - "<information read but not used>"
  inaccuracies:
    - "<incorrect assumption or factual error>"
  impact_on_work:
    - "<description of rework or clarification needed>"
```

**FR-1.3:** If an agent consumes multiple upstream artifacts, it MUST produce one `artifact_evaluation` block per source artifact, all within the same evaluation file.

**FR-1.4:** Evaluation files MUST be stored in `docs/feature/<feature-slug>/artifact-evaluations/` with the naming convention `<evaluator-agent-name>.md` (e.g., `spec.md`, `ct-security.md`, `implementer-<task-id>.md`).

**FR-1.5:** Each evaluating agent's Outputs section in its `.agent.md` file MUST be updated to include the evaluation file path.

**FR-1.6:** Each evaluating agent's Workflow section MUST include an evaluation step that runs after reading upstream artifacts and before the self-verification step (if one exists) or before writing isolated memory.

**FR-1.7:** List fields (`useful_elements`, `missing_information`, `information_not_used`, `inaccuracies`, `impact_on_work`) MUST contain at least one entry each. If no items apply for a field, the agent MUST use a single entry: `"N/A — none identified"`.

**FR-1.8:** The 5 agents excluded from artifact evaluation (researcher ×4, v-build, r-security, r-knowledge) MUST NOT be modified to produce evaluations since they do not consume upstream pipeline-produced artifacts (they consume only initial-request.md, codebase, or git diff).

### FR-2: Orchestrator Telemetry Tracking (PART 2)

**FR-2.1:** The orchestrator MUST capture structured telemetry metadata for every subagent invocation at all 9 dispatch points: Steps 1.1, 2, 3, 3b, 4, 5, 6.1, 6.2, and 7.

**FR-2.2:** Each telemetry entry MUST include the following fields:

```yaml
agent_telemetry:
  agent_name: "<agent-name>"
  pipeline_step: "<step identifier>"
  dispatch_pattern: "<A|B|C|sequential>"
  start_timestamp: "<ISO 8601 or best-effort>"
  end_timestamp: "<ISO 8601 or best-effort>"
  retry_count: <integer, 0 if no retries>
  completion_status: "<DONE|NEEDS_REVISION|ERROR>"
  failure_reason: "<reason string or null>"
  iteration_number: <integer, for Pattern C loops>
  human_intervention_required: <true|false>
```

**FR-2.3:** The `start_timestamp` and `end_timestamp` fields are best-effort — if the LLM runtime does not provide a clock, the orchestrator MUST record the current date context (available via `{{CURRENT_DATE}}` or system context) and note "timestamp precision limited by runtime" in the telemetry file header.

**FR-2.4:** Telemetry MUST be stored in `docs/feature/<feature-slug>/agent-metrics/run-log.md` as a Markdown file containing YAML code blocks, one block per agent invocation. The file MUST include a header with the pipeline run context (feature slug, run date).

**FR-2.5:** For Pattern C replan loops (Steps 5–6), each iteration MUST generate its own telemetry entries with incrementing `iteration_number`.

**FR-2.6:** Cluster-level summary telemetry (total agents dispatched, total errors, cluster outcome) MUST be appended after individual agent entries for each cluster dispatch (Steps 1.1, 3b, 6.2, 7).

**FR-2.7:** Given the orchestrator tool restriction (FR-6), telemetry writing MUST be accomplished via the `memory` tool or delegated to a subagent — the orchestrator MUST NOT use direct file-write tools.

### FR-3: Post-Mortem Agent (PART 3)

**FR-3.1:** A new agent file MUST be created at `.github/agents/post-mortem.agent.md` following the standard `chatagent` format: YAML frontmatter (`name: post-mortem`, `description`), Inputs, Outputs, 6 Operating Rules, Workflow, Completion Contract, Anti-Drift Anchor.

**FR-3.2:** The PostMortem agent's Inputs MUST include:

- `docs/feature/<feature-slug>/artifact-evaluations/*.md` (all evaluation files)
- `docs/feature/<feature-slug>/agent-metrics/run-log.md` (telemetry log)
- `docs/feature/<feature-slug>/memory.md` (shared pipeline memory)
- `docs/feature/<feature-slug>/memory/*.mem.md` (all isolated agent memories)

**FR-3.3:** The PostMortem agent MUST produce a structured report in `docs/feature/<feature-slug>/post-mortems/<run-date>-post-mortem.md` containing a YAML code block with this schema:

```yaml
post_mortem_report:
  run_date: "<date>"
  feature_slug: "<slug>"
  summary: "<one-line summary>"
  recurring_issues:
    - issue: "<description>"
      affected_agents: ["<agent1>", "<agent2>"]
      frequency: <count across evaluations>
  bottlenecks:
    - agent: "<agent-name>"
      reason: "<why it is a bottleneck>"
      retry_count: <total retries>
  most_common_missing_information:
    - item: "<missing info description>"
      reported_by: ["<agent1>", "<agent2>"]
      source_artifact: "<artifact that should have included it>"
  agent_accuracy_scores:
    - agent: "<agent-name>"
      average_usefulness: <float 1-10>
      average_clarity: <float 1-10>
      total_inaccuracies_reported: <integer>
  improvement_recommendations:
    - recommendation: "<actionable recommendation>"
      target_agent: "<agent that should improve>"
      priority: "<high|medium|low>"
      evidence: "<citations from evaluations>"
```

**FR-3.4:** The PostMortem agent MUST also write an isolated memory file to `memory/post-mortem.mem.md` following the standard memory file format.

**FR-3.5:** The PostMortem agent MUST follow the non-blocking pattern established by R-Knowledge: its `ERROR` does not block the pipeline. It MUST use a two-state completion contract (`DONE:` / `ERROR:` only).

**FR-3.6:** The PostMortem agent MUST be read-only with respect to all existing pipeline artifacts and the codebase — it MUST NOT modify any file outside its declared Outputs.

**FR-3.7:** The PostMortem agent MUST implement a self-verification step before returning, verifying: (a) all evaluation files were processed, (b) all telemetry entries were accounted for, (c) recommendations are grounded in specific evaluation data.

**FR-3.8 (R-Knowledge / PostMortem Boundary):** The PostMortem agent focuses on **quantitative cross-agent metrics** (scores, retry rates, missing info frequency) derived from structured evaluation and telemetry data. R-Knowledge focuses on **qualitative pattern analysis** (knowledge evolution, forward-looking suggestions for agent definition changes). Neither agent subsumes the other. The PostMortem agent MUST NOT produce knowledge-suggestions or decisions.md entries. R-Knowledge MUST NOT produce post-mortem reports.

### FR-4: Storage Design (PART 4)

**FR-4.1:** Three new directories MUST be created under `docs/feature/<feature-slug>/`:

- `agent-metrics/` — orchestrator telemetry logs
- `artifact-evaluations/` — structured evaluation files from consuming agents
- `post-mortems/` — PostMortem agent reports

**FR-4.2:** All storage MUST be per-feature scoped (under `docs/feature/<feature-slug>/`), consistent with the existing documentation structure convention.

**FR-4.3:** Data MUST be append-only. No existing files in these directories MUST be overwritten. New pipeline runs MUST create new files (e.g., date-stamped) rather than replacing previous ones.

**FR-4.4:** The orchestrator's Documentation Structure section MUST be updated to include the 3 new directories.

**FR-4.5:** Directory creation MUST occur during Step 0 (Setup) or lazily on first write. If the orchestrator is tool-restricted (FR-6), directory creation MUST be delegated to a setup subagent or handled via the `memory` tool.

### FR-5: Orchestrator Pipeline Extension (PART 3 Integration)

**FR-5.1:** The orchestrator MUST add a new **Step 8 (Post-Mortem)** after Step 7 (R Cluster) in its pipeline workflow.

**FR-5.2:** Step 8 MUST dispatch the PostMortem agent via `runSubagent` and be non-blocking — pipeline success is determined by the R cluster evaluation at Step 7. The PostMortem runs regardless of whether the pipeline result is DONE or includes NEEDS_REVISION cycles.

**FR-5.3:** After the PostMortem agent completes, the orchestrator MUST merge the PostMortem agent's isolated memory into shared `memory.md`.

**FR-5.4:** The orchestrator's Parallel Execution Summary MUST be updated to include Step 8.

**FR-5.5:** The orchestrator's "Orchestrator Expectations Per Agent" table MUST include a row for the PostMortem agent with DONE and ERROR handling expectations.

### FR-6: Orchestrator Tool Restriction (Additional Requirement)

**FR-6.1:** The orchestrator's tool access MUST be restricted to only: `agent`, `agent/runSubagent`, and `memory`. All references to direct file creation, file modification, and file reading tools in the orchestrator agent definition MUST be removed or replaced with delegation via `runSubagent`.

**FR-6.2:** The orchestrator's Step 0 (Setup) operations — creating `initial-request.md`, initializing `memory.md`, and creating the `memory/` directory — MUST be delegated to a setup subagent or handled via the `memory` tool, since direct file-write tools are no longer available.

**FR-6.3:** All memory lifecycle operations that the orchestrator currently performs via direct file writes (merge, prune, invalidate, clean) MUST be restructured to use only the `memory` tool or be delegated to subagents.

**FR-6.4:** The orchestrator's Operating Rules section MUST be updated to explicitly list allowed tools as `[agent, agent/runSubagent, memory]` and state that no other tools are available.

**FR-6.5:** All instructions in the orchestrator that reference making file edits directly MUST be reworded to delegate those operations to subagents via `runSubagent`.

**FR-6.6:** Global Rule 1 ("Never modify code or documentation directly") is already directionally aligned — this restriction formalizes it by removing the tools that would enable direct modification.

### FR-7: Safety Requirements (PART 5)

**FR-7.1:** All changes MUST be additive — no existing agent behaviors, artifact formats, or completion contracts may be altered.

**FR-7.2:** The existing 3-state completion contract (`DONE:` / `NEEDS_REVISION:` / `ERROR:`) MUST remain unchanged for all existing agents.

**FR-7.3:** No new external dependencies (npm packages, APIs, databases, build tools) MUST be introduced.

**FR-7.4:** Agents excluded from artifact evaluation (researcher ×4, v-build, r-security, r-knowledge) MUST NOT be modified.

**FR-7.5:** Existing memory file formats and the memory lifecycle MUST remain unchanged. New memory files (e.g., `post-mortem.mem.md`) follow the existing format.

### FR-8: Deliverables Documentation (PART 6)

**FR-8.1:** After implementation, a summary document MUST be produced covering:

1. A summary of all changes made (which files were modified, which were created)
2. Where evaluation data is stored (path and format)
3. How agents now generate evaluations (workflow step description)
4. How to trigger the PostMortem agent (pipeline step 8 behavior)

---

## Non-Functional Requirements

**NFR-1 (Additive-Only):** All changes must be strictly additive. No existing agent file may have its current behavior altered — only new sections appended. No existing artifact format may change.

**NFR-2 (Structured Output):** All new data artifacts (evaluations, telemetry, post-mortem reports) MUST use YAML within fenced code blocks (` ```yaml ... ``` `) inside Markdown files, preserving the Markdown-first ecosystem while enabling structured parsing.

**NFR-3 (Consistency):** The PostMortem agent MUST follow the identical agent file format pattern as all 19 existing agents: YAML frontmatter, standard sections, 6 Operating Rules (with agent-specific tool preferences), Workflow, Completion Contract, Anti-Drift Anchor.

**NFR-4 (Non-Blocking):** The PostMortem agent and the evaluation workflow step MUST NOT increase pipeline failure modes. Evaluation generation failure within an agent MUST NOT cause the agent to return `ERROR:`. Evaluation is secondary to the agent's primary output.

**NFR-5 (Deterministic Output):** Post-mortem reports MUST be deterministic and structured — no free-form prose in the quantitative sections. Only the `summary` and recommendation `evidence` fields may contain natural-language text.

**NFR-6 (Graceful Degradation):** If artifact evaluation files or telemetry files are missing or corrupted when the PostMortem agent runs, the PostMortem agent MUST note the gaps and proceed with available data, not fail.

**NFR-7 (Per-Feature Isolation):** All new storage (evaluations, metrics, post-mortems) MUST be scoped under the feature directory `docs/feature/<feature-slug>/` to maintain feature isolation and prevent cross-feature contamination.

---

## Constraints & Assumptions

### Constraints

- **C-1:** Agents are defined as `.agent.md` files in `.github/agents/` using the `chatagent` Markdown format with YAML frontmatter.
- **C-2:** All inter-agent communication is via Markdown files stored under `docs/feature/<feature-slug>/`.
- **C-3:** The orchestrator is the sole writer to shared `memory.md`.
- **C-4:** Maximum 4 concurrent subagent invocations (existing concurrency cap).
- **C-5:** The system is prompt-driven (GitHub Copilot agent runtime), not programmatic code — no runtime APIs, timers, or event hooks are available beyond what the LLM runtime provides.
- **C-6:** No test framework or build system exists — the repository contains only agent definitions and documentation.

### Assumptions

- **A-1:** The `memory` tool provided by the GitHub Copilot runtime supports reading and writing files within the `memory/` directory scope, and potentially `memory.md` itself.
- **A-2:** Timestamps are available via `{{CURRENT_DATE}}` or system context at minimum day-level granularity; precise millisecond execution timing is assumed unavailable.
- **A-3:** `runSubagent` returns the completion contract string and the orchestrator can subsequently read the agent's isolated memory file and output files.
- **A-4:** The orchestrator tool restriction (`[agent, agent/runSubagent, memory]`) can be enforced via the agent runtime's tool-access configuration.
- **A-5:** The R-Knowledge agent will not be modified as part of this feature; its existing behavior remains as-is.

---

## Acceptance Criteria

### AC-1: Artifact Evaluation Production

- **Pass:** All 14 specified agent `.agent.md` files contain an evaluation workflow step and an evaluation output in their Outputs section. Each of the 14 agents, when run, produces a Markdown file in `artifact-evaluations/` containing valid YAML code blocks matching the `artifact_evaluation` schema (FR-1.2). All required YAML fields are present with correct types.
- **Fail:** Any of the 14 agents lacks the evaluation step, produces no evaluation file, produces invalid YAML, or is missing required fields.

### AC-2: Evaluation Schema Compliance

- **Pass:** Every `artifact_evaluation` YAML block across all agents contains exactly the fields defined in FR-1.2 with correct types: `evaluator` (string), `source_artifact` (string), `usefulness_score` (integer 1–10), `clarity_score` (integer 1–10), and lists for `useful_elements`, `missing_information`, `information_not_used`, `inaccuracies`, `impact_on_work` (each with ≥1 entry).
- **Fail:** Any evaluation block has missing fields, wrong types (e.g., score outside 1–10), or empty lists.

### AC-3: Excluded Agents Unchanged

- **Pass:** The 5 excluded agents (researcher, v-build, r-security, r-knowledge) have zero modifications to their `.agent.md` files.
- **Fail:** Any excluded agent file is modified.

### AC-4: Orchestrator Telemetry Capture

- **Pass:** After a pipeline run, `agent-metrics/run-log.md` exists and contains YAML telemetry entries for every agent dispatched (at all 9 dispatch points), each entry containing all fields from FR-2.2. Cluster-level summaries exist for cluster dispatches.
- **Fail:** Telemetry file is missing, entries are missing for dispatched agents, or required fields are absent.

### AC-5: Orchestrator Tool Restriction

- **Pass:** The orchestrator `.agent.md` file explicitly lists allowed tools as `[agent, agent/runSubagent, memory]` only. No instructions in the file reference using file-write, file-read, file-create, `grep_search`, `semantic_search`, `read_file`, `run_in_terminal`, or any other tools directly. All former direct file operations are delegated via `runSubagent` or `memory` tool.
- **Fail:** The orchestrator file references or instructs use of any tool outside the allowed set.

### AC-6: PostMortem Agent Exists and Is Correct

- **Pass:** `.github/agents/post-mortem.agent.md` exists, follows the standard `chatagent` format (YAML frontmatter, 6 Operating Rules, Workflow, Completion Contract, Anti-Drift Anchor), declares correct Inputs/Outputs, uses two-state completion contract (`DONE:`/`ERROR:`), includes a self-verification step, and includes a read-only enforcement section.
- **Fail:** File is missing, format diverges from standard, completion contract includes `NEEDS_REVISION`, or read-only enforcement is absent.

### AC-7: PostMortem Agent Is Non-Blocking

- **Pass:** The orchestrator's Step 8 dispatches the PostMortem agent and explicitly states that PostMortem failure (`ERROR:`) does not block pipeline completion. Pipeline success/failure is still determined at Step 7 (R cluster evaluation).
- **Fail:** PostMortem `ERROR` blocks or changes the pipeline outcome.

### AC-8: PostMortem Report Schema

- **Pass:** The PostMortem agent produces a file in `post-mortems/` containing a YAML code block matching the `post_mortem_report` schema (FR-3.3) with all required top-level keys and correct nested structure.
- **Fail:** Report is missing, invalid YAML, or missing required keys.

### AC-9: Storage Directories

- **Pass:** Three directories (`agent-metrics/`, `artifact-evaluations/`, `post-mortems/`) are documented in the orchestrator's Documentation Structure section and are created during pipeline execution (Step 0 or on first write).
- **Fail:** Any of the 3 directories is missing from documentation or not created during execution.

### AC-10: Append-Only Storage

- **Pass:** Multiple pipeline runs on the same feature produce distinct files in all 3 new directories (e.g., date-stamped post-mortem files). No existing file is overwritten.
- **Fail:** A second pipeline run overwrites a file produced by a first run.

### AC-11: No Breaking Changes

- **Pass:** All existing agents' completion contracts are unchanged. All existing artifact formats (feature.md, design.md, plan.md, tasks/_.md, ct-review/_, verification/_, review/_) are produced in the same format as before. All existing agent Inputs sections read the same files. The pipeline can still complete an end-to-end run with the new changes added.
- **Fail:** Any existing completion contract changes, any existing artifact format changes, or the pipeline fails due to the new additions.

### AC-12: R-Knowledge / PostMortem Boundary

- **Pass:** R-Knowledge agent is unmodified. PostMortem agent does not produce `knowledge-suggestions.md` or `decisions.md`. R-Knowledge does not produce post-mortem reports. The two agents have distinct, non-overlapping outputs.
- **Fail:** Either agent produces artifacts belonging to the other's scope.

### AC-13: Evaluation Non-Blocking

- **Pass:** If an agent's evaluation generation fails (e.g., cannot compute a score), the agent still produces its primary output (feature.md, design.md, etc.) and returns `DONE:` with a note about evaluation failure. The agent does NOT return `ERROR:` solely due to evaluation failure.
- **Fail:** An agent returns `ERROR:` because evaluation generation failed, despite primary output being complete.

### AC-14: Deliverables Documentation

- **Pass:** A summary document exists after implementation that covers the 4 items in FR-8.1 (changes made, evaluation storage, evaluation workflow, post-mortem trigger).
- **Fail:** Summary is missing or omits any of the 4 required items.

---

## Edge Cases & Error Handling

### EC-1: Agent Consumes Zero Applicable Upstream Artifacts

- **Input/Condition:** An agent listed as an evaluator has no upstream artifacts available (e.g., research files were not produced due to researcher errors).
- **Expected Behavior:** The agent produces its primary output without an evaluation file, notes "no upstream artifacts available for evaluation" in its isolated memory, and returns `DONE:`.
- **Severity if Missed:** Medium — evaluation data would be silently missing without explanation.

### EC-2: Evaluation YAML Generation Failure

- **Input/Condition:** An agent cannot generate valid YAML for the evaluation (e.g., LLM formatting error, schema violation).
- **Expected Behavior:** The agent completes its primary output, writes a partial or placeholder evaluation file with an error note, and returns `DONE:` (not `ERROR:`). The evaluation file contains an `evaluation_error` field describing the failure.
- **Severity if Missed:** High — agents could fail due to secondary evaluation task, breaking the pipeline.

### EC-3: Missing Telemetry Data (execution_time_ms Unavailable)

- **Input/Condition:** The LLM runtime does not provide precise timestamps for agent execution start/end.
- **Expected Behavior:** The telemetry file header notes "timestamp precision limited by runtime." The `start_timestamp` and `end_timestamp` fields use the best available data (e.g., date only) or `"unavailable"`. The telemetry entry is still produced with all other fields.
- **Severity if Missed:** Low — telemetry would be missing timing data; other fields still provide value.

### EC-4: PostMortem Agent Receives Empty Evaluation Directory

- **Input/Condition:** No agents produced evaluation files (all evaluations failed or the pipeline was cut short).
- **Expected Behavior:** The PostMortem agent produces a report with empty `recurring_issues`, `most_common_missing_information`, and `agent_accuracy_scores` arrays, a note in `summary` that "no evaluation data was available", and returns `DONE:`.
- **Severity if Missed:** Medium — PostMortem agent could crash or return `ERROR:` on a valid edge case.

### EC-5: PostMortem Agent Receives Corrupted YAML in Evaluation Files

- **Input/Condition:** One or more evaluation files contain invalid YAML (malformed code blocks, missing fields).
- **Expected Behavior:** The PostMortem agent skips the corrupted evaluation, logs which files were skipped in the report's `summary`, and processes remaining valid evaluations.
- **Severity if Missed:** High — the PostMortem agent could fail entirely due to one bad input file.

### EC-6: Orchestrator Tool Restriction Breaks Memory Merge

- **Input/Condition:** After restricting the orchestrator to `[agent, agent/runSubagent, memory]`, the orchestrator attempts a memory merge but the `memory` tool does not support the required operation (e.g., cannot write to `memory.md`).
- **Expected Behavior:** The orchestrator delegates the memory operation to a subagent via `runSubagent`. If delegation also fails, the orchestrator logs the failure in its completion output and continues with a degraded memory state (per existing non-blocking memory failure convention).
- **Severity if Missed:** Critical — memory merge is core to pipeline coordination; failure without fallback would break downstream agents.

### EC-7: Pattern C Replan Loop Telemetry Accumulation

- **Input/Condition:** A Pattern C replan loop runs the maximum 3 iterations, producing telemetry entries for each iteration.
- **Expected Behavior:** Each iteration produces separate telemetry entries with incrementing `iteration_number`. The run-log contains all entries (not just the last iteration). Cluster summary reflects total retries across all iterations.
- **Severity if Missed:** Medium — post-mortem bottleneck analysis would be inaccurate without per-iteration data.

### EC-8: Concurrent Agent Evaluations Writing to Same Directory

- **Input/Condition:** During Pattern A parallel dispatch (e.g., 4 CT agents), all 4 agents attempt to write evaluation files to `artifact-evaluations/` simultaneously.
- **Expected Behavior:** Each agent writes to a uniquely named file (`ct-security.md`, `ct-scalability.md`, etc.), preventing write conflicts. No two agents write to the same file.
- **Severity if Missed:** High — file corruption or data loss from concurrent writes to the same file.

### EC-9: Post-Mortem Runs After Pipeline ERROR

- **Input/Condition:** The pipeline ends in `ERROR:` (e.g., R-Security finds a Blocker), but the orchestrator still dispatches Step 8.
- **Expected Behavior:** The PostMortem agent runs normally, analyzing whatever evaluations and telemetry were produced before the error. Its report includes the pipeline error context. Returns `DONE:` regardless of pipeline status.
- **Severity if Missed:** Medium — error runs would lack post-mortem analysis, missing valuable debugging data.

### EC-10: Feature Directory Does Not Have New Storage Directories at Agent Runtime

- **Input/Condition:** An evaluating agent runs but `artifact-evaluations/` does not yet exist (Step 0 did not create it, or lazy creation was expected but failed).
- **Expected Behavior:** The agent creates the directory as part of its file write operation, or includes the evaluation in its primary output file as a fallback, noting the directory creation issue in its memory.
- **Severity if Missed:** Medium — evaluation data could be silently lost.

### EC-11: Second Pipeline Run on Same Feature (Append-Only Constraint)

- **Input/Condition:** A second pipeline run executes on a feature that already has evaluation files, telemetry, and a post-mortem from a prior run.
- **Expected Behavior:** New files are created with distinct names (e.g., date-stamped for post-mortems, run-indexed for telemetry). Prior files remain unmodified.
- **Severity if Missed:** High — overwriting prior run data violates append-only requirement and destroys historical data.

---

## User Stories / Flows

### US-1: Standard Pipeline Run with Evaluations

1. Developer invokes the `feature-workflow.prompt.md` with a feature request.
2. Orchestrator runs Step 0 (Setup) — delegates directory creation (including `agent-metrics/`, `artifact-evaluations/`, `post-mortems/`) to a setup subagent.
3. Steps 1–7 execute normally. Each consuming agent (14 total) produces its primary output AND writes an evaluation file to `artifact-evaluations/`.
4. The orchestrator logs telemetry for every agent dispatch to `agent-metrics/run-log.md`.
5. After Step 7 (R Cluster) completes, the orchestrator dispatches Step 8 — the PostMortem agent.
6. The PostMortem agent reads all evaluation files and the telemetry log, produces a structured report in `post-mortems/<date>-post-mortem.md`, and writes its isolated memory.
7. The orchestrator merges PostMortem memory into `memory.md` and completes.

### US-2: Pipeline Run Where PostMortem Fails

1. Steps 0–7 complete normally; evaluations and telemetry are captured.
2. Step 8: PostMortem agent encounters an error (e.g., cannot parse evaluation files).
3. PostMortem returns `ERROR: <reason>`.
4. Orchestrator logs the PostMortem error but does NOT alter the pipeline outcome — if the pipeline was DONE at Step 7, it remains DONE.

### US-3: Developer Reviews Post-Mortem Report

1. After pipeline completion, developer navigates to `docs/feature/<slug>/post-mortems/`.
2. Developer reads the structured YAML report to identify recurring issues and bottleneck agents.
3. Developer uses `improvement_recommendations` to decide which agent definitions to update (possibly creating new knowledge-suggestions via R-Knowledge on a future run).

---

## Test Scenarios

### TS-1: Evaluation File Structure Validation

Verify each of the 14 modified `.agent.md` files has: (a) an evaluation file listed in Outputs, (b) an evaluation workflow step, (c) evaluation file uses YAML in fenced code block, (d) all schema fields present.

### TS-2: Evaluation Schema Field Validation

Verify every `artifact_evaluation` YAML block: `usefulness_score` is integer 1–10, `clarity_score` is integer 1–10, all list fields have ≥1 entry, `evaluator` matches the agent name, `source_artifact` is a valid relative path.

### TS-3: Excluded Agents Not Modified

Verify `researcher.agent.md`, `v-build.agent.md`, `r-security.agent.md`, `r-knowledge.agent.md` have zero diff from their pre-feature state.

### TS-4: Orchestrator Telemetry File Validation

Verify `agent-metrics/run-log.md` exists after a pipeline run and contains: (a) a header with feature slug and date, (b) one YAML entry per dispatched agent, (c) all fields from FR-2.2 present in each entry, (d) cluster-level summaries present for cluster dispatches.

### TS-5: Orchestrator Tool Restriction Compliance

Verify the orchestrator `.agent.md` file: (a) explicitly declares `[agent, agent/runSubagent, memory]` as its tool set, (b) contains no instructions referencing `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `read_file`, `grep_search`, `semantic_search`, `run_in_terminal`, or `file_search`, (c) all former file operations are replaced with `runSubagent` delegation or `memory` tool usage.

### TS-6: PostMortem Agent Format Compliance

Verify `post-mortem.agent.md`: (a) has YAML frontmatter with `name: post-mortem`, (b) has Inputs listing evaluation files, telemetry, and memory files, (c) has Outputs listing post-mortem report path and `memory/post-mortem.mem.md`, (d) has 6 Operating Rules matching the standard set, (e) has a Workflow with self-verification step, (f) has two-state Completion Contract, (g) has Anti-Drift Anchor, (h) has Read-Only Enforcement section.

### TS-7: PostMortem Report Schema Validation

Verify the PostMortem report file: (a) is in `post-mortems/` directory, (b) contains YAML in fenced code block, (c) has all top-level keys from FR-3.3, (d) nested structures have correct types (arrays of objects with required keys).

### TS-8: PostMortem Non-Blocking Verification

Verify the orchestrator's Step 8: (a) dispatches PostMortem, (b) explicitly states PostMortem ERROR does not block pipeline, (c) pipeline DONE/ERROR is determined at Step 7.

### TS-9: Storage Directories in Documentation Structure

Verify orchestrator's Documentation Structure section includes `agent-metrics/`, `artifact-evaluations/`, and `post-mortems/` with correct descriptions.

### TS-10: Append-Only Storage Verification

Verify that the design specifies unique file naming (timestamps, run indexes) for all new directories and that no agent instruction permits overwriting existing files.

### TS-11: No Breaking Changes Verification

Verify: (a) all existing completion contracts unchanged, (b) all existing Inputs sections unchanged, (c) all existing primary Outputs unchanged (only additions), (d) the deprecated `critical-thinker.agent.md` is not touched.

### TS-12: Evaluation Non-Blocking Verification

Verify each evaluating agent's workflow: (a) the evaluation step is after primary output generation, (b) instructions state that evaluation failure does not cause `ERROR:` return, (c) a fallback for evaluation failure is documented.

### TS-13: R-Knowledge / PostMortem Boundary Verification

Verify: (a) R-Knowledge agent file is unmodified, (b) PostMortem agent outputs do not include `knowledge-suggestions.md` or `decisions.md`, (c) PostMortem anti-drift anchor explicitly states quantitative analysis scope.

### TS-14: Orchestrator Step 8 and Agent Expectations Table

Verify: (a) Step 8 exists in the orchestrator workflow after Step 7, (b) the Orchestrator Expectations Per Agent table has a row for `post-mortem` with DONE and ERROR expectations, (c) Parallel Execution Summary includes Step 8.

### TS-15: Feature-Workflow Prompt Update

Verify `.github/prompts/feature-workflow.prompt.md` references the 3 new artifact directories and the PostMortem step.

---

## Dependencies & Risks

### Dependencies

| Dependency                                                  | Impact                                                                             | Risk Level                                                   |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| GitHub Copilot agent runtime (`runSubagent`, `memory` tool) | All delegation and memory operations depend on runtime tool availability           | Low — these are existing, verified capabilities              |
| LLM YAML generation accuracy                                | YAML code blocks in evaluations and reports must be valid and schema-compliant     | Medium — LLMs can produce malformed YAML                     |
| `memory` tool scope for tool restriction                    | If `memory` tool cannot write to `memory.md`, orchestrator memory operations break | High — must be verified before implementation                |
| 14 agent file modifications                                 | Each agent modification must be individually correct and consistent                | Medium — high volume of similar changes increases error risk |
| R-Knowledge non-blocking pattern                            | PostMortem agent design depends on this existing pattern                           | Low — well-established in codebase                           |

### Risks

| Risk                                                         | Likelihood | Impact                                                                              | Mitigation                                                                        |
| ------------------------------------------------------------ | ---------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| YAML formatting errors in LLM-generated evaluations          | High       | Medium — PostMortem can't parse evaluations                                         | EC-5 error handling, PostMortem skips corrupted files                             |
| `memory` tool insufficient for orchestrator tool restriction | Medium     | High — core memory lifecycle breaks                                                 | Design fallback: delegate all memory file writes to subagents via `runSubagent`   |
| Agent output bloat from evaluation sections                  | Medium     | Low — Markdown files grow but remain manageable                                     | Evaluations stored in separate files, not appended to primary outputs             |
| PostMortem produces low-signal reports                       | Medium     | Low — this is additive, not blocking                                                | Self-verification step validates grounding in evidence                            |
| Inconsistent evaluation implementation across 14 agents      | Medium     | Medium — some agents may miss fields or deviate from schema                         | Use a checklist verification during implementation; TS-1 and TS-2 validate schema |
| Timestamp precision degradation                              | High       | Low — quantitative timing data unavailable but other telemetry fields remain useful | Best-effort timestamps, graceful fallback to "unavailable"                        |
