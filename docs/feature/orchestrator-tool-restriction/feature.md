# Feature Specification: Orchestrator Tool Restriction & Memory Redesign

**Summary:** Restrict the orchestrator agent's tool access to `[agent, agent/runSubagent, memory, read_file, list_dir]` and eliminate the shared `memory.md` file entirely (Pattern B), relying on agent-isolated `memory/<agent>.mem.md` files for all cross-agent context propagation.

---

## Background & Context

The orchestrator currently has broad tool access including 4 discovery/analysis tools (`grep_search`, `semantic_search`, `file_search`, `get_errors`) that are never used in pipeline logic — all orchestrator reads target known deterministic paths. Additionally, the orchestrator is the sole writer to a shared `memory.md` file at ~18 distinct write points per pipeline run, but the VS Code `memory` tool (retained in the tool set) is for cross-session codebase knowledge, not arbitrary file writing. Since the orchestrator has no file-write tools, `memory.md` cannot be maintained without either delegating all writes (10–20+ subagent dispatches) or eliminating the file entirely.

**Research artifacts:**
- [research/architecture.md](research/architecture.md) — orchestrator structure, tool references, memory write points
- [research/impact.md](research/impact.md) — tool removal impact, memory write removal impact, downstream effects
- [research/dependencies.md](research/dependencies.md) — cross-agent memory dependencies, memory merge chains
- [research/patterns.md](research/patterns.md) — memory architecture patterns (A/B/C), orchestrator purity analysis

**Key prior decision:** [decisions.md (self-improvement-system)](../self-improvement-system/decisions.md) Decision #4 — "Tool Restriction: Keep Read Tools for Orchestrator" — established that `read_file` and `list_dir` are acceptable for coordinators (read = observation).

---

## Scope

### In Scope

1. Remove `grep_search`, `semantic_search`, `file_search`, `get_errors` from orchestrator tool access
2. Add a `tools:` field to orchestrator YAML frontmatter as formal enforcement
3. Eliminate shared `memory.md` — remove initialization, merge, prune, invalidation, and all write operations
4. Update all orchestrator prompt sections that reference removed tools or `memory.md` writing
5. Update `feature-workflow.prompt.md` to reflect the new memory architecture
6. Update `dispatch-patterns.md` to remove memory merge references
7. Update all 20 subagent `.agent.md` files to remove `memory.md` from Inputs and Operating Rules
8. Design Lessons Learned propagation mechanism without `memory.md`

### Out of Scope

1. Changing any subagent's tool access (only the orchestrator is affected)
2. Modifying VS Code `memory` tool behavior (it remains as-is for cross-session knowledge)
3. Creating a new memory-merge subagent (Pattern A/C rejected)
4. Changing the isolated `memory/<agent>.mem.md` file format or content
5. Modifying cluster decision logic (already uses `read_file` on known paths)
6. Runtime enforcement testing of VS Code `tools:` YAML field (enforcement is advisory + prose)

---

## Functional Requirements

### FR-1: Orchestrator Tool Restriction

**FR-1.1:** The orchestrator's YAML frontmatter MUST include a `tools:` field listing exactly: `[agent, agent/runSubagent, memory, read_file, list_dir]`.

**FR-1.2:** The orchestrator's prose tool restriction (currently Operating Rule 5, line ~L92) MUST list the same 5 tools as the YAML `tools:` field. The prose and YAML MUST be consistent.

**FR-1.3:** Global Rule 1 (line ~L33) MUST be updated to reference only `read_file` and `list_dir` as read tools. Remove all mentions of `grep_search`, `semantic_search`, `file_search`.

**FR-1.4:** Operating Rule 1 (line ~L83) MUST be updated to remove the recommendation to "prefer `semantic_search` and `grep_search` for discovery." Replace with guidance to use `read_file` with known paths and `list_dir` for directory listing.

**FR-1.5:** The Anti-Drift Anchor MUST be updated to reflect the restricted tool set. Replace "uses read tools freely" with "uses `read_file` and `list_dir` for cluster decisions and routing."

**FR-1.6:** The Anti-Drift Anchor MUST remove any reference to the orchestrator being "sole writer to shared `memory.md`."

### FR-2: Eliminate Shared memory.md

**FR-2.1:** Remove the `memory.md` initialization from Step 0. The orchestrator MUST NOT create or reference `memory.md` at any pipeline step.

