# Plan: Agent Pipeline Improvements

> **Planning Mode:** Initial
> **Feature Slug:** agent-pipeline-improvements
> **Overall Risk:** 🔴
> **Tasks:** 11 | **Waves:** 3

---

## Feature Overview

This feature addresses 8 issues across the agent pipeline system by modifying 15 files (1 new, 14 modified) in `NewAgents/.github/agents/` and `NewAgents/.github/prompts/`. All changes are to markdown-based agent instruction files — no runtime code.

**Issues Addressed:**

1. E2E testing available for all risk levels (decouple e2e_required from workflow_lane)
2. Approval mode clarity and enforcement (remove "default to autonomous" bias)
3. Web fetching capability for agents (fetch_webpage for Researcher, Designer, Spec)
4. No terminal-to-file redirection enforcement (add to Implementer, Verifier)
5. Spec pushback with per-concern granularity (structured ask_questions)
6. E2E via GitHub Copilot Skills integration (copilot-skill format support)
7. SQLite DB auto-archive on new pipeline run (run_id mismatch rename)
8. Fast-track pipeline prompt starting at planning (plan-and-implement.prompt.md)

---

## Success Criteria (from spec-output.yaml)

| AC    | Criterion                                                                | Test Method |
| ----- | ------------------------------------------------------------------------ | ----------- |
| AC-1  | E2E decoupled from risk: 🟢 task + contract → e2e_required=true possible | inspection  |
| AC-2  | E2E optional for low risk: 🟢 + contract → e2e_required=false allowed    | inspection  |
| AC-3  | E2E mandatory for 🔴 + contract (backward compat)                        | inspection  |
| AC-4  | Approval mode authoritative from initial-request.md                      | inspection  |
| AC-5  | No "default to autonomous" phrase in system                              | test        |
| AC-6  | fetch_webpage in tool-access-matrix §1 with 🔒 for 3 agents              | inspection  |
| AC-7  | fetch_webpage docs in researcher, designer, spec agents                  | inspection  |
| AC-8  | implementer no-redirect prohibition                                      | inspection  |
| AC-9  | verifier no-redirect prohibition                                         | inspection  |
| AC-10 | anti-drift anchors include no-redirect                                   | inspection  |
| AC-11 | spec pushback per-concern ask_questions                                  | inspection  |
| AC-12 | pushback_log per-concern user_response                                   | inspection  |
| AC-13 | e2e-integration skill_format field                                       | inspection  |
| AC-14 | Orchestrator Step 0 DB archive                                           | inspection  |
| AC-15 | Orchestrator scope includes archive rename                               | inspection  |
| AC-16 | plan-and-implement.prompt.md exists                                      | inspection  |
| AC-17 | Planner ask_questions access in tool-access-matrix                       | inspection  |
| AC-18 | Planner fast-track fallback input path                                   | inspection  |
| AC-19 | Anti-drift anchor amended for conditional step-skip                      | inspection  |
| AC-20 | feature-workflow.prompt.md unmodified                                    | inspection  |
| AC-21 | orchestrator.agent.md ≤ 550 lines                                        | test        |
| AC-22 | Self-verification checklists updated                                     | inspection  |

---

## Task Index

| #   | Task                                                                                    | Files | Risk | Size     | Deps   | Wave |
| --- | --------------------------------------------------------------------------------------- | ----- | ---- | -------- | ------ | ---- |
| 01  | schemas.md + sql-templates.md — E2E decoupling + archive query                          | 2     | 🟡   | Standard | —      | 1    |
| 02  | tool-access-matrix.md — fetch_webpage + ask_questions + scope                           | 1     | 🟡   | Standard | —      | 1    |
| 03  | e2e-integration.md + dispatch-patterns.md — E2E for all risks + Copilot Skills          | 2     | 🟡   | Standard | —      | 1    |
| 04  | orchestrator.agent.md — all 9 changes (approval, archive, fast-track, E2E, anti-drift)  | 1     | 🔴   | Large    | 01, 02 | 2    |
| 05  | planner.agent.md — E2E decoupling + fast-track fallback + self-verification             | 1     | 🟡   | Standard | 01, 02 | 2    |
| 06  | verifier.agent.md — Tier 5 gate + no-redirect + Copilot skill + approval_mode           | 1     | 🟡   | Standard | 03     | 2    |
| 07  | spec.agent.md — pushback restructure + fetch_webpage + approval_mode                    | 1     | 🟡   | Standard | 02     | 2    |
| 08  | implementer.agent.md — no-redirect + approval_mode                                      | 1     | 🟢   | Standard | —      | 1    |
| 09  | researcher.agent.md + designer.agent.md — fetch_webpage + Copilot skill + approval_mode | 2     | 🟢   | Standard | 02     | 2    |
| 10  | adversarial-reviewer.agent.md + knowledge-agent.agent.md — approval_mode                | 2     | 🟢   | Standard | —      | 1    |
| 11  | plan-and-implement.prompt.md — fast-track pipeline prompt (CREATE)                      | 1     | 🟢   | Standard | 04     | 3    |

---

## Execution Waves

### Wave 1: Foundation + No-Dependency Agents (5 tasks, max concurrent: 4)

All tasks with zero dependencies execute first. Foundation reference documents (schemas, sql-templates, tool-access-matrix, e2e-integration, dispatch-patterns) are updated alongside simple agent changes that have no upstream dependencies.

