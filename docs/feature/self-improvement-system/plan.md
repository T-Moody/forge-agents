# Plan: Self-Improvement System for Forge

**Feature:** self-improvement-system
**Planning Mode:** Initial
**Date:** 2026-02-20

---

## Feature Overview

Add a structured cross-agent evaluation and post-mortem learning system to the Forge 8-step pipeline. Three subsystems are added (Track A) plus one cross-cutting change (Track B):

1. **Artifact Evaluation** — 14 consuming agents gain an evaluation workflow step referencing a shared schema document, producing structured YAML evaluations in `artifact-evaluations/`
2. **Orchestrator Telemetry** — Orchestrator accumulates dispatch metadata in working context, passes to PostMortem at Step 8
3. **PostMortem Analysis** — New non-blocking agent (Step 8) produces quantitative metrics from evaluations and telemetry
4. **Orchestrator Write-Tool Restriction** (Track B, independent) — Remove write/execute tools from orchestrator while retaining read tools

All changes are additive. No existing behaviors, formats, or completion contracts are altered.

---

## Success Criteria

Mapped from feature.md Acceptance Criteria:

| AC    | Description                                                   | Covered By Tasks                  |
| ----- | ------------------------------------------------------------- | --------------------------------- |
| AC-1  | 14 agents have evaluation workflow step and output            | 05, 06, 07, 08, 09                |
| AC-2  | Evaluation schema document exists with correct fields         | 01                                |
| AC-3  | 5 excluded agents unchanged                                   | (implicit — no task targets them) |
| AC-4  | Orchestrator telemetry capture in run-log                     | 10                                |
| AC-5  | Orchestrator tool restriction (revised — keep read tools)     | 03                                |
| AC-6  | PostMortem agent exists and is correct                        | 02                                |
| AC-7  | PostMortem is non-blocking                                    | 02, 10                            |
| AC-8  | PostMortem report schema (no improvement_recommendations)     | 02                                |
| AC-9  | 3 storage directories in orchestrator Documentation Structure | 10                                |
| AC-10 | Append-only storage with sequence suffix collision avoidance  | 01, 02                            |
| AC-11 | No breaking changes to existing agents                        | all tasks                         |
| AC-12 | R-Knowledge / PostMortem boundary maintained                  | 02                                |
| AC-13 | Evaluation is non-blocking (failure ≠ ERROR)                  | 01, 05–09                         |
| AC-14 | Deliverables documentation + prompt update                    | 04, 11                            |

---

## Ordered Task Index

| #   | Task                                                               | Files Touched | Track | AC                      |
| --- | ------------------------------------------------------------------ | ------------- | ----- | ----------------------- |
| 01  | Create shared evaluation schema document                           | 1 new         | A     | AC-2, AC-10, AC-13      |
| 02  | Create PostMortem agent definition                                 | 1 new         | A     | AC-6, AC-7, AC-8, AC-12 |
| 03  | Orchestrator write-tool restriction                                | 1 modified    | B     | AC-5                    |
| 04  | Update feature-workflow prompt                                     | 1 modified    | A     | AC-14                   |
| 05  | Agent evaluation — spec, designer, planner                         | 3 modified    | A     | AC-1, AC-13             |
| 06  | Agent evaluation — ct-security, ct-scalability, ct-maintainability | 3 modified    | A     | AC-1, AC-13             |
| 07  | Agent evaluation — ct-strategy, implementer, documentation-writer  | 3 modified    | A     | AC-1, AC-13             |
| 08  | Agent evaluation — v-tests, v-tasks, v-feature                     | 3 modified    | A     | AC-1, AC-13             |
| 09  | Agent evaluation — r-quality, r-testing                            | 2 modified    | A     | AC-1, AC-13             |
| 10  | Orchestrator Step 8, telemetry, and documentation structure        | 1 modified    | A     | AC-4, AC-7, AC-9        |
| 11  | Summary documentation of all changes                               | 1 new         | A     | AC-14                   |

---

## Execution Waves

### Wave 1 (parallel — no dependencies)

- 01-create-evaluation-schema (depends_on: none)
- 02-create-postmortem-agent (depends_on: none)
- 03-orchestrator-tool-restriction (depends_on: none)
- 04-update-feature-workflow-prompt (depends_on: none)

### Wave 2 (parallel — depends on Wave 1 items)

- 05-agent-eval-spec-designer-planner (depends_on: 01)
- 06-agent-eval-ct-security-scalability-maintainability (depends_on: 01)
- 07-agent-eval-ct-strategy-implementer-docwriter (depends_on: 01)
- 08-agent-eval-v-cluster (depends_on: 01)
- 09-agent-eval-r-cluster (depends_on: 01)
- 10-orchestrator-step8-telemetry (depends_on: 02, 03)

