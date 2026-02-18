# Task 08: Planner — Replan Mode, Cross-Referencing, Isolated Memory

## Task Goal

Update `planner.agent.md` to use memory-first inputs, replace `verifier.md`-based replan detection with the new `MODE: REPLAN` signal, add the cross-referencing workflow for replan mode, and add isolated memory output.

## depends_on

none

## agent

implementer

## In-Scope

- **Inputs (normal mode)**: Remove `analysis.md`. Add primary: `memory/designer.mem.md`, `memory/spec.mem.md`. Add selective: `design.md`, `feature.md`, `research/*.md` (read sections per artifact indexes). Add (if present): `memory/ct-*.mem.md` → selective `ct-review/ct-*.md` reads (planning constraints from CT findings)
- **Inputs (replan mode)**: Remove `verifier.md`. Add primary: `memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md`. Add selective: `verification/v-tasks.md`, `verification/v-tests.md`, `verification/v-feature.md` (read sections per V memory artifact indexes). Remove `design_critical_review.md`
- **Outputs**: Add `docs/feature/<feature-slug>/memory/planner.mem.md`
- **Replan Mode Detection**: Replace "`verifier.md` exists with failures" with two-part mechanism:
  1. Primary: orchestrator provides `MODE: REPLAN` signal with V memory paths
  2. Secondary (fallback): `verification/v-tasks.md` exists with failing task IDs
- **Replan Cross-Referencing Workflow**: Add new subsection with 6-step procedure: 0. Read V sub-agent memories → review key findings and artifact indexes → identify failure sections
  1. Read targeted sections of `verification/v-tasks.md` → extract failing task IDs and reasons
  2. Read targeted sections of `verification/v-tests.md` → match test failures to task IDs
  3. Read targeted sections of `verification/v-feature.md` → match unmet criteria to task IDs
  4. Prioritize tasks (both test failures AND unmet criteria → highest; test only → medium; criteria only → lower)
  5. Produce remediation plan targeting prioritized task list; do NOT re-plan completed tasks
- **Workflow (normal mode)**: Add memory-first reading workflow: read designer + spec memories → artifact indexes → targeted reads. Replace "Update `memory.md`" with "Write Isolated Memory"
- **Anti-Drift Anchor**: Update to reference isolated memory, remove any aggregator references

## Out-of-Scope

- Orchestrator changes for MODE: REPLAN dispatch (task 05 handles Step 6.4)
- Other agent files

## Acceptance Criteria

- AC-3: No `analysis.md`, `design_critical_review.md`, or `verifier.md` (as artifact path) references
- AC-5: Outputs list `memory/planner.mem.md`; workflow has "Write Isolated Memory" step
- AC-10: Inputs list upstream memory files as primary + artifact files for selective reads
- AC-12: Replan mode references individual V memories and V artifacts (not `verifier.md`)
- AC-16: Anti-Drift Anchor references isolated memory, no aggregator references
- AC-19: Section ordering preserved
- AC-20: Completion contract format unchanged
- Replan mode detection uses `MODE: REPLAN` signal (primary) + V artifact fallback (secondary)
- Cross-referencing workflow includes all 6 steps (0–5) from design.md

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `planner.agent.md` fully to identify Inputs, Outputs, Workflow, Mode Detection, Anti-Drift Anchor
2. **Inputs (normal)**: Remove `analysis.md`. Add designer + spec memory files as primary. Add `design.md`, `feature.md`, `research/*.md` for selective reads. Add CT memories if present
3. **Inputs (replan)**: Remove `verifier.md`. Add V memory files as primary. Add `verification/v-tasks.md`, `verification/v-tests.md`, `verification/v-feature.md` for selective reads. Remove `design_critical_review.md`
4. **Outputs**: Add `docs/feature/<feature-slug>/memory/planner.mem.md (isolated memory)`
5. **Mode Detection** (around line 59): Replace "`verifier.md` exists with failures" with:
   - "Replan mode: orchestrator provides `MODE: REPLAN` signal with V memory paths (`memory/v-*.mem.md`)"
   - "Fallback: `verification/v-tasks.md` exists with failing task IDs (non-empty Failures section)"
6. **Workflow (normal)**: Add memory-first reading step at beginning of workflow per design.md
7. **Workflow (replan)**: Add "Replan Cross-Referencing Steps" subsection with the full 6-step procedure from design.md
8. Replace "Update `memory.md`" workflow step with "Write Isolated Memory" (Status, Key Findings, Highest Severity: N/A, Decisions Made, Artifact Index with §Section pointers)
9. **Anti-Drift Anchor**: Update to "You write only to your isolated memory file (`memory/planner.mem.md`), never to shared `memory.md`." Remove any aggregator references
10. Verify no `analysis.md`, `design_critical_review.md`, `verifier.md` (as artifact), or aggregator references remain

## Completion Checklist

- [x] No `analysis.md` references
- [x] No `design_critical_review.md` references
- [x] No `verifier.md` (as artifact path) references
- [x] No aggregator references
- [x] Inputs (normal): designer + spec memories as primary, selective artifact reads
- [x] Inputs (replan): V memories as primary, selective V artifact reads
- [x] Mode detection: `MODE: REPLAN` signal (primary) + fallback
- [x] Cross-referencing workflow: all 6 steps (0–5) present
- [x] Outputs: `memory/planner.mem.md` listed
- [x] "Write Isolated Memory" workflow step present
- [x] Anti-Drift Anchor references isolated memory
- [x] Section ordering preserved
- [x] Completion contract format unchanged
