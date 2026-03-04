# Plan: TDD & E2E Testing Enforcement

**Feature Slug:** `tdd-e2e-enforcement`
**Run ID:** `2026-03-04T00:00:00Z`
**Planning Mode:** Revision (Targeted — Playwright CLI Integration)
**Overall Risk:** 🔴
**Total Tasks:** 16
**Total Waves:** 4

---

## Revision History

| Date              | Type     | Reason                                                                                                                | Affected Tasks                                   |
| ----------------- | -------- | --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| 2026-03-04T10:30Z | Targeted | Integrate `@playwright/cli` (github.com/microsoft/playwright-cli) with built-in SKILLS system for browser E2E testing | TASK-001, TASK-004, TASK-008, TASK-009, TASK-013 |

**Revision Detail:** User research confirmed the correct browser automation tool is `@playwright/cli` (NOT `@playwright/mcp`). Key properties: `playwright-cli` CLI binary, `playwright-cli install --skills` for agent-discoverable skill guides, `-s=<session-name>` for session isolation, `.playwright/cli.config.json` for per-project configuration. Token-efficient by design — purpose-built CLI commands avoid dumping page/accessibility data into LLM context. Officially recommended by Microsoft for coding agents.

---

## Feature Overview

Massive overhaul of the agent system (`NewAgents/.github/agents/`) to make TDD and E2E testing core enforcement mechanisms. The design (Direction C — Hybrid Extended Verifier + Reference Document, revised R2) was reviewed by 3 adversarial reviewers; all 3 Critical, 18 Major, and 12 Minor findings were addressed in revision round 2.

**Core changes:**

- Mandatory RED-GREEN-VERIFY TDD cycle with independent verification
- Verifier gains Tier 5 E2E verification with live app interaction (browser/API/CLI)
- Skills system with step-by-step interaction procedures (pre-defined, deterministic)
- **Playwright CLI (`@playwright/cli`) for browser E2E** — `playwright-cli` commands (open/goto/click/fill/screenshot) with session isolation (`-s=verify-{task-id}`), installed skills (`playwright-cli install --skills`), token-efficient design
- Script generation retained as fallback for complex test suites (Playwright .spec.ts from YAML skill definitions)
- Risk-based workflow lanes (unit-only / unit-integration / full-tdd-e2e)
- Structured E2E contract commands (not raw shell) for security
- Evidence sanitization pipeline (SQL escape, HAR stripping, env scrubbing)
- New `e2e-integration.md` reference document (~500 lines)

---

## Success Criteria (from spec-output.yaml)

| ID    | Criterion                                                                             | Test Method   |
| ----- | ------------------------------------------------------------------------------------- | ------------- |
| AC-1  | TDD evidence in implementation report (tests_written_first, initial_run_failures > 0) | test          |
| AC-2  | tdd-compliance SQL check is blocking for code tasks                                   | test          |
| AC-5  | E2E contract schema complete in schemas.md                                            | inspection    |
| AC-9  | E2E skills schema complete in schemas.md                                              | inspection    |
| AC-10 | Verifier performs actual exploratory interaction (not just test commands)             | test          |
| AC-11 | Per-variation adversarial SQL records                                                 | test          |
| AC-13 | Implementer prohibited from E2E execution                                             | inspection    |
| AC-14 | All E2E lifecycle phases recorded as SQL INSERTs                                      | test          |
| AC-23 | Every task has workflow_lane field                                                    | inspection    |
| AC-30 | Orchestrator run_in_terminal restricted to SQLite/git                                 | inspection    |
| AC-33 | E2E completion gate blocks task completion when e2e_required                          | test          |
| AC-38 | Evidence proves actual app interaction (screenshots, response logs)                   | demonstration |

---

## Ordered Task Index

