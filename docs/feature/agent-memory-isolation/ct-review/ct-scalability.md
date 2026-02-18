# Critical Review: Scalability & Performance

## Summary

The design delivers genuine pipeline step reduction by eliminating 4 sequential LLM invocations (3 aggregators + synthesis researcher), but materially understates the complexity transfer to the orchestrator and creates a new category of risk: the planner in replan mode losing structured task-failure mapping. The ≤30-line memory file limit is adequate for routing but may be too tight for the V cluster, where task ID lists are the critical input for targeted replanning. The "+10 lines net" estimate for orchestrator growth is misleadingly low — it counts lines but ignores that the orchestrator absorbs dense decision logic previously elaborated across ~750 lines of aggregator instructions.

**Overall Risk Level:** Medium — No blocking issues, but several assumptions about orchestrator capacity and memory file adequacy should be validated before planning proceeds.

---

## Findings

### [Severity: High] Orchestrator Complexity Transfer Is Understated

- **What:** The design estimates "+30 lines for decision logic, −20 lines for removing aggregator invocation steps" for a net ~+10 lines (NFR-1 section). This is technically correct on line count but misleading about complexity. The three aggregator agents currently total ~750+ lines of detailed workflow instructions (ct-aggregator: ~200 lines, v-aggregator: ~258 lines, r-aggregator: ~271 lines). These instructions describe nuanced merge, dedup, cross-referencing, and tension-surfacing procedures. The design compresses their _routing decision logic_ into ~25 lines of compact tables in the orchestrator — but those ~25 lines carry the cognitive weight of what was previously spread across ~750 lines of explicit procedure. The orchestrator's LLM must now apply the CT severity check, V decision table (10 rows with priority rules and mixed-state handling), R priority-ordered decision rules (9 lines with security override), AND memory merge protocol — all in-line with its existing 399-line prompt. The risk is not prompt size but prompt density: the orchestrator's error surface for routing decisions increases.
- **Where:** Design section "Non-Functional Requirements → NFR-1: Orchestrator Prompt Size"; Orchestrator Workflow Steps 3b, 6, 7
- **Likelihood:** M
- **Impact:** H
- **Assumption at risk:** A central assumption is that compact decision tables are sufficient to replicate aggregator routing logic with equal reliability. The aggregators had extensive step-by-step workflows with explicit edge case handling (e.g., ct-aggregator steps 3–10 cover input validation, dedup, tension surfacing, coverage merge). The orchestrator gets only the final routing decision, not the procedural scaffolding that made aggregators reliable.

### [Severity: High] Planner Replan Without Structured Task-Failure Mapping

- **What:** The v-aggregator currently produces a structured "Actionable Items" section in `verifier.md` that cross-references failing task IDs from `v-tasks.md` with test failures from `v-tests.md` and feature criteria gaps from `v-feature.md`. This is explicitly called out in the v-aggregator as "the most critical aggregation step" — the planner depends on it for targeted replanning. Under the new design, the planner must self-assemble this cross-referencing by reading 3 individual V artifacts directly. The design acknowledges this in Assumption A-6 and Tradeoff 3 but treats it as acceptable without quantifying the degradation. If there are 15+ tasks, cross-referencing three separate files to identify which tasks need fixes, which tests map to which tasks, and which feature criteria are unmet is substantial analytical work for the planner agent.
- **Where:** Design Tradeoff 3; feature.md EC-11; v-aggregator.agent.md Workflow Step 5
- **Likelihood:** M
- **Impact:** H
- **Assumption at risk:** Assumption A-6 — "The planner in replan mode can assemble task ID failure mapping by reading `verification/v-tasks.md` directly, without needing the cross-referencing previously provided by `v-aggregator`." This assumes the planner LLM can reliably perform multi-document cross-referencing that was previously a dedicated agent's sole job.

### [Severity: Medium] ≤30-Line Memory File May Truncate V-Tasks Findings

