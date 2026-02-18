# Critical Review: Strategy

## Summary

The design conflates two independent improvements — isolated memory files and aggregator removal — into a single change, and builds its justification on an unexamined assumption about where the bottleneck actually lies. Aggregators add one lightweight sequential step _after_ all parallel agents are already done; the real pipeline latency comes from the parallel agents themselves and the fully sequential core path (spec → designer → planner). Removing aggregators yields marginal latency savings while sacrificing valuable cross-agent synthesis (deduplication, contradiction detection, task-failure mapping), expanding the orchestrator into a single point of cognitive complexity, and touching 24 files for a gain the design never quantifies.

## Overall Risk Level

**High** — The design solves a real but minor problem (sequential merge steps) by removing a structural responsibility layer (aggregators) that provides value beyond just sequencing. The orchestrator absorbs responsibilities it was explicitly designed not to have, violating its own "never modify content directly" principle. The planner replan quality regression (loss of cross-referenced task-failure mapping) is a concrete, high-impact consequence that the design acknowledges but does not adequately mitigate.

---

## Fundamental Concerns

### 1. The Bottleneck Diagnosis Is Unexamined

**What:** The initial request states the goal is to "remove bottlenecks by eliminating aggregator agents and synthesis steps." But the design never quantifies _how much_ latency the aggregators actually add, nor compares it to other sources of pipeline latency.

**Why this matters:** The 8-step pipeline has a fully sequential critical path: Research → Spec → Designer → (CT) → Planner → Implementation → V-cluster → R-cluster. The aggregators run _after_ all parallel agents in their cluster have returned. They are lightweight merge-only operations — the ct-aggregator reads 4 files and compiles a combined document. This is fast relative to the CT sub-agents themselves, which each perform deep design analysis.

The actual sequential bottleneck is the core pipeline itself: spec, designer, and planner must run in strict sequence with no parallelism possible. Removing aggregators shaves one lightweight step from three parallel clusters but does nothing about the dominant sequential spine of the pipeline.

**Assumption at risk:** That aggregators are a _material_ bottleneck rather than a minor sequential tax after already-expensive parallel work.

### 2. Two Independent Concerns Are Unnecessarily Coupled

**What:** The design bundles two changes that could be implemented independently:

1. **Isolated memory files** — add `memory/<agent>.mem.md` as a compact output from every agent, with the orchestrator as the sole `memory.md` writer.
2. **Aggregator removal** — eliminate ct-aggregator, v-aggregator, r-aggregator, and researcher synthesis mode.

These are orthogonal. You could add isolated memory files while keeping aggregators — aggregators would simply _also_ write a `.mem.md` file. The orchestrator would still read compact memories for routing decisions, and aggregators would still produce combined artifacts for downstream consumption. This would achieve the "orchestrator reads memories, not full artifacts" goal without sacrificing the synthesis value that aggregators provide.

**Assumption at risk:** That isolated memory and aggregator removal must happen together to achieve the stated goal.

### 3. The Orchestrator Is Being Asked to Do Too Much

**What:** The orchestrator's role is explicitly defined as a _coordinator_: "You NEVER write code, tests, or documentation directly." Its current responsibilities are dispatch, memory lifecycle, and routing based on completion contracts. The design adds:

- CT cluster severity evaluation (reading 4 memory files, applying decision logic)
- V cluster decision table evaluation (a 10-row decision matrix)
- R cluster evaluation with security override (a 9-line priority-ordered decision sequence)
- Memory merge protocol (reading isolated memories, extracting fields, appending to `memory.md`)

This transforms the orchestrator from a router into an analyst. It now reads agent outputs and makes content-level judgments (severity assessment, cross-referencing statuses). The design estimates a net growth of +10 lines, but the _cognitive complexity_ growth is much larger — the orchestrator must now understand severity taxonomies (Critical/High/Medium/Low, Blocker/Major/Minor, PASS/FAIL), apply different decision logic per cluster, and parse structured memory files.

**Where:** [design.md](design.md) — "Component Responsibilities" table; NFR-1 analysis.

**Assumption at risk:** That LLM-based agents handle increased complexity within a single prompt proportionally to line count, rather than disproportionately degrading with added decision branches.

---

## Risks by Category

### Strategic Risks