**FR-2.2:** Remove all 10 memory merge operations (Steps 1.1m, 2m, 3m, 3b.2, 4m, between-waves, 6.3, 7.3, 8.2) from the orchestrator prompt. These steps currently read `*.mem.md` files and write merged content into `memory.md`.

**FR-2.3:** Remove all 3 prune operations (Steps 1.1m, 2m, 4m) that remove stale entries from `memory.md`.

**FR-2.4:** Remove all self-verification log writes to `memory.md` (CT, V, R cluster decision logging). Self-verification logic (reading memories, applying decision tables) MUST be preserved — only the persistent write to `memory.md` is removed. Self-verification results remain in the orchestrator's context window and are captured by telemetry context tracking (Global Rule 13).

**FR-2.5:** Remove the invalidation mechanism (`[INVALIDATED — <reason>]` markers in `memory.md`). For revision loops, the orchestrator MUST communicate invalidation context via dispatch prompt instructions (e.g., "ignore prior V cluster results; revision in progress").

**FR-2.6:** Remove the Memory Lifecycle Actions table from the orchestrator prompt entirely. The 7-phase lifecycle (Initialize, Merge, Prune, Extract Lessons, Invalidate, Clean, Validate) is no longer applicable. Retain only the Validate phase as inline guidance: "Check `memory/<agent>.mem.md` existence after each agent dispatch; treat missing memory as worst-case for cluster routing."

**FR-2.7:** Remove all references to `memory.md` from every step's merge sub-step (the "m" suffix steps: 1.1m, 2m, 3m, etc.). Replace with a brief read-and-proceed instruction: "Read `memory/<agent>.mem.md` for status confirmation. Proceed to next step."

### FR-3: Update Subagent Dispatch Prompts

**FR-3.1:** All subagent dispatch instructions in the orchestrator MUST remove `memory.md` from the input file list. Each dispatch MUST list only the specific upstream `memory/<agent>.mem.md` files the subagent needs.

**FR-3.2:** The orchestrator's dispatch instructions for each pipeline step MUST specify the exact upstream memory files. The mapping is:

| Pipeline Step | Agent(s) | Upstream Memory File Paths to Include |
|---|---|---|
| Step 1 (Research) | researcher ×4 | `initial-request.md` only (no upstream memories exist yet) |
| Step 2 (Spec) | spec | `memory/researcher-*.mem.md` (4 files) |
| Step 3 (Design) | designer | `memory/spec.mem.md`, `memory/researcher-*.mem.md` |
| Step 3b (CT) | ct-* ×4 | `memory/designer.mem.md`, `memory/spec.mem.md` |
| Step 4 (Plan) | planner | `memory/designer.mem.md`, `memory/spec.mem.md` |
| Step 5 (Implement) | implementer, documentation-writer | `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md` |
| Step 6 (Verify) | v-build, v-tests, v-tasks, v-feature | `memory/planner.mem.md` (v-build); `memory/v-build.mem.md` + `memory/planner.mem.md` (others) |
| Step 7 (Review) | r-quality, r-security, r-testing, r-knowledge | Relevant upstream `memory/*.mem.md` files per agent definition |
| Step 8 (PostMortem) | post-mortem | All `memory/*.mem.md` files produced during the run |

**FR-3.3:** For implementation waves beyond wave 1, the orchestrator MUST include Lessons Learned context in the dispatch prompt for wave N+1 agents. The orchestrator reads Lessons Learned entries from wave N implementer `*.mem.md` files (a read-only operation using `read_file`) and includes them as plain text in the dispatch prompt.

**FR-3.4:** For V cluster and R cluster dispatches, the orchestrator MUST include any accumulated Lessons Learned from implementation waves as additional dispatch context (read from implementer `*.mem.md` files).

### FR-4: Update Subagent Definitions

**FR-4.1:** All 20 subagent `.agent.md` files MUST have `memory.md` removed from their Inputs section. Each agent's inputs MUST list only the specific upstream `memory/<agent>.mem.md` files it reads.

**FR-4.2:** All 20 subagent Operating Rule 6 ("Memory-first reading") MUST be updated to remove the instruction to "Read `memory.md` FIRST." Replace with: "Read upstream memory files FIRST (as listed in Inputs) before accessing any artifact. Use their Artifact Indexes to navigate directly to relevant sections."

**FR-4.3:** All 20 subagent Operating Rule 6 fallback text MUST be updated from "If `memory.md` is missing, log a warning and proceed with direct artifact reads" to "If upstream memory files are missing, log a warning and proceed with direct artifact reads."

