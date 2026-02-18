# Plan: Agent-Isolated Memory System

## Planning Mode

**Initial** — no prior plan exists.

## Feature Overview

Replace the shared `memory.md` write model and aggregator agents with a branch-merge memory architecture. Each subagent writes a compact isolated memory file (`memory/<agent-name>.mem.md`), the orchestrator merges these into shared `memory.md`, and the orchestrator absorbs all aggregator decision logic. Three aggregator agents and the researcher synthesis mode are removed. All changes are to markdown prompt files in `NewAgentsAndPrompts/`.

## Success Criteria

Mapped from feature.md acceptance criteria:

| AC    | Description                                                                                         | Implementing Tasks      |
| ----- | --------------------------------------------------------------------------------------------------- | ----------------------- |
| AC-1  | Aggregator files deprecated (no active workflow)                                                    | 01                      |
| AC-2  | No references to aggregator agents in active files                                                  | 01, 02–05, 06–10, 11–15 |
| AC-3  | No references to removed artifacts (analysis.md, design_critical_review.md, verifier.md, review.md) | 02–05, 06, 07, 08, 10   |
| AC-4  | No researcher synthesis mode                                                                        | 06                      |
| AC-5  | All sub-agents produce isolated memory                                                              | 06–09, 11–15            |
| AC-6  | Orchestrator memory merge implemented                                                               | 02, 04, 05              |
| AC-7  | Orchestrator absorbs CT decision logic                                                              | 03                      |
| AC-8  | Orchestrator absorbs V decision logic                                                               | 03, 05                  |
| AC-9  | Orchestrator absorbs R decision logic with security override                                        | 03, 09                  |
| AC-10 | Downstream agents read research files via memory-first pattern                                      | 07, 08                  |
| AC-11 | Designer reads CT files directly in revision mode                                                   | 07                      |
| AC-12 | Planner reads V files directly in replan mode                                                       | 08                      |
| AC-13 | Dispatch patterns updated (no aggregator steps)                                                     | 10                      |
| AC-14 | Memory write safety updated (orchestrator sole writer)                                              | 02                      |
| AC-15 | Documentation structure updated                                                                     | 02                      |
| AC-16 | Anti-drift anchors updated (isolated memory, no aggregator refs)                                    | 05, 06–09, 11–15        |
| AC-17 | Feature workflow updated                                                                            | 10                      |
| AC-18 | Step 1.2 (synthesis) removed                                                                        | 04                      |
| AC-19 | Convention consistency preserved                                                                    | All                     |
| AC-20 | Completion contracts unchanged for sub-agents                                                       | 06–09, 11–15            |

---

## Ordered Task Index

| #   | Task                                                                       | Files                                                                      | Effort | Depends On |
| --- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ------ | ---------- |
| 01  | Deprecate aggregator agents                                                | ct-aggregator.agent.md, v-aggregator.agent.md, r-aggregator.agent.md       | Low    | none       |
| 02  | Orchestrator — Global Rules, Documentation Structure, Memory Lifecycle     | orchestrator.agent.md                                                      | Medium | none       |
| 03  | Orchestrator — Cluster Decision Logic (CT, V, R)                           | orchestrator.agent.md                                                      | Medium | 02         |
| 04  | Orchestrator — Workflow Steps 0–4                                          | orchestrator.agent.md                                                      | Medium | 03         |
| 05  | Orchestrator — Workflow Steps 5–7, Completion Contract, Anti-Drift, Tables | orchestrator.agent.md                                                      | Medium | 04         |
| 06  | Researcher — Remove Synthesis Mode, Add Isolated Memory                    | researcher.agent.md                                                        | Medium | none       |
| 07  | Spec + Designer — Memory-First Inputs, Isolated Memory                     | spec.agent.md, designer.agent.md                                           | Medium | none       |
| 08  | Planner — Replan Mode, Cross-Referencing, Isolated Memory                  | planner.agent.md                                                           | Medium | none       |
| 09  | R-Security — Pipeline Blocker Updates, Isolated Memory                     | r-security.agent.md                                                        | Medium | none       |
| 10  | Support Files — Dispatch Patterns + Feature Workflow                       | dispatch-patterns.md, feature-workflow.prompt.md                           | Medium | none       |
| 11  | CT Sub-Agents — Isolated Memory (Batch 1)                                  | ct-security.agent.md, ct-scalability.agent.md, ct-maintainability.agent.md | Low    | none       |
| 12  | CT-Strategy + V-Build + V-Tests — Isolated Memory                          | ct-strategy.agent.md, v-build.agent.md, v-tests.agent.md                   | Low    | none       |
| 13  | V-Tasks + V-Feature + R-Quality — Isolated Memory                          | v-tasks.agent.md, v-feature.agent.md, r-quality.agent.md                   | Low    | none       |
| 14  | R-Testing + R-Knowledge + Implementer — Isolated Memory                    | r-testing.agent.md, r-knowledge.agent.md, implementer.agent.md             | Low    | none       |
| 15  | Documentation-Writer — Isolated Memory                                     | documentation-writer.agent.md                                              | Low    | none       |

