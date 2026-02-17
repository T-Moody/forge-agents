# Analysis: Architecture Upgrade Remaining Concerns

**Summary:** Four targeted changes address self-defeating memory pruning, orchestrator bloat, implicit memory-write contracts, and r-knowledge context pressure. All four converge on `orchestrator.agent.md` (489 lines, merge-conflict bottleneck). Change 3 has the widest blast radius (22 agent files). Recommended implementation order: 1 → 3 → 4 → 2.

---

## Executive Summary

The forge-architecture-upgrade introduced a memory system, parallel dispatch clusters, and a 489-line orchestrator. Four remaining concerns were identified post-upgrade:

1. **Remove Memory Line Limits** — the 200-line emergency prune cap is self-defeating; it forces agents back to expensive direct artifact reads.
2. **Reduce Orchestrator Complexity** — the orchestrator exceeds its own 450-line target; dispatch pattern repetition is the largest contributor.
3. **Add Memory Write Safety Markers** — the parallel read-only / sequential write contract is enforced only by prose instruction; a machine-checkable `memory_access` YAML field makes it explicit.
4. **Scope r-knowledge Inputs** — r-knowledge receives 2× the inputs of other R sub-agents (6 artifacts vs. 3), risking context-window pressure.

All four changes modify `orchestrator.agent.md`. Changes 1 and 2 are mutually reinforcing (both reduce orchestrator size). Changes 1 and 3 are complementary (removing the prune cap lets memory grow, making write-safety markers more important). No blocking conflicts exist between any pair, but sequencing matters for merge-conflict avoidance.

---

## Change 1: Remove Memory Line Limits

### Findings

The 200-line emergency prune rule appears in **exactly 3 locations** in `orchestrator.agent.md`:

| Location                              | Line | Content                                                                                                                         |
| ------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------- |
| Global Rule 6                         | L34  | "Emergency prune if `memory.md` exceeds 200 lines (keep only Lessons Learned + Artifact Index + current-phase entries)."        |
| Memory Lifecycle Actions table, row 6 | L416 | Emergency prune row: When memory exceeds 200 lines → Remove all except Lessons Learned + Artifact Index + current-phase entries |
| Anti-Drift Anchor                     | L485 | "You manage the memory lifecycle (init, prune, invalidate, emergency prune)."                                                   |

**No other agent file references the 200-line memory limit.** The "~200 lines per call" references found in all 22 agent files' Operating Rules are `read_file` tool usage guidelines — entirely unrelated to memory pruning.

Regular checkpoint pruning (after Steps 1.2, 2, 4) is preserved — it already implements the structural growth controls the initial request calls for.

### Scope

| File                         | Edit                                                           |
| ---------------------------- | -------------------------------------------------------------- |
| `orchestrator.agent.md` L34  | Remove emergency-prune clause from Global Rule 6               |
| `orchestrator.agent.md` L416 | Remove emergency-prune row from Memory Lifecycle Actions table |
| `orchestrator.agent.md` L485 | Remove "emergency prune" from Anti-Drift Anchor                |

**Files affected: 1.** Lines changed: ~3 edits across 3 regions.

### Risks

- **Memory growth without cap:** `memory.md` can now grow beyond 200 lines. All agents reading memory may encounter larger files. Mitigated by: (a) existing checkpoint pruning keeps entries scoped to current ±1 phase, (b) agents already use Artifact Index for targeted navigation, (c) structured markdown is still cheaper than re-reading multiple full artifacts.
- **False-positive edits:** The "~200 lines per call" `read_file` guideline in 22 agent files must NOT be touched — it is unrelated.
- **Historical docs:** `docs/feature/forge-architecture-upgrade/verifier.md` and `tasks/task-14-orchestrator-rewrite.md` reference the emergency prune rule. These are historical artifacts documenting a completed feature and should NOT be modified.

---

## Change 2: Reduce Orchestrator Complexity

