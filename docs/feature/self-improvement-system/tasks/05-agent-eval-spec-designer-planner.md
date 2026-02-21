# Task 05: Agent Evaluation — Spec, Designer, Planner

**Task Goal:** Add artifact evaluation output and workflow step to `spec.agent.md`, `designer.agent.md`, and `planner.agent.md`.

**depends_on:** 01

**agent:** implementer

---

## In-Scope

- Modify `.github/agents/spec.agent.md`:
  - Add evaluation output to Outputs section: `artifact-evaluations/spec.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md`

- Modify `.github/agents/designer.agent.md`:
  - Add evaluation output to Outputs section: `artifact-evaluations/designer.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation target: `feature.md`

- Modify `.github/agents/planner.agent.md`:
  - Add evaluation output to Outputs section: `artifact-evaluations/planner.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation targets: `design.md`, `feature.md`

## Out-of-Scope

- Modifying any other agent files
- Creating the evaluation schema document (Task 01)
- Modifying the orchestrator
- Changing any existing workflow steps, completion contracts, or inputs

---

## Acceptance Criteria

1. Each of the 3 agent files has a new entry in its Outputs section for the evaluation file path (marked as secondary, non-blocking)
2. Each has a new numbered workflow step titled "Evaluate Upstream Artifacts" inserted after primary work completion and before self-verification (or memory writing if no self-verification)
3. The workflow step references `.github/agents/evaluation-schema.md` (not inlined schema)
4. The workflow step includes: rules about non-blocking behavior, evaluation failure not causing ERROR, evaluation being secondary to primary output
5. Each agent's evaluation targets match the per-agent table in design.md §Per-Agent Evaluation Targets
6. The workflow step text is consistent across all 3 agents (same template, agent-specific file paths only)
7. No existing Outputs entries modified or removed
8. No existing workflow steps renumbered — new step uses the next available number
9. Existing completion contracts unchanged

---

## Estimated Effort

Low (~40 lines added per agent × 3 = ~120 lines total)

---

## Test Requirements (TDD Fallback — Configuration-Only)

For each of the 3 agent files:

1. **Verify Outputs addition:** grep for `artifact-evaluations/<agent-name>.md` in Outputs section
2. **Verify workflow step:** grep for "Evaluate Upstream Artifacts" in Workflow section
3. **Verify schema reference:** grep for `evaluation-schema.md` in the new workflow step
4. **Verify non-blocking rule:** grep for "MUST NOT cause" or "must not cause" near "ERROR" in the evaluation step
5. **Verify evaluation targets:** grep for correct source artifact paths per agent
6. **Verify no existing changes:** existing Outputs entries and workflow steps still present and unmodified

---

## Implementation Steps

1. Read design.md §Evaluating Agent Changes for the exact workflow step template text
2. Read design.md §Per-Agent Evaluation Targets for per-agent source artifacts
3. Verify `.github/agents/evaluation-schema.md` exists (dependency on Task 01)
4. For `spec.agent.md`:
   a. Read current file, locate Outputs section and Workflow section
   b. Add evaluation output entry to Outputs
   c. Determine correct step number (after primary work, before self-verification/memory)
   d. Insert evaluation workflow step with targets: research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md
5. For `designer.agent.md`:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output entry
   c. Insert evaluation workflow step with target: feature.md
6. For `planner.agent.md`:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output entry
   c. Insert evaluation workflow step with targets: design.md, feature.md
7. Verify all 3 files have consistent evaluation step wording (same template)

---

## Completion Checklist

- [x] spec.agent.md: evaluation output added, workflow step added, targets correct (4 research files)
- [x] designer.agent.md: evaluation output added, workflow step added, target correct (feature.md)
- [x] planner.agent.md: evaluation output added, workflow step added, targets correct (design.md, feature.md)
- [x] All 3 use identical template wording (agent-specific paths differ)
- [x] Schema referenced by path, not inlined
- [x] Non-blocking rule included in each
- [x] No existing content modified or removed
- [x] Existing completion contracts unchanged

TDD skipped: agent definition markdown file modification
