# Critical Review: Scalability & Performance

## Findings

### [Severity: High] Dispatch Count Claim (18-22) Only Holds for 6-Task Features; Scales Linearly to 40+ for Realistic Features

- **What:** The design's dispatch count analysis in §Decision 3 uses a "typical 6-task feature (2 waves of 3)" to derive the 18-22 dispatch claim. But NFR-3 explicitly requires handling "10+ tasks, 50+ files." For a 20-task feature (5 waves of 4), the dispatch count becomes: 4 (research) + 1 (spec) + 1 (designer) + 3 (design review, high-risk) + 1 (planner) + 20 (implementers) + 5 (verifiers, 1/wave) + 3 (code review, high-risk) + 1 (knowledge) = **39 dispatches minimum**. Worst case with 3 replan iterations: 4 + 1 + 1 + 6 (design review + revision) + 1 + 1 (replan) + 3×(20 + 5) + 2×3 + 1 = **~91 dispatches**. The 18-22 headline number significantly understates the cost for the features the system is required to handle.
- **Where:** design.md §Decision 3: Pipeline Structure → Dispatch Count Analysis
- **Likelihood:** High — any non-trivial feature generates 10+ tasks; the requirement (NFR-3) explicitly mandates this scale.
- **Impact:** Medium — the dispatch count itself is not the problem, but it misleads planning and sets incorrect expectations for orchestrator context window budgets and pipeline latency.
- **Assumption at risk:** That 6 tasks with 2 waves is "typical." In practice, features requiring a 9-agent pipeline with adversarial review are complex by definition.

---

### [Severity: High] Unified Verifier Context Window Explosion at Scale

- **What:** The Verifier is dispatched once per implementation wave and must verify all tasks in that wave (up to 4). For each task, it runs Tier 1-3 checks: IDE diagnostics, build, type checker, linter, tests, import/load, smoke execution. That is potentially 4 tasks × ~8 checks = **32 tool calls per Verifier dispatch**, each producing output (build logs, test output, diagnostic lists) that accumulates in the Verifier's context window. A large test suite's output alone can be thousands of lines. 32 tool calls × 100+ lines average output = 3200+ lines of tool output, plus the 4 implementation reports, spec-output.yaml, plan-output.yaml, the verification ledger, and the pipeline manifest. The Verifier is by far the most context-hungry agent in the system, yet the design rates it 🟢 on risk ("bounded scope — tool execution, not reasoning"). Tool execution with accumulating output IS the context window problem.
- **Where:** design.md §Decision 6: Verification Architecture, §Decision 2: Agent Inventory → Verifier detail
- **Likelihood:** High — any wave with 4 tasks and a real build/test system generates substantial tool output.
- **Impact:** High — context window overflow causes silent truncation (EC-8 says "MUST NOT silently truncate" but the design provides no mechanism to prevent it for the Verifier specifically). The Verifier losing context mid-cascade means incomplete verification evidence, which undermines the entire evidence-gating strategy.
- **Assumption at risk:** That the Verifier has "bounded scope" because it does "tool execution, not reasoning." Tool output accumulation is unbounded regardless of reasoning complexity.

---

### [Severity: High] Orchestrator Context Window Grows Monotonically Across Full Pipeline

- **What:** The orchestrator accumulates telemetry in its context window during Steps 0-8 and maintains the pipeline manifest. The research notes the existing Forge orchestrator at 544 lines is "pushing context limits" even before runtime accumulation. The new orchestrator eliminates memory merge dispatches but replaces them with: (a) reading and validating every agent's typed YAML output after each dispatch, (b) maintaining a growing pipeline manifest, (c) evaluating evidence gates by reading the verification ledger, and (d) accumulating telemetry for the entire run. For a 20-task feature with 39+ dispatches, the orchestrator must process 39+ agent completion contracts, validate 39+ YAML schemas, evaluate 10+ evidence gates, and manage routing decisions — all within a single persistent context window. There is no mechanism for the orchestrator to shed context between steps.
- **Where:** design.md §Decision 2 (Orchestrator agent detail — tool restrictions, inputs), §Decision 5 (zero-merge model — "orchestrator reads typed outputs directly"), §Decision 3 (pipeline structure — telemetry accumulation)
- **Likelihood:** High — the orchestrator's context is the most loaded in the system and never gets a fresh start.
- **Impact:** High — orchestrator context drift or truncation causes incorrect routing decisions, missed evidence gate failures, or dropped security blockers.
- **Assumption at risk:** That eliminating memory merge dispatches solves the orchestrator complexity problem. The merge dispatches are gone, but the orchestrator now inlines all that work within its own context.

