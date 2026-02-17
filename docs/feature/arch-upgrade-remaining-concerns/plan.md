# Plan: Architecture Upgrade Remaining Concerns

**Planning Mode:** Initial — no prior plan or verifier exists.

---

## Feature Overview

Five targeted changes to the agent architecture addressing post-upgrade concerns:

1. **Change 1 — Remove Memory Line Limits:** Eliminate the self-defeating 200-line emergency prune cap from `orchestrator.agent.md`. Replace with explicit structural growth controls (existing checkpoint pruning).
2. **Change 3 — Memory Write Safety Markers:** Add `memory_access` YAML field to 21 agent files (15 read-only, 6 read-write). Add validation logic at 5 orchestrator dispatch points. Update documentation-writer to read-only.
3. **Change 4 — Scope r-knowledge Inputs:** Reduce r-knowledge's orchestrator-provided inputs from 6 artifacts to 2 (`memory.md` + `initial-request.md`). Navigate others via Artifact Index.
4. **Change 2 — Reduce Orchestrator Complexity:** Extract Patterns A and B to `NewAgentsAndPrompts/dispatch-patterns.md`. Compress orchestrator to ≤400 lines.
5. **Change 5 — README Update:** Fix stale researcher counts (×3→×4), update project layout.

**Application order:** 1 → 3 → 4 → 2 → 5 (per design — minimizes merge conflicts on `orchestrator.agent.md`).

---

## Success Criteria

| ID    | Criterion                                                               | Source         |
| ----- | ----------------------------------------------------------------------- | -------------- |
| SC-1  | `emergency prune` string absent from `orchestrator.agent.md`            | AC-1.1, AC-1.2 |
| SC-2  | Memory Lifecycle Actions table has exactly 6 rows                       | AC-1.3         |
| SC-3  | `orchestrator.agent.md` ≤400 lines                                      | AC-2.1         |
| SC-4  | `NewAgentsAndPrompts/dispatch-patterns.md` exists with Patterns A and B | AC-2.2         |
| SC-5  | 15 agents have `memory_access: read-only` in YAML                       | AC-3.1, AC-3.3 |
| SC-6  | 6 agents have `memory_access: read-write` in YAML                       | AC-3.2         |
| SC-7  | Orchestrator has validation logic at 5 dispatch points                  | AC-3.4         |
| SC-8  | r-knowledge dispatch row uses `tier, initial-request.md, memory.md`     | AC-4.1         |
| SC-9  | r-knowledge Inputs section uses Artifact Index navigation               | AC-4.2, AC-4.3 |
| SC-10 | README reflects ×4 researchers and full project layout                  | AC-5.1, AC-5.2 |

---

## Ordered Task Index

| #   | Task                                                                                             | Files                                                            | Change | Effort |
| --- | ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------- | ------ | ------ |
| 01  | Remove emergency prune from orchestrator                                                         | 1 (orchestrator.agent.md)                                        | 1      | Low    |
| 02  | Add `memory_access: read-only` to CT sub-agents                                                  | 4 (ct-security, ct-scalability, ct-maintainability, ct-strategy) | 3      | Low    |
| 03  | Add `memory_access: read-only` to V sub-agents                                                   | 4 (v-build, v-tests, v-tasks, v-feature)                         | 3      | Low    |
| 04  | Add `memory_access: read-only` to R sub-agents                                                   | 4 (r-quality, r-security, r-testing, r-knowledge)                | 3      | Low    |
| 05  | Add `memory_access: read-write` to aggregator agents                                             | 3 (ct-aggregator, v-aggregator, r-aggregator)                    | 3      | Low    |
| 06  | Add `memory_access: read-write` to sequential agents                                             | 3 (spec, designer, planner)                                      | 3      | Low    |
| 07  | Add `memory_access: read-only` to implementer, researcher, doc-writer + update doc-writer step 7 | 3 (implementer, researcher, documentation-writer)                | 3      | Low    |
| 08  | Add memory_access validation logic to orchestrator + update Step 5.2                             | 1 (orchestrator.agent.md)                                        | 3      | Medium |
| 09  | Create dispatch-patterns.md with Patterns A and B                                                | 1 (dispatch-patterns.md — new)                                   | 2      | Low    |
| 10  | Update r-knowledge.agent.md inputs and workflow                                                  | 1 (r-knowledge.agent.md)                                         | 4      | Low    |
| 11  | Update orchestrator r-knowledge dispatch row                                                     | 1 (orchestrator.agent.md)                                        | 4      | Low    |
| 12  | Simplify orchestrator — extract patterns, compress sections                                      | 1 (orchestrator.agent.md)                                        | 2      | Medium |
| 13  | Update README.md                                                                                 | 1 (README.md)                                                    | 5      | Low    |