### Findings

**Current orchestrator: 489 lines** (exceeds its own 450-line target).

#### Section Breakdown

| Section                        | Lines   | Line Range  | % of File |
| ------------------------------ | ------- | ----------- | --------- |
| YAML frontmatter               | 5       | 1–5         | 1.0%      |
| Title + intro                  | 11      | 6–16        | 2.2%      |
| Inputs                         | 4       | 17–20       | 0.8%      |
| Outputs                        | 6       | 21–26       | 1.2%      |
| Global Rules                   | 14      | 27–40       | 2.9%      |
| Documentation Structure        | 28      | 41–68       | 5.7%      |
| Operating Rules                | 13      | 69–81       | 2.7%      |
| **Cluster Dispatch Patterns**  | **40**  | **82–121**  | **8.2%**  |
| **Workflow Steps**             | **245** | **122–366** | **50.1%** |
| NEEDS_REVISION Routing Table   | 13      | 368–380     | 2.7%      |
| Expectations Per Agent         | 26      | 381–406     | 5.3%      |
| Memory Lifecycle Actions       | 14      | 407–420     | 2.9%      |
| **Parallel Execution Summary** | **53**  | **421–473** | **10.8%** |
| Completion Contract            | 9       | 474–482     | 1.8%      |
| Anti-Drift Anchor              | 7       | 483–489     | 1.4%      |

**Workflow Steps alone is 245 lines (50.1%)** — the dominant section. Within it, the cluster dispatch steps (1, 3b, 6, 7) consume 161 lines (65.7% of Workflow Steps, 32.9% of the entire file).

#### Dispatch Pattern Repetition

| Pattern                                | Definition (lines)   | Repeated Usage in Steps           | Total          |
| -------------------------------------- | -------------------- | --------------------------------- | -------------- |
| Pattern A (Fully Parallel)             | L86–92 (~6 lines)    | Steps 1, 3b, 7 (~114 lines)       | ~120           |
| Pattern B (Sequential Gate + Parallel) | L95–103 (~7 lines)   | Step 6 (~44 lines)                | ~51            |
| Pattern C (Replan Loop)                | L105–115 (~14 lines) | Step 6 (~44 lines, shared with B) | overlaps B     |
| **Total**                              | **~40 lines**        | **~158 lines**                    | **~198 lines** |

The step-level descriptions repeat substantial detail from the pattern definitions, particularly error handling and concurrency rules.

**No other agent file references Pattern A, B, or C by name.** Sub-agents have no knowledge of which dispatch pattern invoked them. `r-knowledge.agent.md` references "Pattern Captures" — a wholly different concept (knowledge evolution). Extraction to `docs/patterns.md` is safe: no cross-references will break.

#### Line Reduction Estimates

| Action                                                                                | Lines Saved            |
| ------------------------------------------------------------------------------------- | ---------------------- |
| Extract pattern definitions (40 lines) → `docs/patterns.md`                           | ~30 (keeping pointers) |
| Remove emergency prune (Change 1)                                                     | ~3                     |
| Simplify step descriptions to reference extracted patterns                            | ~40–60 (estimated)     |
| Remove/compress Parallel Execution Summary (53 lines, partially redundant with steps) | ~20–30                 |
| **Total potential**                                                                   | **~93–123 lines**      |

From 489 lines, this would bring the orchestrator to **~366–396 lines** — within the ≤400-line target. However, Change 3 adds ~10–15 lines of validation logic, so the net reduction needs to account for that.

### Scope

| File                                      | Edit                                                                                |
| ----------------------------------------- | ----------------------------------------------------------------------------------- |
| `orchestrator.agent.md` L82–121           | Replace pattern definitions with one-line summaries + pointer to `docs/patterns.md` |
| `orchestrator.agent.md` Steps 1, 3b, 6, 7 | Simplify inline pattern re-descriptions to reference extracted definitions          |
| `orchestrator.agent.md` L421–473          | Compress or remove Parallel Execution Summary                                       |
| `orchestrator.agent.md` L407–420          | Simplify Memory Lifecycle Actions table (post-Change 1)                             |
| `docs/patterns.md` (new file)             | Create extracted dispatch pattern definitions                                       |

