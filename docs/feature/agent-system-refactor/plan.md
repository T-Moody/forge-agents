# Agent System Refactor (Run 3) — Implementation Plan

## Feature Overview

**Feature:** v2 Targeted Improvements — 10 Improvements to Existing Agent System  
**Planning Mode:** Initial  
**Run ID:** 2026-03-09T18:00:00Z  
**Overall Risk:** 🟡

Targeted modifications to the existing 9 v2 agent files (8 agents + global-rules.md) implementing 10 improvements: correct tool names (D-1/D-2), orchestrator-mediated interactive questioning (D-3), git safety extraction (D-4), two-source selective staging (D-5), exploratory QA (D-6), doc-update optional step (D-7), architect web research guidance (D-8), implementer TDD best practices (D-9), Step 8 decomposition into 8a-8e (D-10). All agents must remain under 150-line budgets. No new agent files created. Prompts unchanged.

---

## Success Criteria (from spec-output.yaml)

| AC    | Criterion                                                              | Mapped To           |
| ----- | ---------------------------------------------------------------------- | ------------------- |
| AC-1  | Orchestrator Step 1 E2E skill scan                                     | task-02             |
| AC-2  | Interactive E2E prompt with 3+ options                                 | task-02             |
| AC-3  | Autonomous mode E2E warning (no prompt)                                | task-02             |
| AC-4  | All tool names flat camelCase across all 8 agents                      | tasks 02-09         |
| AC-5  | Orchestrator tools: 'agent' + agents: 7 workers                        | task-02             |
| AC-6  | Orchestrator body: runSubagent dispatch instruction                    | task-02             |
| AC-7  | All 7 workers: user-invocable: false                                   | tasks 03-09         |
| AC-8  | Architect: clarification step gated by interactive mode                | task-04             |
| AC-9  | DR-1: No structured question tool exists; orchestrator mediates        | task-02 (mediation) |
| AC-10 | Planner: plan_summary output for orchestrator                          | task-05             |
| AC-11 | DR-2: No structured question tool; orchestrator mediates plan approval | task-02 (mediation) |
| AC-12 | Implementer: no git add/commit; allowlist = git diff, git status       | task-03             |
| AC-13 | global-rules.md: git safety rule (orchestrator-only staging)           | task-01             |
| AC-14 | Orchestrator Step 8d: selective pathspecs (not git add -A)             | task-02             |
| AC-15 | Orchestrator Step 8e: interactive Commit/Review/Unstage                | task-02             |
| AC-16 | Autonomous: stage only, no commit                                      | task-02             |
| AC-17 | Orchestrator: doc update Apply/Review/Skip choice                      | task-02             |
| AC-18 | Knowledge: MUST NOT modify .agent.md in doc-update mode                | task-08             |
| AC-19 | Architect: Web Research step gated by web_research_enabled             | task-04             |
| AC-20 | Architect: fetch tool name is 'fetch'                                  | task-04             |
| AC-21 | Tester: Exploratory QA with 4 categories                               | task-06             |
| AC-22 | Tester: exploratory findings with category + severity                  | task-06             |
| AC-23 | Tester: ≤150 lines after changes                                       | task-06             |
| AC-24 | Implementer: REFACTOR step (RED-GREEN-REFACTOR-REPORT)                 | task-03             |
| AC-25 | Implementer: YAGNI, KISS, DRY, functional, minimal code                | task-03             |
| AC-26 | REFACTOR: re-run tests + revert on failure                             | task-03             |
| AC-27 | All 8 agents ≤150 lines each                                           | Cross-cutting (all) |

---

## Ordered Task Index

| #   | Task                                                     | Risk | Size     | Files | Wave |
| --- | -------------------------------------------------------- | ---- | -------- | ----- | ---- |
| 1   | Update global-rules.md — Git Safety + Plan Refinement    | 🟢   | Standard | 1     | 1    |
| 2   | Update orchestrator.agent.md — Full Overhaul             | 🟡   | Standard | 1     | 2    |
| 3   | Update implementer.agent.md — Git Safety + TDD Refactor  | 🟡   | Standard | 1     | 2    |
| 4   | Update architect.agent.md — Clarification + Web Research | 🟡   | Standard | 1     | 2    |
| 5   | Update planner.agent.md — Plan Summary + Approval        | 🟢   | Standard | 1     | 2    |
| 6   | Update tester.agent.md — Exploratory QA + Audit          | 🟡   | Standard | 1     | 3    |
| 7   | Update reviewer.agent.md — Audit Pattern Update          | 🟢   | Standard | 1     | 3    |
| 8   | Update knowledge.agent.md — Doc-Update Mode + Memory     | 🟡   | Standard | 1     | 3    |
| 9   | Update researcher.agent.md — Tool Name Fix               | 🟢   | Standard | 1     | 3    |

---

## Execution Waves

### Wave 1 — Foundation (no dependencies)

| Task    | File Modified                     | Changes                                   | Est. Lines |
| ------- | --------------------------------- | ----------------------------------------- | ---------- |
| task-01 | v2/.github/agents/global-rules.md | Git safety rule (FR-5.3), plan refinement | 70→80      |

**Parallelism:** 1 task. Foundation for all subsequent tasks.

### Wave 2 — Core Agent Modifications (depend on task-01)

| Task    | File Modified                           | Key Changes                                           | Est. Lines |
| ------- | --------------------------------------- | ----------------------------------------------------- | ---------- |
| task-02 | v2/.github/agents/orchestrator.agent.md | Tool names, E2E scan, mediation, Step 8 decomposition | 117→142    |
| task-03 | v2/.github/agents/implementer.agent.md  | Tool names, git safety, TDD REFACTOR, quality         | 99→115     |
| task-04 | v2/.github/agents/architect.agent.md    | Tool names, clarification step, web research          | 130→145    |
| task-05 | v2/.github/agents/planner.agent.md      | Tool names, plan_summary output                       | 105→116    |