> **Note:** Tasks 02–04 each touch 4 files, exceeding the standard 3-file limit. This is a conscious deviation — each edit is a single identical YAML line addition (~1 line per file), making these tasks trivially small and focused. Splitting further would create 3 additional micro-tasks with no reviewability benefit.

---

## Execution Waves

### Wave 1 (1 task)

- **01** — Remove emergency prune from orchestrator (depends_on: none)

### Wave 2 (8 tasks, parallel)

- **02** — CT sub-agent YAML read-only (depends_on: none)
- **03** — V sub-agent YAML read-only (depends_on: none)
- **04** — R sub-agent YAML read-only (depends_on: none)
- **05** — Aggregator YAML read-write (depends_on: none)
- **06** — Sequential YAML read-write (depends_on: none)
- **07** — Implementer/researcher/doc-writer YAML + body (depends_on: none)
- **08** — Orchestrator validation logic (depends_on: 01)
- **09** — Create dispatch-patterns.md (depends_on: none)

### Wave 3 (2 tasks, parallel)

- **10** — r-knowledge body changes (depends_on: 04)
- **11** — Orchestrator r-knowledge dispatch row (depends_on: 08)

### Wave 4 (1 task)

- **12** — Simplify orchestrator (depends_on: 09, 11)

### Wave 5 (1 task)

- **13** — Update README (depends_on: 12)

---

## Dependency Graph

```
Wave 1:  01
          │
Wave 2:  02  03  04  05  06  07  08  09
                  │               │
Wave 3:          10              11
                                  │   │
Wave 4:                          12 ◄─┘
                                  │
Wave 5:                          13

Legend:
  01 ──→ 08 ──→ 11 ──→ 12 ──→ 13   (orchestrator.agent.md chain)
  04 ──→ 10                          (r-knowledge.agent.md chain)
  09 ──→ 12                          (dispatch-patterns.md must exist before orchestrator references it)
  02, 03, 05, 06, 07                 (independent — no deps, no downstream)
```

**Same-file sequencing (enforced via depends_on):**

- `orchestrator.agent.md`: Task 01 → 08 → 11 → 12 (4 sequential modifications)
- `r-knowledge.agent.md`: Task 04 → 10 (YAML frontmatter then body)

---

## Implementation Specification

### New Files Created

| File                                       | Created By | Content                                                            |
| ------------------------------------------ | ---------- | ------------------------------------------------------------------ |
| `NewAgentsAndPrompts/dispatch-patterns.md` | Task 09    | Full definitions of Patterns A and B (extracted from orchestrator) |

### Files Modified by Multiple Tasks

| File                    | Tasks (sequential) | Changes                                                        |
| ----------------------- | ------------------ | -------------------------------------------------------------- |
| `orchestrator.agent.md` | 01 → 08 → 11 → 12  | Remove prune → add validation → update dispatch row → compress |
| `r-knowledge.agent.md`  | 04 → 10            | Add YAML field → update inputs/workflow body                   |

### Files Modified by Single Task

| File                                | Task | Change                           |
| ----------------------------------- | ---- | -------------------------------- |
| 4 CT sub-agents                     | 02   | YAML `memory_access: read-only`  |
| 4 V sub-agents                      | 03   | YAML `memory_access: read-only`  |
| 3 aggregator agents                 | 05   | YAML `memory_access: read-write` |
| spec, designer, planner             | 06   | YAML `memory_access: read-write` |
| implementer, researcher, doc-writer | 07   | YAML + doc-writer body           |
| README.md                           | 13   | Fix stale data                   |

### Files NOT Modified

| File                                        | Reason                            |
| ------------------------------------------- | --------------------------------- |
| `critical-thinker.agent.md`                 | Deprecated, malformed frontmatter |
| `feature-workflow.prompt.md`                | Prompt file, not an agent         |
| `docs/feature/forge-architecture-upgrade/*` | Historical artifacts              |
| `docs/feature/agent-improvements/*`         | Historical artifacts              |

---

## Risks & Mitigations

