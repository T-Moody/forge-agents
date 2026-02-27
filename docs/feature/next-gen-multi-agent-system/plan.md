# Plan: Next-Gen Multi-Agent System

## Title & Feature Overview

Implement the complete Next-Generation Deterministic Multi-Agent Pipeline for VS Code GitHub Copilot as specified in design.md v4. The system consists of 9 agent definitions, 3 reference documents, 1 prompt file, and 1 README — 14 files total, all output to `NewAgents/`.

**Planning Mode:** Initial (fresh plan from scratch — previous plan scrapped).

**Output Location:** All agent files → `NewAgents/.github/agents/`, prompts → `NewAgents/.github/prompts/`, README → `NewAgents/README.md`.

---

## Success Criteria

Mapped from design.md §Implementation Checklist and feature.md §Acceptance Criteria:

| ID    | Criterion                                                                                 | Verified By        |
| ----- | ----------------------------------------------------------------------------------------- | ------------------ |
| AC-2  | 9 unique agent definitions (within 4–15 range)                                            | File count         |
| AC-3  | Every agent has a completion contract (DONE / NEEDS_REVISION / ERROR)                     | Grep all .agent.md |
| AC-7  | All agents use unified severity taxonomy (Blocker/Critical/Major/Minor)                   | Grep all .agent.md |
| AC-8  | All agents define typed YAML output schemas referencing schemas.md                        | Grep all .agent.md |
| AC-12 | All agents have self-verification section                                                 | Grep all .agent.md |
| AC-14 | Correct file locations: agents in `.github/agents/`, prompt in `.github/prompts/`, README | Directory listing  |
| CR-12 | Anti-drift anchor in every agent                                                          | Grep all .agent.md |
| P0    | schemas.md, dispatch-patterns.md, severity-taxonomy.md created before agent definitions   | Wave ordering      |
| P1    | All 9 agent .agent.md files + feature-workflow.prompt.md created                          | File listing       |
| P2    | README.md with Mermaid pipeline diagram created                                           | File inspection    |

---

## Ordered Task Index

| #   | Task ID                       | File(s) Produced                                                        | Priority | Agent                |
| --- | ----------------------------- | ----------------------------------------------------------------------- | -------- | -------------------- |
| 01  | 01-schemas-reference          | `NewAgents/.github/agents/schemas.md`                                   | P0       | implementer          |
| 02  | 02-reference-documents        | `NewAgents/.github/agents/dispatch-patterns.md`, `severity-taxonomy.md` | P0       | implementer          |
| 03  | 03-researcher-agent           | `NewAgents/.github/agents/researcher.agent.md`                          | P1       | implementer          |
| 04  | 04-spec-agent                 | `NewAgents/.github/agents/spec.agent.md`                                | P1       | implementer          |
| 05  | 05-designer-agent             | `NewAgents/.github/agents/designer.agent.md`                            | P1       | implementer          |
| 06  | 06-planner-agent              | `NewAgents/.github/agents/planner.agent.md`                             | P1       | implementer          |
| 07  | 07-implementer-agent          | `NewAgents/.github/agents/implementer.agent.md`                         | P1       | implementer          |
| 08  | 08-verifier-agent             | `NewAgents/.github/agents/verifier.agent.md`                            | P1       | implementer          |
| 09  | 09-adversarial-reviewer-agent | `NewAgents/.github/agents/adversarial-reviewer.agent.md`                | P1       | implementer          |
| 10  | 10-knowledge-agent            | `NewAgents/.github/agents/knowledge-agent.agent.md`                     | P1       | implementer          |
| 11  | 11-orchestrator-agent         | `NewAgents/.github/agents/orchestrator.agent.md`                        | P0       | implementer          |
| 12  | 12-feature-workflow-prompt    | `NewAgents/.github/prompts/feature-workflow.prompt.md`                  | P1       | implementer          |
| 13  | 13-readme-documentation       | `NewAgents/README.md`                                                   | P2       | documentation-writer |

---

## Execution Waves

### Wave 1 — Foundation (parallel)

- **01-schemas-reference** (depends_on: none) — All 10 typed YAML schemas with producer/consumer table, examples, evolution strategy
- **02-reference-documents** (depends_on: none) — Dispatch Patterns A/B/C + Severity Taxonomy (Blocker/Critical/Major/Minor)

### Wave 2 — Pipeline Agents (parallel, sub-waves of ≤4 for concurrency cap)