---

### [Severity: High] YAML Verification Ledger Append-Only Growth Without Size Bounds

- **What:** The YAML fallback verification ledger is append-only (design.md §Decision 6). For a 20-task feature: 20 tasks × 2 phases (baseline + after) × ~8 checks = **320 records** in a single YAML file. With 3 replan iterations, this becomes **~960 records**. Each record has ~10 fields. The orchestrator counts records for evidence gating by scanning this file. The Verifier appends to it on every check. Every evidence gate evaluation is an O(N) scan of the full file. With 960 records and multiple gate evaluations, this is an N+1 pattern: for each of 20 tasks, scan 960 records to count matching entries = 19,200 record comparisons. The SQL primary path handles this efficiently via indexed queries, but the YAML fallback — which is the only guaranteed-available path — degrades to O(tasks × total_records).
- **Where:** design.md §Decision 6: Verification Architecture → YAML Schema (Fallback), Evidence Gating Mechanism
- **Likelihood:** Medium — only triggered when SQLite is unavailable, but the design explicitly treats YAML as the guaranteed fallback per EC-4.
- **Impact:** High — the evidence gating mechanism is the core enforcement layer for verification. If the YAML fallback makes evidence gating slow or causes context overflow when the orchestrator reads the ledger, the pipeline's primary safety mechanism degrades.
- **Assumption at risk:** That YAML is an adequate substitute for SQL for evidence gating at scale. YAML lacks indexes, query optimization, and efficient filtering.

---

### [Severity: Medium] Every Non-Research Agent Reads Full design-output.yaml and spec-output.yaml

- **What:** The zero-merge memory model means there is no curated summary of upstream artifacts. Every downstream agent reads the full typed YAML outputs of upstream agents. Specifically, design-output.yaml (the typed equivalent of a 1588-line design.md) and spec-output.yaml (typed equivalent of a 1091-line feature.md) are consumed by: Planner, every Implementer instance, Verifier, and Adversarial Reviewer. For a 20-task feature with 20 Implementer dispatches, that's 20 reads of these two large documents. Each Implementer's context budget is dominated by upstream YAML that is largely irrelevant to its specific task (an Implementer working on task-07 does not need the full design rationale for all 10 decisions). The design acknowledges this risk in §Decision 5: "downstream agents need to read more files (upstream outputs) instead of one curated summary" but the mitigation ("pipeline manifest provides an efficient orientation entry point") only helps navigate TO the files, not reduce their size.
- **Where:** design.md §Decision 5: Memory System Redesign → "What Gets Persisted and Why," §Decision 2: Agent Inventory → Implementer inputs
- **Likelihood:** High — this is the default behavior, not an edge case.
- **Impact:** Medium — context waste reduces the budget available for actual implementation work. For an Implementer, reading two 500+ line YAML documents plus its task spec plus the pipeline manifest before writing any code may leave insufficient context for complex tasks.
- **Assumption at risk:** That "typed YAML outputs ARE the pipeline state" is strictly better than curated summaries. It eliminates merge overhead but replaces it with read-amplification overhead.

---

### [Severity: Medium] Concurrency Cap of 4 Creates Linear Scaling Wall for Implementation

