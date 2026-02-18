# Critical Review: Security & Backwards Compatibility

## Summary

The design is thorough and well-structured, but has **one high-severity gap** in the R-Security pipeline blocker preservation path: the R-Security agent's own prompt still references the R Aggregator as the enforcement mechanism for the pipeline override, and the design's Tier 4 changes underspecify the updates needed to that agent's Pipeline Blocker Override Rule section. There is also a **high-severity backwards compatibility gap** in the planner's replan mode detection, where the file-existence trigger (`verifier.md`) is removed without defining a replacement detection mechanism. These issues are addressable but must be resolved before implementation.

**Overall Risk Level: Medium** — The core decision logic is correctly migrated. The primary risks are implementation fidelity gaps that could silently weaken safety guardrails.

---

## Findings

### [Severity: High] R-Security Pipeline Blocker Override — Prompt Reference Stale After Aggregator Removal

- **What:** The R-Security agent's prompt contains an explicit `Pipeline Blocker Override Rule` section (r-security.agent.md lines 61–67) that states: _"the R Aggregator MUST override its aggregated result to ERROR."_ Its completion contract section (line 225) also says: _"If any finding has Blocker severity, the R Aggregator will override the pipeline result to ERROR."_ Its Anti-Drift Anchor (line 229) says: _"Your Critical/Blocker findings block the entire pipeline."_ After aggregator removal, the R Aggregator no longer exists. If these references are not updated, R-Security's self-understanding of how its findings propagate is wrong. The agent may not prioritize populating the `Highest Severity` field in its new memory file because it believes the R Aggregator (which reads its full artifact) handles escalation. The design's Tier 4 changes list r-security as receiving the "same set of changes" as all 14 sub-agents — generic memory output addition, anti-drift update, and aggregator reference removal. But r-security is NOT a generic sub-agent. It has a unique `Pipeline Blocker Override Rule` section that specifically describes the R Aggregator's role. This section needs targeted updates beyond the generic pattern.
- **Where:** Design section "Implementation Checklist — Tier 4", r-security.agent.md lines 61–67 and 225
- **Likelihood:** H
- **Impact:** H — If R-Security doesn't correctly populate `Highest Severity` in its memory file, the orchestrator cannot enforce the pipeline blocker. Security-critical findings could pass silently.
- **Assumption at risk:** The design assumes generic Tier 4 changes are sufficient for r-security. They are not — r-security has unique pipeline-blocker language that needs explicit rewriting to reference the orchestrator (not aggregator) as the enforcement mechanism, AND to emphasize that the `Highest Severity` memory field is the new vehicle for conveying this.

### [Severity: High] Planner Replan Mode Detection Breaks Without `verifier.md`

- **What:** The planner's mode detection logic (planner.agent.md line 59) currently reads: _"Replan mode: `verifier.md` exists with failures → create a remediation plan."_ The orchestrator invokes the planner "in replan mode with verifier.md" (orchestrator.agent.md line 101). The design removes `verifier.md` entirely (FR-3.2, FR-6.2). The design's Tier 3 planner changes say to "Replace `verifier.md` (replan) with `verification/v-tasks.md`, `verification/v-tests.md`, `verification/v-feature.md`" and "Update replan mode workflow to read individual V artifacts." But the design does NOT explicitly define the **new mode detection trigger**. How does the planner know it's in replan mode? Currently: `verifier.md` exists → replan. In the new model: individual V files (`verification/v-tasks.md`) might exist whether V passed or failed. The file's existence alone no longer signals replan mode. The planner needs updated mode detection logic — e.g., "verification/v-tasks.md exists **with failures**" or "orchestrator provides explicit mode signal."
- **Where:** Design section "APIs & Interfaces — Downstream Input Changes" (planner.agent.md), Feature spec FR-6.2
- **Likelihood:** H
- **Impact:** H — If the planner cannot reliably detect replan mode, it may attempt full replanning when only targeted fixes are needed (wasting tokens) or skip replanning when it's required. This breaks the Pattern C replan loop.
- **Assumption at risk:** Assumption A-6 states the planner can self-assemble failure mapping from individual V artifacts. Even if true, the mode detection mechanism itself is unaddressed.

### [Severity: High] Task-Failure Cross-Referencing Quality Degradation

