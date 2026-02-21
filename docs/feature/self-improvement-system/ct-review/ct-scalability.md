# Critical Review: Scalability & Performance (Revision 1 Re-Review)

> **Re-Review of Revision 1.** Previous review identified 9 findings (2 High, 5 Medium, 2 Low). The designer addressed 1 Critical and 6 High findings from across the CT cluster. This re-review assesses which previous findings are resolved, which remain, and identifies any new concerns introduced by the revision.

## Previous Finding Disposition

| #   | Previous Finding                                   | Previous Severity | Disposition                                                                                                                                                                                                       |
| --- | -------------------------------------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| F1  | memory.md telemetry grows without bound            | High              | **RESOLVED** — Telemetry moved to orchestrator context passing; memory.md no longer contains telemetry                                                                                                            |
| F2  | PostMortem context window saturation               | High              | **DOWNGRADED to Medium** — Memory-first reading with targeted grep_search reduces per-file context usage, but stale-file accumulation and lack of run correlation leave ingestion volume unbounded across re-runs |
| F3  | Orchestrator agent definition size growth          | High              | **DOWNGRADED to Medium** — Track B separation reduces single-change impact; Track A still adds ~4–6 KB to an already 37.5 KB definition                                                                           |
| F4  | N+1 file reads in PostMortem                       | Medium            | **DOWNGRADED to Low** — grep_search extraction reduces full-read count, though discovery + N grep calls remains                                                                                                   |
| F5  | Evaluation file accumulation with no cleanup       | Medium            | **UNCHANGED (Medium)** — No cleanup, archival, or run-correlation mechanism added                                                                                                                                 |
| F6  | Telemetry recording overhead on every dispatch     | Medium            | **RESOLVED** — No memory tool calls for telemetry; accumulation is in orchestrator context                                                                                                                        |
| F7  | Stale evaluation data pollution across re-runs     | Medium            | **UNCHANGED (Medium)** — No run-correlation ID; PostMortem still discovers ALL evaluation files via file_search                                                                                                   |
| F8  | Cumulative evaluation context pressure (14 agents) | Low               | **UNCHANGED (Low)** — Inherent to the design; non-blocking mitigates pipeline-failure risk                                                                                                                        |
| F9  | Pattern C replan loop amplification                | Low               | **NARROWED to Low** — Telemetry overhead removed; amplification now limited to file accumulation and PostMortem ingestion                                                                                         |

**Summary:** 2 of 9 findings fully resolved (F1, F6). 3 downgraded in severity (F2, F3, F4). 3 unchanged (F5, F7, F8). 1 narrowed in scope (F9). 1 new finding identified (F10).

---

## Findings

### [Severity: Medium] Evaluation File Accumulation Without Run Correlation (F5+F7 — Unchanged)

- **What:** The append-only storage design (§Storage Architecture → Append-Only Semantics) creates evaluation files with bare agent names (`spec.md`) on first run and sequence suffixes (`spec-2.md`, `spec-3.md`) on subsequent runs. PostMortem discovers all files in `artifact-evaluations/` via `file_search` (§PostMortem Workflow step 4). After N re-runs, the directory contains N × 14+ evaluation files with no run-correlation ID linking them to a specific pipeline execution. PostMortem's telemetry data (passed from orchestrator context) identifies which agents ran but does NOT reference evaluation files by their actual sequence-suffixed filenames. This disconnect means PostMortem cannot reliably determine which evaluation files belong to the current run.
- **Where:** design.md §Storage Architecture → Append-Only Semantics; §PostMortem Agent Definition → Workflow steps 4–5; §Telemetry Accumulation → Telemetry dispatch prompt format (no evaluation file path references).
- **Likelihood:** Medium — features undergoing active development commonly have 3–5+ re-runs.
- **Impact:** Medium — PostMortem computes aggregate metrics (averages, frequency counts) across stale AND current evaluation data, producing misleading accuracy scores and recurring-issue analysis. The memory-first reading pattern cannot mitigate this because grep_search for scores returns hits from all files regardless of run provenance.
- **Assumption at risk:** That PostMortem processes only current-run evaluation files. Nothing in the design constrains discovery to current-run files.

### [Severity: Medium] PostMortem Ingestion Volume Remains Unbounded Across Re-Runs (F2 — Downgraded from High)

- **What:** The memory-first reading pattern (§PostMortem Agent Definition → Memory-First Reading Pattern) significantly reduces per-file context usage by using grep_search for score extraction before selective full reads. However, the total number of files PostMortem must process is unbounded across re-runs. After 5 re-runs on a feature with 20 implementation tasks: 5 × (14 base evaluations + ~20 implementer/doc-writer evaluations) = **~170 evaluation files** in the directory. Even with grep-only extraction, this means ~170 grep_search calls for score extraction plus discovery overhead. The design's instruction to "read full content only of evaluation files that indicate problems" still requires scanning all files to identify which ones have problems.
- **Where:** design.md §PostMortem Agent Definition → Memory-First Reading Pattern steps 3–4; §Storage Architecture → Append-Only Semantics.
- **Likelihood:** Low-Medium — extreme case requires many re-runs with large task counts.
- **Impact:** Medium when triggered — tool call volume scales linearly with accumulated files; context window fills with grep results from stale files.
- **Assumption at risk:** That grep_search-based extraction is efficient at any file count. At 170+ files, the cumulative tool call cost and result volume becomes significant.

