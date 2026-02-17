# Impact Analysis — Architecture Upgrade Remaining Concerns

## Focus Area

**Impact** — Affected files, modules, and components for each of the 4 proposed changes; what needs to change and where; cross-cutting interactions.

## Summary

All 4 changes primarily affect `orchestrator.agent.md` and `r-knowledge.agent.md`. Change 3 (memory_access markers) has the widest blast radius, touching all 22 active agent files. The researcher agent presents a unique dual-mode challenge for memory_access markers. Changes 1 and 2 are mutually reinforcing (both simplify the orchestrator). No blocking conflicts between the 4 changes were found.

---

## 1. Impact of Removing Memory Line Limits

### Files Directly Affected

| File                    | Lines                                 | What Changes                                                                                                                                                                |
| ----------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `orchestrator.agent.md` | L34 (Global Rule 6)                   | Remove "Emergency prune if `memory.md` exceeds 200 lines (keep only Lessons Learned + Artifact Index + current-phase entries)."                                             |
| `orchestrator.agent.md` | L416 (Memory Lifecycle Actions table) | Remove entire "Emergency prune" row: `\| Emergency prune \| When memory exceeds 200 lines \| Remove all except Lessons Learned + Artifact Index + current-phase entries \|` |
| `orchestrator.agent.md` | L485 (Anti-Drift Anchor)              | Remove "emergency prune" from: "You manage the memory lifecycle (init, prune, invalidate, emergency prune)."                                                                |

### Files NOT Affected (False Positives)

All 20+ agent files reference "~200 lines per call" in their Operating Rules (context-efficient reading for `read_file`). This is a **tool usage guideline**, not a memory limit — it is unrelated to the emergency prune cap and must NOT be changed.

- Example (appears in every agent): `"Use targeted line-range reads with read_file (limit ~200 lines per call)."`
- This is confirmed in: `v-tests.agent.md` L26, `v-tasks.agent.md` L34, `v-feature.agent.md` L33, `v-build.agent.md` L25, `v-aggregator.agent.md` L31, `spec.agent.md` L29, `researcher.agent.md` L44/L77, `r-testing.agent.md` L28, `r-security.agent.md` L31, `r-quality.agent.md` L30, `r-knowledge.agent.md` L37, `r-aggregator.agent.md` L37, `planner.agent.md` L32, `orchestrator.agent.md` L71, `implementer.agent.md` L36, `documentation-writer.agent.md` L32, `designer.agent.md` L29, `ct-strategy.agent.md` L29, `ct-security.agent.md`, `ct-scalability.agent.md`, `ct-maintainability.agent.md`

### Other Prune References (KEEP — Not Emergency Prune)

The following prune references are for **regular checkpoint pruning** and must be preserved:

- `orchestrator.agent.md` L180: "Orchestrator: prune memory" (after Step 1.2)
- `orchestrator.agent.md` L192: "Orchestrator: prune memory." (after Step 2)
- `orchestrator.agent.md` L246: "Orchestrator: prune memory." (after Step 4)
- `orchestrator.agent.md` L412: Regular "Prune" row (After Steps 1.2, 2, 4) — KEEP this row
- `feature-workflow.prompt.md` L24: "Prune at checkpoints to keep context manageable" — KEEP

### Indirect Impacts

- With no emergency prune, `memory.md` could grow beyond 200 lines. All agents reading memory may encounter larger files. Since all agents already use the Artifact Index for targeted navigation, this should be manageable.
- The initial request proposes replacing emergency prune with **structural growth controls**: keeping only current and previous phase entries at checkpoints. This is already partially implemented in the regular prune logic (L412).

---

## 2. Impact of Reducing Orchestrator Complexity

### Pattern References Within Orchestrator

The orchestrator (489 lines) contains three inline dispatch patterns starting at L86:

