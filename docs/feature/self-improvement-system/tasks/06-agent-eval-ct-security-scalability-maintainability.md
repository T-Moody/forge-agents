# Task 06: Agent Evaluation — CT-Security, CT-Scalability, CT-Maintainability

**Task Goal:** Add artifact evaluation output and workflow step to `ct-security.agent.md`, `ct-scalability.agent.md`, and `ct-maintainability.agent.md`.

**depends_on:** 01

**agent:** implementer

---

## In-Scope

- Modify `.github/agents/ct-security.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/ct-security.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `design.md`, `feature.md`

- Modify `.github/agents/ct-scalability.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/ct-scalability.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `design.md`, `feature.md`

- Modify `.github/agents/ct-maintainability.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/ct-maintainability.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `design.md`, `feature.md`

## Out-of-Scope

- Modifying ct-strategy.agent.md (that's Task 07)
- Modifying any non-CT agent files
- Creating the evaluation schema document (Task 01)
- Changing existing workflow steps, completion contracts, or inputs

---

## Acceptance Criteria

1. Each of the 3 CT agent files has a new Outputs entry for `artifact-evaluations/ct-<type>.md` (secondary, non-blocking)
2. Each has a new numbered workflow step "Evaluate Upstream Artifacts" after primary work, before self-verification or memory writing
3. Workflow step references `.github/agents/evaluation-schema.md`
4. Workflow step includes non-blocking rules (evaluation failure ≠ ERROR, secondary to primary output)
5. Evaluation targets for all 3: `design.md`, `feature.md` (per design.md §Per-Agent Evaluation Targets)
6. Template wording consistent with Tasks 05, 07, 08, 09
7. No existing content modified or removed
8. Existing completion contracts unchanged

---

## Estimated Effort

Low (~40 lines per agent × 3 = ~120 lines total)

---

## Test Requirements (TDD Fallback — Configuration-Only)

For each of the 3 agent files:

1. **Verify Outputs addition:** grep for `artifact-evaluations/ct-<type>.md`
2. **Verify workflow step:** grep for "Evaluate Upstream Artifacts"
3. **Verify schema reference:** grep for `evaluation-schema.md`
4. **Verify evaluation targets:** grep for `design.md` and `feature.md` in evaluation step
5. **Verify non-blocking:** grep for "MUST NOT cause" near "ERROR"

---

## Implementation Steps

1. Read design.md §Evaluating Agent Changes for the workflow step template
2. Read design.md §Per-Agent Evaluation Targets — all 3 CT agents evaluate `design.md` and `feature.md`
3. Verify `.github/agents/evaluation-schema.md` exists (Task 01 dependency)
4. For each of the 3 CT agent files:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output entry to Outputs
   c. Determine correct step number for the evaluation step
   d. Insert evaluation workflow step with targets: design.md, feature.md
5. Verify consistent wording across all 3 files and with Task 05's template

---

## Completion Checklist

- [x] ct-security.agent.md: evaluation output added, workflow step added
- [x] ct-scalability.agent.md: evaluation output added, workflow step added
- [x] ct-maintainability.agent.md: evaluation output added, workflow step added
- [x] All 3 evaluate design.md and feature.md
- [x] Schema referenced by path, not inlined
- [x] Non-blocking rule included
- [x] Template consistent with other evaluation tasks
- [x] No existing content modified or removed

TDD skipped: configuration-only Markdown task (no behavioral code changes)