### Wave 3 (sequential — depends on Wave 2)

- 11-summary-documentation (depends_on: 10, 09)

---

## Dependency Graph

```
Wave 1:  [01] ──┬──────────────────────────────────── [02] ────────┐
                 │                                                  │
         [03] ──┼──────────────────────────────────────────────────┤
                 │                                                  │
         [04]   │ (independent)                                    │
                 │                                                  │
Wave 2:  [05] ◄─┤                                                  │
         [06] ◄─┤                                                  │
         [07] ◄─┤                                                  │
         [08] ◄─┤                                                  │
         [09] ◄─┘                                     [10] ◄───────┘
                                                        │
Wave 3:                                               [11] ◄── [09]
```

Key:

- Tasks 05–09 depend on 01 (evaluation schema must exist before agents reference it)
- Task 10 depends on 02 (PostMortem agent contract needed for Step 8) and 03 (both modify orchestrator.agent.md — must be sequenced)
- Task 11 depends on 10 and 09 (sentinel tasks — wave structure ensures all Wave 2 tasks complete first)

---

## Implementation Specification

### New Files to Create

| File                                  | Task | Description                                                           |
| ------------------------------------- | ---- | --------------------------------------------------------------------- |
| `.github/agents/evaluation-schema.md` | 01   | Shared evaluation YAML schema reference document                      |
| `.github/agents/post-mortem.agent.md` | 02   | PostMortem agent definition (quantitative metrics only, non-blocking) |

### Files to Modify

