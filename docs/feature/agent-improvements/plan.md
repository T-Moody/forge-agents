# Implementation Plan: Comprehensive Forge Agent Improvements

## Feature Overview

Rewrite all 9 Forge agents, the workflow prompt, and create a new documentation-writer agent — incorporating 30+ improvements from Gem Team analysis, bug fixes, and cross-cutting enhancements. All deliverables are markdown files (`.agent.md` and `.prompt.md`) output to `NewAgentsAndPrompts/`. No files in `.github/` are modified.

**Key architectural decisions preserved:** Fixed deterministic 8-step pipeline, markdown artifact chain, orchestrator as sole router, three-state text completion contracts (`DONE:`/`NEEDS_REVISION:`/`ERROR:`), dedicated spec/design/critical-thinker stages, dedicated verifier, full autonomy by default.

**Key improvements being implemented:**

- TDD enforcement in implementer (Cluster A — 3 files)
- Pre-mortem analysis, task size limits, mode detection in planner
- Security thread across designer, implementer, reviewer
- Technology-agnostic verifier
- Per-task agent routing (Cluster B — 3 files)
- Optional human approval gates (Cluster C — 2 files)
- Shared operating rules, anti-drift anchors, completion contracts in ALL agents
- New documentation-writer agent
- Bug fixes for critical-thinker (missing completion contract, missing output spec)

---

## Success Criteria

| ID    | Criterion                                                                                                              | Source |
| ----- | ---------------------------------------------------------------------------------------------------------------------- | ------ |
| SC-1  | All 11 files exist in `NewAgentsAndPrompts/` and are non-empty                                                         | AC-1   |
| SC-2  | Critical-thinker has completion contract + output file specification                                                   | AC-2   |
| SC-3  | TDD Cluster A consistency: implementer TDD workflow + verifier integration-level role + orchestrator global rule agree | AC-3   |
| SC-4  | Concurrency cap of 4 in orchestrator + prompt                                                                          | AC-4   |
| SC-5  | Planner has pre-mortem, task size limits, mode detection                                                               | AC-5   |
| SC-6  | Security thread: implementer rules + reviewer OWASP + designer section                                                 | AC-6   |
| SC-7  | Verifier is technology-agnostic (no hardcoded project refs)                                                            | AC-7   |
| SC-8  | All 10 agent files have anti-drift anchors                                                                             | AC-8   |
| SC-9  | All agents have DONE/ERROR contracts, no contradictions                                                                | AC-9   |
| SC-10 | Per-task agent routing coordinated across planner + orchestrator + prompt                                              | AC-10  |
| SC-11 | Approval gates conditional on APPROVAL_MODE, default autonomous                                                        | AC-11  |
| SC-12 | Documentation-writer exists, not in core pipeline                                                                      | AC-12  |

---

## Ordered Task Index

| #   | Task                             | Output File                     | Clusters      | Wave |
| --- | -------------------------------- | ------------------------------- | ------------- | ---- |
| 01  | Write researcher agent           | `researcher.agent.md`           | None          | 1    |
| 02  | Write spec agent                 | `spec.agent.md`                 | None          | 1    |
| 03  | Write designer agent             | `designer.agent.md`             | None          | 1    |
| 04  | Write critical-thinker agent     | `critical-thinker.agent.md`     | None          | 1    |
| 05  | Write implementer agent          | `implementer.agent.md`          | A (primary)   | 2    |
| 06  | Write verifier agent             | `verifier.agent.md`             | A (secondary) | 2    |
| 07  | Write planner agent              | `planner.agent.md`              | B (primary)   | 2    |
| 08  | Write reviewer agent             | `reviewer.agent.md`             | None          | 2    |
| 09  | Write orchestrator agent         | `orchestrator.agent.md`         | A, B, C       | 3    |
| 10  | Write workflow prompt            | `feature-workflow.prompt.md`    | B, C          | 3    |
| 11  | Write documentation-writer agent | `documentation-writer.agent.md` | B (consumer)  | 3    |

---

## Execution Waves

### Wave 1 (parallel — 4 tasks, independent agents)

- 01-researcher (depends_on: none)
- 02-spec (depends_on: none)
- 03-designer (depends_on: none)
- 04-critical-thinker (depends_on: none)

**Rationale:** These 4 agents have zero cluster dependencies. They follow the canonical structure from design.md and have no cross-agent coordination requirements beyond the shared Operating Rules block (which is specified verbatim in design.md).

### Wave 2 (parallel — 4 tasks, cluster-defining agents)

