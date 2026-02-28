# Agent System Overhaul — Implementation Plan

## Feature Overview

Overhaul the 9-agent NewAgents pipeline system to extract shared patterns into reference documents, restore missing Forge capabilities (evaluations, telemetry, instruction files) as SQLite-backed features, restructure adversarial review for multi-perspective prompt diversity, and reduce agent file sizes to 250–350 lines (orchestrator ≤550).

**Planning Mode:** Initial  
**Date:** 2026-02-27  
**Total Tasks:** 18  
**Total Waves:** 5  
**Overall Risk:** 🔴

---

## Success Criteria (from spec-output.yaml)

| FR    | Title                                       | Covered By                   |
| ----- | ------------------------------------------- | ---------------------------- |
| FR-1  | Parallel Execution Investigation            | TASK-008, TASK-010           |
| FR-2  | Multi-Perspective Adversarial Review        | TASK-004, TASK-008, TASK-011 |
| FR-3  | All-Category Review Coverage                | TASK-004, TASK-008, TASK-011 |
| FR-4  | YAML+MD Dual Output Rationalization         | TASK-007, TASK-011–018       |
| FR-5  | Artifact Evaluation System (SQLite)         | TASK-005, 006, 007, 011–014  |
| FR-6  | Pipeline Telemetry Population & Post-Mortem | TASK-006, TASK-010, TASK-012 |
| FR-7  | Terminal-Only Testing Enforcement           | TASK-001, TASK-013, TASK-014 |
| FR-8  | Global No-File-Redirect Rule                | TASK-001, TASK-011, TASK-013 |
| FR-9  | Conditional Context7 MCP Integration        | TASK-005, TASK-013, TASK-018 |
| FR-10 | Agent File Size Reduction via Shared Docs   | TASK-001–003, 006, 010–018   |
| FR-11 | Verifier Tool Contradiction Resolution      | TASK-003, TASK-014, TASK-018 |
| FR-12 | Review Verdict File Path Consistency        | TASK-007, 010, 011, 017      |
| FR-13 | Deterministic Schema & Contract Enforcement | TASK-002, 006, 007, 010      |
| FR-14 | Expanded SQLite Usage                       | TASK-006, 007, 010, 012      |
| FR-15 | Instruction File Infrastructure             | TASK-001, TASK-012           |
| FR-16 | Frontmatter Standardization                 | TASK-009, TASK-010–018       |

---

## Ordered Task Index

| #   | Task ID  | Title                                       | Risk | Wave | Type           | Dependencies       |
| --- | -------- | ------------------------------------------- | ---- | ---- | -------------- | ------------------ |
| 1   | TASK-001 | Create Instruction File Infrastructure      | 🟢   | 1    | infrastructure | —                  |
| 2   | TASK-002 | Create Global Operating Rules Reference Doc | 🟡   | 1    | documentation  | —                  |
| 3   | TASK-003 | Create Tool Access Matrix Reference Doc     | 🟡   | 1    | documentation  | —                  |
| 4   | TASK-004 | Create Review Perspectives Reference Doc    | 🟢   | 1    | documentation  | —                  |
| 5   | TASK-005 | Create Context7 + Evaluation Schema Docs    | 🟢   | 2    | documentation  | —                  |
| 6   | TASK-006 | Create SQL Templates Reference Doc          | 🟡   | 2    | documentation  | TASK-002           |
| 7   | TASK-007 | Update schemas.md (SQLite + Routing Matrix) | 🟡   | 2    | infrastructure | TASK-002           |
| 8   | TASK-008 | Update dispatch-patterns.md (Perspectives)  | 🟡   | 3    | documentation  | TASK-004           |
| 9   | TASK-009 | Add Frontmatter to severity-taxonomy.md     | 🟢   | 2    | infrastructure | —                  |
| 10  | TASK-010 | Restructure Orchestrator Agent              | 🔴   | 4    | code           | TASK-006, TASK-007 |
| 11  | TASK-011 | Restructure Adversarial Reviewer Agent      | 🟡   | 4    | code           | TASK-004, TASK-006 |
| 12  | TASK-012 | Restructure Knowledge Agent                 | 🔴   | 4    | code           | TASK-005, TASK-006 |
| 13  | TASK-013 | Restructure Implementer Agent               | 🟡   | 4    | code           | TASK-005, TASK-006 |
| 14  | TASK-014 | Restructure Verifier Agent                  | 🟡   | 3    | code           | TASK-003, TASK-005 |
| 15  | TASK-015 | Restructure Planner Agent                   | 🟢   | 3    | code           | TASK-002, TASK-003 |
| 16  | TASK-016 | Restructure Spec Agent                      | 🟢   | 3    | code           | TASK-002, TASK-003 |
| 17  | TASK-017 | Restructure Designer Agent                  | 🟢   | 5    | code           | TASK-003, TASK-008 |
| 18  | TASK-018 | Restructure Researcher Agent                | 🟢   | 5    | code           | TASK-003, TASK-005 |