- **What:** The v-aggregator's Step 5 ("Map Failures to Task IDs") performs critical cross-referencing: it correlates failing tests from `v-tests.md` with task IDs from `v-tasks.md`, and feature-level gaps from `v-feature.md` with responsible task IDs. The Actionable Items table it produces is the planner's primary input for targeted replanning (v-aggregator.agent.md line 103-107, 156-161). The design acknowledges this in Tradeoff 3 and rates it High severity / Medium likelihood. But the mitigation ("Planner may need enhanced instructions") is deferred — the design says "Monitor replan quality in early usage." For a safety-critical pipeline, "monitor and hope" is insufficient as a mitigation for a known High-severity risk.
- **Where:** Design section "Tradeoffs — Tradeoff 3", v-aggregator.agent.md Steps 5-6, planner.agent.md replan workflow
- **Likelihood:** M
- **Impact:** H — Without cross-referenced failure mapping, the planner in replan mode must independently correlate test failures, task failures, and feature gaps across three separate files. If it does this poorly, replanning targets the wrong tasks, wasting implementation cycles within the 3-iteration Pattern C limit. Worst case: 3 iterations exhausted without fixing the right issues.
- **Assumption at risk:** Assumption A-6 assumes the planner can self-assemble the cross-reference. This is untested. The v-aggregator was created precisely because this cross-referencing is non-trivial.

### [Severity: Medium] Severity Taxonomy Confusion: Blocker vs. Critical in R-Security Memory