**Files affected: 2** (1 modified, 1 created).

### Risks

- **Orchestrator loses self-contained context:** If the AI running as orchestrator doesn't read `docs/patterns.md`, it may not recall pattern details. The pointer must be explicit.
- **Competing pressure with Change 3:** Change 3 adds validation logic (~10–15 lines). Net reduction is ~80–110 lines, not ~93–123.
- **≤400-line target feasibility:** Achievable only if step descriptions are also compressed, not just pattern definitions extracted. Pattern extraction alone yields only ~30 lines saved.

---

## Change 3: Memory Write Safety Markers

### Findings

All 22 active agent files use exactly 2 YAML frontmatter fields: `name` and `description`. **No `memory_access` field exists anywhere.**

Memory read/write rules are currently enforced only by prose:

- Anti-drift anchors in sub-agents (e.g., "You do NOT write to `memory.md`")
- Orchestrator dispatch step instructions (e.g., "Sub-agents read memory but do NOT write to it")

#### Complete Categorization

**`memory_access: read-only` — 14 agents (dispatched in parallel):**

| Agent                              | Evidence                                                            | Cluster                            |
| ---------------------------------- | ------------------------------------------------------------------- | ---------------------------------- |
| ct-security.agent.md               | Anti-drift L138: "You do NOT write to `memory.md`"                  | CT                                 |
| ct-scalability.agent.md            | Anti-drift L139: "You do NOT write to `memory.md`"                  | CT                                 |
| ct-maintainability.agent.md        | Anti-drift L138: "You do NOT write to `memory.md`"                  | CT                                 |
| ct-strategy.agent.md               | Anti-drift L149: "You do NOT write to `memory.md`"                  | CT                                 |
| v-tests.agent.md                   | No Memory Write step 6 (L144–146)                                   | V                                  |
| v-tasks.agent.md                   | "MUST NOT write to `memory.md`" (L51)                               | V                                  |
| v-feature.agent.md                 | "MUST NOT write to `memory.md`" (L50)                               | V                                  |
| v-build.agent.md                   | No Memory Write step 5 (L115–117)                                   | V (sequential gate, but read-only) |
| r-quality.agent.md                 | "do NOT write to `memory.md`" (L12)                                 | R                                  |
| r-security.agent.md                | "do NOT write to `memory.md`" (L14)                                 | R                                  |
| r-testing.agent.md                 | No Memory Write step 9 (L149–151)                                   | R                                  |
| r-knowledge.agent.md               | No Memory Write step 11 (L253–255)                                  | R                                  |
| researcher.agent.md (focused mode) | Orchestrator L172: "Sub-agents read memory but do NOT write to it." | Research                           |
| implementer.agent.md               | Orchestrator L268: "Sub-agents read memory but do NOT write to it." | Impl waves                         |

**`memory_access: read-write` — 8 agents (dispatched sequentially):**

| Agent                           | Evidence                                                           | Context                      |
| ------------------------------- | ------------------------------------------------------------------ | ---------------------------- |
| ct-aggregator.agent.md          | "the only CT cluster participant that writes to `memory.md`" (L12) | After CT parallel wave       |
| v-aggregator.agent.md           | "the only V cluster agent that writes to `memory.md`" (L10)        | After V parallel wave        |
| r-aggregator.agent.md           | "write to `memory.md` (sequential agent — safe)" (L157)            | After R parallel wave        |
| researcher.agent.md (synthesis) | "Update `memory.md`" step 7 (L114)                                 | After research parallel wave |
| spec.agent.md                   | "Update `memory.md`" step 7 (L65)                                  | Sequential Step 2            |
| designer.agent.md               | "Update `memory.md`" step 12 (L63)                                 | Sequential Step 3            |
| planner.agent.md                | "Update `memory.md`" step 11 (L77)                                 | Sequential Step 4            |
| documentation-writer.agent.md   | "Update `memory.md`" step 7 (L67)                                  | Sequential Step 5 (per-task) |

