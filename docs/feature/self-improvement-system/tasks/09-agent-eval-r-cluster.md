# Task 09: Agent Evaluation — R-Quality, R-Testing

**Task Goal:** Add artifact evaluation output and workflow step to `r-quality.agent.md` and `r-testing.agent.md`.

**depends_on:** 01

**agent:** implementer

---

## In-Scope

- Modify `.github/agents/r-quality.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/r-quality.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation target: `design.md`

- Modify `.github/agents/r-testing.agent.md`:
  - Add evaluation output to Outputs: `artifact-evaluations/r-testing.md`
  - Add evaluation workflow step referencing `evaluation-schema.md`
  - Evaluation target: `feature.md`

## Out-of-Scope

- Modifying r-security.agent.md (excluded per AC-3 — consumes only git diff/codebase)
- Modifying r-knowledge.agent.md (excluded per AC-3/AC-12)
- Modifying other agent files
- Creating the evaluation schema (Task 01)

---

## Acceptance Criteria

1. r-quality.agent.md has a new Outputs entry for `artifact-evaluations/r-quality.md`
2. r-quality.agent.md has a new workflow step evaluating `design.md`
3. r-testing.agent.md has a new Outputs entry for `artifact-evaluations/r-testing.md`
4. r-testing.agent.md has a new workflow step evaluating `feature.md`
5. Workflow steps reference `.github/agents/evaluation-schema.md`
6. Non-blocking rules included
7. Template wording consistent with other evaluation tasks
8. No existing content modified or removed
9. r-security.agent.md and r-knowledge.agent.md are NOT modified

---

## Estimated Effort

Low (~40 lines per agent × 2 = ~80 lines total)

---

## Test Requirements (TDD Fallback — Configuration-Only)

For each of the 2 agent files:

1. **Verify Outputs addition:** grep for `artifact-evaluations/r-<type>.md`
2. **Verify workflow step:** grep for "Evaluate Upstream Artifacts"
3. **Verify schema reference:** grep for `evaluation-schema.md`
4. **Verify evaluation targets:** r-quality → design.md, r-testing → feature.md
5. **Verify excluded agents unchanged:** r-security.agent.md and r-knowledge.agent.md not modified

---

## Implementation Steps

1. Read design.md §Evaluating Agent Changes and §Per-Agent Evaluation Targets
2. Verify `.github/agents/evaluation-schema.md` exists (Task 01 dependency)
3. For `r-quality.agent.md`:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output and workflow step (target: design.md)
4. For `r-testing.agent.md`:
   a. Read current file, locate Outputs and Workflow sections
   b. Add evaluation output and workflow step (target: feature.md)
5. Verify r-security.agent.md and r-knowledge.agent.md were NOT modified
6. Verify consistent template with other evaluation tasks

---

## Completion Checklist

- [x] r-quality.agent.md: output and step added (target: design.md)
- [x] r-testing.agent.md: output and step added (target: design.md, feature.md)
- [x] r-security.agent.md NOT modified
- [x] r-knowledge.agent.md NOT modified
- [x] Schema referenced by path
- [x] Non-blocking rule included
- [x] Template consistent with other evaluation tasks
- [x] No existing content modified or removed