- **03-researcher-agent** (depends_on: 01, 02)
- **04-spec-agent** (depends_on: 01, 02)
- **05-designer-agent** (depends_on: 01, 02)
- **06-planner-agent** (depends_on: 01, 02)
- **07-implementer-agent** (depends_on: 01, 02)
- **08-verifier-agent** (depends_on: 01, 02)
- **09-adversarial-reviewer-agent** (depends_on: 01, 02)
- **10-knowledge-agent** (depends_on: 01, 02)

> **Sub-wave note:** With a concurrency cap of 4, Wave 2 executes as 2 sub-waves: Tasks 03–06, then Tasks 07–10. All 8 are independent of each other.

### Wave 3 — Orchestrator + Prompt (parallel)

- **11-orchestrator-agent** (depends_on: 01, 02) — Most complex agent; references all other agents by name; placed after Wave 2 for consistency
- **12-feature-workflow-prompt** (depends_on: 01) — Entry point prompt binding to orchestrator

### Wave 4 — Documentation

- **13-readme-documentation** (depends_on: 11, 12) — Pipeline overview + Mermaid diagram referencing all agents

---

## Dependency Graph

```
Wave 1:  [01-schemas]  [02-references]
              │  │           │  │
              ▼  ▼           ▼  ▼
Wave 2:  [03] [04] [05] [06] [07] [08] [09] [10]   (all depend on 01 + 02)
              │                    │
              ▼                    ▼
Wave 3:  [11-orchestrator]  [12-prompt]              (11 depends on 01,02; 12 depends on 01)
              │                  │
              ▼                  ▼
Wave 4:  [13-readme]                                 (depends on 11, 12)
```

---

## Implementation Specification

### Code Structure Overview

All files are new (greenfield build at `NewAgents/`):

```
NewAgents/
├── .github/
│   ├── agents/
│   │   ├── schemas.md                        # Task 01: 10 typed YAML schemas
│   │   ├── dispatch-patterns.md              # Task 02: Pattern A/B/C
│   │   ├── severity-taxonomy.md              # Task 02: Blocker/Critical/Major/Minor
│   │   ├── researcher.agent.md               # Task 03
│   │   ├── spec.agent.md                     # Task 04
│   │   ├── designer.agent.md                 # Task 05
│   │   ├── planner.agent.md                  # Task 06
│   │   ├── implementer.agent.md              # Task 07
│   │   ├── verifier.agent.md                 # Task 08
│   │   ├── adversarial-reviewer.agent.md     # Task 09
│   │   ├── knowledge-agent.agent.md          # Task 10
│   │   └── orchestrator.agent.md             # Task 11
│   └── prompts/
│       └── feature-workflow.prompt.md        # Task 12
└── README.md                                  # Task 13
```

### Source References (from design.md §Migration)

| Source File                                  | Purpose for Implementers                                       |
| -------------------------------------------- | -------------------------------------------------------------- |
| `.github/agents/orchestrator.agent.md`       | Dispatch patterns, cluster decision logic, tool restrictions   |
| `.github/agents/researcher.agent.md`         | Research focus partitioning, output format                     |
| `.github/agents/planner.agent.md`            | Task decomposition, dependency-aware waves, pre-mortem         |
| `.github/agents/implementer.agent.md`        | TDD workflow, code quality principles                          |
| `.github/agents/v-build.agent.md`            | Build system detection logic                                   |
| `.github/agents/v-tests.agent.md`            | Test execution patterns                                        |
| `.github/agents/r-knowledge.agent.md`        | Knowledge evolution, decision logging, safety filter           |
| `Anvil/anvil.agent.md`                       | Verification cascade, SQL ledger, adversarial review, pushback |
| `.github/agents/dispatch-patterns.md`        | Pattern A/B/C formal definitions                               |
| `.github/agents/evaluation-schema.md`        | YAML schema pattern (template for schemas.md)                  |
| `.github/prompts/feature-workflow.prompt.md` | Entry point prompt pattern                                     |

### Agent Definition Template (from design.md §Decision 10)

Every `.agent.md` file MUST follow this structure:

1. Header block (Type, Pipeline Step, Inputs, Outputs)
2. Role & Purpose
3. Input Schema (referencing schemas.md)
4. Output Schema (referencing schemas.md)
5. Workflow (ordered steps)
6. Completion Contract (DONE / NEEDS_REVISION / ERROR)
7. Operating Rules
8. Self-Verification
9. Tool Access (allowed tools list)
10. Anti-Drift Anchor