---

## Execution Waves

### Wave 1 — Foundation (4 tasks, parallel)

Creates the core shared reference documents and instruction infrastructure that all subsequent tasks depend on.

| Task     | Files                                                                             | Risk | Est. Lines |
| -------- | --------------------------------------------------------------------------------- | ---- | ---------- |
| TASK-001 | `.github/copilot-instructions.md`, `.github/instructions/pipeline-conventions.md` | 🟢   | 100        |
| TASK-002 | `NewAgents/.github/agents/global-operating-rules.md`                              | 🟡   | 130        |
| TASK-003 | `NewAgents/.github/agents/tool-access-matrix.md`                                  | 🟡   | 110        |
| TASK-004 | `NewAgents/.github/agents/review-perspectives.md`                                 | 🟢   | 130        |

### Wave 2 — Templates & Schemas (4 tasks, parallel)

Creates SQL templates (with sanitization from SEC-1/SEC-2), updates schemas.md, and adds remaining reference docs.

| Task     | Files                                                    | Risk | Est. Lines |
| -------- | -------------------------------------------------------- | ---- | ---------- |
| TASK-005 | `context7-integration.md`, `evaluation-schema.md`        | 🟢   | 130        |
| TASK-006 | `NewAgents/.github/agents/sql-templates.md`              | 🟡   | 160        |
| TASK-007 | `NewAgents/.github/agents/schemas.md` (modify)           | 🟡   | 300        |
| TASK-009 | `NewAgents/.github/agents/severity-taxonomy.md` (modify) | 🟢   | 10         |

### Wave 3 — Simple Agent Modifications + Dispatch Update (4 tasks, parallel)

Low-risk agent restructures and dispatch-patterns update. These agents have minimal new features — primarily extraction to shared docs and frontmatter.

| Task     | Files                                                    | Risk | Est. Lines |
| -------- | -------------------------------------------------------- | ---- | ---------- |
| TASK-008 | `NewAgents/.github/agents/dispatch-patterns.md` (modify) | 🟡   | 50         |
| TASK-014 | `NewAgents/.github/agents/verifier.agent.md` (modify)    | 🟡   | 180        |
| TASK-015 | `NewAgents/.github/agents/planner.agent.md` (modify)     | 🟢   | 80         |
| TASK-016 | `NewAgents/.github/agents/spec.agent.md` (modify)        | 🟢   | 60         |

### Wave 4 — Core Agent Modifications (4 tasks, parallel)

Highest-risk wave. Orchestrator evidence gate rewrite, adversarial-reviewer perspective restructure, knowledge-agent 9-responsibility rewrite, implementer aggressive extraction.

| Task     | Files                                                          | Risk | Est. Lines |
| -------- | -------------------------------------------------------------- | ---- | ---------- |
| TASK-010 | `NewAgents/.github/agents/orchestrator.agent.md` (modify)      | 🔴   | 400        |
| TASK-011 | `NewAgents/.github/agents/adversarial-reviewer.agent.md` (mod) | 🟡   | 150        |
| TASK-012 | `NewAgents/.github/agents/knowledge-agent.agent.md` (modify)   | 🔴   | 250        |
| TASK-013 | `NewAgents/.github/agents/implementer.agent.md` (modify)       | 🟡   | 180        |