| Pattern                                    | Definition Lines | Usage References (within orchestrator)                                            |
| ------------------------------------------ | ---------------- | --------------------------------------------------------------------------------- |
| **Pattern A — Fully Parallel**             | L86–L92          | L155 (Research step), L159, L201 (CT cluster), L207, L219, L320 (R cluster), L330 |
| **Pattern B — Sequential Gate + Parallel** | L95–L103         | L275 (V cluster), L277                                                            |
| **Pattern C — Replan Loop**                | L105–L115        | L277, L309 (Handle V Result)                                                      |
| All patterns                               | —                | L485 (Anti-Drift Anchor)                                                          |

The patterns definition block (L86–L115) is ~30 lines. Extracting it would save those lines plus allow simplifying inline references.

### External References to Patterns

**No other agent file references Pattern A, B, or C by name.** Specifically:

- `r-knowledge.agent.md` L111/L157/L179 references "Pattern Captures" and "pattern-capture" — this is a **different concept** (knowledge evolution pattern captures, not dispatch patterns). No conflict.
- `feature-workflow.prompt.md` references "Cluster parallelization" generically (L25) but never uses Pattern A/B/C names.
- Sub-agents (CT, V, R clusters) have no knowledge of which dispatch pattern invoked them.

### Extraction Safety Assessment

**Extracting patterns to `docs/patterns.md` will NOT break any agent cross-references.** Only the orchestrator itself references the dispatch patterns by name. The orchestrator would need:

1. A pointer line (e.g., "See `docs/patterns.md` for full definitions.")
2. One-line summaries per pattern (replacing the current ~30-line block)
3. Updated inline references throughout Steps 1, 3b, 6, 7 to use summary language

### Memory Lifecycle Actions Table Impact

The table at L407–L417 has 7 rows. Removing the "Emergency prune" row (from Change 1) simplifies it to 6 rows. No additional simplification is proposed for the table itself beyond that.

### Line Count Estimate

Current: 489 lines. Removing pattern definitions (~30 lines) + emergency prune references (~3 lines) + table row (~1 line) = ~455 lines before any further simplification. Reaching ≤400 lines requires additional compression (e.g., simplifying step descriptions, removing redundant annotations).

---

## 3. Impact of Adding Memory Access Markers

### Current YAML Frontmatter State

All active agent files have YAML frontmatter with `name` and `description` fields. **No file currently has a `memory_access` field** — confirmed by grep returning zero matches for `memory_access` across all agent files.

### Complete Categorization

#### `memory_access: read-only` (14 agents — dispatched in parallel)

| Agent File                           | Evidence                                                                                                | Cluster                                  |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `ct-security.agent.md`               | Anti-drift anchor: "You do NOT write to `memory.md`" (L138)                                             | CT                                       |
| `ct-scalability.agent.md`            | Anti-drift anchor: "You do NOT write to `memory.md`" (L139)                                             | CT                                       |
| `ct-maintainability.agent.md`        | Anti-drift anchor: "You do NOT write to `memory.md`" (L138)                                             | CT                                       |
| `ct-strategy.agent.md`               | Anti-drift anchor: "You do NOT write to `memory.md`" (L149)                                             | CT                                       |
| `v-tests.agent.md`                   | "No Memory Write" step 6 (L144–146)                                                                     | V                                        |
| `v-tasks.agent.md`                   | "MUST NOT write to `memory.md`" (L51); anti-drift (L159)                                                | V                                        |
| `v-feature.agent.md`                 | "MUST NOT write to `memory.md`" (L50); anti-drift (L173)                                                | V                                        |
| `v-build.agent.md`                   | "No Memory Write" step 5 (L115–117)                                                                     | V (sequential gate, but still read-only) |
| `r-quality.agent.md`                 | "do NOT write to `memory.md`" (L12); anti-drift (L174)                                                  | R                                        |
| `r-security.agent.md`                | "do NOT write to `memory.md`" (L14); anti-drift (L228)                                                  | R                                        |
| `r-testing.agent.md`                 | "No Memory Write" step 9 (L149–151)                                                                     | R                                        |
| `r-knowledge.agent.md`               | "No Memory Write" step 11 (L253–255)                                                                    | R                                        |
| `researcher.agent.md` (focused mode) | Orchestrator L172: "Sub-agents read memory but do NOT write to it."                                     | Research                                 |
| `implementer.agent.md`               | Orchestrator L268: "Sub-agents read memory but do NOT write to it." No memory write step in agent file. | Impl waves                               |