- **What:** The memory file format limits "Key Findings" to ≤5 bullet points within a ≤30-line file. For most agents, this is adequate — the orchestrator only needs severity for routing. However, `v-tasks.mem.md` needs to communicate which specific tasks failed. In a project with 10-20 tasks, 5 bullet points cannot enumerate all failing task IDs. The orchestrator's routing decision for the V cluster (DONE vs. NEEDS_REVISION) only needs the overall status — but between the memory file and the full `verification/v-tasks.md` artifact, there's a gap: the planner needs the failing task IDs for replan targeting, and the orchestrator provides artifact paths to the planner. The design correctly states the planner reads full artifacts (not memories) for replan. So the question narrows to: does the orchestrator need per-task detail for its V routing decision? It does not — the Status field (DONE/NEEDS_REVISION/ERROR) suffices. But the 5-bullet limit may still cause information loss if the orchestrator needs to log meaningful merge content into `memory.md` from V sub-agents.
- **Where:** Design "Data Models & DTOs → Isolated Memory File Format"; feature.md NFR-2
- **Likelihood:** M
- **Impact:** M
- **Assumption at risk:** That ≤5 bullet points per memory file always contain sufficient information for the orchestrator's merge into `memory.md` Recent Updates. For V-Tasks with many failing tasks, the merge entry may be too terse to help downstream agents orient via memory-first reading.

### [Severity: Medium] Synthesis Work Shifts From 1 Agent to 3 Downstream Agents

- **What:** Currently, the researcher synthesis mode (Step 1.2) reads 4 research files and produces a single `analysis.md`. Spec, designer, and planner each read this 1 file. Under the new design, each of these 3 agents must independently read all 4 research files and mentally synthesize them. This means (a) synthesis is performed 3 times instead of once, and (b) each synthesis may produce different interpretations, since there's no canonical merged analysis. The total LLM token consumption for the 3 downstream agents increases: instead of reading ~1 file each, they read ~4 files each — roughly 3× the input tokens for the research phase across those 3 agents. The design correctly removes one sequential LLM invocation (the synthesis researcher), but replaces it with 3× increased input processing in downstream agents.
- **Where:** Design "Sequence / Interaction Notes → Updated Pipeline Flow"; initial-request.md section 3
- **Likelihood:** H
- **Impact:** M
- **Assumption at risk:** That 4 individual research files are no more burdensome to process than 1 combined analysis.md. While the total content volume is similar, the lack of deduplication and synthesis means each downstream agent processes more raw, potentially overlapping material.

### [Severity: Medium] New Sequential Merge Steps Add Latency Between Clusters

- **What:** The design introduces explicit merge steps after every cluster (Steps 1.1m, 2m, 3m, 3bm, 4m, 5m, 6.3m, 7m). Each merge step requires the orchestrator to: read N memory files, extract artifacts/decisions/findings, and append to `memory.md`. While each individual merge is fast (file reads + appends), the cumulative effect is 8+ new merge operations inserted into the pipeline. For an implementation phase with 5 waves of 4 tasks, that's 5 additional merge operations during Step 5 alone. These are not LLM invocations (just file I/O by the orchestrator's execution context), so they are fast — but they are additional sequential steps that didn't exist before. The net pipeline step reduction is: −4 full LLM invocations (aggregators + synthesis) and +8 lightweight merge operations. This is likely a net positive, but the design should acknowledge the tradeoff explicitly rather than presenting it as pure elimination.
- **Where:** Design "Sequence / Interaction Notes → Updated Pipeline Flow" (all "m" steps)
- **Likelihood:** L
- **Impact:** L
- **Assumption at risk:** That merge operations are negligible in latency compared to full agent invocations. This is almost certainly true, but the design claims outright elimination of bottlenecks rather than replacement with lighter-weight operations.

### [Severity: Low] Loss of Deduplication May Increase Downstream Agent Confusion

