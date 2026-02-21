# Post-Mortem Report

**Feature:** orchestrator-tool-restriction
**Run Date:** 2025-07-17

```yaml
post_mortem_report:
  run_date: "2025-07-17"
  feature_slug: "orchestrator-tool-restriction"
  summary: "32 dispatches, 0 errors, 0 retries; CT cluster required 3 iterations (scope reduction + contradiction fixes); 16 evaluations processed across 8 producer agents; spec's feature.md had lowest scores (avg usefulness 6.6) with 12 reported inaccuracies driven by missing Phase 1 scoping"

  recurring_issues:
    - issue: "feature.md missing Phase 1 vs Phase 2 scope annotations — full 23-file/15-AC scope described without phasing guidance"
      affected_agents: ["designer", "ct-security", "ct-scalability", "ct-strategy", "ct-maintainability", "planner"]
      frequency: 6

    - issue: "NFR-3.1 token estimate inaccuracy — spec claimed 100-250 tokens per .mem.md file; CT measured avg 488 tokens (2-5x underestimate)"
      affected_agents: ["designer", "ct-scalability", "ct-strategy"]
      frequency: 3

    - issue: "AC-3 pass criterion overly strict — says 'returns zero matches' for prohibited tools but design adds tool names to prohibition contexts"
      affected_agents: ["v-feature", "r-testing"]
      frequency: 2

    - issue: "AC-12/AC-13/AC-14 pass criteria assume Phase 2 behavior — criteria reference memory.md removal but Phase 1 preserves memory.md"
      affected_agents: ["planner", "r-testing"]
      frequency: 2

    - issue: "Design missing edit targets for residual merge language — Global Rule 6, Memory Lifecycle Table Merge rows, Parallel Execution Summary not included in change scope"
      affected_agents: ["ct-maintainability", "r-quality"]
      frequency: 2

  bottlenecks:
    - agent: "designer"
      reason: "Required 3 dispatches (initial + 2 revisions) due to CT iteration loop — scope coupling and memory tool contradictions required sequential redesign"
      retry_count: 0

    - agent: "CT cluster (3b)"
      reason: "3 full iterations spanning T+3 to T+7 (5 time steps) — iter 1 found scope coupling (High), iter 2 found memory tool contradictions (High/Medium), iter 3 passed"
      retry_count: 0

  most_common_missing_information:
    - item: "No Phase 1 vs Phase 2 scoping in feature.md — readers must cross-reference design.md §16.3 to determine which FRs/ACs apply"
      reported_by: ["designer", "ct-security", "ct-scalability", "ct-strategy", "ct-maintainability", "planner"]
      source_artifact: "feature.md"

    - item: "Missing edit targets in design.md for residual merge language (Global Rule 6, Memory Lifecycle Merge rows, Parallel Execution Summary)"
      reported_by: ["ct-maintainability", "r-quality"]
      source_artifact: "design.md"

    - item: "Spec did not provide exact replacement text for research-identified changes — only identified what needs to change, not how"
      reported_by: ["spec"]
      source_artifact: "research/architecture.md"

    - item: "T-N1/T-N2 test descriptions ambiguous — names don't specify search text, leading to v-tests remapping"
      reported_by: ["r-testing"]
      source_artifact: "design.md"

  agent_accuracy_scores:
    - agent: "researcher-architecture"
      average_usefulness: 9.0
      average_clarity: 9.0
      total_inaccuracies_reported: 0

    - agent: "researcher-impact"
      average_usefulness: 9.0
      average_clarity: 9.0
      total_inaccuracies_reported: 0

    - agent: "researcher-dependencies"
      average_usefulness: 8.0
      average_clarity: 8.0
      total_inaccuracies_reported: 0

    - agent: "researcher-patterns"
      average_usefulness: 10.0
      average_clarity: 9.0
      total_inaccuracies_reported: 0

    - agent: "spec"
      average_usefulness: 6.6
      average_clarity: 7.5
      total_inaccuracies_reported: 12

    - agent: "designer"
      average_usefulness: 8.5
      average_clarity: 9.1
      total_inaccuracies_reported: 1

    - agent: "planner"
      average_usefulness: 9.0
      average_clarity: 9.4
      total_inaccuracies_reported: 1

    - agent: "v-build"
      average_usefulness: 9.0
      average_clarity: 9.0
      total_inaccuracies_reported: 0
```

## Evaluation Coverage

| Evaluation File | Evaluator | Source Artifacts Evaluated | Blocks |
|-----------------|-----------|--------------------------|--------|
| spec.md | spec | research/architecture.md, research/impact.md, research/dependencies.md, research/patterns.md | 4 |
| designer.md | designer | feature.md | 1 |
| ct-security.md | ct-security | design.md, feature.md | 2 |
| ct-scalability.md | ct-scalability | design.md, feature.md | 2 |
| ct-strategy.md | ct-strategy | design.md, feature.md | 2 |
| ct-maintainability.md | ct-maintainability | design.md, feature.md | 2 |
| planner.md | planner | design.md, feature.md | 2 |
| implementer-01.md | implementer-01 | tasks/01, design.md, feature.md | 3 |
| implementer-02.md | implementer-02 | tasks/02, design.md, feature.md | 3 |
| implementer-03.md | implementer-03 | tasks/03, design.md, feature.md | 3 |
| implementer-04.md | implementer-04 | tasks/04, design.md, feature.md | 3 |
| v-tasks.md | v-tasks | v-build.md, plan.md, tasks/01-04 | 6 |
| v-feature.md | v-feature | v-build.md, feature.md | 2 |
| v-tests.md | v-tests | v-build.md | 1 |
| r-quality.md | r-quality | design.md | 1 |
| r-testing.md | r-testing | design.md, feature.md | 2 |
| **Total** | **16 files** | | **39 blocks** |

## Inaccuracy Detail

| Source Artifact | Reporter | Inaccuracy Description |
|-----------------|----------|----------------------|
| feature.md | designer | Token estimate 100-250 vs actual 488 tokens (2-5x understated) |
| feature.md | designer | Token neutrality claim for Pattern B — CT shows token-negative |
| feature.md | designer | FR-6.1/FR-6.2 tension — remove vs disambiguate without specifying exact references |
| feature.md | ct-security | EC-7 says 'No memory.md file exists' — inaccurate for Phase 1 |
| feature.md | ct-scalability | NFR-3.1 claims 100-250 tokens — measured avg 488 |
| feature.md | ct-scalability | Scope section doesn't reflect Phase 1 reduction |
| feature.md | ct-strategy | NFR-3.1 token estimate 2-5x too low |
| feature.md | ct-strategy | Assumption 3 repeats same underestimate |
| feature.md | planner | AC-12 pass criteria describes Phase 2 behavior |
| feature.md | planner | AC-13/AC-14 not marked as Phase 2 |
| feature.md | v-feature | AC-3 strict criterion conflicts with design intent |
| feature.md | r-testing | AC-12 pass criterion inaccurate for Phase 1 |
| design.md | planner | AC-13/AC-14 misclassified as 'In scope (no changes needed)' |
| tasks/02 | implementer-02 | T-12 claims zero matches but 3 out-of-scope occurrences remain |