- 05-implementer (depends_on: none — but defines Cluster A TDD terminology)
- 06-verifier (depends_on: 05 — must match implementer's TDD responsibility split)
- 07-planner (depends_on: none — but defines Cluster B agent field)
- 08-reviewer (depends_on: none — independent, but complex)

**Rationale:** The implementer and verifier form Cluster A (TDD). Placing them in the same wave ensures they reference the same design.md sections and use consistent TDD terminology. The planner defines the `agent` field (Cluster B primary). The reviewer is independent but complex enough to benefit from having Wave 1's simpler agents as exemplars.

**Cluster A coordination note:** Tasks 05 and 06 MUST use consistent terminology per design.md: "Implementers perform unit-level TDD" / "Verifier performs integration-level verification." Both implementers should reference design.md sections 7 (implementer) and 8 (verifier) and the Cross-Cutting Patterns.

### Wave 3 (parallel — 3 tasks, orchestrator + prompt + new agent)

- 09-orchestrator (depends_on: 05, 06, 07, 08 — touches all clusters, must be consistent with all other agents)
- 10-prompt (depends_on: 09 — reinforces orchestrator rules for Clusters B, C)
- 11-documentation-writer (depends_on: 07 — requires Cluster B agent field convention from planner)

**Rationale:** The orchestrator is the most complex file — it touches Cluster A (TDD global rule), Cluster B (agent routing), and Cluster C (approval gates). Writing it last ensures all other agents' contracts are finalized. The prompt reinforces orchestrator rules. The documentation-writer depends on the agent routing convention established by the planner (Cluster B).

---

## Dependency Graph

```
Wave 1:  01-researcher ──────────────┐
         02-spec ────────────────────┤
         03-designer ────────────────┤── all independent, parallel
         04-critical-thinker ────────┘
                                     │
Wave 2:  05-implementer ─────┐      │
         06-verifier ────────┤──────── Cluster A (TDD)
         07-planner ─────────┤──────── Cluster B primary (agent field)
         08-reviewer ────────┘       │
                                     │
Wave 3:  09-orchestrator ────────────┤── depends on 05,06,07,08 (all clusters)
         10-prompt ──────────────────┤── depends on 09 (Clusters B,C)
         11-documentation-writer ────┘── depends on 07 (Cluster B)
```

### Cluster Coordination Map

| Cluster | Theme          | Files                               | Defining Task     | Key Consistency Requirement                                                                        |
| ------- | -------------- | ----------------------------------- | ----------------- | -------------------------------------------------------------------------------------------------- |
| A       | TDD            | implementer, verifier, orchestrator | 05 (implementer)  | "unit-level TDD" vs "integration-level verification" terminology                                   |
| B       | Agent Routing  | planner, orchestrator, prompt       | 07 (planner)      | `agent` field name, valid values (`implementer`, `documentation-writer`), default behavior         |
| C       | Approval Gates | orchestrator, prompt                | 09 (orchestrator) | `{{APPROVAL_MODE}}` variable name, pause points (post-research, post-planning), default=autonomous |

### Three-State Completion Consistency

| Agent                | Uses NEEDS_REVISION? | Routing Target        |
| -------------------- | -------------------- | --------------------- |
| Researcher           | No                   | —                     |
| Spec                 | No                   | —                     |
| Designer             | No                   | —                     |
| Critical Thinker     | **Yes**              | Designer              |
| Planner              | No                   | —                     |
| Implementer          | No                   | —                     |
| Verifier             | **Yes**              | Planner (replan loop) |
| Reviewer             | **Yes**              | Implementer(s)        |
| Documentation Writer | No                   | —                     |

---

## Risks & Mitigations

| Risk                                                                              | Likelihood | Impact | Mitigation                                                                                                             |
| --------------------------------------------------------------------------------- | ---------- | ------ | ---------------------------------------------------------------------------------------------------------------------- |
| Cluster A inconsistency between implementer/verifier/orchestrator TDD terminology | Medium     | High   | Both Wave 2 tasks 05/06 reference identical design.md sections; orchestrator (task 09) cross-checks both               |
| Cluster B field name mismatch between planner and orchestrator                    | Low        | High   | Design.md specifies exact field name `agent` and valid values; orchestrator task 09 must verify against planner output |
| Shared Operating Rules block diverges across agents                               | Medium     | Medium | Design.md provides verbatim block; only Rule 5 (tool preferences) varies per agent per the table                       |
| Anti-drift anchor text wrong for an agent                                         | Low        | Low    | Design.md provides exact anchor text per agent in a lookup table                                                       |
| Implementer produces file that contradicts design.md                              | Low        | High   | Each task's acceptance criteria include specific content checks against design.md                                      |
| Three-state completion contract applied to wrong agents                           | Medium     | High   | Only critical-thinker, reviewer, verifier use NEEDS_REVISION — all others use only DONE/ERROR per design.md            |

---

## Deliverables & Acceptance

All 11 files must exist in `NewAgentsAndPrompts/`:

1. `orchestrator.agent.md`
2. `researcher.agent.md`
3. `spec.agent.md`
4. `designer.agent.md`
5. `critical-thinker.agent.md`
6. `planner.agent.md`
7. `implementer.agent.md`
8. `verifier.agent.md`
9. `reviewer.agent.md`
10. `feature-workflow.prompt.md`
11. `documentation-writer.agent.md`

**Verification:** After all tasks complete, the verifier should run test scenarios TS-1 through TS-15 from feature.md to confirm cross-file consistency.