### [Severity: Medium] Orchestrator Context Window Pressure from Telemetry Accumulation (New — F10)

- **What:** The revised design moves telemetry from memory.md to "the orchestrator's working context" (§Telemetry Accumulation). The orchestrator is an LLM — it does not have structured working memory. Telemetry observations are embedded in the conversation history as the orchestrator processes each step. By Step 7, the orchestrator's conversation includes: its 37.5 KB agent definition, all tool calls/responses from Steps 0–7 (memory file reads, completion contract observations, memory merges), AND the accumulated telemetry observations. When the orchestrator must format the telemetry for the PostMortem dispatch at Step 8, it must recall observations scattered across its entire conversation history. For a standard run, the telemetry table is ~1.7 KB — modest. For a Pattern C worst case (64 dispatches), the telemetry table is ~5.3 KB. The risk is not the table size but the **recall accuracy**: observations from Step 1 are far back in the conversation by Step 8, and LLMs exhibit degraded recall for information in the middle of long contexts.
- **Where:** design.md §Telemetry Accumulation ("orchestrator accumulates telemetry data in its working context"); §Step 8 Addition ("includes full telemetry dataset").
- **Likelihood:** Medium — standard runs have modest telemetry volume; Pattern C runs with many tasks push the boundary.
- **Impact:** Medium — the orchestrator may produce incomplete or inaccurate telemetry in the PostMortem dispatch prompt, particularly for early-pipeline dispatches (Steps 1.1, 2) by the time it reaches Step 8. Since the telemetry is ephemeral (no persistent checkpoint), there is no way to verify completeness.
- **Assumption at risk:** That the orchestrator LLM can accurately recall and reconstruct dispatch metadata for 19–64 agents from its conversation history spanning Steps 0–7.

### [Severity: Medium] Orchestrator Definition Size Continues to Grow (F3 — Downgraded from High)

- **What:** The orchestrator agent definition is 37.5 KB / 485 lines — already more than double the next-largest agent (planner at 15.9 KB). Track A additions include: Step 8 section (~1 KB), telemetry accumulation instructions at 9 dispatch points (~2 KB), Documentation Structure additions (~0.5 KB), Expectations table row, Parallel Execution Summary update, Completion Contract update, Anti-Drift Anchor update, Memory Lifecycle update (~1 KB). Estimated Track A growth: **~4–6 KB**, pushing the orchestrator to **~42–44 KB**. Track B is separated (good — CT-2 resolution), but if both tracks are eventually implemented, total growth is ~8–10 KB to **~46–48 KB**.
- **Where:** design.md §Implementation Checklist, deliverables A4–A6 (Track A orchestrator changes); B1–B3 (Track B orchestrator changes); current orchestrator.agent.md at 485 lines / 37.5 KB.
- **Likelihood:** High — the additions are specified in the design.
- **Impact:** Medium — every token of the orchestrator definition is loaded into context for every tool call and reasoning step. At 42–44 KB (Track A only), the orchestrator definition consumes a significant fraction of available context, leaving less room for the conversation history where telemetry data must be recalled. This compounds with finding F10 (telemetry recall accuracy).
- **Assumption at risk:** That the orchestrator LLM's reasoning quality does not degrade as its agent definition grows from 37.5 KB toward 44+ KB.

### [Severity: Low] N+1 Tool Call Pattern Reduced but Not Eliminated in PostMortem (F4 — Downgraded)

- **What:** The memory-first reading pattern replaces wholesale file reads with grep_search extraction, reducing context per-read. However, the workflow (§PostMortem Workflow steps 4–5) still follows a discovery-then-iterate pattern: 1 file_search + N grep_search calls for score extraction + selective full reads. For a standard run (14+ evaluation files), this is ~16+ tool calls for evaluation processing alone. With memory file reads (19+ files) and telemetry extraction, PostMortem issues **~25–35 tool calls** before analysis — down from the previous estimate of 35–50+ but still substantial.
- **Where:** design.md §PostMortem Agent Definition → Workflow steps 3–5; §Memory-First Reading Pattern.
- **Likelihood:** High — every pipeline run.
- **Impact:** Low — reduced from previous Medium severity because grep_search calls are lighter-weight than full file reads. Still a linear-scaling pattern.
- **Assumption at risk:** That 25–35 tool calls is within comfortable operating range for a single agent invocation.

### [Severity: Low] Telemetry Data Lost on Failed Runs — Exactly When Most Needed (New Observation)