**FR-4.4:** All 20 subagent Anti-Drift Anchors that reference "never to shared `memory.md`" MUST remove that clause (there is no shared `memory.md` to reference).

### FR-5: Update Supporting Documents

**FR-5.1:** `feature-workflow.prompt.md` line ~L24 (memory system rule) MUST be updated to describe the new architecture: "Each subagent writes an isolated `memory/<agent>.mem.md` file. The orchestrator reads these isolated memories for routing decisions. There is no shared `memory.md` — agents read upstream memories directly."

**FR-5.2:** `feature-workflow.prompt.md` Key Artifacts table (line ~L39) MUST remove the `memory.md` entry. The `memory/*.mem.md` entry remains.

**FR-5.3:** `dispatch-patterns.md` MUST be updated to remove all references to "orchestrator merges into `memory.md`" and "memory-first pattern reading `memory.md`." Replace merge references with "orchestrator reads `memory/<agent>.mem.md` for status confirmation."

### FR-6: Disambiguate `memory` Tool

**FR-6.1:** All orchestrator prompt references to the `memory` tool MUST clarify it is the VS Code cross-session codebase knowledge tool, not a pipeline file-writing mechanism. Add a one-line clarification near the tool list: "`memory` is VS Code's cross-session knowledge store for codebase facts — it does NOT write to pipeline files."

**FR-6.2:** Remove any prompt language that implies the `memory` tool can create or modify `memory.md` or other pipeline files.

---

## Non-Functional Requirements

### NFR-1: Backward Compatibility

**NFR-1.1:** The pipeline MUST produce the same functional output (feature.md, design.md, plan.md, tasks, implementations, verifications, reviews, post-mortem) with and without these changes. The changes affect orchestration mechanics, not pipeline outcomes.

**NFR-1.2:** Subagent fallback behavior ("if upstream memory files are missing, proceed with direct artifact reads") MUST remain intact as a safety net.

### NFR-2: No New Subagents

**NFR-2.1:** This change MUST NOT introduce any new subagent definitions (e.g., no memory-merge agent). The goal is simplification, not indirection.

### NFR-3: Token Efficiency

**NFR-3.1:** Removing `memory.md` SHOULD reduce total pipeline token usage by eliminating 10–20+ merge operations. The slight increase from agents reading 2–5 upstream `*.mem.md` files (100–250 tokens each) instead of 1 merged `memory.md` MUST be offset by the elimination of all merge subagent overhead.

### NFR-4: Consistency

**NFR-4.1:** The YAML `tools:` field and prose tool restriction MUST be consistent — they MUST list identical tools.

**NFR-4.2:** All 20 subagent files MUST be updated atomically — no agent file should reference `memory.md` in Inputs while others do not.

### NFR-5: No Functional Regression in Cluster Decisions

**NFR-5.1:** CT, V, and R cluster decision flows MUST continue to function correctly. They already use `read_file` on known `memory/<agent>.mem.md` paths and do not depend on `memory.md`.

---

## Constraints & Assumptions

### Constraints

1. **VS Code YAML `tools:` enforcement is advisory.** The `tools:` field in YAML frontmatter may not be enforced by the VS Code runtime. Prose-level restrictions in Operating Rules serve as the primary enforcement mechanism. The YAML field serves as declaration and documentation.
2. **No file-write tools for orchestrator.** The orchestrator cannot use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, or `run_in_terminal`. All file mutations are subagent-delegated.
3. **Files to modify span the entire agent suite.** Changes touch 21 agent files + 2 supporting documents (23 files total).

### Assumptions

1. The VS Code `memory` tool is strictly for cross-session codebase knowledge and CANNOT write to arbitrary file paths like `docs/feature/<slug>/memory.md`.
2. All subagent `*.mem.md` files use a consistent format with `## Status`, `## Key Findings`, `## Highest Severity`, `## Decisions Made`, `## Artifact Index` sections.
3. Subagent token budgets can accommodate reading 2–5 isolated `.mem.md` files (~100–250 tokens each) without significant impact.
4. Telemetry Context Tracking (Global Rule 13) captures all dispatch metadata and self-verification results in the orchestrator's context window, making persistent `memory.md` logging redundant.
5. The documentation-writer agent already operates in a Pattern B-like mode (its memory-first rule reads upstream memories directly, not `memory.md`).