| Risk                                                     | Likelihood | Impact   | Mitigation                                                                         |
| -------------------------------------------------------- | ---------- | -------- | ---------------------------------------------------------------------------------- |
| YAML `memory_access` field rejected by runtime           | Low        | Critical | Standard YAML ignores unknown fields; rollback is trivial (remove 1 line per file) |
| Orchestrator AI doesn't read `dispatch-patterns.md`      | Medium     | High     | Explicit read instruction in orchestrator; Pattern C kept inline for safety        |
| Orchestrator exceeds 400 lines after all changes         | Low        | Low      | Parallel Execution Summary compression is the relief valve (~35 lines recoverable) |
| r-knowledge produces weaker analysis with reduced inputs | Medium     | Medium   | Fallback to grep/semantic_search included in design; r-aggregator compensates      |
| Memory grows unbounded without emergency prune           | Low        | Medium   | Checkpoint pruning at 3 pipeline points constrains growth structurally             |
| Doc-writer memory updates lost after read-only change    | Low        | Low      | Orchestrator between-wave update extended; retries available                       |

---

## Plan Validation

### Circular Dependency Check

Dependency chains:

- 01 → 08 → 11 → 12 → 13 (linear)
- 04 → 10 (linear)
- 09 → 12 (linear, merges with chain above at 12)
- 02, 03, 05, 06, 07 (no deps, no downstream)

**No circular dependencies.** ✓

### Task Size Validation

- All tasks estimated Low or Medium effort ✓
- All tasks ≤ 2 dependencies ✓
- Tasks 02–04 exceed 3-file limit (4 files each) — noted as conscious deviation for identical 1-line YAML edits ✓
- All tasks estimated ≤ 100 lines of production code changes ✓

### Dependency Existence Check

All `depends_on` references point to tasks that exist:

- 08 → 01 ✓
- 10 → 04 ✓
- 11 → 08 ✓
- 12 → 09, 11 ✓
- 13 → 12 ✓

---

## Pre-Mortem Analysis

| Task  | Failure Scenario                                            | Likelihood | Impact | Mitigation                                                                                             |
| ----- | ----------------------------------------------------------- | ---------- | ------ | ------------------------------------------------------------------------------------------------------ |
| 01    | Wrong lines identified for emergency prune removal          | Low        | Medium | Design provides exact before/after text for all 3 edit locations                                       |
| 02–06 | YAML field inserted at wrong position in frontmatter        | Low        | Low    | Design specifies "third field after name and description"                                              |
| 07    | Doc-writer step 7 removal breaks downstream expectations    | Low        | Medium | Design provides exact replacement text; orchestrator between-wave update compensates                   |
| 08    | Validation logic added at wrong dispatch points             | Medium     | Medium | Design lists exact 5 Step references with agent lists                                                  |
| 09    | Pattern file content drifts from orchestrator understanding | Low        | Low    | Content is verbatim extract from current orchestrator                                                  |
| 10    | Artifact Index navigation instructions too vague            | Medium     | Medium | Design provides exact before/after text with fallback clause                                           |
| 11    | Wrong dispatch table row modified in Step 7.2               | Low        | Medium | Design provides exact before/after for the r-knowledge row                                             |
| 12    | Orchestrator exceeds 400 lines after compression            | Medium     | Low    | Compress Parallel Execution Summary more aggressively; 400-line target is quality goal, not functional |
| 13    | Stale researcher count references missed                    | Low        | Low    | Grep sweep for ×3, x3, "3 concurrent" catches all                                                      |

**Overall Risk Level:** Low — all changes are well-specified with exact before/after text in the design. The highest-risk task is 12 (orchestrator simplification) which requires judgment on compression depth, but the design provides a detailed line budget showing ≤400 is achievable.

**Key Assumptions:**

1. VS Code Copilot runtime ignores unknown YAML frontmatter fields → All YAML tasks (02–08) depend on this. If false, all 21 agent files would fail to load.
2. Existing checkpoint pruning bounds memory growth sufficiently → Task 01 depends on this. If false, memory could grow unbounded.
3. Artifact Index is maintained with sufficient detail by upstream agents → Task 10 depends on this. If false, r-knowledge analysis quality degrades (but fallback exists).

---

## Deliverables & Acceptance

**Completion = all 13 tasks done + all 10 success criteria verified.**

Files delivered:

- 1 new file: `NewAgentsAndPrompts/dispatch-patterns.md`
- 22 modified files: 21 agent files + `orchestrator.agent.md`
- 1 updated file: `README.md`
- Total: 24 files touched