- **What:** The design enforces max 4 concurrent implementers per wave (C-6). For a 20-task feature, implementation requires 5 sequential waves. Each wave must complete and be verified before the next starts (the verification-replan loop is per-iteration, not per-wave in isolation). Total serial time for implementation = 5 × (implementation_time + verification_time). If each wave takes ~15 minutes (generous) and verification takes ~10 minutes, the implementation phase alone is 5 × 25 = 125 minutes. With replan loops (max 3 iterations), worst case is 3 × 125 = 375 minutes for implementation alone. The design does not discuss strategies for reducing wave count (e.g., task grouping, dependency-aware wave construction to minimize waves).
- **Where:** design.md §Decision 3: Pipeline Structure → Step 5-6, Parallelism Map — "≤4 implementers per wave"
- **Likelihood:** Medium — 20+ task features are explicitly in scope per NFR-3, but actual wall-clock time depends on task complexity.
- **Impact:** Medium — linear time scaling with task count makes the pipeline impractical for large features. Users may abandon the pipeline for simpler approaches.
- **Assumption at risk:** That max 4 concurrent is a platform constraint that cannot be worked around. The design doesn't explore whether wave partitioning could be optimized to minimize total waves.

---

### [Severity: Medium] Adversarial Review Model Call Budget for High-Risk Features Is Substantial

- **What:** For a high-risk feature (any 🔴 file), the total adversarial review budget is: Design review = 3 models × (1 round + max 1 revision round) = up to 6 dispatches. Code review = 3 models × max 2 rounds = up to 6 dispatches. **Total: up to 12 adversarial review dispatches.** Each dispatch involves an LLM reading design-output.yaml or git diff + verification evidence + implementation reports. For code review round 2, the reviewer must also read the previous round's verdicts. These are expensive model calls (full design or full diff review). The design does not quantify the total model-call budget or discuss cost implications — it simply states the reviewer count and max rounds.
- **Where:** design.md §Decision 7: Adversarial Review Design → Trigger Conditions, Review Cycling
- **Likelihood:** Medium — high-risk features trigger 3-model dispatch; the revision rounds are worst-case but bounded.
- **Impact:** Medium — 12 model calls for review alone is substantial overhead. Combined with 39+ other dispatches for a 20-task feature, the total model-call budget approaches 50+, each consuming significant tokens.
- **Assumption at risk:** That "always-on adversarial review" with 1 reviewer per standard feature is "minimal overhead compared to eliminated CT/R cluster dispatches." This is true for standard features but the high-risk path (3× models × 2 rounds) is expensive.

---

### [Severity: Medium] Replan Loop Replans Entire Task Scope, Not Just Failing Tasks

- **What:** The verification-replan loop (Steps 5-6) routes to "Planner (replan mode)" when verification finds issues. The design does not specify whether replanning targets only the failing tasks or re-decomposes the entire plan. If the Planner receives the full verification-report showing 2 out of 20 tasks failing, but re-decomposes all 20 tasks, the replan loop cost is O(total_tasks × iterations) rather than O(failing_tasks × iterations). Worst case: 3 iterations × 20 tasks × implementation + verification = enormous wasted effort re-implementing 18 passing tasks. The design mentions "Planner (replan mode) with verification findings" but does not define replan mode's scope.
- **Where:** design.md §Decision 3: Pipeline Structure → Steps 5-6, §Decision 6: Verification Architecture → Verification-Replan Loop
- **Likelihood:** Medium — replan is triggered when verification finds issues, which the design expects to happen.
- **Impact:** High — if replan re-executes all tasks rather than just failing ones, the cost multiplier is severe. For 20 tasks with 2 failing, re-implementing 20 instead of 2 wastes 90% of the computation.
- **Assumption at risk:** That "replan mode" implies targeted replanning. The design does not specify this.

---

### [Severity: Low] Pipeline Manifest as Context Orientation Grows Without Pagination

