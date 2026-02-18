# Feature: Agent-Isolated Memory System

**Summary:** Redesign the multi-agent orchestration memory architecture from a shared `memory.md` with aggregator-mediated writes to a branch-merge model where each subagent writes its own isolated memory file (`memory/<agent-name>.mem.md`), the orchestrator merges those into shared `memory.md`, and all aggregator agents are removed.

---

## Background & Context

The current system uses a single shared `memory.md` file with strict read-only/read-write access control. Parallel subagents are read-only; only aggregator agents and sequential agents may write. Three aggregator agents (ct-aggregator, v-aggregator, r-aggregator) and a researcher synthesis mode serve as sequential bottlenecks that merge parallel outputs into combined artifact files (`design_critical_review.md`, `verifier.md`, `review.md`, `analysis.md`). Downstream agents consume these combined files.

This architecture creates pipeline bottlenecks at aggregation points and adds unnecessary sequential steps. The proposed change eliminates aggregators, has each agent produce a compact isolated memory file alongside its artifact, and has the orchestrator read those memories for routing decisions while passing full artifact paths to downstream agents.

**Research references:**

- [research/architecture.md](research/architecture.md) — Repository structure, agent file conventions, current memory system, pipeline flow, aggregator architecture
- [research/impact.md](research/impact.md) — Per-file impact analysis (tiers 1–5), cascade effects, risk areas
- [research/dependencies.md](research/dependencies.md) — Data flow maps, input/output contracts, aggregator responsibility redistribution, completion contract migration
- [research/patterns.md](research/patterns.md) — Common conventions, memory access patterns, completion contracts, dispatch patterns, error handling, reusable patterns

---

## Functional Requirements

### FR-1: Agent-Isolated Memory Files

**FR-1.1:** Every subagent MUST produce an isolated memory file as an additional output alongside its primary artifact. The memory file path convention is `memory/<agent-name>.mem.md` (e.g., `memory/ct-security.mem.md`, `memory/researcher-architecture.mem.md`).

**FR-1.2:** The isolated memory file MUST be a compact summary containing:

