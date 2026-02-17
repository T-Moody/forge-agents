# Design Critical Review: Architecture Upgrade Remaining Concerns

**Verdict:** The design is thorough and well-structured but contains one spec-violating decision and several inaccuracies that must be resolved before planning.

**Overall Risk Level:** Medium — The design's technical approach is sound, but a file location mismatch with the feature spec will cause acceptance criteria failures, and the documentation-writer safety narrative is misleading.

---

## Fundamental Concerns

### FC-1: Dispatch Patterns File Location Contradicts Feature Spec

**What:** The design creates the extracted patterns file at `NewAgentsAndPrompts/dispatch-patterns.md`, but the feature spec explicitly specifies `docs/patterns.md` in FR-2.1, FR-2.7, AC-2.2, and TS-4. The design acknowledges this deviation with rationale (co-location, naming collision avoidance) but does not request or document a spec amendment.

**Where:** Design "High-Level Architecture" note, "Change 2: Reduce Orchestrator Complexity" section; Feature spec FR-2.1, AC-2.2, TS-4.

**Likelihood:** High — this will happen exactly as described.

**Impact:** High — The implementation will fail AC-2.2 (`docs/patterns.md` exists) and TS-4 (`test -f docs/patterns.md` succeeds). The design's own success criteria (bottom of design.md) internally references `dispatch-patterns.md`, creating an inconsistency between design and spec that the implementer cannot resolve without designer intervention.

**Assumption at risk:** That the designer has authority to override the feature spec's explicit file path. The design's rationale is reasonable (co-location with agents, avoiding collision with per-feature `research/patterns.md`), but the spec must be formally updated to match. As written, an implementer following the design will produce artifacts that fail the spec's acceptance criteria.

**Recommendation:** Either (a) update the feature spec's FR-2.1, FR-2.7, AC-2.2, AC-2.3, and TS-4 to reference `NewAgentsAndPrompts/dispatch-patterns.md`, or (b) change the design to comply with `docs/patterns.md`. The designer's rationale for co-location is compelling — option (a) is preferred, but the spec change must be explicit.

---

## Risks by Category

### Security / Safety

#### R-1: Documentation-Writer "Sequential Within Each Wave — Safe" Claim Is Inaccurate

**What:** The design proposes updating orchestrator Step 5.2 text to: "Sub-agents read memory but do NOT write to it, except `documentation-writer` which updates memory after completing its task (sequential within each wave — safe)." The parenthetical "(sequential within each wave — safe)" is factually incorrect. In Step 5.2, all tasks in a sub-wave are dispatched concurrently (up to 4 in parallel). If two documentation-writer tasks are in the same sub-wave, they run concurrently and could write to `memory.md` simultaneously.

**Where:** Design "Documentation-Writer Contradiction Resolution" section; Orchestrator Step 5.2 (L261–267).

**Likelihood:** Medium — Multiple documentation-writer tasks in the same sub-wave is plausible in documentation-heavy features.

**Impact:** High — Concurrent writes to `memory.md` could corrupt the operational memory for all downstream agents.

**Assumption at risk:** That documentation-writers within a wave execute sequentially. They don't — they're dispatched in parallel sub-waves like implementers. The design's own F-5 failure scenario acknowledges this risk but the proposed resolution text in Step 5.2 does not. The F-5 section says "the orchestrator should serialize their memory writes" but this serialization is not specified in the Step 5.2 instructions.

**Recommendation:** Either (a) explicitly state in Step 5.2 that documentation-writer tasks within the same sub-wave must be serialized (dispatched one at a time), or (b) change the parenthetical to "documentation-writer updates memory after completing its task — ensure at most one documentation-writer runs per sub-wave to avoid concurrent memory writes," or (c) acknowledge the concurrent write risk honestly and rely on the low probability of exact-simultaneous writes. Option (a) is safest.

#### R-2: Validation Warnings for Documentation-Writer Are Permanently Noisy