**Parallelism:** 4 tasks, max_concurrent: 4. No file overlap. Architect is tightest (5-line headroom).

### Wave 3 — Extended Agent Modifications (depend on wave 2 tasks)

| Task    | File Modified                         | Key Changes                              | Est. Lines |
| ------- | ------------------------------------- | ---------------------------------------- | ---------- |
| task-06 | v2/.github/agents/tester.agent.md     | Exploratory QA, audit update, tool names | 129→143    |
| task-07 | v2/.github/agents/reviewer.agent.md   | Audit pattern update, tool names         | 96→99      |
| task-08 | v2/.github/agents/knowledge.agent.md  | Doc-update mode, memory fix, tool names  | 107→118    |
| task-09 | v2/.github/agents/researcher.agent.md | Tool name fix, user-invocable            | 83→84      |

**Parallelism:** 4 tasks, max_concurrent: 4. No file overlap.

---

## Dependency Graph

```
task-01 ──┬──> task-02 ──> task-08
          │
          ├──> task-03 ──┬──> task-06
          │              └──> task-07
          │
          ├──> task-04 ──> task-09
          │
          └──> task-05
```

**Dependency justifications:**

- **task-02 → task-01:** Orchestrator references global-rules git safety rule for Step 8 staging.
- **task-03 → task-01:** Implementer follows global-rules git safety constraints (removes git add).
- **task-04 → task-01:** Architect reads global-rules (incl. plan refinement limit affecting output handling).
- **task-05 → task-01:** Planner reads global-rules (plan refinement limit directly affects workflow).
- **task-06 → task-03:** Tester static audit must match implementer's updated git patterns.
- **task-07 → task-03:** Reviewer command audit must match implementer's updated allowlist.
- **task-08 → task-02:** Knowledge doc-update mode aligns with orchestrator Step 8b mediation.
- **task-09 → task-04:** Both T3 agents share fetch gating pattern; researcher aligns with architect's corrected tool names.

---

## Risk Summary

### Per-Task Breakdown

| Task    | Risk | Rationale                                                                        |
| ------- | ---- | -------------------------------------------------------------------------------- |
| task-01 | 🟢   | Additive: 2 small additions to shared config (~10 lines)                         |
| task-02 | 🟡   | Business logic: orchestrator coordination overhaul (11 change items, 25 net)     |
| task-03 | 🟡   | Business logic: implementer workflow restructure (git removal + TDD refactor)    |
| task-04 | 🟡   | Business logic: architect workflow additions (5-line headroom — tightest agent)  |
| task-05 | 🟢   | Additive: planner gains plan_summary output step (~11 net lines)                 |
| task-06 | 🟡   | Business logic: tester gains exploratory QA phase + audit pattern update         |
| task-07 | 🟢   | Simple: reviewer audit pattern change (git diff\|add\|status → git diff\|status) |
| task-08 | 🟡   | Business logic: knowledge gains doc-update mode + memory tool reword             |
| task-09 | 🟢   | Simple: tool name corrections + user-invocable: false (~1 net line)              |

### Overall: 🟡

Highest task risk is 🟡 (business logic modifications). No auth/crypto/payments/migrations affected. The architect's 5-line headroom (130→145) is the primary implementation risk, noted in pre-mortem.

---

## Design Review Carry-Over

All 3 adversarial reviewers approved in R2:

- **Security Sentinel:** 0 findings. D-11 rejection of disable-model-invocation validated.
- **Architecture Guardian:** 1 Minor carry-over (git safety is advisory, not deterministic enforcement). Accepted as known limitation — defense-in-depth via three-layer trust model.
- **Pragmatic Verifier:** 0 findings. All 6 R1 findings resolved.

---

## Pre-Mortem Analysis

| Task    | Failure Scenario                                             | Likelihood | Impact | Mitigation                                                 |
| ------- | ------------------------------------------------------------ | ---------- | ------ | ---------------------------------------------------------- |
| task-02 | Orchestrator exceeds 150 lines (11 changes, 117→142 est.)    | M          | H      | Concise prose; extract Step 8 detail if budget tight       |
| task-03 | git add removal breaks implementer workflow reference        | L          | M      | Verify all workflow steps renumbered after step 6 removal  |
| task-04 | Architect exceeds 150 lines (5-line headroom at 130→145)     | H          | H      | Extract FR-8 web research to reference doc if budget tight |
| task-05 | plan_summary field missing from planner output schema        | L          | M      | Follow design D-3 assumptions-first pattern exactly        |
| task-06 | Tester exploratory QA too verbose (129→143, 7-line headroom) | M          | M      | 4-category compact checklist; detailed procedures to SKILL |
| task-08 | Knowledge doc-update mode references non-existent files      | L          | L      | Guard: return DONE if instruction-recommendations.md empty |

**Overall Risk Level:** Medium — The architect (task-04) has the highest probability of budget overrun at 5-line headroom. Extraction fallback documented in design D-6.

**Key Assumptions:**

1. **Tool name enforcement works:** VS Code YAML frontmatter `tools:` array with flat camelCase names actually restricts agent tool access at platform level. If not enforced, all tool-restriction changes are advisory only. (Affects: all tasks)
2. **Line count estimates are accurate:** Design-verified line counts (PowerShell `(Get-Content).Count`, 2026-03-09) are still current. If files were modified since measurement, headroom calculations may be wrong. (Affects: task-04 architect most critically)
3. **user-invocable: false hides agents:** This frontmatter property prevents workers from appearing in VS Code dropdown. If not respected, workers could be invoked directly. (Affects: tasks 03-09)
