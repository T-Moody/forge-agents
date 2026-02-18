# Task 05: Orchestrator — Workflow Steps 5–7, Completion Contract, Anti-Drift, Tables

## Task Goal

Update orchestrator Steps 5–7, NEEDS_REVISION Routing Table, Orchestrator Expectations table, Parallel Execution Summary, Completion Contract, and Anti-Drift Anchor. Replace aggregator invocations in V and R clusters with orchestrator evaluation, add merge steps, and update all reference tables.

## depends_on

04

## agent

implementer

## In-Scope

- **Step 5** (Implementation): Add memory merge between waves — orchestrator reads implementer isolated memories, merges, extracts Lessons Learned
- **Step 6** (V Cluster):
  - Step 6.1/6.2: Keep V sub-agent dispatch (update to note they produce isolated memories)
  - Step 6.3: Replace V Aggregator invocation with orchestrator V decision evaluation (reference V decision flow from task 03). Add merge sub-step (6.3m)
  - Step 6.4: Update Pattern C replan — planner invocation includes explicit `MODE: REPLAN` signal + paths to V memory files (`memory/v-*.mem.md`). Replace `verifier.md` references with individual V artifacts
- **Step 7** (R Cluster):
  - Step 7.1/7.2: Keep R sub-agent dispatch, update error notes (remove aggregator references)
  - Step 7.3: Replace R Aggregator invocation with orchestrator R decision evaluation (reference R decision flow from task 03). Add merge sub-step (7m)
  - Step 7.4: Update R result handling — reference R memories (not `review.md`)
  - Step 7.5: Keep knowledge-suggestions.md and decisions.md preservation (no change)
- **NEEDS_REVISION Routing Table**: Replace all aggregator names with orchestrator evaluation references:
  - "CT Aggregator → Designer" becomes "Orchestrator (CT evaluation) → Designer (with individual `ct-review/ct-*.md`)"
  - "V Aggregator → Planner" becomes "Orchestrator (V evaluation) → Planner (replan mode with V memories + individual artifacts)"
  - "R Aggregator → Implementers" becomes "Orchestrator (R evaluation) → Implementer(s) for affected tasks"
  - "R-Security (via R Aggregator ERROR)" becomes "R-Security (via orchestrator R evaluation)"
  - Remove "All sub-agents route through their aggregator" row
  - Replace "forward Unresolved Tensions as planning constraints" with "forward all High/Critical findings from individual CT artifacts as planning constraints"
  - Replace `verifier.md` references with individual V memory/artifact paths
- **Orchestrator Expectations table**: Remove rows for CT Aggregator, V Aggregator, R Aggregator. Update sub-agent rows to reference isolated memory production and orchestrator routing (not aggregator routing)
- **Parallel Execution Summary**: Update all 4 summary lines:
  - Research: remove "→ Synthesize → analysis.md", add "→ merge memories"
  - CT: remove "→ CT-Aggregator → design_critical_review.md", add "→ orchestrator CT evaluation → merge memories"
  - V: remove "→ V-Aggregator → verifier.md", add "→ orchestrator V evaluation → merge memories"
  - R: remove "→ R-Aggregator → review.md", add "→ orchestrator R evaluation → merge memories"
- **Completion Contract**: Replace "R Aggregator returns DONE" with "R cluster returns DONE" (or equivalent)
- **Anti-Drift Anchor**: Add memory merge responsibility. Remove aggregator dispatch pattern reference. Clarify orchestrator is sole `memory.md` writer

## Out-of-Scope

- Steps 0–4 (handled by task 04)
- Cluster decision logic definitions (handled by task 03)
- Global Rules and Documentation Structure (handled by task 02)

## Acceptance Criteria