#### `memory_access: read-write` (8 agents — dispatched sequentially)

| Agent File                             | Evidence                                                                           | Context                                 |
| -------------------------------------- | ---------------------------------------------------------------------------------- | --------------------------------------- |
| `ct-aggregator.agent.md`               | "the only CT cluster participant that writes to `memory.md`" (L12)                 | Sequential after CT parallel wave       |
| `v-aggregator.agent.md`                | "the only V cluster agent that writes to `memory.md`" (L10); writes at step (L138) | Sequential after V parallel wave        |
| `r-aggregator.agent.md`                | "write to `memory.md` (sequential agent — safe)" (L157); anti-drift (L266)         | Sequential after R parallel wave        |
| `researcher.agent.md` (synthesis mode) | "Update `memory.md`" at synthesis step 7 (L114)                                    | Sequential after research parallel wave |
| `spec.agent.md`                        | "Update `memory.md`" at step 7 (L65)                                               | Sequential (Step 2)                     |
| `designer.agent.md`                    | "Update `memory.md`" at step 12 (L63)                                              | Sequential (Step 3)                     |
| `planner.agent.md`                     | "Update `memory.md`" at step 11 (L77)                                              | Sequential (Step 4)                     |
| `documentation-writer.agent.md`        | "Update `memory.md`" at step 7 (L67)                                               | Sequential (Step 5, per-task)           |

#### Special Cases

| Agent File                   | Status                                                                  |
| ---------------------------- | ----------------------------------------------------------------------- |
| `orchestrator.agent.md`      | **Does not need `memory_access`** — it manages memory, not a sub-agent. |
| `critical-thinker.agent.md`  | **Deprecated** — skip.                                                  |
| `feature-workflow.prompt.md` | **Prompt file, not an agent** — skip.                                   |

### Dual-Mode Complication: Researcher Agent

The `researcher.agent.md` operates in TWO modes:

- **Focused mode**: dispatched in parallel (×4), read-only
- **Synthesis mode**: dispatched sequentially, read-write

A static `memory_access` field in YAML frontmatter cannot represent both. Options:

1. Set `memory_access: read-write` (conservative — synthesis mode needs it) and have the orchestrator know that focused mode is read-only regardless.
2. Create a compound value like `memory_access: mode-dependent` and add a lookup table.
3. Split into two separate agent files (`researcher-focused.agent.md` and `researcher-synthesis.agent.md`).

This is a design decision, not a research finding, but the complication is real and must be addressed.

### Orchestrator Validation Logic Impact

The orchestrator would need new validation logic at each parallel dispatch point:

- Step 1.1 (Research parallel): Validate 4× researcher focused = all read-only
- Step 3b (CT cluster parallel): Validate 4× CT sub-agents = all read-only
- Step 5.2 (Implementation waves): Validate implementer/documentation-writer = all read-only in parallel wave
- Step 6.2 (V parallel): Validate 3× V sub-agents = all read-only
- Step 7.2 (R parallel): Validate 4× R sub-agents = all read-only

This adds logic to the orchestrator, potentially conflicting with Change 2 (reduce orchestrator complexity). However, a single validation function referenced at each dispatch is ~5 lines per dispatch point.

### Documentation-Writer Edge Case

`documentation-writer.agent.md` writes to `memory.md` (step 7, L67) and can run in parallel implementation waves (Step 5.2). The orchestrator says at Step 5.2 L268: "Sub-agents read memory but do NOT write to it." This is a **pre-existing contradiction** — the documentation-writer's own definition says it writes to memory, but the orchestrator says implementation wave sub-agents don't write. Adding `memory_access` markers would surface this contradiction and force a resolution.

---

## 4. Impact of Scoping r-knowledge Inputs

### Current State

**Orchestrator dispatch table (Step 7.2, L332–L335):**