---

## Execution Waves

### Wave 1 (parallel — 12 tasks, all touch different files)

- 01-deprecate-aggregators (depends_on: none)
- 02-orchestrator-global-rules-doc-structure (depends_on: none)
- 06-researcher-remove-synthesis (depends_on: none)
- 07-spec-designer-memory-inputs (depends_on: none)
- 08-planner-replan-mode (depends_on: none)
- 09-r-security-pipeline-blocker (depends_on: none)
- 10-support-files-dispatch-workflow (depends_on: none)
- 11-ct-sub-agents-batch1 (depends_on: none)
- 12-ct-strategy-v-build-v-tests (depends_on: none)
- 13-v-tasks-v-feature-r-quality (depends_on: none)
- 14-r-testing-r-knowledge-implementer (depends_on: none)
- 15-documentation-writer (depends_on: none)

### Wave 2 (sequential — orchestrator chain)

- 03-orchestrator-cluster-decisions (depends_on: 02)

### Wave 3 (sequential — orchestrator chain)

- 04-orchestrator-workflow-steps-0-4 (depends_on: 03)

### Wave 4 (sequential — orchestrator chain)

- 05-orchestrator-workflow-steps-5-7-tables (depends_on: 04)

---

## Dependency Graph

```
Wave 1 (all parallel):
  01 ──┐
  06   │
  07   │  (all independent — different files)
  08   │
  09   │
  10   │
  11   │
  12   │
  13   │
  14   │
  15   │
  02 ──┼──→ 03 (Wave 2) ──→ 04 (Wave 3) ──→ 05 (Wave 4)
       │         │                │                │
       │    orchestrator     orchestrator     orchestrator
       │    cluster logic    steps 0-4        steps 5-7 +
       │    (CT/V/R)                          tables
```

**Critical path:** 02 → 03 → 04 → 05 (orchestrator chain, 4 sequential tasks on same file).

All non-orchestrator tasks are fully independent and complete in Wave 1.

---

## Implementation Specification

### Files Created (none — markdown edits only)

No new files are created in `NewAgentsAndPrompts/`. All tasks edit existing files.

### Files Modified

**Tier 1 — Deprecate (3 files):** `ct-aggregator.agent.md`, `v-aggregator.agent.md`, `r-aggregator.agent.md` — replace contents with deprecation notice.

**Tier 2 — Orchestrator (1 file, 4 tasks):** `orchestrator.agent.md` — major rewrite across 20+ sections. Split into 4 tasks by section groups to stay within 500-line-change limit.

**Tier 3 — Moderate (6 files):** `researcher.agent.md`, `spec.agent.md`, `designer.agent.md`, `planner.agent.md`, `r-security.agent.md`, and support files `dispatch-patterns.md` + `feature-workflow.prompt.md`.

**Tier 4 — Minor (13 files):** All parallel sub-agents + implementer + documentation-writer. Same pattern: add isolated memory output, add memory write step, update Anti-Drift Anchor.

### Integration Points

- **Memory file template:** Defined in design.md Data Models section. All agents reference this template.
- **Cluster decision tables:** Defined in design.md. Orchestrator tasks 03–05 implement these.
- **Memory-first reading pattern:** Defined in design.md Revision 6. Spec, designer, planner, implementer tasks implement this.

---

## Risks & Mitigations

| Risk                                             | Severity | Likelihood | Mitigation                                                                                                                                                                |
| ------------------------------------------------ | -------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Orchestrator exceeds 450-line threshold          | High     | Medium     | Tasks 02–05 use compact decision tables per design. Post-implementation line count check. If >450, extract decision tables to dispatch-patterns.md as blocking follow-up. |
| R-Security pipeline override incorrectly updated | Critical | Low        | Task 09 has explicit instructions for Pipeline Blocker Override Rule rewrite. AC-9 verifies.                                                                              |
| Planner replan mode detection breaks             | High     | Low        | Task 08 includes explicit MODE: REPLAN signal mechanism + fallback detection. AC-12 verifies.                                                                             |
| Stale aggregator references missed               | Medium   | Medium     | All tasks include aggregator reference cleanup. Post-implementation grep verification catches remnants.                                                                   |
| Convention drift across 21 modified files        | Low      | Medium     | Each task's completion checklist includes section ordering verification.                                                                                                  |

---

## Plan Validation

### Circular Dependency Check

Dependency graph: 02 → 03 → 04 → 05. All other tasks have no dependencies. **No circular dependencies.**

### Task Size Validation