| #   | Risk                                                                                                                                                                                                                                                                                                                                                             | Where                                                                                                | Likelihood | Impact | Assumption at Risk                                                                  |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------- | ------ | ----------------------------------------------------------------------------------- |
| S-1 | **Solving the wrong bottleneck.** Aggregators are lightweight post-parallel merge steps; the dominant sequential cost is in the core spec→designer→planner path, which this design does not address.                                                                                                                                                             | [initial-request.md](initial-request.md) — stated goal; [design.md](design.md) — no latency analysis | High       | Medium | Aggregators are a material bottleneck                                               |
| S-2 | **Blast radius disproportionate to value.** 24 files modified, 3 agents removed, 14 agents gain new output steps — for removing 3 lightweight sequential merge operations.                                                                                                                                                                                       | [design.md](design.md) — Implementation Checklist                                                    | High       | Medium | The engineering cost will be justified by the latency savings                       |
| S-3 | **Future cluster types require orchestrator bloat.** Adding any new cluster (e.g., a documentation review cluster) would require adding its decision logic inline to the orchestrator, growing the prompt and making it harder to maintain. With aggregators, you'd just create a new `x-aggregator.agent.md`.                                                   | [design.md](design.md) — "Component Responsibilities" table                                          | Medium     | High   | The system won't need new cluster types                                             |
| S-4 | **The orchestrator becomes a single point of cognitive failure.** All routing decisions, all memory merging, and all cluster evaluation logic in one prompt. If the LLM misinterprets the V decision table or skips the R-Security override check, there is no second agent to catch the error. Currently, aggregators serve as an independent validation layer. | [design.md](design.md) — CT/V/R Decision Flows                                                       | Medium     | High   | The orchestrator LLM will reliably execute all decision branches in a single prompt |

### Scope Risks

| #    | Risk                                                                                                                                                                                                                                                                                                                                                          | Where                                                          | Likelihood | Impact | Assumption at Risk                                                                        |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- | ---------- | ------ | ----------------------------------------------------------------------------------------- |
| SC-1 | **Valuable aggregation capabilities lost without quantified tradeoff.** Deduplication, Unresolved Tensions detection, cross-cutting concern synthesis, and task-failure mapping are all eliminated. The design acknowledges these as tradeoffs (A-4, A-5, A-6) but frames them as acceptable without evidence that downstream agents can replicate the value. | [design.md](design.md) — "Tradeoffs & Alternatives Considered" | High       | Medium | Downstream agents can self-serve the analysis previously provided by aggregators          |
| SC-2 | **Removing `analysis.md` forces 3 downstream agents to each independently parse 4 research files.** Spec, designer, and planner each transition from reading 1 combined file to 4 individual files. This triples the input processing burden at each of the 3 sequential stages and may produce inconsistent interpretations across agents.                   | [design.md](design.md) — "Downstream Input Changes"            | Medium     | Medium | Agents will reach the same conclusions from 4 individual files as from 1 synthesized file |

### Complexity Risks

| #   | Risk                                                                                                                                                                                                                                                                                                                                                                          | Where                                                                          | Likelihood | Impact | Assumption at Risk                                 |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ---------- | ------ | -------------------------------------------------- |
| C-1 | **The "uniform branch-merge model" adds ceremony without benefit for sequential agents.** Spec, designer, and planner run alone, sequentially. Having them write isolated memory files that the orchestrator immediately merges is functionally identical to having them write to `memory.md` directly — but adds an extra file and an extra merge step per sequential agent. | [design.md](design.md) — "Sequential Agent Memory Model Migration"; Tradeoff 4 | High       | Low    | Uniformity is worth the extra file I/O             |
| C-2 | **Memory file proliferation.** A typical pipeline run creates ~20+ `.mem.md` files (4 researchers + spec + designer + 4 CT + planner + N implementers + 4 V + 4 R). These files serve only the orchestrator's routing decisions and have no downstream consumers. After the pipeline completes, they are dead artifacts.                                                      | [design.md](design.md) — "Isolated Memory File Format"                         | High       | Low    | The file count is acceptable overhead              |
| C-3 | **The branch/merge metaphor is misleading.** In git, merges involve conflict detection and resolution. The design's "merge" is sequential append with no conflict resolution. Calling it branch/merge creates false expectations about the system's behavior when agents produce contradictory findings.                                                                      | [design.md](design.md) — High-Level Architecture                               | Low        | Low    | Terminology does not affect implementation quality |

### Integration Risks