### Wave 5 — Final Agent Modifications (2 tasks, parallel)

Completes agent coverage. Designer gets glob-based verdict reading; researcher gets YAML-only output and Context7.

| Task     | Files                                                   | Risk | Est. Lines |
| -------- | ------------------------------------------------------- | ---- | ---------- |
| TASK-017 | `NewAgents/.github/agents/designer.agent.md` (modify)   | 🟢   | 50         |
| TASK-018 | `NewAgents/.github/agents/researcher.agent.md` (modify) | 🟢   | 60         |

---

## Dependency Graph

```
Wave 1 (no deps):
  TASK-001  ──────────────────────────────────────────────────────────────
  TASK-002  ──→ TASK-006 (W2) ──→ TASK-010 (W4), TASK-011 (W4), TASK-012 (W4), TASK-013 (W4)
           ├──→ TASK-007 (W2) ──→ TASK-010 (W4)
           ├──→ TASK-015 (W3)
           └──→ TASK-016 (W3)
  TASK-003  ──→ TASK-014 (W3), TASK-015 (W3), TASK-016 (W3), TASK-017 (W5), TASK-018 (W5)
  TASK-004  ──→ TASK-008 (W3) ──→ TASK-017 (W5)
           └──→ TASK-011 (W4)

Wave 2 (deps on W1):
  TASK-005  ──→ TASK-012 (W4), TASK-013 (W4), TASK-014 (W3), TASK-018 (W5)
  TASK-006  ──→ TASK-010 (W4), TASK-011 (W4), TASK-012 (W4), TASK-013 (W4)
  TASK-007  ──→ TASK-010 (W4)
  TASK-009  ──────────────────────────────────────────────────────────────

Wave 3 (deps on W1-2):
  TASK-008  ──→ TASK-017 (W5)
  TASK-014  (terminal)
  TASK-015  (terminal)
  TASK-016  (terminal)

Wave 4 (deps on W1-3):
  TASK-010  (terminal — orchestrator)
  TASK-011  (terminal — adversarial-reviewer)
  TASK-012  (terminal — knowledge-agent)
  TASK-013  (terminal — implementer)

Wave 5 (deps on W1-3):
  TASK-017  (terminal — designer)
  TASK-018  (terminal — researcher)
```

---

## Risk Summary

| Level | Count | Tasks                                     |
| ----- | ----- | ----------------------------------------- |
| 🟢    | 8     | 001, 004, 005, 009, 015, 016, 017, 018    |
| 🟡    | 8     | 002, 003, 006, 007, 008, 011, 013, 014    |
| 🔴    | 2     | 010 (orchestrator), 012 (knowledge-agent) |

**Overall Risk: 🔴** — Driven by orchestrator rewrite (evidence gates + telemetry + line reduction) and knowledge-agent rewrite (9 responsibilities + new SQL access grant).

---

## Planning Constraints (from Design Review)

1. **SEC-1/SEC-2 — SQL Sanitization**: `sql-templates.md` MUST include mandatory `sql_escape()` wrapper and shell injection prevention (stdin piping to sqlite3) BEFORE any agent uses those templates. TASK-006 precedes all SQL-writing agent tasks in wave ordering.

2. **CORR-1 — Evidence Gate Scope Filter**: All evidence gate queries include `task_id` filter to distinguish Step 3b (design review) from Step 7 (code review). Affects TASK-006 §6 and TASK-010.

3. **CORR-2 — EG-6 Majority Approval**: Use subquery checking ALL 3 category verdicts approve (`HAVING COUNT(CASE WHEN verdict!='approve' THEN 1 END)=0`). Affects TASK-006 §6 and TASK-010.

4. **CORR-3 — SQL Quoting Convention**: Document explicit quoting convention (NULL unquoted, `'value'` single-quoted). Affects TASK-006 §2.

5. **CORR-4 — Routing Matrix Location**: `schemas.md` is the canonical location; `global-operating-rules.md` §5 references it, no duplication. Affects TASK-002 and TASK-007.