**Excluded:**

- `orchestrator.agent.md` — manages memory, not a sub-agent; does not need marker.
- `critical-thinker.agent.md` — deprecated; malformed frontmatter; skip.
- `feature-workflow.prompt.md` — prompt file, not an agent; skip.

### Scope

| File                     | Edit                                                                       |
| ------------------------ | -------------------------------------------------------------------------- |
| 14 read-only agent files | Add `memory_access: read-only` to YAML frontmatter                         |
| 8 read-write agent files | Add `memory_access: read-write` to YAML frontmatter                        |
| `orchestrator.agent.md`  | Add validation logic at 5 dispatch points (Steps 1.1, 3b.1, 5.2, 6.2, 7.2) |

**Files affected: 23** (22 agent files + orchestrator validation logic). This is the widest blast radius of all 4 changes.

### Risks

#### Risk 1: Researcher Dual-Mode Complication (Critical Design Decision)

`researcher.agent.md` operates in **two modes** from a single file:

- **Focused mode**: dispatched ×4 in parallel → must be read-only
- **Synthesis mode**: dispatched ×1 sequentially → must be read-write

A static `memory_access` YAML field cannot represent both. Options:

1. Set `memory_access: read-write` (conservative) and have the orchestrator know focused mode is read-only by override.
2. Set `memory_access: read-only` (default) and have the orchestrator override for synthesis mode.
3. Add compound fields: `memory_access_focused: read-only` + `memory_access_synthesis: read-write`.
4. Split into two files: `researcher-focused.agent.md` and `researcher-synthesis.agent.md`.

#### Risk 2: Documentation-Writer Memory Write Contradiction (Pre-existing)

`documentation-writer.agent.md` writes to `memory.md` at step 7 (L67). But the orchestrator says at Step 5.2 (L268): "Sub-agents read memory but do NOT write to it." This contradiction exists today. Adding `memory_access` markers forces resolution:

- **Option A:** Make documentation-writer truly read-only (remove its memory write step).
- **Option B:** Update orchestrator Step 5.2 to acknowledge documentation-writer as a write exception.

#### Risk 3: v-build Read-Only Despite Sequential Dispatch

v-build is dispatched sequentially (gate agent in Pattern B), but its workflow explicitly says "No Memory Write" (L115–117). It should be `memory_access: read-only` despite being sequential. This is correct behavior but may confuse future maintainers who assume sequential = read-write.

#### Risk 4: YAML Frontmatter Field Validation

If the VS Code Copilot agent runtime strictly validates YAML fields and rejects unknown fields, adding `memory_access` could cause agent load failures. If it ignores unknown fields (more likely), it's safe. This needs verification.

#### Risk 5: Competing Pressure with Change 2

Adding validation logic (~10–15 lines across 5 dispatch points) increases orchestrator size, partially offsetting Change 2's reductions. Net effect is still a reduction, but must be tracked.

---

## Change 4: Scope r-knowledge Inputs

### Findings

R-knowledge has the widest input set of any sub-agent:

| Agent           | Orchestrator-Provided Inputs                                              | Self-Directed Reads                                                |
| --------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| r-quality       | tier, initial-request.md, git diff                                        | —                                                                  |
| r-security      | tier, initial-request.md, git diff                                        | —                                                                  |
| r-testing       | tier, initial-request.md, git diff                                        | —                                                                  |
| **r-knowledge** | **tier, initial-request.md, feature.md, design.md, plan.md, verifier.md** | **decisions.md, .github/instructions/, git diff, entire codebase** |