---

## Acceptance Criteria

### AC-1: Tool Restriction Applied

- **Pass:** The orchestrator's YAML frontmatter contains `tools: [agent, agent/runSubagent, memory, read_file, list_dir]`.
- **Fail:** The `tools:` field is missing, or contains any of `grep_search`, `semantic_search`, `file_search`, `get_errors`.

### AC-2: Prose Tool List Matches YAML

- **Pass:** Operating Rule 5 lists exactly `[agent, agent/runSubagent, memory, read_file, list_dir]` and no other tools.
- **Fail:** The prose tool list differs from the YAML `tools:` field.

### AC-3: Removed Tool References Eliminated

- **Pass:** A text search for `grep_search`, `semantic_search`, `file_search`, and `get_errors` in `orchestrator.agent.md` returns zero matches.
- **Fail:** Any of these strings appear anywhere in the orchestrator file.

### AC-4: No memory.md References in Orchestrator

- **Pass:** A text search for `memory.md` in `orchestrator.agent.md` returns zero matches (excluding any historical comment explaining the removal, if present).
- **Fail:** `memory.md` appears as an input, output, or operational reference in any orchestrator step.

### AC-5: No memory.md References in Subagents

- **Pass:** A text search for `memory.md` in all 20 subagent `.agent.md` files returns zero matches in Inputs or Operating Rules sections.
- **Fail:** Any subagent file still lists `memory.md` as an input or references it in Operating Rules.

### AC-6: No memory.md References in Supporting Documents

- **Pass:** `feature-workflow.prompt.md` and `dispatch-patterns.md` contain zero references to `memory.md` as a pipeline artifact maintained by the orchestrator.
- **Fail:** Either document still describes `memory.md` as part of the pipeline.

### AC-7: Memory Lifecycle Table Removed

- **Pass:** The orchestrator prompt contains no Memory Lifecycle Actions table.
- **Fail:** The table or any lifecycle phase instructions (Initialize, Merge, Prune, Extract, Invalidate, Clean) remain in the orchestrator prompt as operational instructions.

### AC-8: Merge Steps Removed

- **Pass:** Steps 1.1m, 2m, 3m, 3b.2§merge, 4m, between-waves§merge, 6.3§merge, 7.3§merge, 8.2§merge contain no instructions to write to any shared file. They contain at most a read-and-confirm instruction.
- **Fail:** Any merge step still instructs the orchestrator to write, update, or modify `memory.md` or any shared state file.

### AC-9: Subagent Dispatch Includes Upstream Memory Paths

- **Pass:** Every subagent dispatch instruction in the orchestrator lists specific upstream `memory/<agent>.mem.md` file paths (not `memory.md`) as inputs.
- **Fail:** Any dispatch instruction includes `memory.md` as an input, or fails to list the required upstream memory files.

### AC-10: Lessons Learned Propagation Documented

- **Pass:** The orchestrator prompt includes explicit instructions for propagating Lessons Learned from implementer `*.mem.md` files to wave N+1 dispatches and to V/R cluster dispatches.
- **Fail:** No mechanism exists for Lessons Learned to reach later-pipeline agents.

### AC-11: `memory` Tool Disambiguated

- **Pass:** The orchestrator prompt contains a clear statement that the `memory` tool is VS Code's cross-session knowledge store, separate from pipeline memory files.
- **Fail:** The prompt conflates the `memory` tool with pipeline `memory.md` file operations.

### AC-12: Anti-Drift Anchor Updated

- **Pass:** The Anti-Drift Anchor reflects the restricted tool set (`read_file`, `list_dir` only for reading), does not mention `memory.md`, and does not claim the orchestrator is the "sole writer" to any shared file.
- **Fail:** The anchor still references removed tools or `memory.md` writing.

### AC-13: Cluster Decisions Unaffected

- **Pass:** CT, V, and R cluster decision flow sections still read `memory/<agent>.mem.md` files via `read_file`, apply decision tables, and dispatch based on results. Only the self-verification logging to `memory.md` is removed.
- **Fail:** Cluster decision logic is altered, or the read-based routing is broken.

### AC-14: Self-Verification Preserved in Context

- **Pass:** Cluster decision sections retain self-verification logic (evaluate → decide → proceed) but log results to the orchestrator's context window (captured by telemetry tracking) rather than `memory.md`.
- **Fail:** Self-verification logic is entirely removed, or it still references writing to `memory.md`.

