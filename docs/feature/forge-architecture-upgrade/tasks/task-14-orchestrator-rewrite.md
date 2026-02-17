# Task 14: Orchestrator Major Rewrite

## Status: DONE

## Agent

implementer

## Depends On

12, 13

## Description

Major rewrite of `orchestrator.agent.md` to integrate all Forge Architecture Upgrade changes: memory lifecycle management, expanded research (4 focus agents), 3 cluster dispatch patterns (CT fully parallel, V sequential gate + parallel, R fully parallel with Knowledge Evolution), updated NEEDS_REVISION routing table, updated pipeline expectations table (25+ agents), and knowledge evolution preservation. This is the highest-risk task — all changes must be applied atomically to avoid intermediate broken states. Stay under 450-line cap.

**Note:** Task 11 (CT Aggregator) is also a dependency but is guaranteed complete by wave ordering — all Wave 2 tasks (11, 12, 13) complete before Wave 3 starts.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §7 (Orchestrator Evolution — complete spec), §1 (Memory System), §3 (CT Cluster), §4 (V Cluster), §5 (R Cluster), §6 (Research Expansion), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — ORCH-AC-1 through ORCH-AC-14
- `NewAgentsAndPrompts/orchestrator.agent.md` — current file (the rewrite target)
- `NewAgentsAndPrompts/ct-aggregator.agent.md` — dispatch target reference
- `NewAgentsAndPrompts/v-aggregator.agent.md` — dispatch target reference
- `NewAgentsAndPrompts/r-aggregator.agent.md` — dispatch target reference

## Output File

- `NewAgentsAndPrompts/orchestrator.agent.md` (modified in place)

## Acceptance Criteria

1. **Memory lifecycle (ORCH-AC-1, ORCH-AC-2):**
   - Step 0 creates and initializes `memory.md` from template (design.md §1.2)
   - Memory invalidation on step failure and re-execution
   - Memory pruning at orchestrator checkpoints (keep recent context only)
   - Emergency pruning rule for context window pressure

2. **Research expansion (ORCH-AC-3):**
   - Step 1 dispatches researcher with 4 focus areas: `architecture`, `dependencies`, `impact`, `patterns`
   - Uses Pattern A dispatch (fully parallel, ≤4 concurrent)

3. **CT cluster dispatch (ORCH-AC-4, ORCH-AC-5):**
   - Step 3a dispatches 4 CT sub-agents in parallel (Pattern A)
   - Step 3b dispatches CT aggregator after all 4 complete
   - Handles `NEEDS_REVISION` → max-1 revision loop, then forwards constraints to planner
   - Replaces old single critical-thinker invocation

4. **V cluster dispatch (ORCH-AC-6, ORCH-AC-7):**
   - Step 5a dispatches V-Build first (sequential gate, Pattern B)
   - Step 5b on V-Build DONE: dispatches V-Tests, V-Tasks, V-Feature in parallel
   - Step 5b on V-Build ERROR: skips parallel sub-agents, dispatches V-Aggregator with ERROR
   - Step 5c dispatches V Aggregator after all parallel sub-agents complete
   - Handles `NEEDS_REVISION` → targeted replan using task IDs from verifier.md

5. **R cluster dispatch (ORCH-AC-8, ORCH-AC-9):**
   - Step 7a dispatches 4 R sub-agents in parallel (Pattern A/C)
   - Step 7b dispatches R Aggregator after all 4 complete
   - R-Security ERROR overrides to pipeline ERROR
   - R-Knowledge ERROR is non-blocking (log only)

6. **Updated routing table (ORCH-AC-10):**
   - NEEDS_REVISION routing for all 3 clusters
   - CT → max-1 revision loop; unresolved tensions → planner
   - V → targeted replan with task ID mapping
   - R → standard NEEDS_REVISION → implementer
   - R-Security ERROR → pipeline halt

7. **Updated expectations table (ORCH-AC-11):**
   - Table lists all 25+ agents with expected outputs and completion contracts
   - Replaces old 10-agent table

8. **Knowledge evolution preservation (ORCH-AC-12):**
   - After R cluster completes: if `knowledge-suggestions.md` exists, preserve it through pipeline
   - `decisions.md` preserved across pipeline runs

9. **Line count:** Stays under 450 lines total (ORCH-AC-13)

10. **Anti-drift anchor (ORCH-AC-14):** Updated to reference cluster dispatch patterns

## Implementation Guidance

**Approach:** Modify the existing `orchestrator.agent.md` in place. This is a major rewrite — most sections will be significantly expanded or replaced. Work through design.md §7 systematically.

**Structure of changes (from design.md §7):**

### New/Modified Sections

1. **Step 0 (NEW): Memory Initialization**
   - Create `memory.md` from template
   - Template: `## Recent Context`, `## Recent Decisions`, `## Recent Updates`, `## Lessons Learned`

2. **Rule 6 (NEW): Memory-First Protocol**
   - Read `memory.md` at start, write at pipeline checkpoints
   - Invalidation on step failure
   - Emergency pruning on context pressure