### Key Design Constraints for All Tasks

- **Typed YAML schemas:** All agent I/O contracts defined in schemas.md (Task 01)
- **SQLite verification ledger:** `anvil_checks` table as PRIMARY evidence store (WAL + busy_timeout mandatory)
- **Unified severity:** Blocker/Critical/Major/Minor as defined in severity-taxonomy.md (Task 02)
- **Completion contracts:** Every agent returns exactly DONE | NEEDS_REVISION | ERROR
- **Tool restrictions:** Advisory (prompt-level only, no runtime enforcement — honest limitation)
- **Orchestrator `run_in_terminal` scope:** SQLite read queries, DDL at Step 0, git read/commit only (DR-1)
- **Always 3 adversarial reviewers:** No risk-driven reviewer count variance (DR-2)
- **Per-task Verifier dispatch:** One Verifier per completed task, not per wave

---

## Risks & Mitigations

| Risk                                                  | Likelihood | Impact | Mitigation                                                                            |
| ----------------------------------------------------- | ---------- | ------ | ------------------------------------------------------------------------------------- |
| schemas.md exceeds context window for implementer     | Low        | Medium | Split into focused sections; use design.md §Schemas Reference as direct source        |
| Orchestrator prompt too long (references all agents)  | Medium     | Medium | Keep references by name/path; delegate details to schemas.md and dispatch-patterns.md |
| Agent definitions inconsistent with each other        | Medium     | High   | Wave ordering ensures shared refs exist first; agent template enforced per design     |
| Verifier agent complexity (tiered cascade + SQL)      | Medium     | Medium | Large design section provides detailed guidance; implementer references Anvil source  |
| Mermaid diagram in README drifts from actual pipeline | Low        | Low    | README task depends on orchestrator; uses design.md §Decision 10 Mermaid as source    |

---

## Plan Validation

### Circular Dependency Check

Dependency graph walk:

- Wave 1: 01, 02 → no dependencies → OK
- Wave 2: 03–10 → depend on 01, 02 (Wave 1) → no cycles
- Wave 3: 11 → depends on 01, 02 (Wave 1) → no cycles; 12 → depends on 01 (Wave 1) → no cycles
- Wave 4: 13 → depends on 11, 12 (Wave 3) → no cycles

**Result: No circular dependencies.**

### Task Size Validation

| Task | Files | Dependencies | Est. Lines | Effort |
| ---- | ----- | ------------ | ---------- | ------ |
| 01   | 1     | 0            | ~450       | Medium |
| 02   | 2     | 0            | ~250       | Low    |
| 03   | 1     | 2            | ~300       | Medium |
| 04   | 1     | 2            | ~350       | Medium |
| 05   | 1     | 2            | ~300       | Medium |
| 06   | 1     | 2            | ~350       | Medium |
| 07   | 1     | 2            | ~400       | Medium |
| 08   | 1     | 2            | ~500       | Medium |
| 09   | 1     | 2            | ~400       | Medium |
| 10   | 1     | 2            | ~300       | Medium |
| 11   | 1     | 2            | ~500       | Medium |
| 12   | 1     | 1            | ~100       | Low    |
| 13   | 1     | 2            | ~200       | Low    |

**Result: All tasks within limits (≤3 files, ≤2 dependencies, ≤500 lines, ≤Medium effort).**

### Dependency Existence Check

All `depends_on` references point to tasks that exist in this plan:

- 01, 02: none → OK
- 03–10: 01, 02 → exist
- 11: 01, 02 → exist
- 12: 01 → exists
- 13: 11, 12 → exist

**Result: All dependencies valid.**

---

## Pre-Mortem Analysis