### AC-15: Subagent Fallback Preserved

- **Pass:** All 20 subagent files retain a fallback rule: "If upstream memory files are missing, log a warning and proceed with direct artifact reads."
- **Fail:** Any subagent loses its graceful degradation fallback.

---

## Phase 1 Scope

**In-scope (Phase 1):**

| AC | Summary | Rationale |
|----|---------|-----------|
| AC-1 | YAML `tools:` field present with 5 tools | Direct tool restriction |
| AC-2 | Operating Rule 5 matches YAML exactly | Direct tool restriction |
| AC-3 | Zero prohibited tools in allowed contexts | Direct tool restriction |
| AC-11 | `memory` tool disambiguated from pipeline files | Fixes pre-existing contradictions |
| AC-12 | Anti-Drift Anchor reflects restricted tool set | Direct tool restriction |

**Deferred (Phase 2 — Memory Architecture Redesign):**

| AC | Summary | Rationale |
|----|---------|-----------|
| AC-4 through AC-10 | Memory.md elimination, subagent memory architecture | Requires full memory redesign (23-file scope) |
| AC-13 | Upstream memory context provided to all agents | Pass/fail criteria require Phase 2 behavior (memory.md elimination) |
| AC-14 | Memory lifecycle fully managed by subagents | Pass/fail criteria require Phase 2 behavior (memory.md elimination) |
| AC-15 | Pipeline completes end-to-end | Requires Phase 2 changes in place |

**AC-12 Phase 1 Interpretation:**
> AC-12's pass criterion "does not mention memory.md" applies to Phase 2 only. In Phase 1, AC-12 is verified by confirming that the Anti-Drift Anchor (a) reflects the restricted 5-tool set, (b) lists all 8 prohibited tools, (c) provides dual prohibition rationale, and (d) includes `memory` tool disambiguation. Memory.md references are preserved in Phase 1 as the memory architecture is unchanged.

**Reference:** Design §16.3 (AC mapping), §16.5 (scope note). CT-Strategy finding in `memory/ct-strategy.mem.md` (AC-13/AC-14 deferral).

---

## Edge Cases & Error Handling

### EC-1: Missing Upstream Memory Files at Dispatch

- **Input/Condition:** The orchestrator dispatches a subagent, but one or more upstream `memory/<agent>.mem.md` files listed in the dispatch do not exist (e.g., an upstream agent returned ERROR and did not produce a memory file).
- **Expected Behavior:** The subagent's existing fallback logic activates: "log a warning and proceed with direct artifact reads." The orchestrator's Input Validation (cluster decision logic) already treats missing memory as worst-case for routing decisions.
- **Severity if Missed:** High — pipeline would stall if no fallback exists.

### EC-2: Malformed Memory File

- **Input/Condition:** A `memory/<agent>.mem.md` file exists but lacks the expected `## Highest Severity` or `## Status` section that cluster decision logic depends on.
- **Expected Behavior:** The orchestrator's Input Validation treats malformed memory as worst-case (e.g., assumes `critical` severity, `ERROR` status). Cluster decision proceeds with conservative routing.
- **Severity if Missed:** High — orchestrator could make incorrect cluster decisions with incomplete data.

### EC-3: Revision Loop Without Invalidation Mechanism

- **Input/Condition:** A V cluster or R cluster identifies issues requiring revision. Previously, `memory.md` entries were invalidated with `[INVALIDATED]` markers. Now there is no `memory.md` to invalidate.
- **Expected Behavior:** The orchestrator includes explicit invalidation context in the revision dispatch prompt: "Previous V/R cluster results are superseded. Re-evaluate based on the revised artifacts." Subagents receiving the revision dispatch ignore prior cluster memory files from the invalidated run.
- **Severity if Missed:** Medium — revision agents might use stale context from prior cluster runs, leading to incorrect verification/review results.

### EC-4: R-Knowledge Agent Missing Aggregated Artifact Index

- **Input/Condition:** The R-Knowledge agent previously relied most heavily on `memory.md`'s aggregated Artifact Index for navigating `feature.md`, `design.md`, and `plan.md`. Without `memory.md`, it must reconstruct navigation from individual `*.mem.md` files.
- **Expected Behavior:** R-Knowledge reads upstream memory files (which each contain an Artifact Index section) and uses those section-level pointers for targeted reads. Its fallback logic ("proceed with direct artifact reads") applies if memory files are insufficient.
- **Severity if Missed:** Medium — R-Knowledge may read larger portions of artifacts than necessary, increasing token usage but not causing functional failure.

