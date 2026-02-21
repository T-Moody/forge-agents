# Critical Review: Scalability & Performance (Iteration 3)

**Revision context:** Re-review after iteration 2 design revision. The revision fixed memory tool disambiguation contradictions by correcting 5 pre-existing references where the orchestrator prompt incorrectly instructed writing pipeline files via the VS Code `memory` tool (Global Rule 12, Step 0.1, Step 8.2, Memory Lifecycle Table, merge step 1.1m). All 8 merge steps reworded from "orchestrator merges" to "orchestrator dispatches a subagent to merge." Phase 1 scope remains: 2 files, tool restriction only, memory.md preserved.

---

## Prior Findings Disposition

### Iteration 1 → 2 (All Resolved — Unchanged)

| # | Prior Finding | Prior Severity | Disposition |
|---|--------------|----------------|-------------|
| 1 | Token cost estimate 2–5× too low | High | **Resolved.** memory.md preserved; Pattern B token arithmetic is irrelevant to Phase 1. Correct measurements documented in design §10.2 for Phase 2 planning. |
| 2 | PostMortem token explosion (5.2×) | High | **Resolved.** PostMortem still reads one aggregated memory.md, not 33+ individual files. |
| 3 | Orchestrator context window pressure | High | **Resolved.** Self-verification logging to memory.md is preserved. No additional context window burden introduced. |
| 4 | Memory file count O(tasks) scaling | High | **Resolved.** memory.md pruning mechanism is preserved. The orchestrator still garbage-collects via prune operations at Steps 1.1, 2, and 4. |
| 5 | Lessons Learned extraction non-functional | Medium | **Resolved.** The broken keyword heuristic is dropped entirely (not implemented). Existing memory.md-based Lessons Learned accumulation continues. Design §13.2 explicitly documents this. |
| 6 | N+1 read pattern replacing batch reads | Medium | **Resolved.** Merge pattern is preserved unchanged. No new per-file read loops introduced. |
| 7 | Lessons Learned prompt bloat | Medium | **Resolved.** No new Lessons Learned injection mechanism. Existing memory.md accumulation is bounded by the preserved prune operations. |
| 8 | R-Knowledge/R-Testing fan-out | Low | **Resolved.** Subagents unchanged. They continue reading memory.md (pruned, bounded) plus their specific upstream memories. |

### Iteration 2 → 3 (Prior Finding Re-evaluated)

| # | Prior Finding | Prior Severity | Disposition |
|---|--------------|----------------|-------------|
| 9 | Phase 2 scalability risks documented but not gated | Low | **Unchanged.** Design §14.4 acknowledges this as "Accepted." Phase 2 remains a separate future feature with its own design cycle. No new information changes this assessment. |

**Summary:** The iteration 2 revision addresses memory tool disambiguation contradictions and merge step wording. These are documentation/prompt correctness fixes with zero scalability impact. All 9 prior findings remain at their established dispositions.

---

## Iteration 3 — Analysis of Design Revision

### Merge Step Wording Correction — Scalability Assessment

The primary change in this revision is that all 8 merge steps (1.1m, 2m, 3m, 3b.2, 4m, between-waves, 6.3, 7.3) are reworded from "orchestrator merges" to "orchestrator dispatches a subagent to merge." The design states this is a wording correction, not a behavioral change.

**Verified against source:** The current orchestrator.agent.md at L237 says "This is an orchestrator merge operation — no subagent invocation." This line is being removed. If the orchestrator was previously attempting direct file writes at step 1.1m (which it cannot do — it has no file-write tools), then the fix either:
- (a) Documents what already happened (LLM dispatched a subagent anyway, ignoring the instruction) — **zero scalability impact**, or
- (b) Enables a merge that was previously silently failing — **minor positive impact** (one additional subagent dispatch, but the merge now actually works)

In either case, the scalability impact is negligible. One additional subagent dispatch at step 1.1m is bounded and constant-cost.

### Memory Tool Disambiguation — Scalability Assessment

The 5 contradiction fixes (§3.5–§3.9) and 3 disambiguation additions (§3.1, §3.3, §3.4) are pure prompt text changes. They do not add data structures, loops, API calls, or resource consumption. No scalability impact.

### Anti-Drift Rationale Update — Scalability Assessment

The dual justification ("all file writes are delegated to subagents; search/discovery tools are unnecessary because the orchestrator reads known deterministic paths only") is documentation. No scalability impact.

---

## Findings

### [Severity: Low] Phase 2 Scalability Risks Are Documented But Not Gated

- **What:** The design correctly records CT-measured token costs in §10.2 (avg 488 tokens per `.mem.md`, PostMortem 5.2× increase) and lists Phase 2 prerequisites in §15.2. However, these are informational notes, not binding constraints. If Phase 2 proceeds without re-engaging scalability review, the same High-severity findings will recur. The design does not establish a gate (e.g., "Phase 2 MUST address ct-scalability findings before implementation"). Design §14.4 accepts this as-is, which is reasonable since Phase 2 is a separate feature.
- **Where:** design.md §10.2, §15.2, §15.3
- **Likelihood:** Low — this is a future process risk, not a Phase 1 technical risk
- **Impact:** Low — Phase 2 is explicitly a separate future feature with its own design cycle that would include its own CT review
- **Assumption at risk:** "Phase 2 will incorporate CT findings" is stated as intent (§15.3) but not enforced by any mechanism

---

## Cross-Cutting Observations

- **Strategy / Maintainability scope:** The feature.md still describes the full Pattern B scope (FR-2 through FR-5: memory elimination, 20 subagent updates, Lessons Learned mechanism, dispatch prompt rewrites) while design.md only covers Phase 1 (FR-1, FR-6). Design §16.5 now provides guidance for implementer/verifier to use the Phase 1 AC subset — an improvement over iteration 1, but the spec-design mismatch persists. Not a scalability concern — flagging for ct-strategy/ct-maintainability.

---

## Requirement Coverage

| Requirement | Coverage Status | Notes |
| ----------- | --------------- | ----- |
| NFR-3.1 (Token Efficiency) | **N/A for Phase 1** | Phase 1 has zero token cost impact — prompt text changes only. The claim about merge elimination savings applies to deferred Phase 2. |
| FR-1.1–FR-1.6 (Tool Restriction) | **Adequately addressed** | Design §2–§3 covers all edit points. Verified: removed tools appear at L33, L83, L92, L541 in current orchestrator — matching design's edit targets. |
| FR-6.1–FR-6.2 (Memory Disambiguation) | **Adequately addressed** | Disambiguation added in 3 locations (§3.1, §3.3, §3.4) plus 5 contradiction removals (§3.5–§3.9). Net-positive over pre-existing state. |
| FR-2.* (Eliminate memory.md) | **Deferred to Phase 2** | memory.md preserved. All merge, prune, invalidation, self-verification mechanisms continue unchanged. |
| FR-3.3 (Lessons Learned) | **Deferred — existing mechanism preserved** | memory.md-based accumulation continues. Broken keyword heuristic correctly dropped. |
| FR-2.4 (Self-verification) | **Deferred — memory.md logging preserved** | Self-verification writes to memory.md as before. No context window migration. |
| AC-1, AC-2, AC-3, AC-11, AC-12 | **Adequately addressed** | Phase 1 acceptance criteria fully covered by design §2–§3.9. AC-11 coverage strengthened in iteration 2 (contradiction removal). |
| AC-4 through AC-10, AC-13–AC-15 | **Deferred to Phase 2** | These criteria relate to memory.md elimination and subagent updates. |
