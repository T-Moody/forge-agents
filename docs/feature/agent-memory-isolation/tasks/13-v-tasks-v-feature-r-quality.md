# Task 13: V-Tasks + V-Feature + R-Quality — Isolated Memory

## Task Goal

Add isolated memory output, "Write Isolated Memory" workflow step, and updated Anti-Drift Anchors to `v-tasks.agent.md`, `v-feature.agent.md`, and `r-quality.agent.md`. Additionally, replace V Aggregator references with orchestrator/planner references in v-tasks and v-feature.

## depends_on

none

## agent

implementer

## In-Scope

### v-tasks.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/v-tasks.mem.md (isolated memory)`
- **Workflow**: Replace "No Memory Write" step (if exists) with "Write Isolated Memory". V-Tasks uses `Highest Severity: PASS/FAIL`. Status includes DONE/NEEDS_REVISION/ERROR
- **Anti-Drift Anchor**: Update to reference isolated memory. Remove V Aggregator references
- **Additional**: Replace "for the V-Aggregator and planner" (line ~53) with "for the orchestrator and planner". Replace "The V-Aggregator handles memory updates" (line ~59) with isolated memory reference

### v-feature.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/v-feature.mem.md (isolated memory)`
- **Workflow**: Replace "No Memory Write" step (if exists) with "Write Isolated Memory". V-Feature uses `Highest Severity: PASS/FAIL`. Status includes DONE/NEEDS_REVISION/ERROR
- **Anti-Drift Anchor**: Update to reference isolated memory. Remove V Aggregator references
- **Additional**: Replace "for the V-Aggregator and planner" (line ~52) with "for the orchestrator and planner". Replace "The V-Aggregator handles memory updates" (line ~58) with isolated memory reference

### r-quality.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/r-quality.mem.md (isolated memory)`
- **Workflow**: Replace "No Memory Write" step (~line 114) with "Write Isolated Memory". R-Quality uses `Highest Severity: Blocker/Major/Minor` (R taxonomy)
- **Anti-Drift Anchor**: Update to reference isolated memory. Remove R Aggregator references

## Out-of-Scope

- r-security.agent.md (task 09)
- r-testing.agent.md, r-knowledge.agent.md (task 14)
- Orchestrator V/R decision logic (task 03)

## Acceptance Criteria

- AC-5: All 3 files list isolated memory in Outputs and have "Write Isolated Memory" workflow step
- AC-16: All 3 Anti-Drift Anchors reference isolated memory, no aggregator references
- AC-2 (partial): No aggregator references remain in any of the 3 files
- AC-19: Section ordering preserved
- AC-20: Completion contracts unchanged
- v-tasks and v-feature: "V-Aggregator" references replaced with "orchestrator"

## Estimated Effort

Low

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `v-tasks.agent.md` — locate "V-Aggregator and planner" (line ~53), "V-Aggregator handles memory" (line ~59), Anti-Drift Anchor
2. Apply changes: replace V-Aggregator references, add memory output, replace memory write step, update Anti-Drift
3. Read `v-feature.agent.md` — locate same patterns (lines ~52, ~58)
4. Apply identical pattern to v-feature
5. Read `r-quality.agent.md` — locate "No Memory Write" step (~line 114), Anti-Drift Anchor
6. Apply changes: add memory output with R taxonomy (Blocker/Major/Minor), replace memory write step, update Anti-Drift, remove R Aggregator references
7. Verify consistency and no aggregator references remain

## Completion Checklist

- [x] v-tasks.agent.md: memory output, write step (PASS/FAIL), Anti-Drift updated
- [x] v-tasks.agent.md: "V-Aggregator and planner" → "orchestrator and planner"
- [x] v-tasks.agent.md: "V-Aggregator handles memory" removed/replaced
- [x] v-feature.agent.md: memory output, write step (PASS/FAIL), Anti-Drift updated
- [x] v-feature.agent.md: "V-Aggregator and planner" → "orchestrator and planner"
- [x] v-feature.agent.md: "V-Aggregator handles memory" removed/replaced
- [x] r-quality.agent.md: memory output, write step (Blocker/Major/Minor), Anti-Drift updated
- [x] r-quality.agent.md: R Aggregator references removed
- [x] No aggregator references in any file
- [x] Section ordering preserved
- [x] Completion contracts unchanged