### EC-5: Lessons Learned Not Propagated Between Waves

- **Input/Condition:** Implementation wave 1 produces Lessons Learned entries in implementer `*.mem.md` files. Wave 2 dispatches do not include these lessons.
- **Expected Behavior:** The orchestrator reads wave 1 implementer memory files between waves (already a read-only operation), extracts Lessons Learned entries, and includes them as text context in wave 2 dispatch prompts.
- **Severity if Missed:** Medium — wave 2 implementers may repeat mistakes already resolved in wave 1, but the pipeline does not fail.

### EC-6: Concurrent Subagents Reading Same Memory Files

- **Input/Condition:** Parallel cluster subagents (e.g., 4 CT agents) all read the same upstream memory files simultaneously.
- **Expected Behavior:** No conflict — memory files are read-only from the subagent perspective. Each subagent writes only to its own isolated `memory/<agent>.mem.md`. No shared mutable state exists.
- **Severity if Missed:** Low — this is already the current behavior and works correctly.

### EC-7: VS Code `memory` Tool Conflation

- **Input/Condition:** The LLM powering the orchestrator interprets the `memory` tool as a mechanism for writing `memory.md` (the previous behavior) rather than VS Code's cross-session knowledge store.
- **Expected Behavior:** The prompt disambiguation (FR-6.1) prevents this. The `memory` tool is clearly labeled for cross-session codebase facts. No `memory.md` file exists to write to.
- **Severity if Missed:** Critical — orchestrator could attempt to create/write `memory.md` using wrong tools or hallucinate file operations.

### EC-8: PostMortem Agent Missing Aggregated Context