- **What:** The design acknowledges (§Telemetry Accumulation → Resilience trade-off) that if the orchestrator crashes before Step 8 or PostMortem fails, structured telemetry is lost. The justification is that agent memories persist completion statuses and that orchestrator crash terminates the pipeline anyway. However, failed runs produce the most diagnostically valuable telemetry — retry counts, failure reasons, iteration numbers. The fallback (agent memories contain completion statuses) provides only a binary DONE/ERROR per agent, not the structured telemetry (retries, timing, failure reasons) that would help diagnose why the run failed.
- **Where:** design.md §Telemetry Accumulation → "Resilience trade-off" callout; §Failure & Recovery → "Orchestrator telemetry lost" row.
- **Likelihood:** Low — orchestrator crashes are rare; PostMortem failures are mitigated by non-blocking design and self-verification.
- **Impact:** Low in frequency but Medium in value when triggered — the runs where telemetry is lost are precisely the runs where it would be most useful for diagnosis.
- **Assumption at risk:** That agent memory files provide sufficient diagnostic data as a fallback for lost telemetry.

### [Severity: Low] Pattern C Amplification on File Accumulation (F9 — Narrowed)

- **What:** Pattern C replan loops (max 3 iterations) multiply evaluation file creation: each iteration re-runs planner + implementers + V agents, each producing evaluation files with sequence suffixes. In a worst case (3 iterations, 10 tasks): 3 × (1 planner + 10 implementers + 4 V agents) = ~45 additional evaluation files in a single run. Combined with the stale-file accumulation across re-runs (F5), the `artifact-evaluations/` directory can grow to hundreds of files rapidly. The telemetry amplification concern from the previous review is now resolved (context-based telemetry adds no per-dispatch file writes), narrowing this finding to file accumulation only.
- **Where:** design.md §Sequence / Interaction Notes → Pattern C Replan Loop Telemetry; §Storage Architecture → Append-Only Semantics.
- **Likelihood:** Low — 3-iteration Pattern C with 10+ tasks is uncommon.
- **Impact:** Low-Medium when triggered — primarily affects PostMortem ingestion volume (compounds F2) and file_search discovery performance.
- **Assumption at risk:** That the file system and tool infrastructure handle hundreds of files in a single directory without degradation.

---

## Cross-Cutting Observations

- **[CT-Strategy scope]** The stale evaluation data pollution issue (F5+F7) is fundamentally a **correctness** problem, not just scalability. PostMortem reports that blend current-run and prior-run evaluations produce incorrect metrics. A run-correlation mechanism (e.g., run ID in telemetry cross-referenced with evaluation file names) would address both the scalability and correctness dimensions. The design's telemetry dispatch prompt (§Telemetry Accumulation) could include the expected evaluation file paths, enabling PostMortem to filter discoveries.

- **[CT-Maintainability scope]** The orchestrator's growing definition size (F3) is increasingly a maintainability concern. At 37.5 KB, it is 2.4× the next-largest agent. Adding 4–6 KB of self-improvement instrumentation (Track A) compounds an existing structural issue. The design references dispatch-patterns.md as an external reference document — a similar factoring pattern could be applied to telemetry instructions, reducing inline content.

- **[CT-Maintainability scope]** The PostMortem's memory-first reading pattern (5-step strategy in §Memory-First Reading Pattern) adds workflow complexity that is specific to working around context window limits. If the underlying constraint changes (larger context windows, better tool batching), this complexity becomes technical debt.

---

## Requirement Coverage

| Requirement                                                  | Coverage Status                                   | Notes                                                                                                                     |
| ------------------------------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| FR-2.1 (Telemetry at 9 dispatch points)                      | Improved — overhead concern resolved              | Context-based accumulation eliminates per-dispatch tool calls; new concern: recall accuracy at Step 8 (F10)               |
| FR-2.5 (Pattern C per-iteration telemetry)                   | Improved — overhead resolved                      | File amplification remains (F9) but telemetry write overhead eliminated                                                   |
| FR-3.2 (PostMortem reads evaluations + telemetry + memories) | Improved — context saturation partially mitigated | Memory-first reading reduces per-file cost; unbounded file discovery across re-runs remains (F2, F5)                      |
| FR-4.3 (Append-only storage)                                 | Unchanged — no cleanup mechanism                  | File accumulation (F5) and stale data pollution (F5+F7) persist; inherent tension between append-only and scalability     |
| NFR-4 (Non-blocking evaluations)                             | Adequately addressed                              | Non-blocking pattern prevents pipeline failure; context waste is inherent                                                 |
| NFR-6 (Graceful degradation)                                 | Improved                                          | Memory-first reading and telemetry context passing reduce failure modes; telemetry loss on crash acknowledged             |
| NFR-7 (Per-feature isolation)                                | Adequately addressed                              | Per-feature scoping bounds growth; re-run accumulation within a feature remains unbounded                                 |
| EC-7 (Pattern C telemetry accumulation)                      | Improved                                          | No per-dispatch file writes; context accumulation is modest (~5.3 KB worst case); recall accuracy is the residual concern |
| EC-11 (Second pipeline run append-only)                      | Unchanged — stale data concern persists           | Sequence suffix naming prevents overwrites but doesn't enable run correlation                                             |