R-knowledge receives **2× the orchestrator-provided inputs** of other R sub-agents.

Its Inputs section (L18–28) lists 10 sources total. Workflow Step 2 (L80–82) instructs: "Read `initial-request.md`, `feature.md`, `design.md`, `plan.md`, and `verifier.md` for full pipeline history and context." The Artifact Index is mentioned as an optimization hint, not a replacement for full reads.

### Scope

| File                                                       | Edit                                                                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `orchestrator.agent.md` Step 7.2 dispatch table (L332–335) | Change r-knowledge inputs from `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md` to `tier, initial-request.md, memory.md` |
| `r-knowledge.agent.md` Inputs section (L18–28)             | Reduce to `memory.md` + `initial-request.md` as primary; others via Artifact Index                                                              |
| `r-knowledge.agent.md` Workflow Step 2 (L80–82)            | Change from full-file reads to Artifact Index navigation                                                                                        |

**Files affected: 2.**

After the change, r-knowledge's orchestrator-provided inputs (`tier, initial-request.md, memory.md`) become **more uniform** with the other 3 R sub-agents (all get tier + initial-request.md + one other source). This simplifies the dispatch table.

### Risks

- **Artifact Index quality dependency:** If memory.md's Artifact Index doesn't have sufficient detail about artifact contents, r-knowledge may miss relevant sections of `feature.md`, `design.md`, `plan.md`, `verifier.md`. This depends on the quality of Artifact Index maintenance by upstream sequential agents (spec, designer, planner).
- **Behavioral regression:** r-knowledge currently reads 5 artifacts in full at Step 2. Switching to Artifact Index navigation is a behavior change that could miss information not indexed. Risk is moderate — mitigated by the fact that r-knowledge already reads `memory.md` first and uses the Artifact Index as a navigation aid.
- **r-aggregator unaffected:** r-aggregator reads r-knowledge's output (`review/r-knowledge.md`), not its inputs. No cascade.

---

## Cross-cutting Concerns

### 1. Orchestrator as Merge-Conflict Bottleneck

`orchestrator.agent.md` is modified by **all 4 changes**:

| Change   | Edit Regions in orchestrator.agent.md                                                                                       |
| -------- | --------------------------------------------------------------------------------------------------------------------------- |
| Change 1 | Global Rule 6 (L34), Memory Lifecycle table row (L416), Anti-Drift Anchor (L485)                                            |
| Change 2 | Cluster Dispatch Patterns (L82–121), Workflow Steps 1/3b/6/7, Parallel Execution Summary (L421–473), Memory Lifecycle table |
| Change 3 | Dispatch step bodies (Steps 1.1, 3b.1, 5.2, 6.2, 7.2) — adds validation logic                                               |
| Change 4 | Step 7.2 dispatch table row (L332–335)                                                                                      |

Edit regions are **mostly non-overlapping**. The one conflict zone is the Memory Lifecycle Actions table — modified by both Change 1 (remove row) and Change 2 (simplify table). Sequential application (1 before 2) eliminates this conflict.

### 2. Change Interaction Matrix

| Pair  | Interaction                                                                                                | Impact                                                       |
| ----- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| 1 × 2 | **Mutually reinforcing** — both reduce orchestrator size                                                   | Change 1 is prerequisite for Change 2's table simplification |
| 1 × 3 | **Complementary** — removing prune cap lets memory grow, making write-safety markers more important        | No dependency                                                |
| 1 × 4 | **Indirect** — larger memory.md benefits r-knowledge when it relies on memory as primary input             | No dependency                                                |
| 2 × 3 | **Competing line-count pressure** — Change 3 adds ~10–15 lines, Change 2 removes ~93–123                   | Net reduction ~80–110 lines                                  |
| 2 × 4 | **Weak shared region** — both touch Step 7.2; applying 4 before 2 is cleaner                               | No hard dependency                                           |
| 3 × 4 | **No interaction** — both touch r-knowledge.agent.md but in non-overlapping regions (frontmatter vs. body) | Fully independent                                            |