- **Input/Condition:** PostMortem agent (Step 8) previously received `memory.md` as aggregated context for the entire pipeline. Without it, the agent must read all individual `*.mem.md` files.
- **Expected Behavior:** The orchestrator dispatch for PostMortem includes paths to ALL `memory/*.mem.md` files produced during the run. The PostMortem agent reads them individually. Telemetry context (from the orchestrator's context window, per Global Rule 13) supplements the memory files.
- **Severity if Missed:** Medium — PostMortem might miss cross-phase patterns, but individual memory files plus telemetry provide equivalent information.

---

## User Stories / Flows

### Story 1: Standard Pipeline Run (Happy Path)

1. Orchestrator starts at Step 0. No `memory.md` is created.
2. Step 1: Orchestrator dispatches 4 researchers with `initial-request.md`. Each researcher writes its own `memory/researcher-*.mem.md`.
3. Step 1.1m (simplified): Orchestrator reads the 4 researcher `*.mem.md` files using `read_file` to confirm status (DONE/ERROR). No merge operation. Proceeds to Step 2.
4. Step 2: Orchestrator dispatches spec agent with paths to `memory/researcher-*.mem.md` files (no `memory.md`). Spec agent reads upstream memories directly.
5. Steps 3–8 follow the same pattern: dispatch with upstream memory paths, read `*.mem.md` for status, route based on cluster decisions.
6. Step 8: PostMortem receives paths to all `*.mem.md` files plus telemetry context.

### Story 2: Cluster Decision Routing

1. V cluster completes. Orchestrator reads `memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md`.
2. Extracts `## Highest Severity` from each. Applies the V cluster decision table.
3. Result is logged in orchestrator context window (telemetry tracking). No `memory.md` write.
4. If any severity is `critical` or `high`, orchestrator dispatches revision loop with dispatch context: "V cluster found issues: [summary]. Prior V results are superseded."

### Story 3: Multi-Wave Implementation

1. Wave 1: 5 implementer agents dispatched with `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md`.
2. Wave 1 completes. Orchestrator reads all 5 `memory/implementer-*.mem.md` files.
3. Orchestrator extracts Lessons Learned entries from wave 1 memories (read-only).
4. Wave 2: 3 implementer agents dispatched with same upstream paths + Lessons Learned text from wave 1 included in dispatch prompt.

---

## Test Scenarios

### TS-1: Tool Restriction Verification

Search `orchestrator.agent.md` for strings `grep_search`, `semantic_search`, `file_search`, `get_errors`. Expect zero matches. Verify YAML frontmatter contains `tools:` field with exactly 5 tools. (Validates AC-1, AC-2, AC-3)

### TS-2: memory.md Elimination Verification

Search `orchestrator.agent.md` for string `memory.md`. Expect zero matches (or only inside a historical context comment). Verify no step instructs writing, merging, pruning, or invalidating `memory.md`. (Validates AC-4, AC-7, AC-8)

### TS-3: Subagent Input Update Verification

For each of the 20 subagent `.agent.md` files, search for `memory.md` in the Inputs and Operating Rules sections. Expect zero matches. Verify each agent lists specific upstream `memory/<agent>.mem.md` paths instead. (Validates AC-5, AC-15)

### TS-4: Supporting Document Verification

Search `feature-workflow.prompt.md` and `dispatch-patterns.md` for `memory.md` as a pipeline artifact. Expect zero matches for `memory.md` as maintained-by-orchestrator. (Validates AC-6)

### TS-5: Dispatch Prompt Correctness

For each pipeline step in `orchestrator.agent.md`, verify the dispatch instruction lists the correct upstream `memory/<agent>.mem.md` files per the FR-3.2 mapping table. No dispatch should include `memory.md`. (Validates AC-9)

### TS-6: Lessons Learned Propagation

Verify the orchestrator prompt contains explicit instructions between implementation waves to: (a) read implementer `*.mem.md` files, (b) extract Lessons Learned, (c) include extracted lessons in wave N+1 dispatch context. Verify the same mechanism applies to V cluster and R cluster dispatches. (Validates AC-10)

### TS-7: Memory Tool Disambiguation

Search `orchestrator.agent.md` for a statement clarifying that the `memory` tool is VS Code's cross-session knowledge store. Verify no prompt language implies `memory` tool can write pipeline files. (Validates AC-11)

### TS-8: Anti-Drift Anchor Verification

Read the Anti-Drift Anchor section. Verify it: (a) references only `read_file` and `list_dir` as read tools, (b) does not mention `memory.md`, (c) does not claim the orchestrator writes to any shared file, (d) lists prohibited tools including `grep_search`, `semantic_search`, `file_search`, `get_errors`. (Validates AC-12)

### TS-9: Cluster Decision Flow Integrity

Read CT, V, and R cluster decision flow sections. Verify: (a) they still read `memory/<agent>.mem.md` files, (b) they apply decision tables, (c) they do not write to `memory.md`, (d) self-verification results are described as context-window captured (not file-written). (Validates AC-13, AC-14)

### TS-10: Fallback Preservation

For each of the 20 subagent files, verify Operating Rules contain a fallback rule matching: "If upstream memory files are missing, log a warning and proceed with direct artifact reads." (Validates AC-15)

### TS-11: YAML Frontmatter Convention

Verify the orchestrator is the first/only agent with a `tools:` YAML field. Verify no other agent file has gained or lost YAML fields. (Validates AC-1, NFR-4.1)

---

## Dependencies & Risks

### Dependencies

| Dependency | Type | Risk |
|---|---|---|
| All 20 subagent `.agent.md` files | Files to modify | Mass edit — inconsistent updates could leave some agents referencing `memory.md` |
| `feature-workflow.prompt.md` | File to modify | Entry point prompt — must reflect new architecture |
| `dispatch-patterns.md` | File to modify | Reference document — must remove merge pattern references |
| VS Code YAML `tools:` runtime enforcement | Platform behavior | May be advisory only — prose restrictions serve as primary enforcement |
| Telemetry Context Tracking (Global Rule 13) | Existing feature | Self-verification results depend on telemetry capture remaining functional |

### Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Silent quality degradation from missing aggregated Artifact Index | Medium | Low | Each `*.mem.md` contains its own Artifact Index; subagents already read upstream memories directly |
| Lessons Learned not reaching later agents | Medium | Medium | Explicit propagation via dispatch prompt context (FR-3.3, FR-3.4) |
| Inconsistent subagent updates (some still reference memory.md) | Medium | Medium | Use automated text search to verify all 20 files are updated (TS-3) |
| VS Code `tools:` field not enforced at runtime | Low | Low | Prose restrictions in Operating Rules serve as primary enforcement; YAML is declarative |
| LLM interpreting `memory` tool as file writer | Medium | High | Explicit disambiguation in prompt (FR-6.1); no `memory.md` file exists to write to |
| R-Knowledge performance degradation | Low | Low | Agent has fallback; reads upstream memories with Artifact Indexes for navigation |