| #   | Task ID  | Title                                                       | Risk | Size     | Wave | Files |
| --- | -------- | ----------------------------------------------------------- | ---- | -------- | ---- | ----- |
| 1   | TASK-001 | schemas.md — E2E Contract/Skills/Task Schema Extension      | 🟡   | Standard | 1    | 1     |
| 2   | TASK-002 | sql-templates.md — E2E Evidence Gates + TDD Compliance Gate | 🟡   | Standard | 1    | 1     |
| 3   | TASK-003 | global-operating-rules.md — TDD/E2E Safety Rules            | 🟡   | Standard | 1    | 1     |
| 4   | TASK-004 | e2e-integration.md — NEW E2E Reference Document             | 🔴   | Large    | 1    | 1     |
| 5   | TASK-005 | implementer.agent.md — RED-GREEN-VERIFY TDD Cycle           | 🔴   | Large    | 2    | 1     |
| 6   | TASK-006 | verifier.agent.md — TDD Compliance Verification             | 🔴   | Large    | 2    | 1     |
| 7   | TASK-007 | orchestrator.agent.md — E2E Discovery + Lane Gates          | 🔴   | Large    | 2    | 1     |
| 8   | TASK-008 | tool-access-matrix.md — Tier 5 Command Allowlist            | 🟡   | Standard | 2    | 1     |
| 9   | TASK-009 | verifier.agent.md — Tier 5 E2E Lifecycle (5-Phase)          | 🔴   | Large    | 3    | 1     |
| 10  | TASK-010 | planner.agent.md — Workflow Lane Derivation                 | 🟡   | Standard | 3    | 1     |
| 11  | TASK-011 | dispatch-patterns.md — E2E Concurrency + Port Protocol      | 🟡   | Standard | 3    | 1     |
| 12  | TASK-012 | adversarial-reviewer.agent.md — TDD/E2E Review Dimensions   | 🟡   | Standard | 3    | 1     |
| 13  | TASK-013 | researcher.agent.md — Skill Creation Workflow               | 🟡   | Standard | 4    | 1     |
| 14  | TASK-014 | evaluation-schema.md — E2E Evaluation Criteria              | 🟢   | Standard | 4    | 1     |
| 15  | TASK-015 | review-perspectives.md — E2E Review Guidance                | 🟢   | Standard | 4    | 1     |
| 16  | TASK-016 | severity-taxonomy.md — E2E Failure Severity Examples        | 🟢   | Standard | 4    | 1     |

---

## Execution Waves

### Wave 1: Foundation (4 tasks, parallel)

No dependencies. Establishes schemas, SQL templates, global rules, and the E2E reference document that all downstream tasks reference.

| Task     | Target                    | Risk | Est. Lines | Dependencies |
| -------- | ------------------------- | ---- | ---------- | ------------ |
| TASK-001 | schemas.md                | 🟡   | ~290       | none         |
| TASK-002 | sql-templates.md          | 🟡   | ~155       | none         |
| TASK-003 | global-operating-rules.md | 🟡   | ~40        | none         |
| TASK-004 | e2e-integration.md (NEW)  | 🔴   | ~500       | none         |

### Wave 2: Core Agents (4 tasks, parallel)

Depends on Wave 1 foundation. Modifies the three critical agents (implementer, verifier, orchestrator) and the tool access matrix.

| Task     | Target                  | Risk | Est. Lines | Dependencies       |
| -------- | ----------------------- | ---- | ---------- | ------------------ |
| TASK-005 | implementer.agent.md    | 🔴   | ~100       | TASK-001, TASK-003 |
| TASK-006 | verifier.agent.md (TDD) | 🔴   | ~60        | TASK-001, TASK-002 |
| TASK-007 | orchestrator.agent.md   | 🔴   | ~27        | TASK-002, TASK-004 |
| TASK-008 | tool-access-matrix.md   | 🟡   | ~35        | TASK-004           |

### Wave 3: Extended Agents (4 tasks, parallel)

Adds verifier Tier 5 E2E lifecycle, planner workflow lanes, dispatch concurrency rules, and adversarial review dimensions.

