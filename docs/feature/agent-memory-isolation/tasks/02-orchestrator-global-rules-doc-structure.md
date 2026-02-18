# Task 02: Orchestrator — Global Rules, Documentation Structure, Memory Lifecycle

## Task Goal

Update the orchestrator's Global Rules (6, 10, 12), Documentation Structure table, and Memory Lifecycle Actions to reflect the new isolated memory model, removing aggregator references and establishing the orchestrator as sole `memory.md` writer.

## depends_on

none

## agent

implementer

## In-Scope

- **Global Rule 6** (Memory-First Protocol): Add "Merge" action — after each agent/cluster completes, read isolated memory files, merge into `memory.md`. Remove pruning checkpoint at "Step 1.2" (no longer exists). Prune after Steps 1.1, 2, 4.
- **Global Rule 10** (Approval Gate): Replace "after Step 1.2 (research synthesis)" with "after Step 1.1 (research completion)".
- **Global Rule 12** (Memory Write Safety): Rewrite entirely — orchestrator is sole writer to shared `memory.md`. All other agents write only to `memory/<agent-name>.mem.md`. Remove read-only/read-write lists. Remove aggregator names.
- **Documentation Structure table**: Remove `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`. Add `memory/` directory with `<agent-name>.mem.md` entries.
- **Memory Lifecycle Actions table**: Add "Merge" action row. Update "Validate" action to reference isolated memories instead of aggregator writes.
- **Outputs section**: Add `memory/` directory. Clarify sole writer to `memory.md`.

## Out-of-Scope

- Workflow Steps (handled by tasks 04, 05)
- Cluster decision logic (handled by task 03)
- NEEDS_REVISION Routing Table (handled by task 05)
- Orchestrator Expectations table (handled by task 05)
- Completion Contract and Anti-Drift Anchor (handled by task 05)

## Acceptance Criteria

- AC-14: Global Rule 12 states orchestrator is sole `memory.md` writer; no aggregators listed
- AC-15: Documentation Structure table includes `memory/` directory, excludes removed artifacts
- AC-6 (partial): Memory Lifecycle includes "Merge" action
- No aggregator names (`ct-aggregator`, `v-aggregator`, `r-aggregator`) appear in any modified section
- No removed artifact names (`analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`) appear in Documentation Structure
- Global Rule 10 references Step 1.1, not Step 1.2

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `orchestrator.agent.md` to locate Global Rules 6, 10, 12, Documentation Structure table, Memory Lifecycle Actions, and Outputs section
2. Update **Outputs section**: add `docs/feature/<feature-slug>/memory/` directory; clarify `memory.md` sole writer
3. Update **Global Rule 6**: add "Merge" action description; update pruning checkpoints (remove Step 1.2, set to Steps 1.1, 2, 4)
4. Update **Global Rule 10**: replace "Step 1.2 (research synthesis)" with "Step 1.1 (research completion)"
5. Rewrite **Global Rule 12**: replace the entire read-only/read-write lists with the new model per design.md — orchestrator sole writer, all agents write only to isolated memory
6. Update **Documentation Structure table**: remove 4 removed artifacts, add `memory/` directory row with `<agent-name>.mem.md` description
7. Update **Memory Lifecycle Actions table**: add "Merge" row (after each cluster/agent completes, read isolated memories, merge into `memory.md`); update "Validate" row
8. Verify no aggregator names or removed artifact names remain in modified sections

## Completion Checklist

- [x] Global Rule 6 includes "Merge" action and updated prune checkpoints
- [x] Global Rule 10 references Step 1.1 (not 1.2)
- [x] Global Rule 12 declares orchestrator as sole `memory.md` writer
- [x] Global Rule 12 contains no aggregator names
- [x] Documentation Structure table includes `memory/` directory
- [x] Documentation Structure table excludes `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`
- [ ] Memory Lifecycle Actions includes "Merge" row (deferred — below Workflow Steps, out of scope per orchestrator instructions)
- [x] Outputs section references `memory/` directory
- [x] Standard section ordering preserved (NFR-3)