**What:** The orchestrator's validation logic at Step 5.2 will always log a warning for documentation-writer because it has `memory_access: read-write` and is dispatched in a parallel wave. The design acknowledges this as "expected and acknowledged in the step text." But a warning that fires every single pipeline run, by design, erodes the signal value of the warning system — operators learn to ignore Step 5.2 validation warnings entirely.

**Where:** Design "Orchestrator Validation Logic" section; FR-3.8 dispatch point at Step 5.2.

**Likelihood:** High — Every pipeline run with documentation-writer tasks triggers this.

**Impact:** Low — The immediate risk is noise, not incorrect behavior. But the long-term consequence is that a truly dangerous `read-write` agent slipping into a parallel wave at Step 5.2 would be masked by the expected warning.

**Assumption at risk:** That warnings are meaningful signals. A warning that fires every time is not a warning — it's noise.

**Recommendation:** Add documentation-writer as a "known override" at Step 5.2 (similar to the researcher synthesis override at Step 1.2). The validation logic should not warn for documentation-writer at Step 5.2 specifically, since its `read-write` status is acknowledged and handled. This preserves the warning's signal value for unexpected `read-write` agents.

### Scalability

#### R-3: Memory Growth Lacks Quantitative Bounds

**What:** The design removes the 200-line emergency prune without quantifying worst-case memory growth between checkpoints. Checkpoints fire after Steps 1.2, 2, and 4. Between Step 4 (planning) and the next prune point... there is no next prune point. Memory grows through Steps 5, 6, and 7 with no pruning — potentially across multiple implementation waves, replan loops (up to 3 in Pattern C), and review fix loops.

**Where:** Design "Structural Growth Control (Replacement Mechanism)" section; Feature spec EC-1.

**Likelihood:** Medium — Complex features with multiple replan loops will accumulate substantial memory entries post-Step 4.

**Impact:** Medium — Agents may need multiple `read_file` calls for memory, increasing context window usage and potentially degrading analysis quality if the AI truncates or loses track of earlier memory content.

**Assumption at risk:** That three checkpoint prune points are sufficient to bound growth. Steps 5–7 are the most memory-intensive phases (multiple agents writing in sequence) and have zero pruning between them.

**Recommendation:** Consider adding a fourth checkpoint prune point — either between implementation waves (Step 5, between waves) or after V-cluster completion (Step 6). Alternatively, quantify the worst-case memory size to demonstrate that the current three checkpoints produce acceptable bounds (e.g., "in a worst-case 10-task feature with 3 replan loops, memory would reach approximately N lines").

### Maintainability

#### R-4: dispatch-patterns.md Discoverability Risk

**What:** After extracting Pattern A/B/C definitions to an external file, the orchestrator AI must actively decide to read `dispatch-patterns.md` for detailed pattern logic. The one-line summaries retained in the orchestrator are sufficient for routine Pattern A dispatches but insufficient for Pattern C's replan loop (max 3 iterations, invalidation-before-replan sequence, iteration tracking).

**Where:** Design "Change 2" section; Failure scenario F-2.

**Likelihood:** Medium — The AI often follows explicit pointers, but may skip the file read under context pressure or if it believes the one-line summary is sufficient.

**Impact:** High — Incorrect Pattern C execution could cause infinite replan loops (missing iteration cap), memory corruption (missing invalidation step), or premature pipeline termination. Pattern C is the most complex dispatch pattern and the most dangerous to get wrong.

**Assumption at risk:** That a text pointer ("See `dispatch-patterns.md` for full definitions") is sufficient to ensure the AI reads the file. LLMs under context pressure often satisfice with available information rather than making additional tool calls.

**Recommendation:** Consider retaining Pattern C's pseudocode inline in the orchestrator (it's only ~14 lines). Pattern A and B are simple enough for one-line summaries, but Pattern C (replan loop with iteration tracking, invalidation, planner invocation) contains too much operational logic to summarize in one line. This adds ~14 lines but saves the most complex dispatch pattern from discoverability failure. Alternatively, add an explicit instruction in the orchestrator's Operating Rules: "Read `dispatch-patterns.md` before first cluster dispatch" rather than relying on the "when needed" formulation.

