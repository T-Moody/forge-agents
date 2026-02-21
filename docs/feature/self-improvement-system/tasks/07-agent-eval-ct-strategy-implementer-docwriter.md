# Task 07: Agent Evaluation — CT-Strategy, Implementer, Documentation-Writer

**Task Goal:** Add artifact evaluation output and workflow step to `ct-strategy.agent.md`, `implementer.agent.md`, and `documentation-writer.agent.md`.

**depends_on:** 01

**agent:** implementer

---

## In-Scope

- Modify `.github/agents/ct-strategy.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/ct-strategy.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `design.md`, `feature.md`

- Modify `.github/agents/implementer.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/implementer-<task-id>.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `tasks/<task>.md`, `design.md`, `feature.md`
  - **Note:** Uses task-ID naming pattern for evaluation file

- Modify `.github/agents/documentation-writer.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/documentation-writer-<task-id>.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `tasks/<task>.md`, `design.md`, `feature.md`
  - **Note:** Uses task-ID naming pattern for evaluation file

## Out-of-Scope

- Modifying other CT agents (Tasks 06)
- Modifying other agent files
- Creating the evaluation schema (Task 01)
- Changing existing workflow steps, completion contracts, or inputs

---

## Acceptance Criteria

1. ct-strategy.agent.md: Outputs entry for `artifact-evaluations/ct-strategy.md`, evaluation workflow step with targets `design.md` and `feature.md`
2. implementer.agent.md: Outputs entry for `artifact-evaluations/implementer-<task-id>.md`, evaluation workflow step with targets `tasks/<task>.md`, `design.md`, `feature.md`
3. documentation-writer.agent.md: Outputs entry for `artifact-evaluations/documentation-writer-<task-id>.md`, evaluation workflow step with targets `tasks/<task>.md`, `design.md`, `feature.md`
4. Implementer and documentation-writer use task-ID naming: `<agent>-<task-id>.md`
5. Workflow step references `.github/agents/evaluation-schema.md`
6. Non-blocking rules included (evaluation failure ≠ ERROR)
7. Template wording consistent with Tasks 05, 06, 08, 09
8. No existing content modified or removed
9. Existing completion contracts unchanged

---

## Estimated Effort

Low (~40 lines per agent × 3 = ~120 lines total)

---

## Test Requirements (TDD Fallback — Configuration-Only)

For each of the 3 agent files:

1. **Verify Outputs addition:** grep for `artifact-evaluations/` with correct naming pattern
2. **Verify workflow step:** grep for "Evaluate Upstream Artifacts"
3. **Verify schema reference:** grep for `evaluation-schema.md`
4. **Verify evaluation targets:** per-agent correct artifacts
5. **Verify task-ID naming** (implementer and documentation-writer): grep for `<task-id>` in Outputs

---

## Implementation Steps

1. Read design.md §Evaluating Agent Changes and §Per-Agent Evaluation Targets
2. Verify `.github/agents/evaluation-schema.md` exists (Task 01 dependency)
3. For `ct-strategy.agent.md`:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output and workflow step (targets: design.md, feature.md)
4. For `implementer.agent.md`:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output with task-ID naming: `implementer-<task-id>.md`
   c. Add evaluation workflow step (targets: tasks/<task>.md, design.md, feature.md)
5. For `documentation-writer.agent.md`:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output with task-ID naming: `documentation-writer-<task-id>.md`
   c. Add evaluation workflow step (targets: tasks/<task>.md, design.md, feature.md)
6. Verify consistent template wording across all 3 files

---

## Completion Checklist

- [x] ct-strategy.agent.md: evaluation output and workflow step added (targets: design.md, feature.md)
- [x] implementer.agent.md: evaluation output with task-ID naming, workflow step added (targets: task file, design.md, feature.md)
- [x] documentation-writer.agent.md: evaluation output with task-ID naming, workflow step added (targets: task file, design.md, feature.md)
- [x] Task-ID naming pattern correct for implementer and documentation-writer
- [x] Schema referenced by path, not inlined
- [x] Non-blocking rule included
- [x] Template consistent with other evaluation tasks
- [x] No existing content modified or removed