```
| r-knowledge | tier, initial-request.md, feature.md, design.md, plan.md, verifier.md | review/r-knowledge.md + review/knowledge-suggestions.md + decisions.md |
```

**r-knowledge.agent.md Inputs section (L18–L28):**

```
- docs/feature/<feature-slug>/memory.md (read first — operational memory)
- docs/feature/<feature-slug>/initial-request.md
- docs/feature/<feature-slug>/feature.md
- docs/feature/<feature-slug>/design.md
- docs/feature/<feature-slug>/plan.md
- docs/feature/<feature-slug>/verifier.md
- docs/feature/<feature-slug>/decisions.md (if present)
- .github/instructions/ (if present)
- Git diff
- Entire codebase
```

**r-knowledge.agent.md Step 2 (L80–L82):**

```
Read initial-request.md, feature.md, design.md, plan.md, and verifier.md for full pipeline history and context.
```

### Proposed Changes

| Location                    | Current                                                                    | Proposed                                                                                     |
| --------------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Orchestrator Step 7.2 table | `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md`    | `tier, initial-request.md, memory.md`                                                        |
| r-knowledge Inputs section  | 6 artifact files listed                                                    | `memory.md` + `initial-request.md` as primary; others via Artifact Index                     |
| r-knowledge Step 2          | "Read initial-request.md, feature.md, design.md, plan.md, and verifier.md" | "Use memory.md Artifact Index to navigate to relevant sections of other artifacts as needed" |

### Impact on R Cluster Dispatch Logic

The R cluster dispatch (Step 7.2) currently passes **different input sets** per agent:

| Agent       | Current Additional Inputs                                                 |
| ----------- | ------------------------------------------------------------------------- |
| r-quality   | tier, initial-request.md, git diff context                                |
| r-security  | tier, initial-request.md, git diff context                                |
| r-testing   | tier, initial-request.md, git diff context                                |
| r-knowledge | tier, initial-request.md, **feature.md, design.md, plan.md, verifier.md** |

After the change, r-knowledge would take `tier, initial-request.md, memory.md` — making it **more uniform** with the other 3 agents (all get tier + initial-request.md + one other input). This simplifies the dispatch table, not complicates it.

### Effect on r-knowledge Behavior

r-knowledge currently reads 5 artifact files in Step 2. After the change:

- It would read `memory.md` first (already does this at Step 1)
- It would use the Artifact Index to identify which sections of `feature.md`, `design.md`, `plan.md`, `verifier.md` are relevant
- It would use targeted `read_file` calls to read only those sections
- Steps 3–11 are unaffected

The risk: if the Artifact Index in `memory.md` doesn't have sufficient detail about artifact contents, r-knowledge might miss relevant sections. This depends on the quality of Artifact Index maintenance by upstream agents.

### Other Files Affected

- **r-aggregator.agent.md**: Not affected. It reads `review/r-knowledge.md` output, not r-knowledge's inputs.
- **feature-workflow.prompt.md**: Not affected. No agent-specific input details.

---

## 5. Cross-Cutting Impacts

### Change 1 × Change 2: Mutually Reinforcing

Both changes simplify the orchestrator. Removing emergency prune (Change 1) eliminates one row from the Memory Lifecycle Actions table and one clause from Global Rule 6, directly supporting the line count reduction goal (Change 2). These should be implemented together for maximum effect.

### Change 1 × Change 3: Memory Growth Increases Marker Importance

Without emergency pruning, `memory.md` can grow beyond 200 lines. Larger memory files mean:

- More data at risk of corruption if a parallel agent accidentally writes
- `memory_access: read-only` markers become **more important**, not less
- These changes are complementary — Change 1 creates risk that Change 3 mitigates

### Change 2 × Change 3: Competing Pressures on Orchestrator Size

Extracting patterns (Change 2) reduces orchestrator size. Adding `memory_access` validation logic (Change 3) adds to it. Net effect depends on implementation:

- Pattern extraction saves ~30 lines
- Validation logic adds ~10–15 lines (5 dispatch points × 2–3 lines each)
- Net reduction: ~15–20 lines
- **Not a conflict**, but worth tracking during implementation

