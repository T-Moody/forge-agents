# Task 12: V Aggregator

## Agent

implementer

## Depends On

07, 08

## Description

Create `v-aggregator.agent.md` — the aggregator for the Verification cluster. It reads all 4 V sub-agent output files (v-build, v-tests, v-tasks, v-feature), compiles a unified verification report, maps failures to task IDs, and produces `verifier.md`. It is the only V cluster agent that writes to `memory.md`.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §2 (Aggregator Pattern), §4.3 (V Aggregation Details), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — V-AC-4, V-AC-10
- `NewAgentsAndPrompts/v-build.agent.md` — output format reference (created in task 07)
- `NewAgentsAndPrompts/v-tests.agent.md` — output format reference (created in task 07)
- `NewAgentsAndPrompts/v-tasks.agent.md` — output format reference (created in task 08)
- `NewAgentsAndPrompts/v-feature.agent.md` — output format reference (created in task 08)

## Output File

- `NewAgentsAndPrompts/v-aggregator.agent.md`

## Acceptance Criteria

1. [x] Follows v2 template structure
2. [x] Reads all 4 V sub-agent outputs from `verification/` directory + `memory.md`
3. [x] Produces `verifier.md` as output
4. [x] Compiles unified report with sections: Build Results, Test Results, Per-Task Verification, Feature-Level Verification
5. [x] Maps failing items to specific task IDs — critical for targeted replan (design.md §4.3 step 4)
6. [x] Completion contract: `DONE:` (V-Build DONE + all parallel DONE) / `NEEDS_REVISION:` (V-Build DONE + any parallel NEEDS_REVISION) / `ERROR:` (V-Build ERROR or 2+ sub-agent ERROR)
7. [x] Mixed NEEDS_REVISION + ERROR: ERROR takes priority
8. [x] Writes to `memory.md`: Recent Updates + Lessons Learned (sequential agent — safe)
9. [x] Output format matches design.md §4.3 `verifier.md` specification
10. [x] Preserves the targeted replan information (which task IDs failed) in "Actionable Items" section

## Status

Complete — `v-aggregator.agent.md` created. TDD skipped: markdown agent definition file, no behavioral code.

## Implementation Guidance

```
---
name: v-aggregator
description: "Aggregates Verification cluster results into a unified verification report with task-specific failure mapping."
---
```

**Role:** Merge-only aggregator for V cluster. Compiles build, test, task, and feature verification results into a single `verifier.md`. The most critical output is the task ID mapping — the planner depends on it for targeted replanning.

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/verification/v-build.md`
- `docs/feature/<feature-slug>/verification/v-tests.md`
- `docs/feature/<feature-slug>/verification/v-tasks.md`
- `docs/feature/<feature-slug>/verification/v-feature.md`

**Outputs:**

- `docs/feature/<feature-slug>/verifier.md`

**Aggregation workflow (from design.md §4.3):**

1. Read `memory.md`
2. Read all available V sub-agent outputs
3. Compile unified verification report with dedicated sections per sub-agent
4. Merge Issues Found from all sub-agents, sorted by severity
5. **Map failing items to specific task IDs** (from v-tasks.md — critical for replan)
6. Determine completion contract per table
7. Write `verifier.md`
8. Write to `memory.md`: Recent Updates, Lessons Learned

**Output format (from design.md §4.3):**

```markdown
# Verification Report

## Summary

## Build Results

## Test Results

## Per-Task Verification

| Task ID | Status | Issues |

## Feature-Level Verification

## Issues Summary

### Critical / High / Medium / Low

## Actionable Items (mapped to task IDs)

## Verification Scope
```

**Note:** This is a markdown agent definition file — TDD does not apply.