### Backwards Compatibility

#### R-5: YAML Frontmatter Runtime Rejection (Already Flagged)

**What:** The design adds a `memory_access` field to all agent YAML frontmatter. If the VS Code Copilot runtime validates YAML fields strictly (rejects unknown fields), all 22 agents would fail to load.

**Where:** Design "Migration & Backwards Compatibility" section; Feature spec C-2.

**Likelihood:** Low — Standard YAML parsers ignore unknown fields, and VS Code extensions typically follow this convention.

**Impact:** Critical — Complete pipeline failure. All agents become non-functional.

**Assumption at risk:** That the VS Code Copilot agent runtime ignores unknown YAML frontmatter fields.

**Recommendation:** The design correctly flags this and proposes a rollback plan (remove the field from all files). Acceptable as-is, but a pre-deployment validation step (add the field to one agent and test dispatch) would eliminate the risk entirely.

### Edge Cases

#### R-6: Researcher Dual-Mode Override Relies on Implicit Convention

**What:** The researcher is marked `memory_access: read-only` with a dispatch-time override for synthesis mode. The override mechanism is described in prose in the orchestrator workflow but is not formalized in the YAML or any structured format. Future modifications to Step 1.2 could accidentally remove the override awareness.

**Where:** Design "Researcher Dual-Mode Handling" section; orchestrator Step 1.2.

**Likelihood:** Low — The override is documented in the design and the orchestrator workflow.

**Impact:** Medium — If the override is lost, the orchestrator would either (a) warn spuriously about researcher write behavior in synthesis mode or (b) block researcher synthesis mode memory writes. Both are recoverable but disruptive.

**Assumption at risk:** That future orchestrator modifications will preserve awareness of the researcher dual-mode override.

**Recommendation:** Add a YAML comment in `researcher.agent.md` frontmatter: `memory_access: read-only  # Synthesis mode: orchestrator overrides to read-write at Step 1.2` — making the override self-documenting in the agent file, not just in the orchestrator's instructions.

### Performance

#### R-7: Validation Logic Adds Overhead at Every Dispatch Point

**What:** The validation logic requires reading each agent's YAML frontmatter before every parallel dispatch. This means 4 file reads at Step 1.1, 4 at Step 3b.1, N at Step 5.2, 3 at Step 6.2, and 4 at Step 7.2 — potentially 15+ additional file reads per pipeline run.

**Where:** Design "Orchestrator Validation Logic" section.

**Likelihood:** High — This will always occur.

**Impact:** Low — File reads of YAML frontmatter (~5 lines) are trivial. The overhead is negligible compared to the total pipeline cost of dispatching 20+ agent invocations.

**Assumption at risk:** N/A — overhead is negligible. This is a minor observation, not a risk.

---

## Requirement Coverage Gaps

### Gap-1: README Changes Not Designed

The feature spec (FR-5.1, AC-5.1, AC-5.2) requires substantial README updates:

- "Why Forge?" table: "3 concurrent agents" → "4 concurrent agents"
- "Stages at a Glance" table: still shows "researcher ×3 + synthesis" and "3 concurrent"
- Workflow diagram: shows 3 researcher boxes, not 4
- Agent roster: lists 10 agents, doesn't mention 12 cluster sub-agents
- "Project Layout": lists only 10 files, not the 22+ current agent files

The design references "Change 5" in the Files Modified table and implementation checklist but contains no detailed "Change 5" section. An implementer has no design guidance for these README changes and must reverse-engineer the required edits from the feature spec and current codebase state.

**Recommendation:** Add a "Change 5: README Update" section to the design with before/after content for each stale section, or explicitly state that the implementer should follow FR-5.1 from the feature spec directly.

### Gap-2: feature-workflow.prompt.md Not Assessed