| Task                          | Failure Scenario                                                         | Likelihood | Impact | Mitigation                                                                    |
| ----------------------------- | ------------------------------------------------------------------------ | ---------- | ------ | ----------------------------------------------------------------------------- |
| 01-schemas-reference          | Schema definitions inconsistent with design.md dual schema sections      | Medium     | High   | Implementer must reconcile §Decision 6 and §Data Storage schemas (R2 finding) |
| 02-reference-documents        | Dispatch pattern definitions too vague for orchestrator to use           | Low        | Medium | Reference existing `.github/agents/dispatch-patterns.md` as template          |
| 03-researcher-agent           | Missing typed YAML output schema details                                 | Low        | Low    | schemas.md (01) provides full schema; design §Decision 2 has agent detail     |
| 04-spec-agent                 | Pushback system underspecified in agent definition                       | Low        | Medium | design.md §Decision 2 (Spec detail) + Anvil pushback pattern                  |
| 05-designer-agent             | Justification scoring format unclear                                     | Low        | Low    | design.md §Decision 8 has explicit decision-record schema                     |
| 06-planner-agent              | relevant_context pointer mechanism underspecified                        | Medium     | Medium | design.md §Decision 5 has example task schema with relevant_context           |
| 07-implementer-agent          | Self-fix loop + git staging + revert mode complexity                     | Medium     | Medium | design.md v4 H3/H8/H9 define each mechanism; Anvil source for baseline        |
| 08-verifier-agent             | Most complex agent — 4-tier cascade + SQL + baseline cross-check         | Medium     | High   | Per-task dispatch bounds context; Anvil source provides cascade pattern       |
| 09-adversarial-reviewer-agent | YAML verdict output + SQL INSERT + review_focus parameterization complex | Medium     | Medium | design.md §Decision 2 (AR detail) has full schema and focus definitions       |
| 10-knowledge-agent            | Evidence bundle assembly (Step 8b) underspecified                        | Low        | Low    | Non-blocking; design.md §Pipeline Overview Step 8b defines components         |
| 11-orchestrator-agent         | Decision table + SQL queries + tool restrictions make this the longest   | High       | High   | Delegate schema details to schemas.md; keep orchestrator focused on routing   |
| 12-feature-workflow-prompt    | Prompt format incompatible with VS Code agent framework                  | Low        | Medium | Reference existing `.github/prompts/feature-workflow.prompt.md` as template   |
| 13-readme-documentation       | Mermaid diagram inconsistent with actual pipeline                        | Low        | Low    | Copy Mermaid from design.md §Decision 10; verify step names match agents      |

### Overall Risk Level

**Medium** — The orchestrator (Task 11) and verifier (Task 08) are the two highest-complexity tasks. Both have extensive design specification but require careful synthesis of multiple design sections. The dual schema definitions in design.md (§Decision 6 vs §Data Storage) are a known inconsistency that must be reconciled during Task 01.

### Key Assumptions

| Assumption                                                             | Impact if Wrong                                    | Dependent Tasks |
| ---------------------------------------------------------------------- | -------------------------------------------------- | --------------- |
| `.agent.md` is the correct VS Code agent definition format             | All 9 agent files would need format changes        | 03–11           |
| VS Code `runSubagent` supports concurrent dispatch up to 4             | Dispatch patterns would need restructuring         | 02, 11          |
| SQLite is available on target Windows system without installation      | Verification architecture needs YAML-only fallback | 01, 08, 11      |
| LLMs can reliably produce well-formed YAML matching schema definitions | Retry/fallback logic becomes critical path         | 01, all agents  |
| `run_in_terminal` is available to orchestrator agent                   | Evidence gate verification impossible (DR-1 fails) | 11              |

---

## Deliverables & Acceptance

### Completion Criteria

All 14 files created at correct paths under `NewAgents/`:

- 9 `.agent.md` files in `NewAgents/.github/agents/`
- 3 `.md` reference documents in `NewAgents/.github/agents/`
- 1 `.prompt.md` file in `NewAgents/.github/prompts/`
- 1 `README.md` at `NewAgents/`

### Verification Checklist

- [ ] All 9 agents have: completion contract, self-verification, anti-drift anchor, tool access list
- [ ] All agents reference schemas.md for their output schema
- [ ] All agents use unified severity taxonomy (Blocker/Critical/Major/Minor)
- [ ] Orchestrator tool restrictions match DR-1 (no create_file, no grep_search, etc.)
- [ ] Orchestrator decision table covers all routing conditions from design.md
- [ ] Verifier includes 4-tier cascade with SQL evidence ledger
- [ ] Adversarial reviewer supports review_focus parameterization (security/architecture/correctness)
- [ ] schemas.md contains all 10 schemas with producer/consumer table
- [ ] README includes Mermaid pipeline diagram matching design.md
- [ ] feature-workflow.prompt.md binds to orchestrator with APPROVAL_MODE variable
