# Agent System Refactor — Implementation Plan

## Feature Overview

**Feature:** Clean-Slate Multi-Agent Pipeline Rebuild  
**Planning Mode:** Initial  
**Run ID:** 2026-03-08T12:00:00Z  
**Overall Risk:** 🔴

A full clean-slate rebuild of the agent system: 8 agents + 1 shared reference + 2 prompt files in `v2/.github/`. Zero SQLite, YAML file-based evidence, VS Code native tool restrictions via YAML frontmatter, DAG-based task execution, completion-contract-only routing, ≤1,500 total lines.

---

## Success Criteria (from spec-output.yaml)

| AC    | Criterion                                                       | Mapped To                 |
| ----- | --------------------------------------------------------------- | ------------------------- |
| AC-1  | Total system ≤1,500 lines                                       | Cross-cutting (all tasks) |
| AC-2  | No agent file >150 lines                                        | Each agent task           |
| AC-3  | Exactly 8 .agent.md files                                       | Cross-cutting             |
| AC-4  | ≤2 shared reference docs                                        | task-01                   |
| AC-5  | YAML frontmatter: name, description, tools                      | Each agent task           |
| AC-6  | Orchestrator agents field lists all 7                           | task-10                   |
| AC-7  | Non-orchestrator agents: []                                     | tasks 03-09               |
| AC-8  | No sqlite/sqlite3/verification-ledger references                | All tasks                 |
| AC-9  | YAML-only routing in orchestrator                               | task-10                   |
| AC-10 | 8 named steps in orchestrator                                   | task-10                   |
| AC-11 | Code Review (Step 7) non-skippable                              | task-10                   |
| AC-12 | Planner output schema: id, description, depends_on, files, risk | task-05                   |
| AC-13 | DAG-based dispatch (not wave-based)                             | task-10                   |
| AC-14 | No file overlap in parallel dispatches                          | task-10                   |
| AC-15 | Implementer tools exclude app start/E2E/curl                    | task-07                   |
| AC-16 | Tester has run_in_terminal                                      | task-08                   |
| AC-17 | Knowledge cannot write .agent.md files                          | task-09                   |
| AC-18 | No § cross-references                                           | All tasks                 |
| AC-19 | Researcher has fetch_webpage                                    | task-03                   |
| AC-20 | New directory (not modifying existing)                          | All tasks                 |
| AC-21 | Industry traceability                                           | All tasks                 |
| AC-22 | Completion contract in every output schema                      | All agent tasks           |

---

## Ordered Task Index

| #   | Task                                    | Risk | Size     | Files | Wave |
| --- | --------------------------------------- | ---- | -------- | ----- | ---- |
| 1   | Create global-rules.md shared reference | 🟢   | Standard | 1     | 1    |
| 2   | Create pipeline prompt files            | 🟢   | Standard | 2     | 1    |
| 3   | Create researcher.agent.md              | 🟢   | Standard | 1     | 2    |
| 4   | Create architect.agent.md               | 🟢   | Standard | 1     | 2    |
| 5   | Create planner.agent.md                 | 🟢   | Standard | 1     | 2    |
| 6   | Create reviewer.agent.md                | 🟢   | Standard | 1     | 2    |
| 7   | Create implementer.agent.md             | 🟡   | Standard | 1     | 3    |
| 8   | Create tester.agent.md                  | 🟡   | Standard | 1     | 3    |
| 9   | Create knowledge.agent.md               | 🟡   | Standard | 1     | 3    |
| 10  | Create orchestrator.agent.md            | 🔴   | Large    | 1     | 4    |

---

## Execution Waves

### Wave 1 — Foundation (no dependencies)

| Task    | File(s) Created                                                                       | Est. Lines |
| ------- | ------------------------------------------------------------------------------------- | ---------- |
| task-01 | v2/.github/agents/global-rules.md                                                     | ~100       |
| task-02 | v2/.github/prompts/feature-workflow.prompt.md, v2/.github/prompts/quick-fix.prompt.md | ~55        |

**Parallelism:** 2 tasks, max_concurrent: 2. No file overlap.

### Wave 2 — Tier 3 Read-Only Agents (depend on task-01)

| Task    | File Created                          | Est. Lines |
| ------- | ------------------------------------- | ---------- |
| task-03 | v2/.github/agents/researcher.agent.md | ~100       |
| task-04 | v2/.github/agents/architect.agent.md  | ~140       |
| task-05 | v2/.github/agents/planner.agent.md    | ~130       |
| task-06 | v2/.github/agents/reviewer.agent.md   | ~110       |

**Parallelism:** 4 tasks, max_concurrent: 4. No file overlap. All Tier 3 (read-only + create) agents.

### Wave 3 — Tier 2 Standard Trust Agents + Knowledge (depend on task-01)

| Task    | File Created                           | Est. Lines |
| ------- | -------------------------------------- | ---------- |
| task-07 | v2/.github/agents/implementer.agent.md | ~130       |
| task-08 | v2/.github/agents/tester.agent.md      | ~140       |
| task-09 | v2/.github/agents/knowledge.agent.md   | ~100       |

**Parallelism:** 3 tasks, max_concurrent: 3. No file overlap. Agents with security-scoped constraints (command allowlist, singleton mode, output path allowlist).

### Wave 4 — Orchestrator (depend on task-01)

| Task    | File Created                            | Est. Lines |
| ------- | --------------------------------------- | ---------- |
| task-10 | v2/.github/agents/orchestrator.agent.md | ~150       |