| Task | Files | Est. Lines Changed | Dependencies | Effort | Within Limits? |
| ---- | ----- | ------------------ | ------------ | ------ | -------------- |
| 01   | 3     | ~120               | 0            | Low    | Yes            |
| 02   | 1     | ~80                | 0            | Medium | Yes            |
| 03   | 1     | ~100               | 1            | Medium | Yes            |
| 04   | 1     | ~120               | 1            | Medium | Yes            |
| 05   | 1     | ~120               | 1            | Medium | Yes            |
| 06   | 1     | ~100               | 0            | Medium | Yes            |
| 07   | 2     | ~120               | 0            | Medium | Yes            |
| 08   | 1     | ~150               | 0            | Medium | Yes            |
| 09   | 1     | ~80                | 0            | Medium | Yes            |
| 10   | 2     | ~100               | 0            | Medium | Yes            |
| 11   | 3     | ~90                | 0            | Low    | Yes            |
| 12   | 3     | ~90                | 0            | Low    | Yes            |
| 13   | 3     | ~90                | 0            | Low    | Yes            |
| 14   | 3     | ~100               | 0            | Low    | Yes            |
| 15   | 1     | ~30                | 0            | Low    | Yes            |

All tasks satisfy: ≤3 files, ≤500 lines, ≤2 dependencies, ≤Medium effort. **All pass.**

### Dependency Existence Check

- Task 03 depends on 02 — exists. ✓
- Task 04 depends on 03 — exists. ✓
- Task 05 depends on 04 — exists. ✓
- All other tasks depend on none. ✓

**All dependency references valid.**

---

## Pre-Mortem Analysis

| Task  | Failure Scenario                                                                       | Likelihood | Impact | Mitigation                                                                                    |
| ----- | -------------------------------------------------------------------------------------- | ---------- | ------ | --------------------------------------------------------------------------------------------- |
| 01    | Deprecation notice format inconsistent across 3 files                                  | L          | L      | Use identical template for all 3 files                                                        |
| 02    | Global Rule 12 rewrite doesn't fully remove aggregator read-write lists                | M          | M      | Completion checklist verifies no aggregator names remain in Global Rules                      |
| 03    | V decision table incorrectly transcribed (10 rows with mixed-state handling)           | M          | H      | Cross-reference design.md V Cluster Decision Flow exactly; include self-verification step     |
| 04    | Step 1.2 removal leaves orphaned references (step numbering, approval gate)            | M          | M      | Search for "1.2", "synthesis", "analysis.md" in orchestrator after edits                      |
| 05    | NEEDS_REVISION routing table still references aggregator names                         | M          | H      | Completion checklist verifies each routing table row references orchestrator, not aggregators |
| 06    | Synthesis mode content boundaries unclear — partial removal leaves dangling references | M          | M      | Search for "Mode 2", "synthesis", "analysis.md" after removal                                 |
| 07    | Spec/Designer memory-first reading workflow too vague for implementer to execute       | M          | M      | Task provides explicit step-by-step reading workflow text from design.md                      |
| 08    | Replan cross-referencing steps omitted or incomplete                                   | M          | H      | Task includes full 5-step cross-referencing procedure from design.md                          |
| 09    | R-Security severity vocabulary not explicitly constrained to Blocker/Major/Minor       | M          | H      | Task explicitly specifies vocabulary constraint text; completion checklist verifies           |
| 10    | Dispatch patterns still reference combined artifacts                                   | M          | M      | Completion checklist includes grep for removed artifact names                                 |
| 11–15 | "No Memory Write" replacement missed in some agents                                    | M          | L      | Each task's steps explicitly list finding and replacing this text                             |

**Overall Risk Level:** Medium — The orchestrator chain (tasks 02–05) concentrates the highest-risk changes. Each task's decision logic must precisely match design.md specifications. Post-implementation grep verification for aggregator and removed-artifact references provides a safety net.

**Key Assumptions:**

| Assumption                                                                           | Impact If Wrong                                                | Dependent Tasks                         |
| ------------------------------------------------------------------------------------ | -------------------------------------------------------------- | --------------------------------------- |
| A-1: Memory file naming convention `memory/<agent-name>.mem.md` is stable            | All tasks would need renaming updates                          | 06–15 (all agents adding memory output) |
| A-2: Sequential agents (spec, designer, planner) use isolated memory (uniform model) | Tasks 07, 08 would simplify to direct memory.md writes         | 07, 08                                  |
| A-3: Orchestrator merges at cluster boundaries, not per-agent                        | Orchestrator workflow steps would need additional merge points | 04, 05                                  |
| A-6: Planner can self-assemble task-failure mapping from individual V artifacts      | Task 08's cross-referencing steps would be insufficient        | 08                                      |
| Design.md is the authoritative source for all prompt text changes                    | If design.md contains errors, they propagate to all tasks      | All tasks                               |

---

## Deliverables & Acceptance

**Deliverables:**

- 24 modified `.agent.md` and `.md` files in `NewAgentsAndPrompts/`
- 3 deprecated aggregator files
- 1 modified orchestrator file (across 4 tasks)
- 6 modified Tier 3 agent/support files
- 13 modified Tier 4 agent files

**Acceptance:**

- All AC-1 through AC-20 from feature.md pass
- Grep verification: zero matches for aggregator names and removed artifact paths in active files
- All modified files retain standard section ordering (NFR-3)
- All completion contracts unchanged (AC-20)
