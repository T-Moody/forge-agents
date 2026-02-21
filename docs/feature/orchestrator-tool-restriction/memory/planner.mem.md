# Memory: planner

## Status

DONE: 4 tasks created, 2 waves. Phase 1 scoped to 5 ACs (AC-1, AC-2, AC-3, AC-11, AC-12). AC-13/AC-14 deferred per CT-Strategy. feature-workflow.prompt.md change made required per CT-Maintainability.

## Key Findings

- Phase 1 is a 2-production-file change (orchestrator.agent.md + feature-workflow.prompt.md) plus 1 spec annotation (feature.md) — 4 tasks total, low overall risk
- Tasks 01 and 02 both edit orchestrator.agent.md so must be sequenced (Wave 1 → Wave 2) to avoid concurrent-edit conflicts; Tasks 03 and 04 edit separate files and run in parallel in Wave 1
- CT-Strategy's AC-13/AC-14 deferral reduces Phase 1 from 7 ACs to 5 ACs — these two ACs have pass/fail criteria requiring memory.md elimination (Phase 2 behavior)
- AC-12's feature.md pass criteria ("does not mention memory.md") conflicts with Phase 1 scope (memory.md preserved) — Task 04 adds Phase 1 interpretation note
- Design §3.1–§3.9 provides exact current/new text for all edit points, making implementation low-risk despite 13+ edit locations in orchestrator.agent.md

## Highest Severity

N/A

## Decisions Made

- **Task grouping by concern:** Task 01 groups all tool-restriction edits (YAML + 4 prose sections including Anti-Drift); Task 02 groups all memory-tool contradiction fixes + merge wording corrections. Rationale: logical cohesion — Task 01 is "what we're restricting", Task 02 is "fixing pre-existing bugs exposed by the restriction."
- **Same-file sequencing over parallel:** Tasks 01→02 are sequential despite being logically independent, to avoid concurrent-edit conflicts on orchestrator.agent.md. Accepted shallow chain (depth 2) as acceptable tradeoff.
- **feature-workflow.prompt.md as required task:** Per CT-Maintainability constraint, overriding design §4.3's "Optional" label. Prevents documentation drift.
- **5 ACs not 7:** Adopted CT-Strategy recommendation to defer AC-13/AC-14 whose pass/fail criteria describe Phase 2 behavior.

## Artifact Index

- [plan.md](../plan.md)
  - §Feature Overview — Phase 1 scope summary
  - §Success Criteria — 5 ACs mapped to design sections
  - §Execution Waves — 2 waves, dependency graph
  - §Pre-Mortem Analysis — per-task failure scenarios, overall risk Low
- [tasks/01-tool-restriction-prose.md](../tasks/01-tool-restriction-prose.md)
  - §Implementation Steps — YAML frontmatter, Global Rule 1, Operating Rules 1/5, Anti-Drift Anchor
  - §Acceptance Criteria — AC-1, AC-2, AC-3 (partial), AC-11 (partial), AC-12
- [tasks/02-memory-disambiguation-merge-wording.md](../tasks/02-memory-disambiguation-merge-wording.md)
  - §Implementation Steps — Global Rule 12, Step 0.1, Step 8.2, Memory Lifecycle Table, 8 merge steps
  - §Acceptance Criteria — AC-11 (completion), merge step verification
- [tasks/03-feature-workflow-update.md](../tasks/03-feature-workflow-update.md)
  - §Implementation Steps — Add tool restriction note to Rules section
- [tasks/04-feature-spec-phase1-scope.md](../tasks/04-feature-spec-phase1-scope.md)
  - §Implementation Steps — Phase 1 scope section with AC classification and AC-12 interpretation note