### 3. No Changes Needed in These Files

| File                                        | Reason                                                                                                    |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `feature-workflow.prompt.md`                | Sufficiently abstract; no mention of emergency prune, pattern names, memory_access, or r-knowledge inputs |
| `docs/feature/forge-architecture-upgrade/*` | Historical artifacts; should not be modified                                                              |
| `docs/feature/agent-improvements/*`         | Historical artifacts; not affected                                                                        |
| `docs/optimization-from-gem-team.md`        | Not affected                                                                                              |
| `docs/comparison-forge-vs-gem-team.md`      | Not affected                                                                                              |

### 4. Pre-existing README Staleness

The README has stale data predating these changes:

- "Why Forge?" table says "3 concurrent agents" for research — should be 4
- "Stages at a Glance" table says "researcher ×3" — should be ×4
- Workflow diagram shows 3 researcher boxes — should be 4
- "Project Layout" lists 10 agent files but doesn't list cluster sub-agents

These should be fixed when updating README after implementing the 4 changes.

---

## Recommended Implementation Order

**1 → 3 → 4 → 2**

| Order      | Change                                       | Rationale                                                                                                                                                                                                                                                                               |
| ---------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **First**  | **Change 1: Remove Memory Line Limits**      | Highest impact. Prerequisite for Change 2's table simplification. Only 3 edits in 1 file. Lowest risk.                                                                                                                                                                                  |
| **Second** | **Change 3: Memory Write Safety Markers**    | Widest blast radius (22 files) but each edit is mechanical (add 1 YAML field). Must be applied before Change 2 so the orchestrator's final line count accounts for validation logic additions. Surfaces the researcher dual-mode and documentation-writer contradiction for resolution. |
| **Third**  | **Change 4: Scope r-knowledge Inputs**       | 2 files, moderate effort. Benefits from Change 1 (larger memory available) and is cleaner before Change 2 restructures the orchestrator's step descriptions.                                                                                                                            |
| **Last**   | **Change 2: Reduce Orchestrator Complexity** | Highest effort. Must come last to (a) incorporate Change 1's prune removal, (b) account for Change 3's validation additions, and (c) work with Change 4's modified Step 7.2. Final line-count target (≤400) is assessed against the cumulative state.                                   |

**Post-implementation:** Update `README.md` (fix stale researcher counts + document the 4 changes).

---

## Concrete Metrics

### Current State

| Metric                                           | Value                            |
| ------------------------------------------------ | -------------------------------- |
| Orchestrator total lines                         | 489                              |
| Orchestrator Workflow Steps lines                | 245 (50.1%)                      |
| Orchestrator Cluster Dispatch Patterns lines     | 40 (8.2%)                        |
| Orchestrator Parallel Execution Summary lines    | 53 (10.8%)                       |
| Orchestrator Memory Lifecycle Actions table rows | 7                                |
| Emergency prune references in orchestrator       | 3 locations (L34, L416, L485)    |
| r-knowledge orchestrator-provided inputs         | 6 (vs. 3 for other R sub-agents) |
| r-knowledge total input sources                  | 10                               |
| Agent files with `memory_access` field           | 0 / 22                           |
| Agent files needing `memory_access: read-only`   | 14                               |
| Agent files needing `memory_access: read-write`  | 8                                |

### Per-Change File Counts

| Change                 | Files Modified                | Files Created          | Total Files |
| ---------------------- | ----------------------------- | ---------------------- | ----------- |
| Change 1               | 1                             | 0                      | 1           |
| Change 2               | 1                             | 1 (`docs/patterns.md`) | 2           |
| Change 3               | 23 (22 agents + orchestrator) | 0                      | 23          |
| Change 4               | 2                             | 0                      | 2           |
| README update          | 1                             | 0                      | 1           |
| **Total unique files** | **24**                        | **1**                  | **25**      |