- **task-01:** schemas.md + sql-templates.md
- **task-02:** tool-access-matrix.md
- **task-03:** e2e-integration.md + dispatch-patterns.md
- **task-08:** implementer.agent.md (no-redirect, approval_mode)
- **task-10:** adversarial-reviewer.agent.md + knowledge-agent.agent.md (approval_mode)

### Wave 2: Core + Extended Agents (5 tasks, max concurrent: 4)

Agent files that reference foundation documents modified in Wave 1. The orchestrator task (task-04) is the critical path — it's the only 🔴 task and has 9 coordinated changes.

- **task-04:** orchestrator.agent.md ← depends on task-01, task-02
- **task-05:** planner.agent.md ← depends on task-01, task-02
- **task-06:** verifier.agent.md ← depends on task-03
- **task-07:** spec.agent.md ← depends on task-02
- **task-09:** researcher + designer agents ← depends on task-02

### Wave 3: Supplemental (1 task, max concurrent: 1)

The fast-track prompt is created after the orchestrator's fast-track detection mechanism is in place.

- **task-11:** plan-and-implement.prompt.md ← depends on task-04

---

## Dependency Graph

```
task-01 ──┬──→ task-04 ──→ task-11
task-02 ──┤
          ├──→ task-05
          ├──→ task-07
          └──→ task-09
task-03 ────→ task-06

task-08  (independent)
task-10  (independent)
```

**Critical path:** task-01/task-02 → task-04 → task-11 (3 sequential steps)

---

## Risk Summary

| Level             | Count | Tasks                                                           |
| ----------------- | ----- | --------------------------------------------------------------- |
| 🔴 Critical       | 1     | task-04 (orchestrator — auth, pipeline flow, anti-drift anchor) |
| 🟡 Business Logic | 6     | task-01, task-02, task-03, task-05, task-06, task-07            |
| 🟢 Additive       | 4     | task-08, task-09, task-10, task-11                              |

**Overall Risk: 🔴** — driven by orchestrator.agent.md changes (task-04) which modify pipeline flow control, approval mode logic, DB archive behavior, and the anti-drift security anchor.

### 🔴 Classification Rationale (task-04)

The orchestrator controls pipeline step execution, approval mode enforcement, and security review guarantees. The changes include:

- Modifying the anti-drift anchor that guarantees Step 7 (security review) execution
- Adding DB archive logic that runs filesystem operations (`Rename-Item`)
- Rewriting approval mode selection which affects all downstream agent behavior
- Adding fast-track mode detection that enables step skipping

All 9 changes operate on the single most critical file in the system. The design keeps them in one task because they're interleaved in the same file sections with tight line budget constraints (539→550).

---

## Pre-Mortem Analysis

| Task    | Failure Scenario                                                                      | Likelihood | Impact | Mitigation                                                                                    |
| ------- | ------------------------------------------------------------------------------------- | ---------- | ------ | --------------------------------------------------------------------------------------------- |
| task-01 | EG-10 restructure breaks existing evidence gate queries                               | M          | H      | Verify existing EG-9 gate is untouched; additive-only change to EG-10                         |
| task-02 | §1 matrix row formatting breaks table alignment                                       | L          | L      | Use existing row format as template                                                           |
| task-03 | Copilot skill format description too vague for Tier 5 consumption                     | M          | M      | Reference concrete SKILL.md structure from design D-8                                         |
| task-04 | Net line change exceeds 550-line budget                                               | H          | H      | Track line count per edit; DB archive (~8 lines) is first extraction candidate if over budget |
| task-05 | Fast-track fallback path doesn't adequately compensate for missing upstream artifacts | M          | M      | ask_questions prompts must cover scope, requirements, constraints explicitly                  |
| task-06 | Copilot skill → Playwright CLI conversion guidance too abstract                       | M          | L      | Focus on structural mapping (interaction steps → page actions)                                |
| task-07 | Per-concern ask_questions format doesn't match VS Code UX                             | L          | M      | Use existing ask_questions API pattern from spec (questions[] array)                          |
| task-08 | No-redirect rule wording inconsistent with verifier (task-06)                         | L          | L      | Use identical wording from spec FR-4.1/FR-4.2                                                 |
| task-09 | fetch_webpage vs Context7 guidance unclear for designer                               | L          | L      | Keep distinction simple: fetch_webpage=web pages, Context7=library APIs                       |
| task-10 | Approval_mode parameter table doesn't exist in target files                           | L          | L      | Create the table if absent, or add row to existing parameters section                         |
| task-11 | Prompt variable name doesn't match orchestrator detection logic                       | M          | H      | Cross-reference with task-04 implementation; variable name = pipeline_mode                    |

### Overall Risk Level: **Medium**

The highest-impact risk is the orchestrator 550-line budget (task-04). All other risks are manageable through careful text editing. The feature modifies only markdown instruction files — no build, runtime, or test infrastructure is affected.

### Key Assumptions

1. **Orchestrator line count is 539** (verified by design R2 via direct file read + research/impact.yaml). If actual count differs, task-04 line budget arithmetic changes. _(Affects: task-04)_
2. **ask_questions API supports per-question options** with allowFreeformInput. This is the documented API pattern. _(Affects: task-07, task-05)_
3. **No e2e-contract.yaml exists in the workspace.** If one is added before implementation, e2e_required derivation changes for 🔴 tasks. _(Affects: all tasks with e2e_required field)_
4. **feature-workflow.prompt.md will not be modified** by any external change during implementation. _(Affects: task-11, AC-20)_
5. **All 8 subagent files have an Orchestrator-Provided Parameters section** (or equivalent). If not, the implementer must create one. _(Affects: task-05, task-06, task-07, task-08, task-09, task-10)_