6. **SEC-3 — Knowledge Agent Scope Consistency**: `run_in_terminal` scope is "SELECT on all tables + INSERT on instruction_updates only" — not SELECT-only. All occurrences must be consistent. Affects TASK-003 and TASK-012.

7. **ARCH-1 — Knowledge Agent Responsibilities**: Include responsibility inventory table mapping 9 responsibilities to justifications. Affects TASK-012.

8. **ARCH-2 — Schema Evolution Strategy**: SQLite schema evolution strategy in `sql-templates.md` §9. Affects TASK-006.

9. **ARCH-3 — Orchestrator Line Target**: Revised to 550 (from 480) with explicit NFR-1 exception. Affects TASK-010.

10. **SEC-5 — Safety Constraint Filter**: Formally defined in `global-operating-rules.md` §9 with immutable rules, protected phrases, pattern-matching rejection, evaluator separation. Affects TASK-002.

11. **SEC-6 — File Path Validation**: CHECK constraints on `instruction_updates.file_path` and `artifact_evaluations.artifact_path` in revised DDL. Affects TASK-006 and TASK-007.

---

## Pre-Mortem Analysis

| Task     | Failure Scenario                                                    | Likelihood | Impact | Mitigation                                                           |
| -------- | ------------------------------------------------------------------- | ---------- | ------ | -------------------------------------------------------------------- |
| TASK-002 | Global operating rules §9 safety filter definition insufficient     | M          | H      | Cross-reference SEC-5 findings; enumerate immutable rules explicitly |
| TASK-003 | Tool access matrix inconsistent with individual agent modifications | M          | M      | All agent tasks reference TASK-003 output; verify at each task       |
| TASK-006 | sql_escape() wrapper inadequate for all edge cases                  | M          | H      | Follow SEC-1/SEC-2 recommendations exactly; include test examples    |
| TASK-007 | schemas.md SQLite section conflicts with sql-templates.md DDL       | L          | M      | Single source of truth in sql-templates.md; schemas.md references    |
| TASK-010 | Orchestrator exceeds 550-line target after all additions            | H          | M      | Aggressive extraction to shared docs; measure during implementation  |
| TASK-010 | Evidence gate queries still allow Step 3b/7 conflation              | L          | H      | Explicit task_id in every query; CORR-1 constraint in task def       |
| TASK-011 | 3 perspectives produce insufficiently diverse reviews               | M          | M      | Maximally divergent personas with different severity thresholds      |
| TASK-012 | Knowledge Agent exceeds 345-line target with 9 responsibilities     | H          | M      | Extract details to shared docs; responsibility inventory is brief    |
| TASK-012 | Instruction update tracking conflicts with safety filter            | L          | H      | Safety filter defined first (TASK-002); KA references it             |
| TASK-013 | Implementer extraction loses critical inline context                | M          | M      | Verify all section pointers resolve; test with sample task           |
| TASK-014 | create_file scope restriction wording too permissive/restrictive    | L          | M      | Exact glob: `verification-reports/*.yaml`; match TASK-003 matrix     |

**Overall Risk Level:** High — driven by the orchestrator's combined evidence gate fix + telemetry wiring + line reduction, and the knowledge-agent's 9-responsibility rewrite with new SQL access patterns.

**Key Assumptions:**

1. VS Code auto-loads `.github/copilot-instructions.md` for all agents including orchestrator (if wrong: TASK-001 is ineffective, rules must be inlined) — affects TASK-001, all agent tasks.
2. sqlite3 CLI is available on target systems (if wrong: all SQL-based features fail) — affects TASK-006, 010, 011, 012, 013, 014.
3. Section pointer references (`See global-operating-rules.md §3`) are reliably followed by LLM agents (if wrong: shared doc extraction provides no benefit) — affects all agent tasks.
4. The 3-tier content architecture (global → shared → inline) achieves target line counts (if wrong: agents remain over 350 lines) — affects TASK-010, 012, 013, 014.
5. Design-output.yaml revision addressed all critical/major review findings adequately (if wrong: implementation encodes flawed design) — affects all tasks.