| #   | Risk                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Where                                                                                                                                                                      | Likelihood | Impact | Assumption at Risk                                                     |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ------ | ---------------------------------------------------------------------- |
| I-1 | **Planner replan quality regression.** The V-aggregator currently cross-references failing task IDs from `v-tasks.md` with test failures from `v-tests.md` and feature gaps from `v-feature.md`, producing a unified Actionable Items section with prioritized, task-mapped fixes. Without this, the planner must self-assemble the failure mapping from 3 separate files. This is a non-trivial analytical task that the planner was not designed to perform. | [v-aggregator.agent.md](../../NewAgentsAndPrompts/v-aggregator.agent.md) — Workflow Step 5 "Map Failures to Task IDs"; [design.md](design.md) — Tradeoff 3, Assumption A-6 | High       | High   | The planner can replicate the V-aggregator's cross-referencing quality |
| I-2 | **Designer revision quality regression.** The ct-aggregator currently deduplicates CT findings and surfaces contradictions as Unresolved Tensions. Without aggregation, the designer in revision mode reads 4 individual CT review files and must mentally merge, deduplicate, and detect contradictions. This increases the designer's cognitive load during the most critical part of the pipeline (design correction).                                      | [ct-aggregator.agent.md](../../NewAgentsAndPrompts/ct-aggregator.agent.md) — Workflow Steps 5, 7, 8; [design.md](design.md) — Tradeoff 1, 2                                | Medium     | Medium | The designer can effectively self-merge 4 CT review files              |
| I-3 | **Unresolved Tensions between CT sub-agents silently disappear.** The ct-aggregator explicitly surfaces contradictions (e.g., security says "add encryption" while scalability says "encryption adds unacceptable latency"). Without aggregation, these contradictions persist undetected in separate files. The designer might address one CT agent's finding while unknowingly contradicting another's.                                                      | [ct-aggregator.agent.md](../../NewAgentsAndPrompts/ct-aggregator.agent.md) — Workflow Step 8                                                                               | Medium     | Medium | Contradictions between CT agents are rare enough to ignore             |

---

## Requirement Coverage Gaps

| Requirement                                                   | Coverage Status     | Gap                                                                                                                                                                                                                                                                               |
| ------------------------------------------------------------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-4.1 (CT decision logic)                                    | Covered             | —                                                                                                                                                                                                                                                                                 |
| FR-4.2 (V decision table)                                     | Covered but at risk | The decision table is implemented, but the _downstream effect_ (planner replan using individual V files instead of aggregated Actionable Items) is a quality regression not addressed by any FR.                                                                                  |
| FR-4.3 (R decision logic + security override)                 | Covered but fragile | The R-Security override is safety-critical. Moving it from a dedicated agent (r-aggregator) to inline orchestrator logic removes a defense-in-depth layer. If the orchestrator's prompt is long enough that the LLM skips this check, security findings could be silently passed. |
| A-4 / A-5 (Loss of deduplication and contradiction detection) | Explicitly accepted | These are stated as assumptions but never validated. The design asserts downstream agents "can identify contradictions themselves" without evidence or updated workflow instructions for how they would do so.                                                                    |
| EC-11 (Planner replan without aggregated mapping)             | Partially covered   | The design acknowledges the risk and says "Planner may need enhanced instructions for self-assembly." But the design does not include those enhanced instructions — it defers the mitigation.                                                                                     |

---

## Key Assumptions

| #   | Assumption                                                                                              | Validity                                                                                                                                                                                                                                                                                        | If Wrong                                                                                           |
| --- | ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| A-1 | Aggregators are the primary bottleneck in the pipeline.                                                 | **Unvalidated.** No latency data or analysis provided. Aggregators run after parallel work completes and perform lightweight merge operations. The core sequential path (spec → designer → planner) is likely the dominant bottleneck.                                                          | The entire motivation for this change is undermined.                                               |
| A-4 | Loss of deduplication is acceptable because the orchestrator reads only compact memories for routing.   | **Partially valid.** True for orchestrator routing, but false for the designer in revision mode, who now reads 4 undeduped CT files and must mentally merge them.                                                                                                                               | Designer revision quality degrades; duplicate findings waste designer context window.              |
| A-5 | Loss of contradiction detection is acceptable because downstream consumers can identify contradictions. | **Unvalidated.** No downstream agent has instructions for contradiction detection. The designer has no workflow step for "check if CT sub-agents contradict each other."                                                                                                                        | Silent contradictions in design revision; downstream agents implement conflicting recommendations. |
| A-6 | The planner can self-assemble failure-to-task mapping from individual V files.                          | **Risky.** The V-aggregator's Step 5 is a non-trivial 4-step process: extract failing tasks, cross-reference with test failures, cross-reference with feature gaps, produce prioritized Actionable Items. The planner has no equivalent workflow and was not designed for this analytical task. | Replan quality degrades; untargeted fixes; wasted implementation iterations.                       |

---

## Strategic Considerations

### 1. The "Aggregator as Agent" Pattern Has Structural Value Beyond Latency