- **What:** R-Security uses "Blocker" and "Critical" somewhat interchangeably. r-security.agent.md line 66 says: _"Severity: Blocker (Critical)"_ — parenthetical suggests they're synonyms. The design's memory template for R agents specifies `Blocker/Major/Minor` (design.md Data Models section). The design's R Cluster Decision Flow step 3 checks for `Blocker or Critical`. But the memory template only lists `Blocker/Major/Minor` — there's no `Critical` in the R taxonomy. If R-Security writes "Critical" instead of "Blocker" in its memory file (which is plausible given its own prompt says "Critical severity"), the orchestrator would need to match both terms. The design's Input Validation Strategy says "Unexpected values are treated as the worst-case severity for safety" — which would catch this. But relying on fallback validation for the most safety-critical field in the pipeline is fragile.
- **Where:** Design section "Data Models — Isolated Memory File Format", r-security.agent.md line 66, design.md R Cluster Decision Flow step 3
- **Likelihood:** M
- **Impact:** M — Mitigated by worst-case fallback, but adds unnecessary ambiguity to the most critical routing decision in the pipeline.
- **Assumption at risk:** The design assumes R-Security will use "Blocker" (the R taxonomy term) rather than "Critical" (R-Security's own internal language). Both appear in the current r-security prompt.

### [Severity: Medium] Orchestrator Completion Contract Stale Reference

- **What:** The orchestrator's completion contract (orchestrator.agent.md line 387) says: _"Workflow completes only when the final review (R Aggregator) returns DONE."_ The design's Implementation Checklist item 22 for orchestrator says: _"Update Completion Contract: 'R cluster returns DONE' (not 'R Aggregator')."_ This is noted but is safety-relevant: if the orchestrator's completion contract still says "R Aggregator returns DONE" after implementation, and no R Aggregator exists, the orchestrator has a logical impossibility in its termination condition. This could cause the orchestrator to loop indefinitely or error out.
- **Where:** Design section "Implementation Checklist — Tier 2", orchestrator.agent.md line 387
- **Likelihood:** L
- **Impact:** H — Pipeline cannot complete if the termination condition references a non-existent agent.
- **Assumption at risk:** Assumes implementer will correctly apply item 22 of a 25-item orchestrator change list. One miss in a safety-critical location.

### [Severity: Medium] Unresolved Tensions Detection Loss May Mask Security-Performance Tradeoffs

- **What:** The design explicitly accepts losing Unresolved Tensions detection (Tradeoff 2, Assumption A-5). However, the CT-aggregator specifically detects contradictions between sub-agents — for example, when ct-security flags a requirement as critical while ct-scalability flags the same requirement's performance cost. These tensions directly inform design decisions. Without detection, the designer in revision mode reads 4 individual CT files and must independently notice contradictions across them. For security vs. performance tradeoffs specifically, missing a tension could mean the designer resolves the concern in one direction without knowing the other sub-agent raised the opposite concern.
- **Where:** Design sections "Tradeoffs — Tradeoff 2", ct-aggregator.agent.md Step 8
- **Likelihood:** M
- **Impact:** M — Affects design quality rather than pipeline safety. Security constraints could be weakened by performance optimizations the designer sees in ct-scalability without seeing ct-security's opposing view.
- **Assumption at risk:** Assumption A-5 states downstream consumers can "identify contradictions themselves." For LLM agents reading 4 separate files, cross-file contradiction detection is unreliable.

### [Severity: Low] V-Build Gate ERROR Forwarding Path Changes

- **What:** Currently, when V-Build errors (orchestrator.agent.md line 256): "On ERROR: retry once. If persistent → skip parallel, forward to V-Aggregator." The V-Aggregator then records this in verifier.md and triggers replan. In the new model, the orchestrator handles this directly. The design's updated Pattern B (design.md) says orchestrator reads memories and determines cluster result. But if V-Build errors before any parallel agents run, there are NO memory files from V-Tests/V-Tasks/V-Feature. The orchestrator must handle "V-Build ERROR, zero parallel memories available" as a special case. The design's V decision table row says "ERROR | any | any | any → ERROR" which covers this functionally, but the merge protocol expects to read memory files that don't exist.
- **Where:** Design section "V Cluster Decision Flow", orchestrator.agent.md line 256
- **Likelihood:** L
- **Impact:** L — The decision table correctly returns ERROR. The merge simply has nothing to merge, which is handled by the non-blocking memory design.
- **Assumption at risk:** None significant — existing fallback handles this.

### [Severity: Low] Sequential Agent Memory Model — Unnecessary Indirection

- **What:** The design mandates that spec, designer, and planner write to isolated memory files instead of directly to `memory.md`, even though they run sequentially with no concurrency risk. The orchestrator then immediately merges. This adds one file write + one merge step per sequential agent (3 agents × 2 extra operations = 6 operations) for architectural purity. The tradeoff (Tradeoff 4) explicitly accepts this. But it also means the pipeline is slightly more fragile: if memory merge fails for a sequential agent, downstream agents lose context they would have had under the old direct-write model. The non-blocking fallback applies, but it's a net-negative reliability change motivated by uniform modeling rather than functional need.
- **Where:** Design section "Tradeoffs — Tradeoff 4", design.md Sequential Agent Memory Model Migration
- **Likelihood:** L
- **Impact:** L — Non-blocking fallback makes this benign in practice.
- **Assumption at risk:** Assumption A-2 that uniformity is worth the extra indirection.

---

## Cross-Cutting Observations

1. **Orchestrator prompt growth risk:** The orchestrator is currently 399 lines. The design estimates net +10 lines. This seems optimistic. Adding CT decision flow (6 lines), V decision table (10 lines), R decision flow (9 lines), memory merge protocol (5 lines), updated Pattern C (adjustment), memory directory creation, updated NEEDS_REVISION table, updated Expectations table, and pruning checkpoint changes likely exceeds +10 lines net. The design should have a concrete fallback plan (stated as "extract to `dispatch-patterns.md` if needed") with a firm threshold rather than "if the orchestrator exceeds ~450 lines."

2. **Missing rollback story:** The design has no rollback plan. If the new architecture fails in practice (e.g., replan quality degrades severely), there's no documented path to revert. Since these are prompt files, git revert is trivial, but noting this as a conscious decision would strengthen the design.

3. **Memory file cleanup not addressed:** The `memory/` directory accumulates `*.mem.md` files across pipeline runs. No cleanup or archival strategy is defined. Over many features, this could lead to stale memory files from prior runs confusing agents that read `memory/` directory contents.

---

## Requirement Coverage

| Requirement                                       | Coverage Status       | Notes                                                                                                                                                              |
| ------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| FR-1 (Isolated memory files)                      | Fully covered         | Template, naming, workflow step all specified                                                                                                                      |
| FR-2 (Orchestrator memory merge)                  | Fully covered         | Merge sequence, sole-writer rule defined                                                                                                                           |
| FR-3 (Remove aggregators)                         | Fully covered         | Deprecation notices specified                                                                                                                                      |
| FR-4.1 (CT decision logic)                        | Fully covered         | Decision flow specified                                                                                                                                            |
| FR-4.2 (V decision logic)                         | Fully covered         | Decision table replicated from feature spec                                                                                                                        |
| FR-4.3 (R decision logic with security override)  | **Partially covered** | Logic is correct but R-Security's own prompt update is underspecified — Pipeline Blocker Override Rule section needs targeted update beyond generic Tier 4 changes |
| FR-4.5 (CT memory highest severity)               | Fully covered         | Template includes field                                                                                                                                            |
| FR-4.6 (R-Security memory explicit severity)      | **Partially covered** | Template includes field, but r-security prompt update doesn't explicitly instruct the agent on the criticality of populating this field for pipeline safety        |
| FR-5 (Remove analysis.md / synthesis)             | Fully covered         |                                                                                                                                                                    |
| FR-6.2 (Planner reads V files directly in replan) | **Partially covered** | Inputs updated but mode detection mechanism not redefined                                                                                                          |
| FR-7 (Dispatch patterns)                          | Fully covered         | Patterns A/B updated, Pattern C updated                                                                                                                            |
| FR-8 (NEEDS_REVISION routing)                     | Fully covered         | Table updates specified                                                                                                                                            |
| FR-9 (Memory write safety)                        | Fully covered         |                                                                                                                                                                    |
| FR-10-15                                          | Fully covered         |                                                                                                                                                                    |
| NFR-1 (Orchestrator prompt size)                  | Covered with risk     | Net growth estimate of +10 lines may be optimistic                                                                                                                 |
| NFR-5 (Backwards-compatible severity taxonomies)  | **Partially covered** | Blocker vs. Critical terminology overlap in R-Security not resolved                                                                                                |
| AC-9 (R decision logic with security override)    | Covered               | Logic correct, but fidelity depends on r-security prompt update quality                                                                                            |
| EC-11 (Planner replan without aggregated mapping) | **Gap**               | Design acknowledges risk but mitigation is "monitor" — no concrete enhancement to planner instructions                                                             |

---

## Key Assumptions to Validate

| #   | Assumption                                                                      | Risk if Wrong                                                                      | Validation Method                                                                                                                      |
| --- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| A-4 | Deduplication loss is acceptable                                                | Downstream agents see duplicate findings, potentially overweighting a single issue | Verify designer/planner can handle duplicates in CT/V files without confusion                                                          |
| A-5 | Tensions detection loss is acceptable                                           | Security-performance tradeoffs in CT reviews go undetected by designer             | Review whether designer prompt already handles multi-file contradiction detection                                                      |
| A-6 | Planner can self-assemble task-failure mapping                                  | Replan targets wrong tasks, Pattern C iterations wasted                            | **Must validate** — planner.agent.md replan instructions need enhancement to describe how to cross-reference V-Tasks/V-Tests/V-Feature |
| —   | R-Security will use "Blocker" (not "Critical") in Highest Severity memory field | Orchestrator doesn't match on correct string for pipeline override                 | Explicitly constrain R-Security's memory template values in its prompt                                                                 |
| —   | Planner can detect replan mode without `verifier.md`                            | Mode detection fails → wrong planning behavior                                     | Define new mode detection trigger in planner prompt update                                                                             |

---

## Recommendations (Prioritized)

1. **[Must Fix] Explicitly update r-security's Pipeline Blocker Override Rule section** — Rewrite lines 61–67 and line 225 of r-security.agent.md to reference the orchestrator (not R Aggregator) as the enforcement mechanism, and emphasize that the `Highest Severity` field in the memory file is now the vehicle for pipeline override communication. Do not treat r-security as a generic Tier 4 change — it needs a targeted Tier 3 update.

2. **[Must Fix] Define planner replan mode detection trigger** — Replace `verifier.md` exists → replan`with an explicit new trigger. Options: (a) check`verification/v-tasks.md`exists with`## Failing Task IDs` section non-empty; (b) orchestrator passes explicit mode signal (add to planner invocation contract); (c) both for redundancy. Document the chosen approach in the design.

3. **[Should Fix] Enhance planner replan instructions for self-assembly** — Add explicit workflow steps to planner.agent.md for cross-referencing V-Tasks (failing task IDs), V-Tests (test failures), and V-Feature (criteria gaps) in replan mode. Don't defer this as "monitor" — provide minimum viable cross-referencing guidance now.

4. **[Should Fix] Normalize R-Security severity vocabulary** — In r-security's memory template instructions, explicitly constrain `Highest Severity` to use `Blocker/Major/Minor` (the R taxonomy). Add a comment: "Use `Blocker` (not `Critical`) to align with R cluster taxonomy." Update the orchestrator's R decision flow to match only on `Blocker` (not `Blocker or Critical`) to reduce ambiguity.

5. **[Nice to Have] Define orchestrator prompt size threshold and extraction trigger** — Change "if the orchestrator exceeds ~450 lines" to a firm threshold with a defined action (e.g., "If post-implementation orchestrator exceeds 430 lines, extract CT/V/R decision tables to `dispatch-patterns.md` as a blocking follow-up before merging").