- Agent name and role
- Completion status (DONE / NEEDS_REVISION / ERROR)
- Key findings (≤5 bullet points)
- Highest severity finding (if applicable, using the agent's severity taxonomy)
- Decisions made (if any)
- Artifacts produced (file paths)

**FR-1.3:** The isolated memory file is NOT a replacement for the agent's primary artifact. The primary artifact (e.g., `ct-review/ct-security.md`, `research/architecture.md`) is still produced with full content.

**FR-1.4:** Each agent's Outputs section MUST list the isolated memory file.

**FR-1.5:** Each agent's Workflow section MUST include a step to write key findings to the isolated memory file. This step occurs after the agent completes its primary analysis but before returning the completion contract.

**FR-1.6:** All agents that previously had a "No Memory Write" workflow step MUST replace it with an "Write Own Memory" step that writes to `memory/<agent-name>.mem.md`.

### FR-2: Orchestrator Memory Merge

**FR-2.1:** After each agent completes (or after each parallel cluster completes), the orchestrator MUST read the agent's isolated memory file and merge its contents into the shared `memory.md`.

**FR-2.2:** The orchestrator MUST be the sole writer to shared `memory.md`. No other agent writes to `memory.md` directly.

**FR-2.3:** The Memory Lifecycle Actions in `orchestrator.agent.md` MUST include a new "Merge" action: after each agent completes, read isolated memory → update Artifact Index, Recent Decisions, Recent Updates in `memory.md`.

**FR-2.4:** For parallel clusters (CT ×4, V ×3+gate, R ×4, Research ×4, Implementation waves), the orchestrator MUST merge all isolated memories from the cluster after all agents in the cluster return, before making routing decisions.

**FR-2.5:** The memory-first protocol (Operating Rule 6) remains: agents still read `memory.md` first to orient, but write only to their isolated memory file.

### FR-3: Remove Aggregator Agents

**FR-3.1:** The files `ct-aggregator.agent.md`, `v-aggregator.agent.md`, and `r-aggregator.agent.md` MUST be removed (deleted or marked as deprecated with clear deprecation notice).

**FR-3.2:** No combined artifact files are produced:

- No `design_critical_review.md` (was produced by ct-aggregator)
- No `verifier.md` (was produced by v-aggregator)
- No `review.md` (was produced by r-aggregator)

**FR-3.3:** All references to aggregator agents MUST be removed from `orchestrator.agent.md`, `dispatch-patterns.md`, `feature-workflow.prompt.md`, and any sub-agent files that mention them.

### FR-4: Orchestrator Absorbs Aggregator Decision Logic

**FR-4.1 — CT Cluster Completion Logic:** The orchestrator MUST read CT sub-agent isolated memories and determine:

- Any finding rated Critical or High severity → `NEEDS_REVISION` (route to designer)
- All findings Medium or Low → `DONE`
- <2 sub-agent outputs → `ERROR`

**FR-4.2 — V Cluster Completion Logic:** The orchestrator MUST read V sub-agent isolated memories and determine pass/fail using the existing decision table:

| V-Build | V-Tests        | V-Tasks        | V-Feature      | Result                   |
| ------- | -------------- | -------------- | -------------- | ------------------------ |
| DONE    | DONE           | DONE           | DONE           | DONE                     |
| DONE    | NEEDS_REVISION | any            | any            | NEEDS_REVISION           |
| DONE    | any            | NEEDS_REVISION | any            | NEEDS_REVISION           |
| DONE    | any            | any            | NEEDS_REVISION | NEEDS_REVISION           |
| ERROR   | any            | any            | any            | ERROR                    |
| DONE    | ERROR          | not ERROR      | not ERROR      | Proceed with available   |
| DONE    | not ERROR      | ERROR          | not ERROR      | Proceed with available   |
| DONE    | not ERROR      | not ERROR      | ERROR          | Proceed with available   |
| any     | —              | —              | —              | 2+ missing/ERROR → ERROR |

**FR-4.3 — R Cluster Completion Logic:** The orchestrator MUST read R sub-agent isolated memories and determine:

1. R-Security ERROR or Blocker/Critical findings → pipeline `ERROR` (safety-critical override)
2. R-Security missing → `ERROR` (mandatory agent)
3. 2+ non-knowledge sub-agents ERROR/missing → `ERROR`
4. ≥1 Major finding (no Blocker) → `NEEDS_REVISION`
5. All remaining → `DONE`
6. R-Knowledge status MUST NOT affect the overall result (non-blocking)

**FR-4.4:** The orchestrator MUST read only compact memory summaries for these decisions, NOT full artifact files.

**FR-4.5:** Each CT sub-agent's isolated memory MUST include its highest-severity finding so the orchestrator can evaluate FR-4.1 without reading full CT artifacts.

**FR-4.6:** R-Security's isolated memory MUST include an explicit highest-severity field so the orchestrator can evaluate the pipeline override (FR-4.3, rule 1).

### FR-5: Remove `analysis.md` and Researcher Synthesis Mode

**FR-5.1:** The researcher's synthesis mode (Mode 2) MUST be removed from `researcher.agent.md`. Only focused mode remains.

**FR-5.2:** `analysis.md` is no longer produced. All references to `analysis.md` MUST be removed across all agent files.

**FR-5.3:** Downstream agents that previously read `analysis.md` MUST read individual research files directly:

- `spec.agent.md` — Inputs change to include `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md`
- `designer.agent.md` — Same input change
- `planner.agent.md` — Same input change

**FR-5.4:** The orchestrator MUST remove Step 1.2 (Synthesize Research) entirely. After research agents complete (Step 1.1), the orchestrator reads their isolated memories and merges to `memory.md`, then proceeds directly to Step 2 (Spec).

### FR-6: Remove Combined Cluster Output Files

**FR-6.1:** `design_critical_review.md` is no longer produced. The designer in revision mode MUST read individual `ct-review/ct-*.md` files directly.

**FR-6.2:** `verifier.md` is no longer produced. The planner in replan mode MUST read individual V sub-agent artifacts directly (`verification/v-tasks.md` for failing task IDs, `verification/v-tests.md` for test failures, `verification/v-feature.md` for criteria gaps).

**FR-6.3:** `review.md` is no longer produced. The orchestrator reads R sub-agent memories to determine pipeline result.

**FR-6.4:** All references to `design_critical_review.md`, `verifier.md`, and `review.md` MUST be removed from all agent files, `dispatch-patterns.md`, and `feature-workflow.prompt.md`.

### FR-7: Update Dispatch Patterns

**FR-7.1:** Pattern A (Fully Parallel) MUST be updated:

- Remove step: "invoke aggregator"
- Remove step: "check aggregator completion contract"
- Replace with: "orchestrator reads sub-agent isolated memories and determines cluster result"
- The ≥2 outputs threshold still applies for the orchestrator's decision

**FR-7.2:** Pattern B (Sequential Gate + Parallel) MUST be updated:

- Remove: "forward to aggregator"
- Remove: "invoke aggregator with all available outputs"
- Remove: "check aggregator completion contract"
- Replace with: "orchestrator reads sub-agent isolated memories and determines cluster result"

**FR-7.3:** Pattern C (Replan Loop) MUST be updated:

- Replace "If aggregator DONE" with "If orchestrator determines DONE from V memories"
- Replace `verifier.md` references with individual V sub-agent artifacts
- Replace "Invoke planner with verifier.md" with "Invoke planner with individual V sub-agent artifacts"
- Max 3 iterations constraint remains unchanged

### FR-8: Update NEEDS_REVISION Routing

**FR-8.1:** The NEEDS_REVISION Routing Table in `orchestrator.agent.md` MUST be updated:

- Replace "CT Aggregator → Designer" with "Orchestrator (CT cluster evaluation) → Designer"
- Replace "V Aggregator → Planner" with "Orchestrator (V cluster evaluation) → Planner"
- Replace "R Aggregator → Implementers" with "Orchestrator (R cluster evaluation) → Implementers"
- Remove the row "All sub-agents route through their aggregator"

### FR-9: Update Memory Write Safety Rules

**FR-9.1:** Global Rule 12 (Memory Write Safety) in `orchestrator.agent.md` MUST be updated:

- Remove `ct-aggregator`, `v-aggregator`, `r-aggregator`, `researcher (synthesis mode)` from the read-write list
- Clarify that the orchestrator is the sole writer to shared `memory.md`
- Clarify that all sub-agents write to their own isolated memory files only

### FR-10: Update Documentation Structure

**FR-10.1:** The Documentation Structure table in `orchestrator.agent.md` MUST be updated to:

- Remove: `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`
- Add: `memory/` directory containing `<agent-name>.mem.md` files

### FR-11: Update Agent Anti-Drift Anchors

**FR-11.1:** All sub-agent Anti-Drift Anchors that say "You do NOT write to `memory.md`" MUST be updated to "You write only to your isolated memory file (`memory/<agent-name>.mem.md`), never to shared `memory.md`" or equivalent.

**FR-11.2:** All sub-agent Anti-Drift Anchors that reference aggregators (e.g., "The V-Aggregator handles memory updates") MUST be updated to remove aggregator references.

### FR-12: Update Orchestrator Expectations Per Agent Table

**FR-12.1:** Remove rows for CT Aggregator, V Aggregator, R Aggregator from the Orchestrator Expectations table.

**FR-12.2:** Update sub-agent rows to reflect that they no longer route through aggregators and now produce isolated memory files.

### FR-13: Update Parallel Execution Summary

**FR-13.1:** Remove "→ Synthesize → analysis.md" from the research step.

**FR-13.2:** Remove "→ CT-Aggregator → design_critical_review.md" from the CT step.

**FR-13.3:** Remove "→ V-Aggregator → verifier.md" from the V step.

**FR-13.4:** Remove "→ R-Aggregator → review.md" from the R step.

**FR-13.5:** Add memory merge steps at appropriate points in the parallel execution summary.

### FR-14: Update `feature-workflow.prompt.md`

**FR-14.1:** Update workflow description to reflect the removal of aggregator agents and synthesis step.

**FR-14.2:** Update Key Artifacts table to remove `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`.

**FR-14.3:** Add description of the isolated memory model.

### FR-15: Researcher Focused Mode Memory

**FR-15.1:** Each focused researcher agent (×4 parallel) MUST produce an isolated memory file using a naming convention that includes the focus area (e.g., `memory/researcher-architecture.mem.md`, `memory/researcher-impact.mem.md`).

**FR-15.2:** The researcher's focused mode description MUST be updated to remove any reference to synthesis mode.

---

## Non-Functional Requirements

### NFR-1: Orchestrator Prompt Size

The orchestrator is already 399 lines. Absorbing aggregator decision logic MUST NOT make the orchestrator unwieldy. Decision tables and completion contract logic SHOULD be concisely expressed (e.g., compact tables rather than lengthy prose).

### NFR-2: Memory File Compactness

Isolated memory files MUST be compact summaries (target: ≤30 lines). They are optimized for orchestrator consumption, not human reading. Full detail belongs in the primary artifact.

### NFR-3: Convention Consistency

All modifications MUST preserve the existing agent file conventions documented in [research/patterns.md](research/patterns.md):

- YAML frontmatter format
- Standard section ordering (Title, Role, Prohibitions, Inputs, Outputs, Operating Rules, Workflow, Completion Contract, Anti-Drift Anchor)
- Identical Operating Rules 1–4 across all agents
- Completion Contract format

### NFR-4: Memory System Non-Blocking

The memory system MUST remain non-blocking. If an agent cannot create its isolated memory file, or if the orchestrator cannot merge it, the pipeline MUST continue. Agents fall back to direct artifact reads per the existing robustness pattern.

### NFR-5: Backward-Compatible Severity Taxonomies

The existing severity scales MUST be preserved:

- CT cluster: Critical / High / Medium / Low
- R cluster: Blocker / Major / Minor
- V cluster: Binary PASS/FAIL + task-level status

### NFR-6: Concurrency Safety

The isolated memory model MUST preserve concurrency safety. Parallel agents write only to their own isolated files — no shared-state conflicts. The orchestrator merges sequentially after all parallel agents return.

---

## Constraints & Assumptions

### Constraints

**C-1:** This is a prompt-file-only change. All changes are to markdown agent definition files in `NewAgentsAndPrompts/`. No source code, runtime infrastructure, or test code exists or is modified.

**C-2:** Only files in `NewAgentsAndPrompts/` are in scope. The older `.github/agents/` directory is NOT modified.

**C-3:** The `critical-thinker.agent.md` is already deprecated and requires no changes.

**C-4:** The maximum concurrency cap of 4 parallel subagents per wave (Global Rule 8) remains unchanged.

**C-5:** The 8-step pipeline structure remains (Steps 0–7), with Step 1.2 removed and aggregator invocation sub-steps removed from Steps 3b, 6, and 7.

### Assumptions

**A-1:** The isolated memory file naming convention `memory/<agent-name>.mem.md` is centralized in a `memory/` subdirectory under the feature directory (i.e., `docs/feature/<feature-slug>/memory/`).

**A-2:** Sequential agents (spec, designer, planner) also use isolated memory files (branch-merge) rather than writing directly to `memory.md`. This maintains a uniform model where the orchestrator is the sole `memory.md` writer.

**A-3:** The orchestrator merges isolated memories at cluster boundaries (after all parallel agents return), not after each individual agent in a parallel wave.

**A-4:** Deduplication of findings across sub-agents (previously handled by aggregators) is acceptable to lose. Individual artifacts retain their full findings. The orchestrator reads only compact memories for routing decisions.

**A-5:** Unresolved Tensions detection (contradictions between sub-agent findings, previously surfaced by aggregators) is acceptable to lose for orchestrator routing. Downstream consumers reading full artifacts can identify contradictions themselves.

**A-6:** The planner in replan mode can assemble task ID failure mapping by reading `verification/v-tasks.md` directly, without needing the cross-referencing previously provided by `v-aggregator`.

---

## Acceptance Criteria

### AC-1: Aggregator Files Removed

**Pass:** `ct-aggregator.agent.md`, `v-aggregator.agent.md`, `r-aggregator.agent.md` are either deleted or contain only a deprecation notice. They contain no active workflow instructions.
**Fail:** Any aggregator file contains active workflow, inputs, outputs, or operating rules.

### AC-2: No References to Aggregator Agents

**Pass:** A text search for `ct-aggregator`, `v-aggregator`, `r-aggregator`, `CT Aggregator`, `V Aggregator`, `R Aggregator`, `CT-Aggregator`, `V-Aggregator`, `R-Aggregator` across all non-deprecated files in `NewAgentsAndPrompts/` returns zero matches (excluding deprecation notices in the aggregator files themselves, if retained).
**Fail:** Any active agent file references an aggregator agent by name.

### AC-3: No References to Removed Artifacts

**Pass:** A text search for `analysis.md`, `design_critical_review.md`, `verifier.md` (as an artifact path, not the word "verifier"), and `review.md` (as an artifact path produced by r-aggregator) across all active files in `NewAgentsAndPrompts/` returns zero matches.
**Fail:** Any active agent file references a removed combined artifact.

### AC-4: No Researcher Synthesis Mode

**Pass:** `researcher.agent.md` contains no "Mode 2", "Synthesis", "synthesis mode", or `analysis.md` references. It defines only focused mode.
**Fail:** `researcher.agent.md` retains synthesis mode content.

### AC-5: All Sub-Agents Produce Isolated Memory

**Pass:** Every non-deprecated agent file in `NewAgentsAndPrompts/` (excluding `critical-thinker.agent.md` and removed aggregator files) lists an isolated memory file (`memory/<agent-name>.mem.md`) in its Outputs section AND has a workflow step to write to it.
**Fail:** Any active agent file lacks isolated memory in Outputs or lacks a memory write workflow step.

### AC-6: Orchestrator Memory Merge

**Pass:** `orchestrator.agent.md` includes:

1. A "Merge" action in the Memory Lifecycle section
2. Steps after each cluster completion to read isolated memories and merge into `memory.md`
3. Explicit statement that the orchestrator is the sole writer to `memory.md`
   **Fail:** Any of the three items is missing.

### AC-7: Orchestrator Absorbs CT Decision Logic

**Pass:** `orchestrator.agent.md` contains the CT cluster completion logic: reads CT memories, checks for Critical/High findings, returns DONE/NEEDS_REVISION/ERROR accordingly. The logic matches FR-4.1.
**Fail:** CT completion logic is missing, incomplete, or still references ct-aggregator.

### AC-8: Orchestrator Absorbs V Decision Logic

**Pass:** `orchestrator.agent.md` contains the V cluster completion decision table (matching FR-4.2) and handles Pattern C replan loop using individual V memories/artifacts. References `verification/v-tasks.md` (not `verifier.md`) for replan input to planner.
**Fail:** V completion logic is missing, incomplete, references v-aggregator, or references `verifier.md`.

### AC-9: Orchestrator Absorbs R Decision Logic with Security Override

**Pass:** `orchestrator.agent.md` contains the R cluster completion logic (matching FR-4.3) including the R-Security pipeline override (Critical/Blocker → ERROR). R-Knowledge is explicitly non-blocking.
**Fail:** R completion logic is missing, R-Security override is absent, or R-Knowledge ERROR blocks the pipeline.

### AC-10: Downstream Agents Read Research Files Directly

**Pass:** `spec.agent.md`, `designer.agent.md`, and `planner.agent.md` each list `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md` (or `research/*.md` glob) in their Inputs section. None references `analysis.md`.
**Fail:** Any of the three agents references `analysis.md` or does not list research files.

### AC-11: Designer Reads CT Files Directly in Revision Mode

**Pass:** `designer.agent.md` references individual `ct-review/ct-*.md` files as input for the revision cycle. Does not reference `design_critical_review.md`.
**Fail:** `designer.agent.md` references `design_critical_review.md`.

### AC-12: Planner Reads V Files Directly in Replan Mode

**Pass:** `planner.agent.md` references individual verification artifacts (specifically `verification/v-tasks.md` for task failures) as input in replan mode. Does not reference `verifier.md`.
**Fail:** `planner.agent.md` references `verifier.md`.

### AC-13: Dispatch Patterns Updated

**Pass:** `dispatch-patterns.md` defines Pattern A and Pattern B without any aggregator invocation steps. No references to aggregator agents, `design_critical_review.md`, `verifier.md`, or `review.md`.
**Fail:** `dispatch-patterns.md` still references aggregators or combined artifacts.

### AC-14: Memory Write Safety Updated

**Pass:** Global Rule 12 in `orchestrator.agent.md` does not list any aggregator as a read-write agent. The rule explicitly states that the orchestrator is the sole writer to `memory.md` and all sub-agents write only to their isolated memory files.
**Fail:** Aggregators are still listed, or the rule does not clarify the new write model.

### AC-15: Documentation Structure Updated

**Pass:** The Documentation Structure table in `orchestrator.agent.md` includes `memory/<agent-name>.mem.md` entries and does not include `analysis.md`, `design_critical_review.md`, `verifier.md`, or `review.md`.
**Fail:** Removed artifacts still appear, or memory directory is missing.

### AC-16: Anti-Drift Anchors Updated

**Pass:** No active agent file's Anti-Drift Anchor contains "You do NOT write to `memory.md`" without qualification. Instead, they reference writing to the agent's own isolated memory file. No Anti-Drift Anchor references an aggregator agent.
**Fail:** Any active agent retains the old memory-write prohibition or aggregator reference in its Anti-Drift Anchor.

### AC-17: Feature Workflow Updated

**Pass:** `feature-workflow.prompt.md` does not reference aggregator agents, `analysis.md`, `design_critical_review.md`, `verifier.md`, or `review.md`. It describes the isolated memory model.
**Fail:** `feature-workflow.prompt.md` retains old references.

### AC-18: Step 1.2 Removed

**Pass:** `orchestrator.agent.md` does not contain a Step 1.2 (Synthesize Research) or any instruction to invoke a researcher in synthesis mode.
**Fail:** Step 1.2 or synthesis researcher invocation remains.

### AC-19: Convention Consistency Preserved

**Pass:** All modified agent files retain the standard section ordering (YAML frontmatter, Title, Role, Prohibitions, Inputs, Outputs, Operating Rules, Workflow, Completion Contract, Anti-Drift Anchor) and identical Operating Rules 1–4.
**Fail:** Any agent file deviates from the standard structure or has inconsistent Operating Rules.

### AC-20: Completion Contracts Unchanged for Sub-Agents

**Pass:** Sub-agent completion contracts remain in their existing format (DONE/ERROR or DONE/NEEDS_REVISION/ERROR depending on agent). No new fields are required in the completion contract line itself.
**Fail:** Sub-agent completion contract formats are altered.

---

## Edge Cases & Error Handling

### EC-1: Agent Fails to Write Isolated Memory File

- **Input/Condition:** A sub-agent encounters an error and cannot create its `memory/<agent-name>.mem.md` file.
- **Expected Behavior:** The orchestrator logs a warning and proceeds. The agent's findings are still available in its primary artifact. Memory merge for that agent is skipped. Pipeline continues (non-blocking, per NFR-4).
- **Severity if Missed:** Medium — orchestrator loses compact routing data for one agent but can still read the primary artifact.

### EC-2: Fewer Than 2 Sub-Agent Outputs in Parallel Cluster

- **Input/Condition:** In a CT, R, or Research cluster, fewer than 2 sub-agents return successfully (after retries).
- **Expected Behavior:** Orchestrator declares cluster `ERROR`. This is the same threshold as the current aggregator-based system.
- **Severity if Missed:** High — proceeding with too few inputs could produce unreliable routing decisions.

### EC-3: R-Security Agent Fails or Missing

- **Input/Condition:** R-Security does not return (ERROR after retry) or its isolated memory file indicates Critical/Blocker findings.
- **Expected Behavior:** Orchestrator declares pipeline `ERROR`. R-Security is mandatory and its failures block the pipeline.
- **Severity if Missed:** Critical — security-critical findings could be silently bypassed.

### EC-4: R-Knowledge Agent Fails

- **Input/Condition:** R-Knowledge returns ERROR or its isolated memory file is missing.
- **Expected Behavior:** Orchestrator ignores the failure and proceeds. R-Knowledge is explicitly non-blocking. The overall R cluster result is determined by the remaining sub-agents.
- **Severity if Missed:** Low — R-Knowledge provides enhancement suggestions, not safety-critical findings.

### EC-5: V-Build Gate Fails

- **Input/Condition:** V-Build returns ERROR (after retry) in Pattern B.
- **Expected Behavior:** Orchestrator skips remaining V sub-agents (v-tests, v-tasks, v-feature), declares V cluster `ERROR`, and proceeds to Pattern C replan loop if iterations remain.
- **Severity if Missed:** High — running verification sub-agents without a passing build produces unreliable results.

### EC-6: Pattern C Exhausts 3 Iterations

- **Input/Condition:** V cluster returns NEEDS_REVISION or ERROR for 3 consecutive iterations.
- **Expected Behavior:** Orchestrator proceeds with findings documented in individual V sub-agent artifacts. Pipeline does NOT halt — it moves to Step 7 (Review) with known verification issues.
- **Severity if Missed:** Medium — pipeline would stall indefinitely or mask verification failures.

### EC-7: Missing Research File for Downstream Agent

- **Input/Condition:** One of the 4 research files (e.g., `research/patterns.md`) does not exist because the researcher for that focus area ERROR'd.
- **Expected Behavior:** The downstream agent (spec, designer, planner) notes the gap (per Operating Rule 2: "Missing context: Note the gap and proceed with available information") and proceeds with the available research files.
- **Severity if Missed:** Medium — the downstream agent would halt or produce incomplete output without graceful degradation.

### EC-8: Orchestrator Memory Merge Conflict

- **Input/Condition:** Two agents produce isolated memories that contain contradictory information (e.g., different severity assessments of the same finding).
- **Expected Behavior:** Orchestrator performs a sequential merge (no conflict resolution needed at merge — both entries are appended). The orchestrator's routing decision is based on the worst-case severity across all memories.
- **Severity if Missed:** Medium — contradictory findings could lead to inconsistent routing decisions.

### EC-9: Sequential Agent (Spec/Designer/Planner) Isolated Memory

- **Input/Condition:** Sequential agents that previously wrote directly to `memory.md` now write to isolated memory files instead.
- **Expected Behavior:** The orchestrator merges their isolated memory into `memory.md` after they return, the same as for parallel agents. This is consistent with the universal branch-merge model.
- **Severity if Missed:** Low — sequential writes to shared memory are safe, but inconsistency in the model adds complexity.

### EC-10: Memory Directory Does Not Exist

- **Input/Condition:** The `memory/` directory under the feature directory has not been created.
- **Expected Behavior:** The first agent to write an isolated memory file creates the `memory/` directory (or the orchestrator creates it during Step 0 initialization). Operating Rule 2 (persistent errors) applies if directory creation fails.
- **Severity if Missed:** Medium — agents cannot write isolated memory files, falling back to non-blocking degradation.

### EC-11: Planner Replan Without Aggregated Task-Failure Mapping

- **Input/Condition:** Planner enters replan mode, but there is no `verifier.md` with a consolidated Actionable Items section. The planner must read individual V artifacts.
- **Expected Behavior:** Planner reads `verification/v-tasks.md` directly for failing task IDs and `verification/v-tests.md` for test failures. The planner assembles its own understanding of what needs replanning from these individual files.
- **Severity if Missed:** High — without clear failure-to-task mapping, replanning becomes untargeted and may not fix the right tasks.

---

## User Stories / Flows

### Flow 1: Happy Path (Full Pipeline, No Revisions)

1. Orchestrator initializes `initial-request.md`, `memory.md`, and `memory/` directory (Step 0).
2. Orchestrator dispatches 4 researchers in parallel (Pattern A, Step 1.1). Each produces `research/<focus>.md` and `memory/researcher-<focus>.mem.md`.
3. Orchestrator reads 4 researcher memories, merges into `memory.md`. No synthesis step.
4. Orchestrator dispatches spec agent. Spec reads 4 research files directly, produces `feature.md` + `memory/spec.mem.md`. Orchestrator merges memory.
5. Orchestrator dispatches designer. Designer reads 4 research files + `feature.md`, produces `design.md` + `memory/designer.mem.md`. Orchestrator merges memory.
6. Orchestrator dispatches 4 CT agents in parallel (Pattern A, Step 3b). Each produces `ct-review/ct-<domain>.md` + `memory/ct-<domain>.mem.md`.
7. Orchestrator reads 4 CT memories. All findings ≤ Medium severity → DONE. Orchestrator merges CT memories into `memory.md`.
8. Orchestrator dispatches planner. Planner reads 4 research files + `feature.md` + `design.md`, produces `plan.md` + `tasks/*.md` + `memory/planner.mem.md`. Orchestrator merges memory.
9. Implementation waves (Step 5). Each implementer/doc-writer produces code/tests/docs + `memory/implementer-<task-id>.mem.md`. Orchestrator merges between waves.
10. V cluster (Pattern B+C, Step 6). V-Build gate → V-Tests/Tasks/Feature in parallel. Each produces `verification/v-<domain>.md` + `memory/v-<domain>.mem.md`. Orchestrator reads V memories → all DONE → pipeline verification passes.
11. R cluster (Pattern A, Step 7). 4 R agents in parallel. Each produces `review/r-<domain>.md` + `memory/r-<domain>.mem.md`. Orchestrator reads R memories → R-Security no blockers, all findings minor → DONE. Pipeline completes.

### Flow 2: CT Cluster Triggers Design Revision

1. Steps 1–6 same as Flow 1.
2. Orchestrator reads CT memories. `memory/ct-security.mem.md` reports highest severity: Critical.
3. Orchestrator determines: NEEDS_REVISION → route to designer.
4. Designer re-reads individual `ct-review/ct-*.md` files for full findings, revises `design.md`.
5. Orchestrator re-runs CT cluster (max 1 revision loop).

### Flow 3: V Cluster Triggers Replan (Pattern C)

1. Steps 1–9 same as Flow 1.
2. V cluster completes. Orchestrator reads V memories. `memory/v-tasks.mem.md` reports NEEDS_REVISION with failing task IDs.
3. Orchestrator determines: NEEDS_REVISION → invoke planner in replan mode.
4. Planner reads `verification/v-tasks.md` directly for failing task IDs, produces updated tasks.
5. Implementers fix tasks.
6. V cluster re-runs (iteration 2 of max 3).

---

## Test Scenarios

Since all files are markdown agent definitions, "testing" means structural verification of the modified files. The following scenarios define verification checks.

### TS-1: Aggregator Removal Verification

**Verifies:** AC-1, AC-2
**Method:** Check that `ct-aggregator.agent.md`, `v-aggregator.agent.md`, `r-aggregator.agent.md` are deleted or contain only a deprecation notice. Grep all remaining `.agent.md` and `.md` files in `NewAgentsAndPrompts/` for aggregator name strings. Zero active matches expected.

### TS-2: Combined Artifact Reference Removal

**Verifies:** AC-3
**Method:** Grep all files in `NewAgentsAndPrompts/` for the strings `analysis.md`, `design_critical_review.md`, `verifier.md` (as artifact references). Zero matches expected in active agent files. Note: `verifier.md` grep must distinguish from partial matches like "verifier" in generic text.

### TS-3: Synthesis Mode Removal

**Verifies:** AC-4, AC-18
**Method:** Read `researcher.agent.md` and verify no "Mode 2", "Synthesis", "synthesis mode", or `analysis.md` strings exist. Read `orchestrator.agent.md` and verify no Step 1.2 or synthesis invocation exists.

### TS-4: Isolated Memory Output in All Agents

**Verifies:** AC-5
**Method:** For each active agent file, verify:

1. The Outputs section contains a `memory/<agent-name>.mem.md` entry
2. The Workflow section contains a step to write findings to the isolated memory file
   **Files to check:** All 21 active agent files (24 minus 3 removed aggregators).

### TS-5: Orchestrator Memory Merge

**Verifies:** AC-6
**Method:** Read `orchestrator.agent.md` and verify:

1. Memory Lifecycle section includes a "Merge" action
2. After each cluster dispatch (Steps 1.1, 3b, 5, 6, 7), there is a step to read isolated memories and merge
3. The text explicitly states the orchestrator is the sole writer to `memory.md`

### TS-6: CT Decision Logic in Orchestrator

**Verifies:** AC-7
**Method:** Read `orchestrator.agent.md` CT cluster handling (Step 3b) and verify it contains:

1. Instructions to read CT sub-agent memories
2. Severity-based decision (Critical/High → NEEDS_REVISION)
3. ≥2 outputs threshold
4. No aggregator reference

### TS-7: V Decision Logic in Orchestrator

**Verifies:** AC-8
**Method:** Read `orchestrator.agent.md` V cluster handling (Step 6) and verify it contains:

1. The V completion decision table
2. Pattern C replan loop using individual V artifacts
3. Planner replan input is `verification/v-tasks.md` (not `verifier.md`)
4. No aggregator reference

### TS-8: R Decision Logic with Security Override

**Verifies:** AC-9
**Method:** Read `orchestrator.agent.md` R cluster handling (Step 7) and verify it contains:

1. R-Security pipeline override (Critical/Blocker → ERROR)
2. R-Security mandatory check
3. R-Knowledge non-blocking
4. ≥2 non-knowledge outputs threshold
5. No aggregator reference

### TS-9: Downstream Agent Input Updates

**Verifies:** AC-10, AC-11, AC-12
**Method:** Read `spec.agent.md`, `designer.agent.md`, `planner.agent.md` Inputs sections and verify:

1. Each lists individual research files (not `analysis.md`)
2. Designer lists `ct-review/ct-*.md` for revision (not `design_critical_review.md`)
3. Planner lists `verification/v-tasks.md` for replan (not `verifier.md`)

### TS-10: Dispatch Patterns Updated

**Verifies:** AC-13
**Method:** Read `dispatch-patterns.md` and verify Pattern A and Pattern B contain no aggregator steps and no combined artifact references.

### TS-11: Memory Write Safety Rule

**Verifies:** AC-14
**Method:** Read Global Rule 12 in `orchestrator.agent.md` and verify:

1. No aggregator listed as read-write
2. Orchestrator stated as sole `memory.md` writer
3. All sub-agents write only to isolated memory

### TS-12: Documentation Structure Table

**Verifies:** AC-15
**Method:** Read Documentation Structure in `orchestrator.agent.md` and verify:

1. `memory/` directory with `*.mem.md` entries present
2. No `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md` entries

### TS-13: Anti-Drift Anchor Consistency

**Verifies:** AC-16
**Method:** For each active agent file, read the Anti-Drift Anchor and verify:

1. No unqualified "do NOT write to `memory.md`" statement
2. No aggregator agent references
3. If memory write restriction exists, it references isolated memory

### TS-14: Feature Workflow Prompt

**Verifies:** AC-17
**Method:** Read `feature-workflow.prompt.md` and verify:

1. No aggregator references
2. No `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md` references
3. Isolated memory model mentioned

### TS-15: Convention Adherence

**Verifies:** AC-19
**Method:** For each modified agent file, verify section ordering matches the standard convention: YAML frontmatter → Title → Role → Prohibitions → Inputs → Outputs → Operating Rules → Workflow → Completion Contract → Anti-Drift Anchor. Verify Operating Rules 1–4 text is identical across agents.

### TS-16: Completion Contract Preservation

**Verifies:** AC-20
**Method:** For each sub-agent file, read the Completion Contract section and verify the format matches the existing pattern (DONE/ERROR or DONE/NEEDS_REVISION/ERROR). No new required fields in the completion line.

---

## Dependencies & Risks

### Dependencies

| Component                                                | Dependency Type      | Description                                                                             |
| -------------------------------------------------------- | -------------------- | --------------------------------------------------------------------------------------- |
| `orchestrator.agent.md`                                  | Primary target       | Most heavily modified file; absorbs aggregator logic, gains memory merge responsibility |
| `researcher.agent.md`                                    | Major change         | Synthesis mode removal; affects Step 1.2 elimination                                    |
| `dispatch-patterns.md`                                   | Reference doc        | Patterns A/B/C updated; orchestrator references must align                              |
| `spec.agent.md`, `designer.agent.md`, `planner.agent.md` | Input changes        | Switch from combined files to individual research/artifact files                        |
| 14 sub-agent files                                       | Minor changes        | All need isolated memory output added                                                   |
| `feature-workflow.prompt.md`                             | Workflow entry point | Must reflect new architecture                                                           |

### Risks

| Risk                                                                                                                                                                                                                                       | Severity | Likelihood             | Mitigation                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **R-Security pipeline override lost** — If the orchestrator does not correctly implement the R-Security override, security-critical findings could be silently passed.                                                                     | Critical | Low (if spec followed) | AC-9 explicitly tests for this. Implementation must include a mandatory R-Security check before any R cluster DONE determination.                                     |
| **Task ID failure mapping degraded** — The planner replan mode loses the cross-referenced task-failure mapping previously produced by v-aggregator. Reading `v-tasks.md` directly may not provide the same quality of targeted replanning. | High     | Medium                 | AC-12 verifies planner reads V artifacts directly. Planner may need enhanced instructions for self-assembling failure mapping. Monitor replan quality in early usage. |
| **Orchestrator complexity explosion** — Absorbing 3 aggregators' decision logic + memory merge into the already-large orchestrator (399 lines) could exceed prompt context limits or reduce LLM reliability.                               | High     | Medium                 | NFR-1 requires concise expression. Decision tables should be compact. Consider extracting cluster decision tables into `dispatch-patterns.md` if needed.              |
| **Downstream agents overwhelmed by input expansion** — Spec/designer/planner switch from reading 1 combined file to 4 individual research files, increasing input burden.                                                                  | Medium   | Low                    | Memory-first protocol still applies — agents use `memory.md` Artifact Index to navigate. Individual files are the same total content, just not merged.                |
| **Memory file naming collisions** — Multiple implementers could produce memory files with conflicting names if task IDs are reused across waves.                                                                                           | Medium   | Low                    | Use task ID in name: `memory/implementer-<task-id>.mem.md`. Task IDs are unique within a pipeline run.                                                                |
| **Unresolved Tensions detection lost** — Aggregators currently cross-check sub-agents for contradictions. Without aggregators, contradictions may go undetected.                                                                           | Medium   | Medium                 | Acceptable per A-5. Downstream consumers reading full artifacts can identify contradictions. Document this as a known limitation.                                     |
| **Convention drift during modification** — With 21+ files being modified, some may drift from the standard section ordering or Operating Rules text.                                                                                       | Low      | Medium                 | TS-15 verifies convention adherence. Use search/replace patterns to make consistent changes across all sub-agent files.                                               |
