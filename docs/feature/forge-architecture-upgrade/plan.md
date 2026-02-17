# Implementation Plan: Forge Architecture Upgrade

## Planning Mode

**Initial** — No existing `plan.md` found. Creating full plan from scratch.

## Summary

16 tasks organized into 4 execution waves. Creates 15 new agent files (sub-agents + aggregators for CT, V, R clusters) and modifies 11 existing files (memory integration, research expansion, orchestrator rewrite, feature-workflow prompt update, 3 deprecation notices). Estimated total: ~1,900 lines of new markdown across 15 new files, ~300 lines of modifications across 11 existing files. All tasks assigned to `implementer` agent. All files authored in `NewAgentsAndPrompts/` per initial request scope.

**Key constraints:**

- The orchestrator is the single serialization bottleneck — placed atomically in Wave 3 after all cluster agents are finalized.
- Parallel sub-agents do NOT write to `memory.md` (R1 concurrency fix). Only aggregators and sequential agents write.
- Max 4 concurrent subagent invocations per sub-wave.

**Deployment note:** All files are authored in `NewAgentsAndPrompts/`. A separate sync step (outside this plan's scope) must copy all new and modified files to `.github/agents/` and `.github/prompts/` for VS Code Copilot runtime discovery.

## Success Criteria

| ID    | Criterion                                                                             | Traces To                                           |
| ----- | ------------------------------------------------------------------------------------- | --------------------------------------------------- |
| SC-1  | All 15 new agent files exist in `NewAgentsAndPrompts/` following v2 template          | design.md Appendix A                                |
| SC-2  | All existing agents include memory protocol (read first, write for sequential agents) | MEM-INT-AC-1 through MEM-INT-AC-7                   |
| SC-3  | Researcher dispatches 4 focus areas including `patterns`                              | RES-AC-1, RES-AC-2                                  |
| SC-4  | CT cluster: 4 sub-agents + aggregator produce `design_critical_review.md`             | CT-AC-1 through CT-AC-9                             |
| SC-5  | V cluster: v-build gate + 3 parallel sub-agents + aggregator produce `verifier.md`    | V-AC-1 through V-AC-10                              |
| SC-6  | R cluster: 4 sub-agents + aggregator produce `review.md` + `knowledge-suggestions.md` | R-AC-1 through R-AC-11, KE-SAFE-1 through KE-SAFE-7 |
| SC-7  | Orchestrator coordinates memory lifecycle, 3 cluster patterns, updated routing        | ORCH-AC-1 through ORCH-AC-14                        |
| SC-8  | Parallel sub-agents do NOT write to `memory.md`                                       | R1 concurrency fix                                  |
| SC-9  | Superseded agents marked as deprecated                                                | design.md §11 Migration Phase 4                     |
| SC-10 | Feature-workflow prompt references memory, clusters, knowledge evolution              | MEM-INT-AC-7                                        |

## Ordered Task Index

| #   | Task                                         | Agent       | Files                                                           | Effort | Depends On |
| --- | -------------------------------------------- | ----------- | --------------------------------------------------------------- | ------ | ---------- |
| 01  | Researcher: 4th focus area + memory protocol | implementer | researcher.agent.md                                             | Low    | none       |
| 02  | Spec + Designer: memory protocol             | implementer | spec.agent.md, designer.agent.md                                | Low    | none       |
| 03  | Planner: memory protocol                     | implementer | planner.agent.md                                                | Low    | none       |
| 04  | Implementer + Doc Writer: memory protocol    | implementer | implementer.agent.md, documentation-writer.agent.md             | Low    | none       |
| 05  | CT sub-agents: Security + Scalability        | implementer | ct-security.agent.md, ct-scalability.agent.md                   | Medium | none       |
| 06  | CT sub-agents: Maintainability + Strategy    | implementer | ct-maintainability.agent.md, ct-strategy.agent.md               | Medium | none       |
| 07  | V sub-agents: Build + Tests                  | implementer | v-build.agent.md, v-tests.agent.md                              | Medium | none       |
| 08  | V sub-agents: Tasks + Feature                | implementer | v-tasks.agent.md, v-feature.agent.md                            | Medium | none       |
| 09  | R sub-agents: Quality + Security             | implementer | r-quality.agent.md, r-security.agent.md                         | Medium | none       |
| 10  | R sub-agents: Testing + Knowledge            | implementer | r-testing.agent.md, r-knowledge.agent.md                        | Medium | none       |
| 11  | CT Aggregator                                | implementer | ct-aggregator.agent.md                                          | Medium | 05, 06     |
| 12  | V Aggregator                                 | implementer | v-aggregator.agent.md                                           | Medium | 07, 08     |
| 13  | R Aggregator                                 | implementer | r-aggregator.agent.md                                           | Medium | 09, 10     |
| 14  | Orchestrator: major rewrite                  | implementer | orchestrator.agent.md                                           | Medium | 12, 13     |
| 15  | Feature workflow prompt update               | implementer | feature-workflow.prompt.md                                      | Low    | 14         |
| 16  | Deprecate superseded agents                  | implementer | critical-thinker.agent.md, verifier.agent.md, reviewer.agent.md | Low    | 14         |

## Execution Waves

### Wave 1: Foundation — Memory Integration + Sub-Agent Creation (10 tasks, parallel)

All tasks in this wave are fully independent with no inter-dependencies. The orchestrator partitions into sub-waves of ≤4 concurrent tasks (3 sub-waves: 4 + 4 + 2).

| Task | File(s)                                             | Agent       | Dependencies |
| ---- | --------------------------------------------------- | ----------- | ------------ |
| 01   | researcher.agent.md                                 | implementer | none         |
| 02   | spec.agent.md, designer.agent.md                    | implementer | none         |
| 03   | planner.agent.md                                    | implementer | none         |
| 04   | implementer.agent.md, documentation-writer.agent.md | implementer | none         |
| 05   | ct-security.agent.md, ct-scalability.agent.md       | implementer | none         |
| 06   | ct-maintainability.agent.md, ct-strategy.agent.md   | implementer | none         |
| 07   | v-build.agent.md, v-tests.agent.md                  | implementer | none         |
| 08   | v-tasks.agent.md, v-feature.agent.md                | implementer | none         |
| 09   | r-quality.agent.md, r-security.agent.md             | implementer | none         |
| 10   | r-testing.agent.md, r-knowledge.agent.md            | implementer | none         |

### Wave 2: Aggregators (3 tasks, parallel)

Each aggregator depends on its cluster's sub-agents being created. Aggregators must read sub-agent files to ensure output format alignment.

| Task | File(s)                | Agent       | Dependencies |
| ---- | ---------------------- | ----------- | ------------ |
| 11   | ct-aggregator.agent.md | implementer | 05, 06       |
| 12   | v-aggregator.agent.md  | implementer | 07, 08       |
| 13   | r-aggregator.agent.md  | implementer | 09, 10       |

### Wave 3: Orchestrator Rewrite (1 task, sequential)

Atomic rewrite of the orchestrator to integrate all clusters, memory lifecycle, and updated routing. Depends on V and R aggregators (explicit); CT aggregator is also complete by wave ordering.

| Task | File(s)               | Agent       | Dependencies |
| ---- | --------------------- | ----------- | ------------ |
| 14   | orchestrator.agent.md | implementer | 12, 13       |

### Wave 4: Finalization (2 tasks, parallel)

| Task | File(s)                                                         | Agent       | Dependencies |
| ---- | --------------------------------------------------------------- | ----------- | ------------ |
| 15   | feature-workflow.prompt.md                                      | implementer | 14           |
| 16   | critical-thinker.agent.md, verifier.agent.md, reviewer.agent.md | implementer | 14           |

## Dependency Graph

```
Wave 1 (all independent):                  Wave 2:           Wave 3:        Wave 4:
  01 (researcher) ──────────────────────────────────────────────────────────────────
  02 (spec+designer) ───────────────────────────────────────────────────────────────
  03 (planner) ─────────────────────────────────────────────────────────────────────
  04 (impl+docwriter) ──────────────────────────────────────────────────────────────
  05 (ct-sec+scal) ──────┐
                         ├──→ 11 (ct-agg) ──────┐
  06 (ct-maint+strat) ──┘                       │
  07 (v-build+tests) ───┐                       │
                        ├──→ 12 (v-agg) ────────┼──→ 14 (orch) ──→ 15 (prompt)
  08 (v-tasks+feat) ───┘                        │                 ──→ 16 (deprecate)
  09 (r-qual+sec) ─────┐                       │
                       ├──→ 13 (r-agg) ────────┘
  10 (r-test+know) ───┘
```

## Plan Validation

1. **Circular Dependency Check:** ✓ No cycles. All dependencies flow strictly forward (lower task IDs → higher task IDs).

2. **Task Size Validation:**

| Task | Files | Deps | Est. Lines | Effort | Valid |
| ---- | ----- | ---- | ---------- | ------ | ----- |
| 01   | 1     | 0    | ~15        | Low    | ✓     |
| 02   | 2     | 0    | ~20        | Low    | ✓     |
| 03   | 1     | 0    | ~10        | Low    | ✓     |
| 04   | 2     | 0    | ~18        | Low    | ✓     |
| 05   | 2     | 0    | ~180       | Medium | ✓     |
| 06   | 2     | 0    | ~180       | Medium | ✓     |
| 07   | 2     | 0    | ~200       | Medium | ✓     |
| 08   | 2     | 0    | ~180       | Medium | ✓     |
| 09   | 2     | 0    | ~180       | Medium | ✓     |
| 10   | 2     | 0    | ~230       | Medium | ✓     |
| 11   | 1     | 2    | ~110       | Medium | ✓     |
| 12   | 1     | 2    | ~120       | Medium | ✓     |
| 13   | 1     | 2    | ~120       | Medium | ✓     |
| 14   | 1     | 2    | ~200       | Medium | ✓     |
| 15   | 1     | 1    | ~10        | Low    | ✓     |
| 16   | 3     | 1    | ~30        | Low    | ✓     |

3. **Dependency Existence Check:** ✓ All `depends_on` references (05, 06, 07, 08, 09, 10, 12, 13, 14) point to tasks that exist in this plan.

## Implementation Specification

### New Files (15 agent definitions in `NewAgentsAndPrompts/`)

**CT Cluster:**

- `ct-security.agent.md` — Security vulnerabilities + backwards compatibility
- `ct-scalability.agent.md` — Scalability bottlenecks + performance implications
- `ct-maintainability.agent.md` — Maintainability + complexity + integration risks
- `ct-strategy.agent.md` — Strategic risks + scope + edge cases + fundamental approach
- `ct-aggregator.agent.md` — Merges 4 CT outputs → `design_critical_review.md`

**V Cluster:**

- `v-build.agent.md` — Build system detection + compilation (sequential gate)
- `v-tests.agent.md` — Test suite execution + analysis
- `v-tasks.agent.md` — Per-task acceptance criteria verification
- `v-feature.agent.md` — Feature-level criteria verification
- `v-aggregator.agent.md` — Merges V outputs → `verifier.md`

**R Cluster:**

- `r-quality.agent.md` — Code quality + readability + conventions
- `r-security.agent.md` — Security scanning + OWASP compliance
- `r-testing.agent.md` — Test quality + coverage adequacy
- `r-knowledge.agent.md` — Knowledge evolution (suggestions only)
- `r-aggregator.agent.md` — Merges R outputs → `review.md`

### Modified Files (11)

| File                          | Change Scope                                                                                              | Est. Lines Changed |
| ----------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------ |
| researcher.agent.md           | Add `patterns` focus area + memory protocol                                                               | ~15                |
| spec.agent.md                 | Memory protocol (input, rule 6, workflow steps)                                                           | ~10                |
| designer.agent.md             | Memory protocol                                                                                           | ~10                |
| planner.agent.md              | Memory protocol                                                                                           | ~10                |
| implementer.agent.md          | Memory protocol (read-only, no write step)                                                                | ~8                 |
| documentation-writer.agent.md | Memory protocol                                                                                           | ~10                |
| orchestrator.agent.md         | Major rewrite: memory lifecycle, 4 researchers, 3 cluster dispatch patterns, routing tables, expectations | ~150-200           |
| feature-workflow.prompt.md    | Memory + cluster + knowledge evolution references                                                         | ~10                |
| critical-thinker.agent.md     | Deprecation notice                                                                                        | ~10                |
| verifier.agent.md             | Deprecation notice                                                                                        | ~10                |
| reviewer.agent.md             | Deprecation notice                                                                                        | ~10                |

### Key Integration Points

- All new sub-agents read `memory.md` but do NOT write (R1 concurrency invariant)
- All aggregators consolidate sub-agent findings into `memory.md` root-level sections (sequential — safe)
- Orchestrator dispatches via `runSubagent` using 3 named cluster patterns (A: fully parallel, B: sequential gate + parallel, C: replan loop)
- Sub-agents produce intermediate files in cluster-specific directories (`ct-review/`, `verification/`, `review/`)

## Pre-Mortem Analysis

| Task | Failure Scenario                                                 | Likelihood | Impact | Mitigation                                                                               |
| ---- | ---------------------------------------------------------------- | ---------- | ------ | ---------------------------------------------------------------------------------------- |
| 01   | 4th focus area `patterns` overlaps with `architecture` scope     | Low        | Low    | design.md §6.1 defines distinct non-overlapping scope                                    |
| 02   | Memory protocol inserted at wrong workflow position              | Low        | Medium | design.md §8.2 specifies exact insertion points per agent                                |
| 03   | Planner memory write interacts poorly with replan mode           | Low        | Medium | Memory protocol is purely additive; replan flow unchanged                                |
| 04   | Implementer read-only vs doc-writer read+write inconsistency     | Low        | Low    | Task explicitly documents the design difference                                          |
| 05   | CT sub-agent categories overlap causing redundant findings       | Medium     | Low    | Cross-Cutting Observations section handles boundary issues                               |
| 06   | CT-Strategy scope too broad ("is the approach right?")           | Medium     | Medium | Scoped by specific key questions in design.md §3.1                                       |
| 07   | V-Build doesn't capture sufficient state for downstream agents   | Medium     | High   | design.md §4.1 specifies exact v-build.md output fields                                  |
| 08   | V-Tasks / V-Feature scope overlap on acceptance criteria         | Low        | Medium | Per-task vs feature-level criteria are clearly distinct                                  |
| 09   | R-Security ERROR override too aggressive for pipeline flow       | Low        | Medium | Intentional design: security is non-negotiable (R-AC-4)                                  |
| 10   | R-Knowledge produces vague, non-actionable suggestions           | Medium     | Medium | KE-SAFE-2 mandates specific file/section/diff format                                     |
| 11   | CT Aggregator fails to properly deduplicate cross-agent findings | Medium     | Medium | Dedup by matching Where+What fields (design.md §3.3)                                     |
| 12   | V Aggregator doesn't map failures to task IDs for replan         | Medium     | High   | design.md §4.3 step 4 explicitly requires task ID mapping                                |
| 13   | R Aggregator mishandles R-Knowledge non-blocking status          | Low        | Medium | Clear rules in design.md §5.5 completion contract table                                  |
| 14   | Orchestrator rewrite introduces regressions in existing flow     | High       | High   | Atomic update; design.md §7 provides complete spec; test against existing pipeline steps |
| 15   | Prompt update too minimal to guide users                         | Low        | Low    | Task specifies minimum required content                                                  |
| 16   | Deprecation notices unclear about replacement agents             | Low        | Low    | Task specifies exact format with cluster references                                      |

**Overall Risk Level:** Medium — Task 14 (orchestrator rewrite) is the single highest-risk item, modifying the system's central coordination point with ~150-200 lines across 8+ sections. The design provides detailed specifications (§7), but the orchestrator must correctly coordinate memory lifecycle, 3 cluster dispatch patterns, and 18+ new agent expectations simultaneously. All other tasks are bounded and well-specified.

**Key Assumptions:**

1. `NewAgentsAndPrompts/` is the correct authoring target. Affects: all tasks.
2. v2 agent template (YAML frontmatter, role, inputs/outputs, 5 rules, workflow, output spec, completion contract, anti-drift anchor) is the correct structural pattern. Affects: tasks 05-13.
3. VS Code Copilot runtime supports ≤4 concurrent sub-agents. Affects: task 14.
4. Aggregator "merge-only, 40-60% overhead" scope is achievable. Affects: tasks 11-13.
5. Orchestrator stays under 450-line cap. If not, sub-orchestrator decomposition needed. Affects: task 14.