**Parallelism:** 1 task. The orchestrator is the most complex agent (DAG dispatch, routing, pre-commit validation, trust boundary enforcement). Written last for maximum design context.

---

## Dependency Graph

```
task-01 ──┬──> task-03
          ├──> task-04
          ├──> task-05
          ├──> task-06
          ├──> task-07
          ├──> task-08
          ├──> task-09
          └──> task-10

task-02 (independent)
```

All agent tasks (03-10) depend on task-01 (global-rules.md) because every agent references the completion contract schema and cross-cutting rules defined there. Task-02 (prompt files) is fully independent.

---

## Risk Summary

### Per-Task Breakdown

| Task    | Risk | Rationale                                                                                                     |
| ------- | ---- | ------------------------------------------------------------------------------------------------------------- |
| task-01 | 🟢   | New shared reference doc — additive, no security logic                                                        |
| task-02 | 🟢   | New prompt files — short entry points, no complex logic                                                       |
| task-03 | 🟢   | New agent file — read-only Tier 3, simple workflow                                                            |
| task-04 | 🟢   | New agent file — read-only Tier 3, structured output                                                          |
| task-05 | 🟢   | New agent file — read-only Tier 3, DAG task schema                                                            |
| task-06 | 🟢   | New agent file — read-only Tier 3, findings + verdict                                                         |
| task-07 | 🟡   | Command allowlist (security enforcement), file ownership constraint                                           |
| task-08 | 🟡   | Dual-mode execution, singleton constraint (concurrency), test lifecycle                                       |
| task-09 | 🟡   | Output path allowlist (security gate), post-review write capability                                           |
| task-10 | 🔴   | Pre-commit validation, trust boundary enforcement, git operations, all routing, DAG dispatch with concurrency |

### Overall: 🔴

The orchestrator (task-10) contains the security-critical pre-commit validation gate, trust boundary enforcement, and all pipeline routing decisions. This escalates the feature-level risk to 🔴.

---

## Known Issues from Design Review

These issues were identified in R2/R3 design review and must be addressed during implementation:

1. **KI-1: Pre-commit validation scope** (AG-R2-C-1, AG-R3-C-1 Major)  
   design.md describes entire-git-diff validation. The **correct** behavior is per `trust_boundary_model` in design-output.yaml: validate only Knowledge agent output paths. Orchestrator task (task-10) must use trust_boundary_model as authoritative.

2. **KI-2: Pipeline-log.yaml append mechanism** (PV-R3 Minor)  
   The orchestrator appends dispatch entries incrementally. The implementer should specify the append pattern (create_file for initial, replace_string_in_file for appends to dispatches list).

3. **KI-3: Orchestrator body sections** (AG-R3-A-2 Minor)  
   Orchestrator uses Pipeline Steps + DAG Dispatch instead of the standard D-3 6-section layout. This deviation is accepted due to complexity.

---

## Pre-Mortem Analysis

| Task    | Failure Scenario                                                                                    | Likelihood | Impact | Mitigation                                                   |
| ------- | --------------------------------------------------------------------------------------------------- | ---------- | ------ | ------------------------------------------------------------ |
| task-01 | Completion contract schema too verbose, eating into 100-line budget                                 | L          | M      | Keep to 3 fields (status, summary, output_paths) per D-4     |
| task-02 | Quick-fix step set inconsistent with immutable minimum set                                          | L          | H      | Explicitly enumerate steps {1,5,6,7,8} per D-13              |
| task-04 | Architect agent exceeds 140-line budget with spec+design dual responsibility                        | M          | M      | Separate requirements from design in workflow; tight prose   |
| task-05 | Planner output schema missing DAG fields (depends_on, files)                                        | L          | H      | Reference D-6 task_schema definition directly                |
| task-07 | Command allowlist too restrictive or too permissive                                                 | M          | H      | Match exact patterns from trust_boundary_model               |
| task-08 | Dual-mode tester exceeds 140 lines; triggers D-7 split                                              | M          | M      | If >140 lines, extract to 2 agents per D-7 split_trigger     |
| task-09 | Knowledge output_path_allowlist paths inconsistent with orchestrator pre-commit                     | M          | H      | Both tasks reference same design-output.yaml section         |
| task-10 | Orchestrator exceeds 150-line budget with DAG + routing + pre-commit                                | H          | H      | Extract DAG to dag-dispatch.md per D-8 orchestrator_fallback |
| task-10 | Pre-commit scope implements entire-diff (design.md wording) instead of Knowledge-only (trust model) | M          | H      | KI-1: trust_boundary_model is authoritative                  |
| task-10 | DAG dispatch algorithm unclear, causing ambiguous parallel resolution                               | M          | M      | Follow D-6 alt-1: topological sort + file ownership check    |

**Overall Risk Level:** High — The orchestrator (task-10) has the highest probability of budget overrun and the pre-commit scope issue requires careful attention to use the trust_boundary_model (not design.md) as authoritative.

**Key Assumptions:**

1. VS Code .agent.md YAML frontmatter `tools` field enforces tool restrictions at platform level. If not, the trust boundary model loses its primary enforcement mechanism. (Affects: all tasks)
2. A single tester.agent.md can fit both static and dynamic workflows within 140 lines. If not, D-7 split_trigger activates and agent count becomes 9. (Affects: task-08, task-10)
3. The orchestrator DAG dispatch algorithm fits within ~20 lines of the 150-line budget. If not, D-8 fallback creates dag-dispatch.md. (Affects: task-10, task-01)