| File                                           | Tasks  | Changes                                                                                                                                                                                                       |
| ---------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.github/agents/orchestrator.agent.md`         | 03, 10 | Track B: tool restriction, Step 0 rewrite, Global Rule 1 (Task 03). Track A: Step 8, telemetry context, doc structure, expectations table, parallel summary, completion contract, anti-drift anchor (Task 10) |
| `.github/prompts/feature-workflow.prompt.md`   | 04     | Add evaluation/metrics/PostMortem references to Key Artifacts and Rules                                                                                                                                       |
| `.github/agents/spec.agent.md`                 | 05     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/designer.agent.md`             | 05     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/planner.agent.md`              | 05     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/ct-security.agent.md`          | 06     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/ct-scalability.agent.md`       | 06     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/ct-maintainability.agent.md`   | 06     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/ct-strategy.agent.md`          | 07     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/implementer.agent.md`          | 07     | Add evaluation output + workflow step (task-ID naming)                                                                                                                                                        |
| `.github/agents/documentation-writer.agent.md` | 07     | Add evaluation output + workflow step (task-ID naming)                                                                                                                                                        |
| `.github/agents/v-tests.agent.md`              | 08     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/v-tasks.agent.md`              | 08     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/v-feature.agent.md`            | 08     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/r-quality.agent.md`            | 09     | Add evaluation output + workflow step                                                                                                                                                                         |
| `.github/agents/r-testing.agent.md`            | 09     | Add evaluation output + workflow step                                                                                                                                                                         |

### Files NOT Modified (Safety — AC-3, AC-11)

| File                                       | Reason                               |
| ------------------------------------------ | ------------------------------------ |
| `.github/agents/researcher.agent.md`       | No upstream agent artifacts consumed |
| `.github/agents/v-build.agent.md`          | Evaluates codebase only              |
| `.github/agents/r-security.agent.md`       | Consumes only git diff/codebase      |
| `.github/agents/r-knowledge.agent.md`      | Excluded per FR-1.8 / AC-3           |
| `.github/agents/critical-thinker.agent.md` | Deprecated; do not touch             |

---

## Planning Constraints (from CT Cluster)

These Medium-severity findings inform task acceptance criteria:

1. **Spec-design divergence on AC-5:** feature.md says `[agent, runSubagent, memory]` only; design.md keeps read tools. **Plan per design.md** — the design resolution is correct.
2. **YAML validation gap:** No enforcement mechanism for evaluation YAML. Acceptance criteria note YAML blocks must be well-formed examples.
3. **Evaluation collision avoidance:** Agent workflow templates should mention file-existence checks (sequence suffix for NEEDS_REVISION re-executions).
4. **Spec reference FR-3.3 staleness:** FR-3.3 still mentions `improvement_recommendations`. **Ignore it — follow design.md** (quantitative only).
5. **Schema doc not in agent Inputs:** CT-maintainability F-N2 noted agents reference schema by instruction but it's not in their Inputs section. The evaluation workflow step references the schema path — this is sufficient per design.md.
6. **Context telemetry recall accuracy:** CT-scalability F10 notes orchestrator must maintain telemetry across 20+ dispatches. Design accepts this trade-off; no mitigation needed in planning.

---

## Risks & Mitigations

| Risk                                                          | Likelihood | Impact | Mitigation                                                                                                 |
| ------------------------------------------------------------- | ---------- | ------ | ---------------------------------------------------------------------------------------------------------- |
| Orchestrator file too large after Track A + Track B changes   | Medium     | Medium | Track B applied first (Task 03), then Track A (Task 10). Both tasks have clear section boundaries.         |
| Evaluation step wording inconsistency across 14 agents        | Medium     | Low    | Tasks 05–09 all reference the same design.md template text. Acceptance criteria require identical wording. |
| PostMortem agent definition too large for single task         | Low        | Medium | Task 02 creates the full file at ~250 lines — within Medium effort limit.                                  |
| Track B changes conflict with Track A changes in orchestrator | Medium     | Medium | Sequenced via dependency: Task 03 (Track B) first, Task 10 (Track A) second.                               |

---

## Pre-Mortem Analysis

| Task                                | Failure Scenario                                                              | Likelihood | Impact | Mitigation                                                                                |
| ----------------------------------- | ----------------------------------------------------------------------------- | ---------- | ------ | ----------------------------------------------------------------------------------------- |
| 01-create-evaluation-schema         | Schema fields don't match design.md exactly                                   | Low        | High   | Task specifies exact YAML from design.md §Data Models                                     |
| 02-create-postmortem-agent          | PostMortem agent format diverges from standard chatagent pattern              | Medium     | High   | Task references design.md §PostMortem Agent Definition which provides complete agent text |
| 03-orchestrator-tool-restriction    | Tool restriction accidentally removes read tools needed for cluster decisions | Medium     | High   | Task explicitly lists retained vs. removed tools per design.md §Write-Tool Restriction    |
| 04-update-feature-workflow-prompt   | Missing prompt sections or wrong artifact names                               | Low        | Low    | Small, well-defined change; design.md §Feature Workflow Prompt Update provides exact text |
| 05-agent-eval-spec-designer-planner | Evaluation step inserted at wrong position in workflow                        | Medium     | Medium | Task specifies "after primary work, before self-verification or memory writing"           |
| 06-agent-eval-ct-cluster-1          | CT agents have different workflow structure than expected                     | Low        | Medium | CT agents follow standardized format; task includes verification step                     |
| 07-agent-eval-ct-strategy-impl-doc  | Implementer/documentation-writer task-ID naming pattern wrong                 | Medium     | Medium | Task explicitly specifies `implementer-<task-id>.md` naming per design.md                 |
| 08-agent-eval-v-cluster             | V agents have unique workflow patterns (Pattern B/C)                          | Low        | Low    | Evaluation step is additive regardless of dispatch pattern                                |
| 09-agent-eval-r-cluster             | R agents already have review-specific workflows                               | Low        | Low    | Small task (2 agents); evaluation step is additive                                        |
| 10-orchestrator-step8-telemetry     | Step 8 text conflicts with Track B changes from Task 03                       | Medium     | High   | Task 10 depends on Task 03; implementer reads current file state after Track B changes    |
| 11-summary-documentation            | Summary incomplete — misses some changes                                      | Low        | Low    | Task runs last; can enumerate all modified files from plan.md                             |

**Overall Risk Level:** Medium — The orchestrator file modifications (Tasks 03 and 10) carry the highest risk due to file size and the need to sequence Track A and Track B changes on the same file. Mitigated by explicit dependency ordering.

**Key Assumptions:**

1. All 14 evaluating agent files follow the standard chatagent format with identifiable Outputs and Workflow sections. (Tasks 05–09 depend on this)
2. The orchestrator file has identifiable section boundaries (Documentation Structure, Operating Rules, Workflow steps, etc.) that can be targeted for additive edits. (Tasks 03, 10 depend on this)
3. The `create_file` tool can create new files at `.github/agents/` paths. (Tasks 01, 02 depend on this)
4. `replace_string_in_file` works reliably for Markdown content insertions in large files. (All modification tasks depend on this)

---

## Deliverables & Acceptance

Implementation is complete when:

1. All 11 tasks are marked DONE
2. 2 new files created: `evaluation-schema.md`, `post-mortem.agent.md`
3. 16 existing files modified: 14 evaluating agents + orchestrator + feature-workflow prompt
4. 1 summary documentation file produced
5. 5 excluded agent files have zero modifications
6. All acceptance criteria AC-1 through AC-14 are satisfied
