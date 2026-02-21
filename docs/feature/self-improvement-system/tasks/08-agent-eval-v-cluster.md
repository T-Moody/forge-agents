# Task 08: Agent Evaluation — V-Tests, V-Tasks, V-Feature

**Task Goal:** Add artifact evaluation output and workflow step to `v-tests.agent.md`, `v-tasks.agent.md`, and `v-feature.agent.md`.

**depends_on:** 01

**agent:** implementer

---

## In-Scope

- Modify `.github/agents/v-tests.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/v-tests.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation target: `verification/v-build.md`

- Modify `.github/agents/v-tasks.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/v-tasks.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `verification/v-build.md`, `plan.md`, `tasks/*.md`

- Modify `.github/agents/v-feature.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/v-feature.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `verification/v-build.md`, `feature.md`

## Out-of-Scope

- Modifying v-build.agent.md (excluded per AC-3 — evaluates codebase only)
- Modifying other agent files
- Creating the evaluation schema (Task 01)
- Changing existing workflow steps, completion contracts, or inputs

---

## Acceptance Criteria

1. Each of the 3 V-cluster agent files has a new Outputs entry for `artifact-evaluations/v-<type>.md`
2. Each has a new numbered workflow step "Evaluate Upstream Artifacts"
3. v-tests evaluates: `verification/v-build.md`
4. v-tasks evaluates: `verification/v-build.md`, `plan.md`, `tasks/*.md`
5. v-feature evaluates: `verification/v-build.md`, `feature.md`
6. Workflow step references `.github/agents/evaluation-schema.md`
7. Non-blocking rules included
8. Template wording consistent with other evaluation tasks
9. No existing content modified or removed
10. v-build.agent.md is NOT modified

---

## Estimated Effort

Low (~40 lines per agent × 3 = ~120 lines total)

---

## Test Requirements (TDD Fallback — Configuration-Only)

For each of the 3 agent files:

1. **Verify Outputs addition:** grep for `artifact-evaluations/v-<type>.md`
2. **Verify workflow step:** grep for "Evaluate Upstream Artifacts"
3. **Verify schema reference:** grep for `evaluation-schema.md`
4. **Verify evaluation targets:** per-agent correct artifacts
5. **Verify v-build unchanged:** diff v-build.agent.md shows no changes

---

## Implementation Steps

1. Read design.md §Evaluating Agent Changes and §Per-Agent Evaluation Targets
2. Verify `.github/agents/evaluation-schema.md` exists (Task 01 dependency)
3. For each of the 3 V-cluster agent files:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output entry
   c. Insert evaluation workflow step with agent-specific targets
4. Verify v-build.agent.md was NOT modified
5. Verify consistent template with other evaluation tasks

---

## Completion Checklist

- [x] v-tests.agent.md: output and step added (target: verification/v-build.md)
- [x] v-tasks.agent.md: output and step added (targets: verification/v-build.md, plan.md, tasks/\*.md)
- [x] v-feature.agent.md: output and step added (targets: verification/v-build.md, feature.md)
- [x] v-build.agent.md NOT modified
- [x] Schema referenced by path
- [x] Non-blocking rule included
- [x] Template consistent with other evaluation tasks
- [x] No existing content modified or removed

## Implementation Notes

- TDD skipped: configuration-only task (markdown agent definition changes, no behavioral code)
- All changes are purely additive — no existing content was modified or removed
- Evaluation step template follows design.md §Evaluating Agent Changes exactly