- **What:** Aggregators previously deduplicated findings across sub-agents (Tradeoff 1). Without deduplication, if 2 CT agents flag the same issue with different severity levels, both entries persist in their respective artifacts and memories. The orchestrator uses worst-case severity for routing (correct), but when the designer reads 4 individual CT files during revision, they see the same finding twice without reconciliation. For the CT cluster (4 agents, limited overlap potential), this is minor. For the R cluster where r-quality and r-testing may flag overlapping code quality issues, the downstream consumer faces more noise. The design accepts this (Assumption A-4) and it is a reasonable tradeoff.
- **Where:** Design "Tradeoffs & Alternatives Considered → Tradeoff 1"
- **Likelihood:** M
- **Impact:** L
- **Assumption at risk:** Assumption A-4 — that duplicate findings in individual artifacts don't meaningfully degrade downstream agent performance.

### [Severity: Low] Parallel Dispatch Itself Is Unchanged

- **What:** The design's framing suggests improved parallel dispatch, but the actual parallel execution model is unchanged: sub-agents still run ≤4 concurrently per wave (Global Rule 8 unchanged, per Constraint C-4). What improves is post-parallel processing: the aggregator sequential step is removed. This is a pipeline depth reduction, not a parallelism improvement. The distinction matters because "improved parallel dispatch" could be misinterpreted as more agents running concurrently.
- **Where:** Initial-request.md "improvement to parallel dispatch"; design High-Level Architecture
- **Likelihood:** L
- **Impact:** L
- **Assumption at risk:** That stakeholders understand the improvement is pipeline depth (fewer sequential steps) rather than increased concurrency.

---

## Cross-Cutting Observations

1. **Lost aggregator capabilities beyond routing are not fully catalogued.** The design focuses on routing decisions (DONE/NEEDS_REVISION/ERROR) moving to the orchestrator. But aggregators also produced: deduplication, cross-cutting concern synthesis, unresolved tension detection, requirement coverage merging, and structured output formatting. These are quality-of-analysis capabilities, not routing capabilities. Their loss is acknowledged (Tradeoffs 1 & 2) but the cumulative effect on downstream agent quality is not assessed. This is more of a maintainability/quality concern than a scalability one, but it has indirect scaling implications if downstream agents struggle more with raw inputs.

2. **Memory file format consistency across different agent types.** The design uses one template for all agents but clusters have different critical fields (severity taxonomies). The `Highest Severity` field serves different purposes: for CT it drives revision routing, for R it drives pipeline halting, for V it's PASS/FAIL. The orchestrator must parse the same field with fundamentally different semantics depending on which agent produced it. This is a minor parsing complexity that scales linearly with the number of clusters.

---

## Requirement Coverage

| Requirement                                | Coverage Status                   | Notes                                                                                                                                                                            |
| ------------------------------------------ | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-1 (Isolated Memory Files)               | Fully covered                     | Template defined, path conventions specified, ≤30 line target set                                                                                                                |
| FR-2 (Orchestrator Memory Merge)           | Fully covered                     | Merge protocol defined, sole-writer rule explicit                                                                                                                                |
| FR-3 (Remove Aggregators)                  | Fully covered                     | Deprecation notice strategy defined                                                                                                                                              |
| FR-4 (Orchestrator Absorbs Decision Logic) | Covered but compressed            | CT/V/R decision flows defined; however, edge cases from aggregator workflows (dedup, tensions, coverage merge) are dropped without explicit acknowledgment in the decision flows |
| FR-5 (Remove analysis.md)                  | Fully covered                     | Synthesis mode removed, downstream inputs updated                                                                                                                                |
| FR-6 (Remove Combined Artifacts)           | Fully covered                     | All 4 combined artifact removals specified                                                                                                                                       |
| FR-7 (Update Dispatch Patterns)            | Fully covered                     | Patterns A/B/C updates specified                                                                                                                                                 |
| NFR-1 (Orchestrator Prompt Size)           | Covered but estimate questionable | +10 lines net claim is credible on line count but misleading on complexity density                                                                                               |
| NFR-2 (Memory Compactness)                 | Covered                           | ≤30 lines, ≤5 bullets; adequate for routing, potentially tight for V-tasks merge details                                                                                         |
| NFR-4 (Non-Blocking Memory)                | Fully covered                     | Graceful degradation paths specified for every failure mode                                                                                                                      |
| NFR-6 (Concurrency Safety)                 | Fully covered                     | Isolated writes eliminate shared-state conflicts; sequential merge after parallel return                                                                                         |