The feature spec's C-5 excludes `feature-workflow.prompt.md` from changes. However, this file references "Memory system: Initialize and maintain `memory.md` across all pipeline steps. Prune at checkpoints to keep context manageable" (line 25). After removing the emergency prune, the "Prune at checkpoints" language remains accurate. But the file also doesn't mention the `memory_access` markers or the new dispatch-patterns reference doc. The design does not assess whether `feature-workflow.prompt.md` needs updates — the review instructions specifically asked about this.

**Finding:** `feature-workflow.prompt.md` does not need changes for Change 1 (pruning language is still accurate), Change 3 (prompt doesn't reference memory_access), or Change 4 (prompt doesn't reference r-knowledge inputs). For Change 2, the prompt doesn't reference dispatch patterns by name. **No updates needed** — but the design should explicitly note this assessment rather than silently excluding the file.

### Gap-3: Orchestrator Step 5.2 Line Number Discrepancy

The design references Step 5.2 "Sub-agents read memory but do NOT write to it" at L268. The actual text is at L265 (item 4 of the numbered list within Section 5.2). This is minor but could cause implementer confusion when applying the exact edit.

---

## Key Assumptions

| #   | Assumption                                                                         | Relied On By                   | Validation Status                                                                                                                                    |
| --- | ---------------------------------------------------------------------------------- | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| A-1 | VS Code Copilot runtime ignores unknown YAML fields                                | Change 3 (all 22 agent files)  | **Unvalidated** — must test pre-deployment per design's own recommendation                                                                           |
| A-2 | Artifact Index is maintained with sufficient detail by upstream agents             | Change 4 (r-knowledge scoping) | **Partially validated** — spec/designer/planner agents are instructed to write Artifact Index entries, but index quality is not enforced or measured |
| A-3 | Three checkpoint prune points sufficiently bound memory growth                     | Change 1 (prune removal)       | **Unvalidated** — no quantitative analysis of worst-case memory size provided                                                                        |
| A-4 | One-line pattern summaries + file pointer are sufficient for AI to follow patterns | Change 2 (pattern extraction)  | **Unvalidated** — high risk for Pattern C (most complex dispatch pattern)                                                                            |
| A-5 | Documentation-writer tasks execute sequentially within each wave                   | Change 3 (write safety)        | **Incorrect** — doc-writers are dispatched in parallel sub-waves per Step 5.2 rules                                                                  |
| A-6 | The orchestrator AI will actively read dispatch-patterns.md when needed            | Change 2 (pattern extraction)  | **Unvalidated** — LLMs under context pressure may skip optional file reads                                                                           |

---

## Strategic Considerations

### The Advisory Marker Trap

The `memory_access` markers are explicitly advisory — no technical enforcement exists. This is the right pragmatic choice given platform constraints. However, the design should be cautious about the language used around these markers. Phrases like "machine-checkable" and "validation logic" suggest a level of enforcement that doesn't exist. The "validation" is the orchestrator AI reading YAML and deciding what to do — it's not actually validated in any deterministic way. Future documentation should make this distinction clear to prevent false confidence.

### Pattern Extraction Granularity

The design treats all three patterns equally for extraction, but they differ significantly in complexity:

- **Pattern A** (fully parallel): Simple, easily summarized in one line
- **Pattern B** (sequential gate + parallel): Moderate, one-line summary is adequate for most cases
- **Pattern C** (replan loop): Complex, involves iteration tracking, memory invalidation, planner re-invocation, and conditional exit — hard to compress meaningfully

Extracting all three equally may over-extract Pattern C. The worst failure mode of this design is an orchestrator that mishandles the V cluster replan loop. Consider keeping Pattern C inline and extracting only A and B — this would save ~20 lines instead of ~30, still well within the line budget.

### Long-Term Memory Architecture

Removing the emergency prune is correct for now, but the design doesn't address the longer-term question: what happens when the pipeline is used for very large features with 20+ tasks, 3 replan iterations, and extensive lessons learned? Running a quantitative estimate (even rough) would strengthen the decision and provide a baseline for future monitoring. If the estimate shows memory could reach 400+ lines in extreme cases, that's still acceptable — but the number should be known, not assumed.