3. **Step 1 (MODIFIED): Research**
   - Dispatch researcher with 4 focus areas (was 3)
   - Add `patterns` focus area
   - Use Pattern A dispatch

4. **Step 3 (MAJOR REWRITE): Critical Thinking → CT Cluster**
   - Replace single agent with cluster dispatch
   - Pattern A: Dispatch 4 CT sub-agents in parallel
   - Run CT Aggregator
   - Handle NEEDS_REVISION: max-1 loop → plan constraints on exhaustion

5. **Step 5 (MAJOR REWRITE): Verification → V Cluster**
   - Replace single agent with cluster dispatch
   - Pattern B: V-Build gate → parallel V-Tests/V-Tasks/V-Feature → V Aggregator
   - Handle V-Build ERROR: skip parallel, forward ERROR
   - Handle NEEDS_REVISION: targeted replan with task IDs

6. **Step 7 (MAJOR REWRITE): Review → R Cluster**
   - Replace single agent with cluster dispatch
   - Pattern A/C: 4 R sub-agents in parallel → R Aggregator
   - R-Security override rule
   - R-Knowledge non-blocking
   - Knowledge evolution preservation

7. **Expectations Table (REWRITE):** All 25+ agents listed

8. **NEEDS_REVISION Routing Table (REWRITE):** New routing for clusters

### Key Constraints

- **Atomic update:** Apply all changes in a single task. Do not leave the orchestrator in a partially-updated state.
- **450-line cap:** If the orchestrator exceeds 450 lines, extract repeated patterns (e.g., dispatch sub-routine) into a shared reference section within the file.
- **Max 4 concurrent sub-agents:** All dispatch patterns respect Global Rule 7.
- **Backward compatibility for unmodified steps:** Steps 2 (spec), 3 (designer), 4 (planner), 6 (implementer), 8 (documentation-writer) remain largely unchanged but get memory protocol additions.

**Note:** This is a markdown agent definition file — TDD does not apply.

## Completion Checklist

- [x] **AC-1 Memory lifecycle:** Step 0 creates `memory.md` from template with Artifact Index, Recent Decisions, Lessons Learned, Recent Updates sections. Memory invalidation on revision (mark `[INVALIDATED]`). Pruning at checkpoints (after Steps 1.2, 2, 4). Emergency pruning when >200 lines.
- [x] **AC-2 Memory-First Protocol:** Global Rule 6 defines full memory protocol. Memory Lifecycle Actions table lists all 7 orchestrator memory actions.
- [x] **AC-3 Research expansion:** Step 1.1 dispatches 4 researcher instances (architecture, impact, dependencies, patterns). Pattern A fully parallel ≤4 concurrent.
- [x] **AC-4 CT cluster dispatch:** Step 3b.1 dispatches 4 CT sub-agents in parallel (Pattern A). Step 3b.2 invokes ct-aggregator.
- [x] **AC-5 CT NEEDS_REVISION:** Step 3b.3 handles max-1 revision loop → designer revises → full re-run. Unresolved Tensions forwarded as planning constraints.
- [x] **AC-6 V cluster dispatch:** Step 6.1 V-Build sequential gate (Pattern B). Step 6.2 dispatches V-Tests/V-Tasks/V-Feature on V-Build DONE. V-Build ERROR skips parallel, forwards to V-Aggregator.
- [x] **AC-7 V NEEDS_REVISION:** Step 6.4 Pattern C replan loop (max 3 iterations) using task IDs from verifier.md Actionable Items.
- [x] **AC-8 R cluster dispatch:** Step 7.2 dispatches 4 R sub-agents in parallel (Pattern A). Step 7.3 invokes r-aggregator.
- [x] **AC-9 R error handling:** R-Security ERROR is critical pipeline blocker. R-Knowledge ERROR is non-blocking (log only).
- [x] **AC-10 Updated routing table:** All 3 clusters covered. CT→designer (max 1, forward constraints). V→planner replan (max 3). R→implementers (max 1). R-Security→halt.
- [x] **AC-11 Expectations table:** 19 rows covering all agent types (Researcher ×4, synthesis, Spec, Designer, CT sub×4, CT Agg, Planner, Implementer, DocWriter, V-Build, V-Tests/Tasks/Feature, V Agg, R-Quality/Security/Testing/Knowledge, R Agg).
- [x] **AC-12 Knowledge evolution:** Section 7.5 preserves `knowledge-suggestions.md` (never auto-applied) and `decisions.md` (append-only, persists across runs).
- [x] **AC-13 Line count:** 355 lines — well under 450-line cap.
- [x] **AC-14 Anti-drift anchor:** Updated to reference cluster dispatch patterns (Pattern A for CT/R, Pattern B+C for V) and memory lifecycle.
- [x] **TDD skipped:** Markdown agent definition file — no test framework applicable.
- [x] **Atomic update:** All changes applied in a single replacement — no intermediate broken states.