| Task     | Target                        | Risk | Est. Lines | Dependencies       |
| -------- | ----------------------------- | ---- | ---------- | ------------------ |
| TASK-009 | verifier.agent.md (Tier 5)    | 🔴   | ~180       | TASK-004, TASK-006 |
| TASK-010 | planner.agent.md              | 🟡   | ~45        | TASK-001           |
| TASK-011 | dispatch-patterns.md          | 🟡   | ~50        | TASK-004           |
| TASK-012 | adversarial-reviewer.agent.md | 🟡   | ~50        | TASK-001           |

### Wave 4: Supplemental (4 tasks, parallel)

Low-risk additive changes to supporting reference documents and the researcher agent.

| Task     | Target                 | Risk | Est. Lines | Dependencies |
| -------- | ---------------------- | ---- | ---------- | ------------ |
| TASK-013 | researcher.agent.md    | 🟡   | ~30        | TASK-004     |
| TASK-014 | evaluation-schema.md   | 🟢   | ~20        | none         |
| TASK-015 | review-perspectives.md | 🟢   | ~30        | TASK-012     |
| TASK-016 | severity-taxonomy.md   | 🟢   | ~20        | none         |

---

## Dependency Graph (DAG)

```
Wave 1 (Foundation):
  TASK-001 (schemas) ──────────────┬─→ TASK-005 (implementer)
                                   ├─→ TASK-006 (verifier TDD)
                                   ├─→ TASK-010 (planner)
                                   └─→ TASK-012 (adversarial)

  TASK-002 (sql-templates) ────────┬─→ TASK-006 (verifier TDD)
                                   └─→ TASK-007 (orchestrator)

  TASK-003 (global-rules) ────────────→ TASK-005 (implementer)

  TASK-004 (e2e-integration) ──────┬─→ TASK-007 (orchestrator)
                                   ├─→ TASK-008 (tool-access)
                                   ├─→ TASK-009 (verifier Tier 5)
                                   ├─→ TASK-011 (dispatch)
                                   └─→ TASK-013 (researcher)

Wave 2 (Core):
  TASK-006 (verifier TDD) ────────────→ TASK-009 (verifier Tier 5)

Wave 3 (Extended):
  TASK-012 (adversarial) ─────────────→ TASK-015 (review-perspectives)
```

**Critical path:** TASK-004 → TASK-006 → TASK-009 (e2e-integration → verifier TDD → verifier Tier 5)

---

## Risk Summary

### Overall: 🔴 (High)

**Justification:** 5 of 16 tasks are 🔴 (critical risk), including all three core agent modifications (implementer, verifier, orchestrator) and the foundational E2E reference document. The critical path runs through the verifier's Tier 5 E2E lifecycle — the most complex single change in the system.

### Per-Task Risk Breakdown

| Risk              | Count | Tasks                                                                          |
| ----------------- | ----- | ------------------------------------------------------------------------------ |
| 🔴 Critical       | 5     | TASK-004, TASK-005, TASK-006, TASK-007, TASK-009                               |
| 🟡 Business Logic | 8     | TASK-001, TASK-002, TASK-003, TASK-008, TASK-010, TASK-011, TASK-012, TASK-013 |
| 🟢 Additive       | 3     | TASK-014, TASK-015, TASK-016                                                   |

### 🔴 File Risk Rationale

| File                  | Risk | Rationale                                                                                                                               |
| --------------------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------- |
| e2e-integration.md    | 🔴   | Contains command sanitization rules (D-21), command allowlist (D-22), evidence sanitization pipeline (D-25) — security-critical content |
| implementer.agent.md  | 🔴   | Modifies TDD enforcement (core compliance mechanism), adds E2E prohibition — behavioral correctness                                     |
| verifier.agent.md     | 🔴   | Adds Tier 5 E2E with process management, script generation, PID tracking — most complex change                                          |
| orchestrator.agent.md | 🔴   | Modifies evidence gates (routing decisions), contract validation, line budget at 543/550 — high blast radius                            |

---

## Design Discrepancy Note

