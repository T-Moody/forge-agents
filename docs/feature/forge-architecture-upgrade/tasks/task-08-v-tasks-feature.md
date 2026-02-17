# Task 08: V Sub-Agents — Tasks + Feature

## Status

completed

## Agent

implementer

## Depends On

none

## Description

Create two new Verification cluster sub-agent files: `v-tasks.agent.md` (per-task acceptance criteria verification) and `v-feature.agent.md` (feature-level acceptance criteria verification). Both run after V-Build passes. Both read `memory.md` but do NOT write to it.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §4.1 (V Sub-Agent Definitions), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — V-AC-1 through V-AC-10
- `NewAgentsAndPrompts/verifier.agent.md` — current file as structural reference (Steps 4-5)

## Output File

- `NewAgentsAndPrompts/v-tasks.agent.md`
- `NewAgentsAndPrompts/v-feature.agent.md`

## Acceptance Criteria

1. Both files follow v2 template structure
2. **v-tasks** inherits verifier Step 4 (Task-Level Verification): reads `plan.md`, `tasks/*.md`, verifies per-task acceptance criteria
3. **v-tasks** outputs to `docs/feature/<feature-slug>/verification/v-tasks.md` containing: per-task status (verified/partially-verified/failed), acceptance criteria checklists, specific failing task IDs
4. **v-tasks** includes task IDs in output — critical for targeted replan by the planner
5. **v-feature** inherits verifier Step 5 (Feature-Level Verification): reads `feature.md`, verifies feature-level acceptance criteria
6. **v-feature** outputs to `docs/feature/<feature-slug>/verification/v-feature.md` containing: feature acceptance criteria status, regression check results, overall readiness
7. Both have completion contract: `DONE:` / `NEEDS_REVISION:` / `ERROR:`
8. Both read `v-build.md` for build context
9. Both include read-only enforcement (MUST NOT modify source code)
10. Both do NOT write to `memory.md`

## Implementation Guidance

### v-tasks.agent.md

```
---
name: v-tasks
description: "Per-task acceptance criteria verification for the Verification cluster."
---
```

**Role:** Verifies each completed task's acceptance criteria against the implemented code. Identifies which specific tasks pass or fail — critical for targeted replanning.

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/verification/v-build.md`
- `docs/feature/<feature-slug>/initial-request.md`
- `docs/feature/<feature-slug>/plan.md`
- `docs/feature/<feature-slug>/tasks/*.md`
- Entire codebase

**Outputs:**

- `docs/feature/<feature-slug>/verification/v-tasks.md`

**Key workflow (from verifier Step 4):**

1. Read `memory.md`
2. Read `v-build.md` for build context
3. Read `initial-request.md` to verify alignment with original request
4. For each task file in `tasks/*.md`:
   - Verify code changes match task acceptance criteria
   - Verify required tests were written
   - Cross-reference test results with task test requirements
   - Mark task as verified / partially-verified / failed
5. Write `v-tasks.md` with per-task verification status and specific failing task IDs

**Critical requirement:** The output MUST include specific task IDs for any failures, so the planner can create targeted fix tasks during replan.

### v-feature.agent.md

```
---
name: v-feature
description: "Feature-level acceptance criteria verification for the Verification cluster."
---
```

**Role:** Verifies feature-level acceptance criteria from `feature.md` against the combined implementation. Checks for regressions.

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/verification/v-build.md`
- `docs/feature/<feature-slug>/feature.md`
- `docs/feature/<feature-slug>/design.md`
- Entire codebase

**Outputs:**

- `docs/feature/<feature-slug>/verification/v-feature.md`

**Key workflow (from verifier Step 5):**

1. Read `memory.md`
2. Read `v-build.md` for build context
3. Verify feature-level acceptance criteria from `feature.md`
4. Check for regressions outside feature scope
5. Write `v-feature.md` with acceptance criteria status and readiness assessment

**Note:** These are markdown agent definition files — TDD does not apply.

## Completion Checklist

- [x] AC1: Both files follow v2 template structure (YAML frontmatter → Role → Inputs → Outputs → Operating Rules → Workflow → Output spec → Completion Contract → Anti-Drift Anchor)
- [x] AC2: v-tasks inherits verifier Step 4 — reads plan.md, tasks/\*.md, verifies per-task acceptance criteria
- [x] AC3: v-tasks outputs to verification/v-tasks.md with per-task status, acceptance criteria checklists, specific failing task IDs
- [x] AC4: v-tasks includes task IDs in output (Failing Task IDs section) — critical for targeted replan
- [x] AC5: v-feature inherits verifier Step 5 — reads feature.md, verifies feature-level acceptance criteria
- [x] AC6: v-feature outputs to verification/v-feature.md with feature acceptance criteria status, regression check results, overall readiness
- [x] AC7: Both have completion contract: DONE / NEEDS_REVISION / ERROR
- [x] AC8: Both read v-build.md for build context
- [x] AC9: Both include read-only enforcement (MUST NOT modify source code)
- [x] AC10: Both do NOT write to memory.md (read-only memory protocol enforced)

TDD skipped: markdown agent definition files — no behavioral code changes.
