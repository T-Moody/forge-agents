# Task 04: Feature Spec Phase 1 Scope Annotation

## Task Goal

Add a Phase 1 scope section to `feature.md` that identifies which acceptance criteria are in-scope for Phase 1 and which are deferred, incorporating CT-Strategy's finding that AC-13 and AC-14 must be deferred.

## depends_on

none

## agent

implementer

## In-scope

- Add a "Phase 1 Scope" section to `feature.md` (before or after the Acceptance Criteria section)
- Classify 5 ACs as Phase 1 in-scope: AC-1, AC-2, AC-3, AC-11, AC-12
- Classify AC-13, AC-14 as deferred with rationale: pass/fail criteria require Phase 2 behavior (memory.md elimination)
- Classify AC-4 through AC-10 and AC-15 as deferred to Phase 2 (memory architecture redesign)
- Provide Phase 1 interpretation for AC-12: Anti-Drift updated for tool restriction and memory disambiguation; memory.md references preserved (memory.md is not eliminated in Phase 1)
- Reference design §16.3 and §16.5 as the source for this scoping decision

## Out-of-scope

- Modifying existing AC definitions (the original AC text remains unchanged)
- Changes to `orchestrator.agent.md` (Tasks 01, 02)
- Changes to `feature-workflow.prompt.md` (Task 03)
- Any Phase 2 implementation work

## Acceptance Criteria

1. `feature.md` contains a clearly labeled Phase 1 scope section
2. Exactly 5 ACs are listed as Phase 1 in-scope: AC-1, AC-2, AC-3, AC-11, AC-12
3. AC-13 and AC-14 are explicitly listed as deferred with rationale referencing Phase 2 behavior in their pass/fail criteria
4. AC-4 through AC-10 and AC-15 are listed as deferred to Phase 2
5. AC-12 has a Phase 1 interpretation note explaining that "does not mention memory.md" applies to Phase 2 only; Phase 1 verifies tool restriction updates and memory disambiguation in the Anti-Drift Anchor
6. Original AC definitions are not modified

## Test Requirements

After applying the edit, verify:
- The Phase 1 scope section exists and is well-formatted
- Count of in-scope ACs equals 5
- Count of deferred ACs equals 10 (AC-4 through AC-10, AC-13, AC-14, AC-15)
- AC-12 Phase 1 interpretation note is present
- All original AC text is preserved

## Implementation Steps

1. Read the Acceptance Criteria section of `feature.md` (around L171-250) to understand structure
2. Add a new section "## Phase 1 Scope" after the Acceptance Criteria section (or as a subsection within it)
3. Create an in-scope / deferred table with rationale for each AC classification
4. Add AC-12 Phase 1 interpretation note
5. Reference design §16.3 and §16.5 as source
6. Verify original AC definitions are unchanged

**Reference:** Design §16.3 (AC mapping table) and §16.5 (scope note) in `docs/feature/orchestrator-tool-restriction/design.md`. CT-Strategy finding on AC-13/AC-14 in `docs/feature/orchestrator-tool-restriction/memory/ct-strategy.mem.md`.

## Estimated Effort

Low

## Completion Checklist

- [x] Phase 1 scope section added to `feature.md`
- [x] 5 ACs classified as in-scope (AC-1, AC-2, AC-3, AC-11, AC-12)
- [x] AC-13, AC-14 deferred with Phase 2 behavior rationale
- [x] AC-4–AC-10, AC-15 deferred to Phase 2
- [x] AC-12 Phase 1 interpretation note present
- [x] Original AC definitions preserved
- [x] Design §16.3/§16.5 and CT-Strategy referenced

TDD skipped: documentation-only task (no behavioral code changes).