---

## Recommendations (Prioritized)

1. **[Must Fix] Reconcile dispatch-patterns.md location with feature spec.** The design proposes `NewAgentsAndPrompts/dispatch-patterns.md` but the spec mandates `docs/patterns.md`. Either update the feature spec or comply with it. The design's rationale for the new location is sound — update the spec.

2. **[Must Fix] Correct the documentation-writer safety narrative.** The "sequential within each wave — safe" claim is factually wrong for parallel sub-waves. Either serialize documentation-writer memory writes explicitly, or add documentation-writer as a known override at Step 5.2 validation (suppressing the noise), and acknowledge in the text that concurrent writes are possible but mitigated by low probability.

3. **[Should Fix] Add README change details.** Add a "Change 5" section to the design, or explicitly state that implementers should follow FR-5.1 from the feature spec. The README needs 4+ distinct edits across 4 sections — this shouldn't be left implicit.

4. **[Should Fix] Consider keeping Pattern C inline.** Retain the replan loop pseudocode in the orchestrator to prevent discoverability failures for the most complex dispatch pattern. Extract only Patterns A and B.

5. **[Minor] Add override comment in researcher.agent.md frontmatter.** Self-document the synthesis mode override: `memory_access: read-only  # Synthesis mode: orchestrator overrides to read-write at Step 1.2`.

6. **[Minor] Fix line number reference.** Step 5.2 "Sub-agents read memory" text is at L265, not L268 as stated in the design.

7. **[Minor] Assess feature-workflow.prompt.md explicitly.** The design should note that this file was assessed and no changes are needed, rather than silently excluding it.

---

## Verification of Design Claims

| Claim                                                                | Source               | Verified | Result                                                                                                                                                          |
| -------------------------------------------------------------------- | -------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Emergency prune at exactly 3 locations                               | Design, Analysis     | ✅       | Confirmed: L34 (Global Rule 6), L416 (table row), L485 (Anti-Drift Anchor)                                                                                      |
| "~200 lines per call" in 22 agent files is unrelated                 | Design, Feature spec | ✅       | Confirmed: all 22 matches are `read_file` operating rules, not memory prune                                                                                     |
| No other agent references emergency prune                            | Design               | ✅       | Confirmed: only `orchestrator.agent.md` contains "emergency prune"                                                                                              |
| orchestrator.agent.md is 489 lines                                   | Design, Analysis     | ✅       | Confirmed                                                                                                                                                       |
| documentation-writer writes to memory at step 7                      | Design               | ✅       | Confirmed: `documentation-writer.agent.md` L67-69                                                                                                               |
| Orchestrator Step 5.2 says "Sub-agents read memory but do NOT write" | Design               | ✅       | Confirmed at L265 (design says L268 — minor discrepancy)                                                                                                        |
| r-knowledge receives 6 inputs vs 3 for other R agents                | Design               | ✅       | Confirmed: r-knowledge lists `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md` vs `tier, initial-request.md, git diff context` for others |
| critical-thinker.agent.md has malformed frontmatter                  | Design               | ✅       | Confirmed: content precedes opening `---` delimiter                                                                                                             |
| No sub-agent file references Pattern A/B/C by name                   | Design, Analysis     | ✅       | Confirmed via grep                                                                                                                                              |
| v-build is explicitly read-only per workflow                         | Design               | ✅       | Confirmed: v-build.agent.md L115-117 explicitly states "No Memory Write"                                                                                        |
| Dispatch patterns at L82-121                                         | Design               | ✅       | Confirmed: section header at ~L82, patterns defined through ~L121                                                                                               |
| feature-workflow.prompt.md already references 4 research agents      | Self-verified        | ✅       | Confirmed: line 26 says "Research runs 4 parallel sub-agents"                                                                                                   |