The design's `file_inventory` lists `researcher.agent.md` as **unchanged**, but `agent_modification_map` includes it with changes for D-19 (skill creation workflow). The planner follows the `agent_modification_map` since D-19 explicitly requires researcher modifications. Implementer for TASK-013 should note this discrepancy and verify with the design document.

---

## Pre-Mortem Analysis

| Task     | Failure Scenario                                                                           | Likelihood | Impact | Mitigation                                                                                         |
| -------- | ------------------------------------------------------------------------------------------ | ---------- | ------ | -------------------------------------------------------------------------------------------------- |
| TASK-004 | e2e-integration.md exceeds 500-line estimate, causing context pressure for agents          | M          | H      | Per-agent reading guide (D-2) limits per-agent context load; sections are independently consumable |
| TASK-005 | TDD cycle description conflicts with existing implementer Steps 4-6, causing inconsistency | M          | H      | Implementer must replace Steps 4-6 entirely (not patch), verify no orphaned references             |
| TASK-006 | TDD compliance secondary heuristics (D-13) too strict, causing false failures              | L          | M      | Secondary heuristics are warnings only (not blocking) — audit evidence for adversarial review      |
| TASK-007 | Orchestrator exceeds 550-line budget after changes (target 543/550, 7-line margin)         | H          | H      | EG-9 sub-phase checks can be moved to verifier output (saves ~3 lines) per D-16 analysis           |
| TASK-009 | Verifier Tier 5 section too large, pushing verifier past maintainable size                 | M          | M      | Sub-role organization (D-24) with conditional loading keeps non-E2E verification lean              |
| TASK-009 | Script generation logic (D-20) ambiguous for non-Playwright frameworks                     | M          | M      | Skills schema and contract specify runner; script generation is framework-parameterized            |
| TASK-001 | Schema additions conflict with existing schema numbering or field overlap                  | L          | M      | Additive-only schema changes (CR-5); new fields have sensible defaults                             |
| TASK-002 | EG-2/EG-10 consolidation breaks existing orchestrator gate queries                         | M          | H      | TASK-007 must update orchestrator in same wave as TASK-002 creates new SQL templates               |
| TASK-011 | E2E concurrency cap conflicts with existing 4-agent dispatch cap                           | L          | M      | E2E cap is a sub-limit within the existing 4-agent cap, not a replacement                          |
| TASK-013 | Researcher modifications contradict file_inventory "unchanged" listing                     | L          | L      | Implementer should verify D-19 intent and implement if modification map is authoritative           |

### Overall Risk Level: High

**Justification:** The orchestrator line budget (7-line margin at 543/550) is the highest-risk constraint. Any implementation imprecision in TASK-007 could trigger a full refactoring. The verifier's Tier 5 lifecycle (TASK-009) is the most complex individual change — integrating process management, script generation, evidence capture, and sanitization into a single agent.

### Key Assumptions

1. **Orchestrator 528-line baseline is accurate** — If current orchestrator is larger than 528 lines, the 543/550 target is unachievable without more aggressive extraction. (Affects: TASK-007)
2. **`@playwright/cli` is the browser E2E tool** — Browser interaction uses `playwright-cli` CLI commands (open, goto, click, fill, screenshot) with session isolation (`-s=verify-{task-id}`). Skills are installed via `playwright-cli install --skills`. Script generation (D-20) is retained as fallback for complex suites. If the target project uses a non-browser runner, the CLI command model does not apply. (Affects: TASK-001, TASK-004, TASK-008, TASK-009, TASK-013)
3. **Agent markdown files are treated as production code** — TDD applies to agent file modifications via structural validation checks. (Affects: TASK-005, TASK-006, all tasks)
4. **No E2E contract exists for the agent system itself** — All tasks have e2e_required=false. Verification is via structural inspection, not live app testing. (Affects: all tasks)
5. **Schema changes are purely additive** — No existing schema fields are removed or renamed (CR-5). (Affects: TASK-001)
6. **Design R2 addressed all review findings** — The 26 decisions in design-output.yaml are final. No further design revision expected. (Affects: all tasks)