---

## Key Assumptions to Validate

| #   | Assumption                                                                                                                          | Risk if Wrong                                                                                   | Referenced In   |
| --- | ----------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | --------------- |
| 1   | Compact decision tables in the orchestrator prompt produce the same routing reliability as full aggregator workflows                | Orchestrator misroutes (e.g., proceeds past Critical CT findings or misses R-Security override) | NFR-1, FR-4     |
| 2   | Planner can self-assemble task-failure mapping from 3 V artifacts as reliably as reading v-aggregator's structured Actionable Items | Replan targets wrong tasks; verification loop churns without converging                         | A-6, Tradeoff 3 |
| 3   | ≤5 bullet points per memory file capture sufficient detail for memory.md merge quality                                              | memory.md becomes less useful for downstream agents' memory-first orientation                   | NFR-2, FR-1.2   |
| 4   | 3 downstream agents independently synthesizing 4 research files is not materially worse than 1 synthesis step                       | Inconsistent interpretations of research findings across spec/designer/planner                  | FR-5, A-4       |

---

## Strategic Considerations

1. **The "+10 lines" metric is the wrong measure of orchestrator burden.** The real metric should be decision-point count or error-surface area. The orchestrator gains 3 new decision-point clusters (CT, V, R), each with multi-state evaluation, priority ordering, and safety-critical overrides. This is fundamentally different from 10 lines of static text. If the LLM orchestrator makes a routing error at any of these decision points, the consequences range from unnecessary revision loops (wasted compute) to passed security issues (safety failure). The design should include explicit self-verification steps (e.g., "log reasoning for routing decision before proceeding") as a guard against LLM reasoning errors in the compressed decision tables.

2. **The design's "pipeline step count" accounting is correct but could be more precise.** Net change: −4 full LLM agent invocations, +8 lightweight file-based merge operations, shifted synthesis work to 3 downstream agents. This IS a net improvement in wall-clock time (LLM invocations dominate latency), but the total LLM token expenditure may not decrease substantially because downstream agents read more inputs.

3. **Escape hatch is well-designed.** The design's deferred alternative — extracting decision tables to a separate reference file if the orchestrator exceeds ~450 lines — is a reasonable escape hatch. The design should commit to monitoring rather than hoping the 450-line threshold is never reached.

---

## Recommendations

1. **(High priority)** Acknowledge that orchestrator complexity growth is in decision density, not line count. Add explicit reasoning-logging or self-verification instructions to the orchestrator's CT/V/R decision flows to catch routing errors. Even a single line like "Log the routing decision and reasoning in memory.md before proceeding" would provide auditability.

2. **(High priority)** Validate Assumption A-6 explicitly: consider whether V-Tasks should include a structured "Failing Task IDs" list in its memory file (beyond the 5-bullet Key Findings), or whether the planner's replan workflow instructions should be enhanced with explicit cross-referencing guidance for the 3 V artifacts.

3. **(Medium priority)** Reframe the "+10 lines net" claim in NFR-1 to be transparent about the tradeoff: fewer lines of aggregator-invocation boilerplate replaced with more lines of dense decision logic. The current framing could lead to underestimating implementation risk.

4. **(Low priority)** Clarify that "improved parallel dispatch" in project communications means reduced pipeline depth (fewer sequential steps), not increased concurrency. The concurrency cap (≤4 agents) is unchanged.
