# Task 10: Orchestrator Step 8, Telemetry, and Documentation Structure

**Task Goal:** Add Step 8 (PostMortem dispatch), telemetry context-tracking instructions, documentation structure updates, expectations table row, parallel summary update, completion contract clarification, and anti-drift anchor update to the orchestrator.

**depends_on:** 02, 03

**agent:** implementer

---

## In-Scope

- Modify `.github/agents/orchestrator.agent.md` (Track A additions — applied AFTER Task 03's Track B changes):
  - Add Step 8 (Post-Mortem) to the pipeline workflow with non-blocking dispatch
  - Add Step 8.2 memory merge instructions
  - Add telemetry context-tracking instructions (after each `runSubagent` return, accumulate telemetry in working context)
  - Add telemetry dispatch prompt format documentation
  - Add 3 new directories to Documentation Structure section
  - Add PostMortem row to "Orchestrator Expectations Per Agent" table
  - Add Step 8 to Parallel Execution Summary
  - Add merge action to Memory Lifecycle section
  - Update Completion Contract to clarify Step 8 is non-blocking
  - Update Anti-Drift Anchor to mention PostMortem dispatch, delegation, and read tools
  - Ensure no Telemetry section is added to memory.md initialization

## Out-of-Scope

- Tool restriction changes (already done in Task 03)
- Creating the PostMortem agent file (Task 02)
- Modifying any other agent files
- Adding evaluation workflow steps to the orchestrator (orchestrator doesn't evaluate)

---

## Acceptance Criteria

1. Step 8 exists in the workflow after Step 7, with §8.1 dispatching `post-mortem` agent and §8.2 merging memory
2. Step 8 explicitly states PostMortem ERROR does NOT block pipeline — pipeline success determined at Step 7
3. Telemetry context-tracking instructions present: after each `runSubagent` return, orchestrator notes dispatch metadata in working context
4. Telemetry dispatch prompt format documented (table format with agent dispatches and cluster summaries)
5. Documentation Structure includes `agent-metrics/`, `artifact-evaluations/`, `post-mortems/` with correct descriptions
6. Expectations table has a PostMortem row: DONE and ERROR handling described, non-blocking
7. Parallel Execution Summary includes Step 8
8. Memory Lifecycle has merge action for post-mortem
9. Completion Contract says Step 8 is non-blocking
10. Anti-Drift Anchor mentions: PostMortem dispatch at Step 8, delegation of file operations, read tools for cluster decisions
11. No `## Telemetry` section in memory.md initialization (context-based telemetry, not persistent)
12. No existing workflow steps renumbered
13. All changes are additive — no existing instructions removed

---

## Estimated Effort

Medium (~300 lines added to 1 file)

---

## Test Requirements (TDD Fallback — Configuration-Only)

1. **Verify Step 8 exists:** grep for "### 8." or "Step 8" or "Post-Mortem" in workflow section
2. **Verify non-blocking:** grep for "non-blocking" near Step 8; grep for "does NOT" near "pipeline outcome"
3. **Verify telemetry instructions:** grep for "telemetry" and "context" — should describe accumulation in working context
4. **Verify Documentation Structure:** grep for `agent-metrics/`, `artifact-evaluations/`, `post-mortems/`
5. **Verify Expectations table:** grep for "post-mortem" or "Post-Mortem" in expectations table
6. **Verify Parallel Summary:** grep for "Step 8" in parallel execution summary
7. **Verify Anti-Drift Anchor:** grep for "PostMortem" and "delegate" in anchor
8. **Verify no Telemetry section in memory init:** grep for "Telemetry" in Step 0 — should NOT appear
9. **Verify no step renumbering:** existing Steps 1–7 headers unchanged

---

## Implementation Steps

1. Read current `.github/agents/orchestrator.agent.md` (post–Task 03 changes) to understand current state
2. Read design.md for exact text of all additions:
   - §Step 8 Addition (Revised) for Step 8 text
   - §Telemetry Accumulation (Revised) for telemetry context-tracking instructions and dispatch format
   - §Documentation Structure Update for directory entries
   - §Orchestrator Expectations Table Update for PostMortem row
   - §Parallel Execution Summary Update for Step 8 entry
   - §Memory Lifecycle Update for merge action
   - §Completion Contract Update for non-blocking clarification
   - §Anti-Drift Anchor Update for updated anchor text
3. Add telemetry context-tracking instructions (place near the beginning of the pipeline flow or as a general instruction before Step 1)
4. Add Step 8 after Step 7 in the workflow
5. Add Step 8.2 memory merge
6. Add 3 new directories to Documentation Structure
7. Add PostMortem row to Expectations table
8. Add Step 8 to Parallel Execution Summary
9. Add merge action to Memory Lifecycle
10. Update Completion Contract
11. Update Anti-Drift Anchor
12. Verify no Telemetry section was added to memory.md init in Step 0
13. Verify no existing steps were renumbered

---

## Completion Checklist

- [x] Step 8 added to workflow (8.1 dispatch, 8.2 memory merge)
- [x] Step 8 explicitly non-blocking
- [x] Telemetry context-tracking instructions present
- [x] Telemetry dispatch format documented
- [x] Documentation Structure has 3 new directories
- [x] Expectations table has PostMortem row
- [x] Parallel Summary includes Step 8
- [x] Memory Lifecycle has post-mortem merge action
- [x] Completion Contract updated (Step 8 non-blocking)
- [x] Anti-Drift Anchor updated
- [x] No Telemetry section in memory.md initialization
- [x] No existing workflow steps renumbered
- [x] All changes additive

## Implementation Notes

- TDD Fallback applied: configuration-only task (Markdown agent definition file)
- All changes are additive on top of Task 03's Track B tool restriction changes
- Telemetry rule added as Global Rule 13; existing rules 1-12 unchanged
- Step 8 inserted after Step 7.5 Knowledge Evolution Preservation; Steps 0-7 headers unchanged
- Anti-Drift Anchor updated to include PostMortem dispatch, file delegation, and read tool mentions while preserving Task 03's tool restriction statement
