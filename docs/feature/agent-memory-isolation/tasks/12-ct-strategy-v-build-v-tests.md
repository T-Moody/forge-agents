# Task 12: CT-Strategy + V-Build + V-Tests — Isolated Memory

## Task Goal

Add isolated memory output, "Write Isolated Memory" workflow step, and updated Anti-Drift Anchors to `ct-strategy.agent.md`, `v-build.agent.md`, and `v-tests.agent.md`.

## depends_on

none

## agent

implementer

## In-Scope

For each of the 3 files, apply the universal sub-agent changes:

### ct-strategy.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/ct-strategy.mem.md (isolated memory)`
- **Workflow**: Add "Write Isolated Memory" step. CT taxonomy: `Highest Severity: Critical/High/Medium/Low`
- **Anti-Drift Anchor**: Update to reference isolated memory

### v-build.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/v-build.mem.md (isolated memory)`
- **Workflow**: Replace "No Memory Write" step with "Write Isolated Memory". V-Build uses `Highest Severity: PASS/FAIL` (binary gate)
- **Anti-Drift Anchor**: Update to reference isolated memory. Remove any V Aggregator references

### v-tests.agent.md

- **Outputs**: Add `docs/feature/<feature-slug>/memory/v-tests.mem.md (isolated memory)`
- **Workflow**: Replace "No Memory Write" step with "Write Isolated Memory". V-Tests uses `Highest Severity: PASS/FAIL`. Status includes DONE/NEEDS_REVISION/ERROR
- **Anti-Drift Anchor**: Update to reference isolated memory. Remove any V Aggregator references

## Out-of-Scope

- v-tasks.agent.md, v-feature.agent.md (task 13)
- Orchestrator V decision logic (task 03)

## Acceptance Criteria

- AC-5: All 3 files list isolated memory in Outputs and have "Write Isolated Memory" workflow step
- AC-16: All 3 Anti-Drift Anchors reference isolated memory, no aggregator references
- AC-19: Section ordering preserved
- AC-20: Completion contracts unchanged
- CT-strategy uses CT taxonomy (Critical/High/Medium/Low)
- V-build and V-tests use V taxonomy (PASS/FAIL)

## Estimated Effort

Low

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `ct-strategy.agent.md` relevant sections, apply CT sub-agent changes (same pattern as task 11)
2. Read `v-build.agent.md` relevant sections. Find "No Memory Write" step (line ~117). Replace with "Write Isolated Memory" with V-Build template (PASS/FAIL). Add memory to Outputs. Update Anti-Drift
3. Read `v-tests.agent.md` relevant sections. Find "No Memory Write" step (line ~146). Replace with "Write Isolated Memory" with V-Tests template (PASS/FAIL, DONE/NEEDS_REVISION/ERROR status). Add memory to Outputs. Update Anti-Drift
4. Remove any V Aggregator references from v-build and v-tests Anti-Drift Anchors
5. Verify consistency across all 3 files

## Completion Checklist

- [x] ct-strategy.agent.md: Outputs include `memory/ct-strategy.mem.md`
- [x] ct-strategy.agent.md: "Write Isolated Memory" step with CT taxonomy
- [x] ct-strategy.agent.md: Anti-Drift references isolated memory
- [x] v-build.agent.md: Outputs include `memory/v-build.mem.md`
- [x] v-build.agent.md: "No Memory Write" replaced with "Write Isolated Memory" (PASS/FAIL)
- [x] v-build.agent.md: Anti-Drift references isolated memory, no V Aggregator
- [x] v-tests.agent.md: Outputs include `memory/v-tests.mem.md`
- [x] v-tests.agent.md: "No Memory Write" replaced with "Write Isolated Memory" (PASS/FAIL)
- [x] v-tests.agent.md: Anti-Drift references isolated memory, no V Aggregator
- [x] No aggregator references in any file
- [x] Section ordering preserved
- [x] Completion contracts unchanged
