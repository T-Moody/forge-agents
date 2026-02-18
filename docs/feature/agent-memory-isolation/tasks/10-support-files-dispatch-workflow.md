# Task 10: Support Files — Dispatch Patterns + Feature Workflow

## Task Goal

Update `dispatch-patterns.md` to remove aggregator invocation steps from Patterns A/B/C and update `feature-workflow.prompt.md` to reflect the new isolated memory architecture, removing aggregator and combined artifact references.

## depends_on

none

## agent

implementer

## In-Scope

### dispatch-patterns.md

- **Pattern A (Fully Parallel)**: Remove steps related to "invoke aggregator" and "check aggregator completion contract". Replace with "Orchestrator reads sub-agent isolated memories and determines cluster result". Keep ≥2 outputs threshold
- **Pattern B (Sequential Gate + Parallel)**: Remove "forward to aggregator", "invoke aggregator with all available outputs", "check aggregator completion contract". Replace with orchestrator reads memories
- **Pattern C (Replan Loop)**: Replace "If aggregator DONE" with "If orchestrator determines DONE from V memories". Replace `verifier.md` references with individual V sub-agent artifacts. Replace "Invoke planner with verifier.md" with "Invoke planner with MODE: REPLAN + V memory paths". Keep max 3 iterations
- Remove all references to `design_critical_review.md`, `verifier.md`, `review.md`, `analysis.md` as combined artifacts
- Remove all aggregator agent name references

### feature-workflow.prompt.md

- Remove aggregator references from any rules/descriptions
- Update research description: no synthesis step, no `analysis.md`
- Add isolated memory model description (agents produce `memory/<agent-name>.mem.md`, orchestrator merges)
- Update Key Artifacts table (if present): remove `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`
- Update approval gate reference: "after research completion" not "research synthesis"

## Out-of-Scope

- Orchestrator inline pattern references (handled by task 03)
- Agent-specific modifications

## Acceptance Criteria

- AC-13: `dispatch-patterns.md` Pattern A and B contain no aggregator invocation steps or combined artifact references
- AC-17: `feature-workflow.prompt.md` has no aggregator, `analysis.md`, `design_critical_review.md`, `verifier.md`, or `review.md` references; describes isolated memory model
- No aggregator agent names in either file
- Pattern C references V memories and individual V artifacts, `MODE: REPLAN` signal

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `dispatch-patterns.md` to identify Pattern A, B, C sections and all aggregator references
2. **Pattern A**: Remove aggregator invocation step(s). Add "Orchestrator reads sub-agent isolated memories (`memory/<agent-name>.mem.md`) and applies cluster decision logic to determine result (DONE/NEEDS_REVISION/ERROR)". Retain ≥2 outputs threshold
3. **Pattern B**: Remove aggregator forwarding and invocation steps. Add orchestrator memory reading step. Update gate failure path (no aggregator to forward to — orchestrator handles)
4. **Pattern C**: Replace aggregator references with orchestrator V evaluation. Replace `verifier.md` with V memories + individual artifacts. Add `MODE: REPLAN` signal in planner invocation. Keep max 3 iterations
5. Search for and remove any remaining aggregator names or combined artifact paths
6. Read `feature-workflow.prompt.md` to identify all update targets
7. Remove/replace aggregator agent references
8. Update research phase description: remove synthesis step, remove `analysis.md`
9. Add isolated memory model description: "Each agent produces a compact isolated memory file (`memory/<agent-name>.mem.md`) alongside its primary artifact. The orchestrator reads these memories for routing decisions and merges them into shared `memory.md`."
10. Update Key Artifacts table: remove removed artifact entries
11. Update approval gate reference if present
12. Final verification: grep both files for aggregator names and removed artifact paths

## Completion Checklist

- [x] dispatch-patterns.md: Pattern A has no aggregator invocation
- [x] dispatch-patterns.md: Pattern B has no aggregator invocation
- [x] dispatch-patterns.md: Pattern C references orchestrator V evaluation, V memories, `MODE: REPLAN`
- [x] dispatch-patterns.md: no aggregator names remain
- [x] dispatch-patterns.md: no `design_critical_review.md`, `verifier.md` (artifact), `review.md` (artifact), `analysis.md` references
- [x] feature-workflow.prompt.md: no aggregator references
- [x] feature-workflow.prompt.md: no removed artifact references
- [x] feature-workflow.prompt.md: describes isolated memory model
- [x] feature-workflow.prompt.md: research phase has no synthesis step

## Status

Complete

## Notes

- TDD skipped: markdown-only task, no behavioral code.
- Updated intro line in dispatch-patterns.md to remove stale note about Pattern C being inline-only.