Aggregators serve three purposes: (a) merge/deduplicate sub-agent outputs, (b) detect cross-agent contradictions, and (c) produce a completion contract for routing. The design frames aggregators purely as latency bottlenecks (purpose c), ignoring that purposes (a) and (b) deliver real analytical value that no other agent replaces. Removing them optimizes for latency at the expense of output quality.

### 2. A Smaller Change Could Achieve 80% of the Benefit

If the goal is to reduce sequential steps, consider two simpler alternatives that the design considered but rejected without sufficient justification:

- **Alternative A: Remove synthesis mode only.** Step 1.2 (researcher synthesis → `analysis.md`) is the one aggregation step that genuinely blocks the pipeline with a heavy operation (merging 4 research files into a coherent analysis). Removing it and having downstream agents read research files directly eliminates the most impactful bottleneck. Keep the other aggregators for their synthesis and routing value. This is a 6-8 file change instead of 24.

- **Alternative B: Keep aggregators, add isolated memory files.** Aggregators continue to produce combined artifacts for downstream agents, but also write `.mem.md` files. The orchestrator reads memories for fast routing while downstream agents still get deduplicated, synthesized artifacts. This achieves the "orchestrator reads compact memories, not full artifacts" goal without sacrificing aggregation quality.

### 3. Orchestrator Complexity Is a Maintenance Trap

Every time the team adds a new cluster type or changes routing logic, they must modify the single most critical prompt in the system. With aggregators, new cluster types are encapsulated in new agent files that follow a well-established pattern. Without aggregators, the orchestrator must grow to accommodate each new decision domain. Over time, this creates a "god prompt" that accumulates responsibilities and becomes increasingly fragile.

### 4. The Design Creates Silent Quality Regressions

The most concerning aspect is not what the design _changes_ but what it _loses_ without replacement:

- The designer in revision mode loses a deduplicated, contradiction-surfaced view of CT findings.
- The planner in replan mode loses a cross-referenced, task-mapped view of verification failures.
- These agents must now do additional analytical work they weren't designed for, with no updated workflow instructions in the design to support them.

The design explicitly acknowledges these losses (A-4, A-5, A-6) but frames them as acceptable without evidence. Assumption A-6 (planner can self-assemble failure mapping) is particularly risky given that the V-aggregator's Step 5 is a dedicated 4-step cross-referencing process.

### 5. Memory File Proliferation Creates Noise

A single pipeline run will create 20+ `.mem.md` files that exist solely for the orchestrator's routing decisions. After the pipeline completes, they are dead artifacts with no downstream value. This is not a blocking concern, but it works against the system's goal of clear documentation structure — every pipeline run leaves behind a `memory/` directory full of transient routing metadata mixed in with the feature's real documentation output.

---

## Recommendations

1. **Decouple isolated memory from aggregator removal.** Implement isolated memory files (FR-1, FR-2) as a standalone change. This achieves "orchestrator reads memories, not full artifacts" without the risks of aggregator removal. Evaluate aggregator removal as a separate, subsequent decision.

2. **If aggregators must be removed, preserve the V-aggregator's task-failure mapping.** The planner replan path is the highest-risk regression. Either: (a) add an explicit "Map Failures to Task IDs" step to the planner's replan workflow, or (b) have the orchestrator produce a lightweight failure summary from V memories as input to the planner.

3. **If aggregators must be removed, add contradiction detection instructions to the designer's revision workflow.** The designer should have a workflow step: "Before addressing individual CT findings, scan all 4 CT review files for contradictions between sub-agents. Note any conflicting recommendations and address them explicitly in the revised design."

4. **Quantify the bottleneck.** Before committing to a 24-file change, measure or estimate how much latency each aggregator step actually adds relative to the total pipeline time. If aggregators add <5% of total pipeline time, the complexity and quality cost of removing them may not be justified.

5. **Reconsider the "uniform branch-merge" model for sequential agents.** Spec, designer, and planner gain nothing from isolated memory files — the orchestrator can read their primary artifacts directly after they complete (since they run sequentially, there's no concurrency benefit). Exempting them simplifies the design and reduces file count.

---

## Cross-Cutting Observations

- The design is thorough and well-structured. The acceptance criteria are comprehensive, the testing strategy is solid, and the failure/recovery analysis is detailed.
- The fundamental challenge is that the design has been built backward: it starts from the solution (remove aggregators, add isolated memory) and works outward to justify it, rather than starting from the problem (pipeline is slow) and evaluating solutions against measured data.
- The 648-line design document for modifying prompt files is itself a signal of scope expansion. The complexity of the design reflects the complexity of the change, which reflects an over-scoped solution to an under-analyzed problem.