(Orchestrator is counted once; it appears in all 4 changes.)

---

## Open Design Questions

### Must-Resolve Before Implementation

1. **Researcher dual-mode `memory_access`:** How should a single agent file represent both read-only (focused) and read-write (synthesis) modes? Options: (a) default to `read-only` with orchestrator override, (b) compound fields, (c) split into two agent files. This blocks Change 3 for the researcher.

2. **Documentation-writer memory write contradiction:** Should `documentation-writer.agent.md` stop writing to `memory.md` (become truly read-only), or should the orchestrator's Step 5.2 rule be updated to acknowledge it as a read-write exception in implementation waves? This blocks Change 3 for documentation-writer.

3. **`memory_access` validation strictness:** Should the orchestrator hard-error or just log a warning when a `memory_access: read-write` agent is dispatched in a parallel wave? The initial request says "log a warning" — is that sufficient?

### Should-Resolve Before Implementation

4. **YAML frontmatter compatibility:** Does the VS Code Copilot agent runtime accept unknown YAML fields? If it strictly validates, adding `memory_access` could cause load failures. Needs testing.

5. **Orchestrator ≤400-line feasibility:** Pattern extraction alone saves ~30 lines (489→~459). Reaching ≤400 requires also compressing step descriptions and/or the Parallel Execution Summary. How aggressively should step descriptions be compressed?

6. **Parallel Execution Summary (53 lines):** Is the ASCII pipeline diagram intended as a quick-reference for the AI orchestrator, or is it redundant with the Workflow Steps section? Can it be removed or compressed?

### Nice-to-Resolve

7. **Artifact Index quality guarantees:** Change 4 makes r-knowledge dependent on Artifact Index quality for navigating artifacts it previously read in full. Should upstream agents (spec, designer, planner) have a minimum completeness requirement for their Artifact Index entries?

8. **`docs/patterns.md` read cost:** If patterns are extracted, the orchestrator must read `docs/patterns.md`. Is this additional file read acceptable, or should patterns be included as a pointer only (with the expectation that the orchestrator knows them from training)?

9. **README stale data:** Should the pre-existing researcher count errors (3→4) be fixed as part of this feature or tracked separately?

---

## Appendix: Sources

### Partial Research Files Synthesized

| File                                                 | Focus Area   | Confidence | Key Contribution                                                                                                                     |
| ---------------------------------------------------- | ------------ | ---------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| [research/architecture.md](research/architecture.md) | Architecture | High       | Orchestrator line counts, section breakdown, YAML frontmatter inventory, r-knowledge input architecture, dispatch pattern structure  |
| [research/impact.md](research/impact.md)             | Impact       | High       | Per-change file impact lists, cross-cutting interaction analysis, documentation-writer contradiction, researcher dual-mode discovery |
| [research/dependencies.md](research/dependencies.md) | Dependencies | High       | Inter-change dependency graph, file-level collision map, agent cross-references, README dependencies, historical doc assessment      |

### Conflicts Between Partials

- **Emergency prune location count:** Architecture found 3 locations; Impact listed the same 3 locations. No conflict.
- **v-build classification:** Dependencies listed v-build under `memory_access: read-write` in the sequential/aggregator category; Impact correctly identified v-build as `read-only` despite sequential dispatch. **Resolution:** Impact's analysis is correct — v-build explicitly has a "No Memory Write" step and should be `memory_access: read-only`. Dependencies' categorization appears to have been based on dispatch mode rather than actual memory behavior.
- **Implementation order:** Dependencies recommended 1 → 3 → 4 → 2. The initial request's priority order was 1 → 2 → 3 → 4. This analysis adopts the dependency-informed order (1 → 3 → 4 → 2) because Change 2 must come last to account for all other changes' effect on the orchestrator.