### Change 3 × Change 4: No Interaction

r-knowledge would be `memory_access: read-only` regardless of its input scope. Scoping inputs (Change 4) doesn't change its memory access category. These are independent.

### Change 1 × Change 4: Indirect Interaction

With no emergency pruning (Change 1), `memory.md` may be larger when r-knowledge reads it (Change 4). Since Change 4 makes r-knowledge rely more heavily on `memory.md` as its primary input, a larger memory file could be beneficial (more Artifact Index detail available). However, it also means more context window consumption from memory alone.

### Change 3: Documentation-Writer Contradiction (Pre-existing)

As noted in Section 3, `documentation-writer.agent.md` claims to write to `memory.md` (L67) but the orchestrator says implementation wave sub-agents are memory read-only (L268). This contradiction exists today. Adding `memory_access` markers will force a decision: either documentation-writer becomes truly read-only (removing its memory write step), or the orchestrator's Step 5.2 rule is updated to acknowledge that documentation-writer is an exception.

### Change 3: Researcher Dual-Mode Issue

The researcher agent operates in both parallel read-only mode and sequential read-write mode. A static `memory_access` YAML field cannot cleanly represent both modes. This must be resolved before implementing Change 3 — it's a design decision.

---

## File References

| File                                                                                     | Relevance                                                                                                                |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| [orchestrator.agent.md](../../NewAgentsAndPrompts/orchestrator.agent.md)                 | Primary target for all 4 changes; contains emergency prune rules, pattern definitions, dispatch tables, memory lifecycle |
| [r-knowledge.agent.md](../../NewAgentsAndPrompts/r-knowledge.agent.md)                   | Target for Change 4 (input scoping); affected by Change 3 (memory_access marker)                                         |
| [researcher.agent.md](../../NewAgentsAndPrompts/researcher.agent.md)                     | Dual-mode complication for Change 3                                                                                      |
| [documentation-writer.agent.md](../../NewAgentsAndPrompts/documentation-writer.agent.md) | Pre-existing memory write contradiction surfaced by Change 3                                                             |
| [feature-workflow.prompt.md](../../NewAgentsAndPrompts/feature-workflow.prompt.md)       | References prune/cluster concepts generically; minimal changes needed                                                    |
| All 14 read-only agents                                                                  | Change 3: add `memory_access: read-only` to YAML frontmatter                                                             |
| All 8 read-write agents                                                                  | Change 3: add `memory_access: read-write` to YAML frontmatter                                                            |

---

## Assumptions & Limitations

- **Assumption:** The `memory_access` YAML field is purely declarative — it doesn't enable any runtime file-locking mechanism.
- **Assumption:** The orchestrator validates `memory_access` at dispatch time, not at runtime.
- **Limitation:** Did not analyze whether `docs/patterns.md` would need to be passed as input to any agent. Currently no agent needs to know the dispatch pattern mechanics.
- **Limitation:** Line count estimates for orchestrator reduction are approximate and depend on final formatting choices.

---

## Open Questions

1. **Researcher dual-mode:** How should `memory_access` be handled for an agent with both read-only and read-write modes? Should the agent be split into two files?
2. **Documentation-writer contradiction:** Should `documentation-writer.agent.md` stop writing to memory (remove step 7), or should the orchestrator acknowledge it as an exception in parallel waves?
3. **Artifact Index quality:** If r-knowledge relies on memory.md's Artifact Index for navigation (Change 4), what happens if the Artifact Index entries are stale or incomplete?
4. **Orchestrator line target:** Is ≤400 lines achievable with only pattern extraction + emergency prune removal (~34 lines saved from 489), or does additional compression require restructuring step descriptions?

---

## Research Metadata

- **confidence_level:** high — all agent files were systematically searched and verified
- **coverage_estimate:** 100% of active agent files in `NewAgentsAndPrompts/` examined; all grep searches returned complete results
- **gaps:** None identified. The deprecated `critical-thinker.agent.md` was excluded as it has no active role in the pipeline.