- AC-2 (final): No aggregator names in Steps 5–7, routing table, expectations table, parallel summary, completion contract, or anti-drift anchor
- AC-3 (final): No `verifier.md`, `review.md`, `design_critical_review.md`, `analysis.md` references in modified sections
- AC-6 (final): Merge sub-steps present after Steps 5 (between waves), 6, 7
- AC-8 (partial): V cluster handling references orchestrator V evaluation and Pattern C uses V memories
- AC-9 (partial): R cluster handling references orchestrator R evaluation
- AC-16 (partial): Anti-Drift Anchor references memory merge, sole writer, no aggregator dispatch
- Step 6.4 includes `MODE: REPLAN` signal in planner invocation
- Completion Contract references "R cluster" not "R Aggregator"
- Expectations table has no aggregator rows
- NEEDS_REVISION table references orchestrator evaluations

## Estimated Effort

Medium

## Test Requirements

TDD skipped: markdown-only task, no behavioral code.

## Implementation Steps

1. Read `orchestrator.agent.md` Steps 5–7, NEEDS_REVISION table, Expectations table, Parallel Execution Summary, Completion Contract, Anti-Drift Anchor (approximately lines 230–399)
2. **Step 5.2**: Add "Between implementation waves: read implementer memories (`memory/implementer-*.mem.md`) → merge into `memory.md` → extract Lessons Learned"
3. **Step 6.1/6.2**: Note that V sub-agents produce `memory/v-*.mem.md` alongside their artifacts
4. **Step 6.3**: Replace V Aggregator invocation entirely. New text: "Apply V Cluster Decision (see V decision flow). Read 4 V memories → merge into `memory.md`." Remove all `v-aggregator` references
5. **Step 6.4**: Update Pattern C replan loop — "Invoke planner with `MODE: REPLAN` and paths: `memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md`". Remove `verifier.md` references. Reference individual V artifacts for planner input
6. **Step 7.1/7.2**: Update R-Security error note — remove "aggregator ERROR" reference. Keep "R-Knowledge non-blocking"
7. **Step 7.3**: Replace R Aggregator invocation entirely. New text: "Apply R Cluster Decision (see R decision flow). Read 4 R memories → merge into `memory.md`." Remove all `r-aggregator` and `review.md` references
8. **Step 7.4**: Update R result handling — reference R memories for routing decision
9. Rewrite **NEEDS_REVISION Routing Table** per in-scope specifications above (5 row updates, 1 row removal)
10. Rewrite **Orchestrator Expectations table** — remove 3 aggregator rows, update sub-agent rows
11. Rewrite **Parallel Execution Summary** — 4 line updates removing aggregator/synthesis references
12. Update **Completion Contract** — "R cluster returns DONE" not "R Aggregator"
13. Update **Anti-Drift Anchor** — add memory merge, remove aggregator dispatch, clarify sole writer
14. Final search for any remaining aggregator references or removed artifact names in entire file

## Completion Checklist

- [x] Step 5 has memory merge between implementation waves
- [x] Step 6.3 references orchestrator V evaluation (no v-aggregator)
- [x] Step 6.4 includes `MODE: REPLAN` signal + V memory paths (no `verifier.md`)
- [x] Step 7.3 references orchestrator R evaluation (no r-aggregator, no `review.md`)
- [x] NEEDS_REVISION table: all rows reference orchestrator, no aggregator names
- [x] NEEDS_REVISION table: no "Unresolved Tensions", no `verifier.md`, no `design_critical_review.md`
- [x] Expectations table: no aggregator rows
- [x] Parallel Execution Summary: no aggregator or synthesis references, has merge steps
- [x] Completion Contract: references "R cluster" not "R Aggregator"
- [x] Anti-Drift Anchor: mentions memory merge, sole writer; no aggregator pattern reference
- [x] Grep entire `orchestrator.agent.md` for `aggregator`, `analysis.md`, `design_critical_review.md`, `verifier.md` (as artifact), `review.md` (as artifact) — zero matches expected (2 acceptable historical/negation references remain)
- [x] Standard section ordering preserved