- **What:** The pipeline manifest schema includes a `steps` array with one entry per dispatch (agent, instance, model, status, output_path, retry_count) and an `error_log` array. For 39+ dispatches, the manifest grows to 300-500 lines of YAML. Every downstream agent reads this manifest first for orientation. The manifest has no pagination, summarization, or compact view — it's a flat list of all dispatch records from Step 0 to the current step. Late-pipeline agents (Verifier, Adversarial Reviewer, Knowledge Agent) read a manifest dominated by completed-step noise that provides no orientation value.
- **Where:** design.md §Pipeline Manifest Schema
- **Likelihood:** High — the manifest grows with every dispatch by design.
- **Impact:** Low — the manifest is a small fraction of most agents' context budgets, but it's wasted context that compounds with other bloat sources listed above.
- **Assumption at risk:** That the manifest is "small, typed YAML" suitable for efficient orientation. It grows linearly and is never summarized.

---

### [Severity: Low] No Strategy for Verifier Scaling Across Waves — Linear Ledger Re-reads

- **What:** The Verifier is dispatched once per wave. Each dispatch reads the verification ledger (which includes all previous waves' records) to check evidence gates and detect regressions. By wave 5 of a 20-task feature, the Verifier reads a ledger with 4 waves × 4 tasks × ~16 records = 256 previously accumulated records, plus performs its own 32+ new checks. The ledger is never partitioned by wave. The Verifier's context load increases linearly with each wave, meaning later waves are more likely to hit context limits than earlier ones.
- **Where:** design.md §Decision 6: Verification Architecture → verification-ledger.yaml schema, Baseline Capture Integration
- **Likelihood:** Medium — the ledger grows across waves within a single run.
- **Impact:** Low — the per-wave growth is moderate (64 records per wave), but combined with tool output accumulation it compounds the Verifier context window problem (see High severity finding above).
- **Assumption at risk:** That a flat append-only ledger is efficient for wave-over-wave verification.

---

## Cross-Cutting Observations

- **Maintainability scope:** The Verifier agent definition will be the most complex in the system — it must handle Tier 1-3 cascades, SQL/YAML dual-track writes, baseline comparison, regression detection, and evidence gating for multiple tasks simultaneously. This is the agent most likely to accumulate prompt debt over time. (ct-maintainability)
- **Security scope:** The YAML verification ledger fallback has no integrity protection. Since agents write YAML to a shared append-only file, a misbehaving or hallucinating agent could write fabricated verification records that pass evidence gates. SQL at least offers typed column constraints and CHECK clauses. (ct-security)
- **Strategy scope:** The design optimizes for the "typical 6-task feature" scenario throughout (dispatch counts, wave estimates) but the requirement (NFR-3) demands handling 10+ task / 50+ file features. The design should be stress-tested against the hard requirement, not the "typical" case. (ct-strategy)

---

## Requirement Coverage

| Requirement                                                                  | Coverage Status     | Notes                                                                                                                                                                                                     |
| ---------------------------------------------------------------------------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| NFR-3: Scalability (10+ tasks, 50+ files, no context window overflow)        | Partially Addressed | Wave-based execution prevents combinatorial explosion, but context window overflow for Verifier and Orchestrator at 20+ tasks is unmitigated. EC-8 mechanism ("request summarized context") is undefined. |
| NFR-5: Context Efficiency (summaries before full artifacts, concise prompts) | Partially Addressed | Pipeline manifest provides index, but zero-merge model forces full YAML reads rather than curated summaries. Every Implementer reads full design-output.yaml regardless of task scope.                    |
| EC-8: Agent Context Window Exceeded                                          | Weakly Addressed    | The edge case specifies "request orchestrator for summarized context" but the design defines no summarization mechanism, format, or trigger threshold.                                                    |
| EC-10: Concurrent Subagent Limit Hit                                         | Addressed           | Sub-wave partitioning for >4 tasks is specified.                                                                                                                                                          |
| EC-7: Verification Replan Loop Exhaustion                                    | Addressed           | Max 3 iterations with Confidence: Low proceeding. But replan scope (full vs. targeted) is undefined.                                                                                                      |
| AC-15: Bounded Retries                                                       | Addressed           | All retry budgets documented with explicit maximums.                                                                                                                                                      |
| C-6: Max 4 Concurrent Subagents                                              | Addressed           | Sub-wave partitioning enforced. Linear scaling consequence not discussed.                                                                                                                                 |
