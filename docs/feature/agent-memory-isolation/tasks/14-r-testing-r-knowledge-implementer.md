# Task 14: R-Testing + R-Knowledge + Implementer — Isolated Memory

## Task Goal

Add isolated memory output, "Write Isolated Memory" workflow step, and updated Anti-Drift Anchors to `r-testing.agent.md`, `r-knowledge.agent.md`, and `implementer.agent.md`. Additionally, handle R Aggregator reference removal and implementer-specific task-ID naming.

## depends_on

none

## agent

implementer

## In-Scope

### r-testing.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/r-testing.mem.md (isolated memory)`
- **Workflow**: Replace "No Memory Write" step (~line 151) with "Write Isolated Memory". R-Testing uses `Highest Severity: Blocker/Major/Minor` (R taxonomy)
- **Anti-Drift Anchor**: Update to reference isolated memory. Remove R Aggregator references

### r-knowledge.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/r-knowledge.mem.md (isolated memory)`
- **Workflow**: Replace "No Memory Write" step (~line 269) with "Write Isolated Memory". R-Knowledge uses `Highest Severity: Blocker/Major/Minor` (R taxonomy, though R-Knowledge is non-blocking)
- **Anti-Drift Anchor**: Update to reference isolated memory. Remove R Aggregator references (~line 12 "R Aggregator proceeds without": update to "orchestrator proceeds without")
- **Additional**: If `verifier.md` is referenced for memory navigation, remove or replace with individual V artifacts

### implementer.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/implementer-<task-id>.mem.md (isolated memory)` — note task-ID in filename
- **Workflow**: Add "Write Isolated Memory" step. Implementer uses `Highest Severity: N/A`
- **Workflow**: Add memory-first reading: "Read planner memory (`memory/planner.mem.md`) → consult artifact index → targeted reads of task file + `design.md` sections"
- **Anti-Drift Anchor**: Update to reference isolated memory. Replace "Do NOT write to `memory.md`" with "You write only to your isolated memory file (`memory/implementer-<task-id>.mem.md`), never to shared `memory.md`"

## Out-of-Scope

- r-security.agent.md (task 09)
- documentation-writer.agent.md (task 15)
- Orchestrator changes

## Acceptance Criteria

- AC-5: All 3 files list isolated memory in Outputs and have "Write Isolated Memory" workflow step
- AC-16: All 3 Anti-Drift Anchors reference isolated memory, no aggregator references
- AC-2 (partial): No aggregator references remain
- AC-19: Section ordering preserved
- AC-20: Completion contracts unchanged
- Implementer uses task-ID naming (`implementer-<task-id>.mem.md`)
- Implementer has memory-first reading workflow

## Estimated Effort

Low

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `r-testing.agent.md` — locate "No Memory Write" step (~line 151), Anti-Drift Anchor
2. Apply changes: add memory output (Blocker/Major/Minor), replace memory write step, update Anti-Drift, remove R Aggregator references
3. Read `r-knowledge.agent.md` — locate "No Memory Write" step (~line 269), R Aggregator reference (~line 12), Anti-Drift Anchor
4. Apply changes: add memory output, replace memory write step, update Anti-Drift, replace "R Aggregator proceeds without" with "orchestrator proceeds without". Check for any `verifier.md` references and remove
5. Read `implementer.agent.md` — locate Outputs, Workflow, Anti-Drift Anchor
6. Apply changes: add memory output with task-ID naming, add "Write Isolated Memory" step (N/A severity), add memory-first reading step, update Anti-Drift
7. Verify no aggregator references remain in any file

## Completion Checklist

- [x] r-testing.agent.md: memory output, write step (Blocker/Major/Minor), Anti-Drift updated
- [x] r-testing.agent.md: R Aggregator references removed
- [x] r-knowledge.agent.md: memory output, write step, Anti-Drift updated
- [x] r-knowledge.agent.md: R Aggregator references replaced with orchestrator
- [x] r-knowledge.agent.md: no `verifier.md` references (if any existed)
- [x] implementer.agent.md: memory output with task-ID naming (`implementer-<task-id>.mem.md`)
- [x] implementer.agent.md: "Write Isolated Memory" step
- [x] implementer.agent.md: memory-first reading workflow (planner memory → artifact index → targeted reads)
- [x] implementer.agent.md: Anti-Drift references isolated memory
- [x] No aggregator references in any file
- [x] Section ordering preserved
- [x] Completion contracts unchanged
