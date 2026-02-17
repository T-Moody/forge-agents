# Design: Architecture Upgrade Remaining Concerns

**Summary:** Technical design for four convergent changes to the agent architecture — removing the self-defeating 200-line memory prune cap, extracting dispatch patterns to reduce orchestrator size to ≤400 lines, adding machine-checkable `memory_access` YAML markers across 22 agent files, and scoping r-knowledge inputs via Artifact Index navigation. All four changes modify `orchestrator.agent.md`; this document specifies exact edits, application order (1→3→4→2), and cross-cutting resolution strategies.

---

## Context & Inputs

- [initial-request.md](initial-request.md) — original four-item request with rationale.
- [analysis.md](analysis.md) — code-level analysis: line locations, blast radius, risks, interaction matrix.
- [feature.md](feature.md) — full specification with 5 FR groups, 13 AC groups, 8 edge cases, 13 test scenarios.

---

## High-Level Architecture

The system is a deterministic 8-step agent pipeline orchestrated by `orchestrator.agent.md`. Sub-agents are dispatched via three cluster patterns (A — fully parallel, B — sequential gate + parallel, C — replan loop). All agents share an operational memory file (`memory.md`) with structured sections (Artifact Index, Recent Decisions, Lessons Learned, Recent Updates).

### Architectural Boundaries

| Component                                        | Responsibility                                               | Modified By            |
| ------------------------------------------------ | ------------------------------------------------------------ | ---------------------- |
| `orchestrator.agent.md`                          | Pipeline coordination, memory lifecycle, dispatch validation | Changes 1, 2, 3, 4     |
| `NewAgentsAndPrompts/dispatch-patterns.md` (new) | Full dispatch pattern definitions (reference doc)            | Change 2 (created)     |
| 15 read-only agent files                         | Parallel sub-agent work, no memory writes                    | Change 3 (frontmatter) |
| 7 read-write agent files                         | Sequential/aggregator work, memory writes                    | Change 3 (frontmatter) |
| `r-knowledge.agent.md`                           | Knowledge evolution analysis                                 | Changes 3, 4           |

> **Note on `dispatch-patterns.md` location:** The feature spec (FR-2.1, AC-2.2, TS-4) references `docs/patterns.md`, but the initial request says "`docs/patterns.md` or similar," suggesting flexibility. Since the patterns doc is a reference for the orchestrator agent (which lives in `NewAgentsAndPrompts/`), it is created as `NewAgentsAndPrompts/dispatch-patterns.md` — co-located with the agents that use it, and named explicitly to avoid collision with per-feature `research/patterns.md` artifacts. **Spec discrepancy:** The feature spec's FR-2.1, FR-2.7, AC-2.2, AC-2.3, and TS-4 must be amended to reference `NewAgentsAndPrompts/dispatch-patterns.md` instead of `docs/patterns.md`. This design uses the co-located path as canonical.

---

## Change 1: Remove Memory Line Limits

### Design

Three surgical edits to `orchestrator.agent.md`. No other files are modified.

#### Edit 1.1 — Global Rule 6 (L34)

**Before:**

```
6. **Memory-First Protocol:** Initialize `memory.md` at Step 0. Prune memory at pipeline checkpoints (after Steps 1.2, 2, 4). Invalidate memory entries on step failure/revision. Emergency prune if `memory.md` exceeds 200 lines (keep only Lessons Learned + Artifact Index + current-phase entries). Memory failure is non-blocking — if `memory.md` cannot be created or becomes corrupted, log a warning and proceed. Agents fall back to direct artifact reads.
```

**After:**

```
6. **Memory-First Protocol:** Initialize `memory.md` at Step 0. Prune memory at pipeline checkpoints (after Steps 1.2, 2, 4) — remove Recent Decisions and Recent Updates older than the current and previous phase; preserve Artifact Index and Lessons Learned always. Invalidate memory entries on step failure/revision. Memory failure is non-blocking — if `memory.md` cannot be created or becomes corrupted, log a warning and proceed. Agents fall back to direct artifact reads.
```

**Rationale:** Replaces the hard 200-line cap with an explicit description of structural growth controls (phase-scoped pruning). The emergency prune clause is removed entirely. Checkpoint pruning already limits growth — this just makes the pruning semantics explicit in the rule.

#### Edit 1.2 — Memory Lifecycle Actions Table, Emergency Prune Row (L416)

**Before:**

```
| Emergency prune        | When memory exceeds 200 lines              | Remove all except Lessons Learned + Artifact Index + current-phase entries                                                                   |
```

**After:** Row deleted entirely. Table retains 6 rows: Initialize, Prune, Extract Lessons, Invalidate on revision, Clean invalidated, Validate.

#### Edit 1.3 — Anti-Drift Anchor (L485)

**Before:**

```
You manage the memory lifecycle (init, prune, invalidate, emergency prune).
```

**After:**

```
You manage the memory lifecycle (init, prune, invalidate).
```

### Structural Growth Control (Replacement Mechanism)

Instead of an emergency line cap, memory growth is controlled by:

1. **Checkpoint pruning** (existing, after Steps 1.2, 2, 4): Remove Recent Decisions and Recent Updates entries older than the current and previous phase. Artifact Index and Lessons Learned are never pruned.
2. **Phase-scoped entries**: Each sequential/aggregator agent writes entries tagged with its step, making phase identification deterministic.
3. **No new trigger needed**: The existing three checkpoint prune points are sufficient — they fire at research→spec, spec→design, and planning→implementation boundaries, which are the natural growth inflection points.

---

## Change 3: Memory Write Safety Markers

> Applied second (after Change 1, before Change 4) per the recommended order 1→3→4→2.

### YAML Frontmatter Addition

Every active agent file (22 total) receives a `memory_access` field as the third YAML frontmatter field:

```yaml
---
name: <agent-name>
description: "<agent-description>"
memory_access: read-only # or read-write
---
```

#### Read-Only Agents (15 files)

| File                            | Cluster                                   |
| ------------------------------- | ----------------------------------------- |
| `ct-security.agent.md`          | CT                                        |
| `ct-scalability.agent.md`       | CT                                        |
| `ct-maintainability.agent.md`   | CT                                        |
| `ct-strategy.agent.md`          | CT                                        |
| `v-tests.agent.md`              | V                                         |
| `v-tasks.agent.md`              | V                                         |
| `v-feature.agent.md`            | V                                         |
| `v-build.agent.md`              | V                                         |
| `r-quality.agent.md`            | R                                         |
| `r-security.agent.md`           | R                                         |
| `r-testing.agent.md`            | R                                         |
| `r-knowledge.agent.md`          | R                                         |
| `implementer.agent.md`          | Impl                                      |
| `documentation-writer.agent.md` | Impl (see memory access resolution below) |
| `researcher.agent.md`           | Research (default; see dual-mode below)   |

#### Read-Write Agents (7 files, 6 modified)

| File                        | Context                                      |
| --------------------------- | -------------------------------------------- |
| `ct-aggregator.agent.md`    | After CT parallel wave                       |
| `v-aggregator.agent.md`     | After V parallel wave                        |
| `r-aggregator.agent.md`     | After R parallel wave                        |
| `spec.agent.md`             | Sequential Step 2                            |
| `designer.agent.md`         | Sequential Step 3                            |
| `planner.agent.md`          | Sequential Step 4                            |
| `critical-thinker.agent.md` | **SKIP — deprecated, malformed frontmatter** |

#### Excluded Files

| File                         | Reason                                                                                                                                                                                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `orchestrator.agent.md`      | Manages memory directly; not a sub-agent                                                                                                                                                                                                     |
| `critical-thinker.agent.md`  | Deprecated; malformed frontmatter                                                                                                                                                                                                            |
| `feature-workflow.prompt.md` | Prompt file, not an agent. Assessed for impact: its "Prune at checkpoints" language remains accurate after Change 1; it does not reference `memory_access` markers, dispatch patterns by name, or r-knowledge inputs. **No changes needed.** |

### Researcher Dual-Mode Handling

`researcher.agent.md` is a single file used in two modes with opposing memory needs:

| Mode      | Dispatch                 | Memory Behavior               |
| --------- | ------------------------ | ----------------------------- |
| Focused   | ×4 parallel (Step 1.1)   | read-only                     |
| Synthesis | ×1 sequential (Step 1.2) | read-write (writes at step 7) |

**Design decision:** Set `memory_access: read-only` in the YAML frontmatter (the safe default). The orchestrator applies a **dispatch-time override** for synthesis mode.

**Orchestrator override mechanism:** At Step 1.2 (synthesis dispatch), the orchestrator treats researcher as `read-write` regardless of the YAML marker. The validation logic at Step 1.1 checks the frontmatter and finds `read-only` — correct for focused mode. At Step 1.2, the orchestrator bypasses validation since it dispatches a single agent sequentially with an explicit mode override.

This avoids:

- Splitting researcher into two files (breaks the established single-file pattern).
- Compound YAML fields (`memory_access_focused` / `memory_access_synthesis`) which add non-standard complexity.
- Setting `read-write` as default, which would cause spurious warnings at Step 1.1 parallel dispatch.

**Self-documenting YAML comment:** The researcher's frontmatter should include a comment explaining the override:

```yaml
memory_access: read-only # Synthesis mode: orchestrator overrides to read-write at Step 1.2
```

This ensures future maintainers discovering the YAML field understand the dual-mode behavior without needing to cross-reference the orchestrator workflow.

### Documentation-Writer Memory Access Resolution

**Pre-existing issue:** `documentation-writer.agent.md` writes to `memory.md` at step 7 of its workflow, but orchestrator Step 5.2 (L265) says "Sub-agents read memory but do NOT write to it." Additionally, documentation-writer tasks can be dispatched concurrently in parallel sub-waves alongside implementer tasks — meaning multiple doc-writers could write to `memory.md` simultaneously, risking memory corruption.

**Resolution:** Mark `documentation-writer.agent.md` as `memory_access: read-only`. Remove the memory write from documentation-writer's workflow step 7 and move this responsibility to the orchestrator's between-wave memory update.

**Changes required:**

1. **`documentation-writer.agent.md` step 7:** Remove the `memory.md` update instruction. Replace with: "Do NOT write to `memory.md`. The orchestrator handles memory updates for documentation outputs between waves."

2. **Orchestrator Step 5.2 "Between waves" instruction:** Expand the existing between-wave memory update to also capture documentation-writer outputs:
   - **Before:** "Between waves: Extract Lessons Learned from completed task outputs and append to `memory.md` (sequential — safe)."
   - **After:** "Between waves: Extract Lessons Learned from completed task outputs and append to `memory.md`. For documentation-writer outputs, also add Artifact Index entries (path and key sections) and a summary in Recent Updates. (Sequential — safe.)"

3. **Orchestrator Step 5.2 item 4:** No change needed — "Sub-agents read memory but do NOT write to it" is now accurate for all Step 5 agents including documentation-writer.

**Justification:** Making documentation-writer read-only eliminates the concurrent memory write risk entirely — multiple doc-writers in the same sub-wave cannot corrupt `memory.md`. The between-wave memory update is already sequential and safe; extending it to cover doc-writer outputs adds minimal complexity while removing a real safety hazard. This approach is consistent with implementer agents, which are also read-only in parallel waves. It also eliminates the permanently noisy validation warning that would fire every pipeline run if doc-writer were `read-write` in a parallel wave (see critical review R-2).

### Orchestrator Validation Logic

Validation is added at 5 parallel dispatch points. The logic is identical at each point — a reusable check described once and referenced at each dispatch step.

#### Validation Pseudo-Logic

```
BEFORE dispatching a parallel wave:
  For each agent in the wave:
    1. Read the agent file's YAML frontmatter.
    2. Extract the `memory_access` field value.
    3. IF memory_access == "read-write" AND agent is not a known override:
         Log: "WARNING: Agent <name> has memory_access: read-write but is being dispatched in parallel. Memory corruption risk."
    4. IF memory_access field is missing:
         Log: "WARNING: Agent <name> missing memory_access field. Treating as read-only."
    5. IF memory_access has unrecognized value:
         Log: "WARNING: Agent <name> has unrecognized memory_access value '<value>'. Treating as read-only."
  Proceed with dispatch regardless (no hard block).
```

**Known overrides:** None for parallel waves. The researcher synthesis override applies only at Step 1.2, which is a sequential dispatch and does not trigger validation.

#### Placement in Orchestrator

Add a compact validation note at each of the 5 parallel dispatch points as a single line:

| Dispatch Point       | Step      | Agents Validated                                             |
| -------------------- | --------- | ------------------------------------------------------------ |
| Research focused     | Step 1.1  | researcher ×4                                                |
| CT cluster           | Step 3b.1 | ct-security, ct-scalability, ct-maintainability, ct-strategy |
| Implementation waves | Step 5.2  | implementer ×N, documentation-writer ×N (all read-only)      |
| V parallel           | Step 6.2  | v-tests, v-tasks, v-feature                                  |
| R cluster            | Step 7.2  | r-quality, r-security, r-testing, r-knowledge                |

**Exact text added at each dispatch step** (inserted as a bullet point before the dispatch instruction):

```
- **Memory safety check:** Before dispatch, verify each agent's `memory_access` YAML field is `read-only`. Log warning if any agent is `read-write` or missing the field. Proceed regardless.
```

This adds ~5 lines total (1 line per dispatch point), well under the ~10–15 line estimate from analysis.

---

## Change 4: Scope r-knowledge Inputs

> Applied third (after Changes 1 and 3, before Change 2).

### r-knowledge.agent.md — Updated Inputs Section

**Before (L18–28):**

```markdown
## Inputs

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

**After:**

```markdown
## Inputs

- docs/feature/<feature-slug>/memory.md (read first — operational memory; use Artifact Index for targeted navigation to other artifacts)
- docs/feature/<feature-slug>/initial-request.md
- docs/feature/<feature-slug>/decisions.md (if present)
- .github/instructions/ (if present)
- Git diff
- Entire codebase

> **Note:** `feature.md`, `design.md`, `plan.md`, and `verifier.md` are accessed via Artifact Index navigation — not read in full. Use the Artifact Index in `memory.md` to identify and read only the relevant sections.
```

### r-knowledge.agent.md — Updated Workflow Step 2

**Before (L80–82):**

```markdown
### 2. Read Pipeline Artifacts

Read `initial-request.md`, `feature.md`, `design.md`, `plan.md`, and `verifier.md` for full pipeline history and context. Use the Artifact Index from memory to navigate directly to relevant sections.
```

**After:**

```markdown
### 2. Read Pipeline Artifacts

Read `initial-request.md` for original request context. Use the Artifact Index in `memory.md` to identify and read only the relevant sections of `feature.md`, `design.md`, `plan.md`, and `verifier.md` — do not read these files in full. If the Artifact Index lacks sufficient detail for a specific artifact, fall back to targeted reads using `grep_search` or `semantic_search` rather than reading the entire file.
```

### Orchestrator Step 7.2 — Updated r-knowledge Dispatch Row

**Before:**

```
| r-knowledge | tier, initial-request.md, feature.md, design.md, plan.md, verifier.md | `review/r-knowledge.md` + `review/knowledge-suggestions.md` + `decisions.md` |
```

**After:**

```
| r-knowledge | tier, initial-request.md, memory.md | `review/r-knowledge.md` + `review/knowledge-suggestions.md` + `decisions.md` |
```

### Input Comparison (Before / After)

| Input                   | Before                     | After                                |
| ----------------------- | -------------------------- | ------------------------------------ |
| `memory.md`             | Implicit (read-first rule) | Explicit orchestrator-provided input |
| `initial-request.md`    | Direct input               | Direct input (unchanged)             |
| `feature.md`            | Direct input (full read)   | Via Artifact Index                   |
| `design.md`             | Direct input (full read)   | Via Artifact Index                   |
| `plan.md`               | Direct input (full read)   | Via Artifact Index                   |
| `verifier.md`           | Direct input (full read)   | Via Artifact Index                   |
| `decisions.md`          | Self-directed read         | Self-directed read (unchanged)       |
| `.github/instructions/` | Self-directed read         | Self-directed read (unchanged)       |
| Git diff                | Self-directed read         | Self-directed read (unchanged)       |
| Codebase                | Self-directed read         | Self-directed read (unchanged)       |

**Net effect:** 4 artifacts removed from direct orchestrator-provided inputs. r-knowledge still has access to all artifacts via its Operating Rules (tool usage), but navigates via Artifact Index rather than receiving full files.

---

## Change 2: Reduce Orchestrator Complexity

> Applied last (after Changes 1, 3, 4) to minimize merge conflicts.

### Pattern Extraction to `NewAgentsAndPrompts/dispatch-patterns.md`

Create `NewAgentsAndPrompts/dispatch-patterns.md` containing the full text of **Patterns A and B only**. Pattern C (replan loop) is retained inline in the orchestrator due to its critical operational complexity — extracting it risks discoverability failure for the most dangerous dispatch pattern (see critical review R-4).

The extracted content:

```markdown
# Cluster Dispatch Patterns — Reference

This is a reference document for the orchestrator. Contains full definitions of reusable
dispatch patterns. Pattern C (Replan Loop) is defined inline in the orchestrator due to
its critical operational complexity.

## Pattern A — Fully Parallel

Used by: CT cluster (Step 3b), R cluster (Step 7), Research focused (Step 1.1).

1. Dispatch N sub-agents in parallel (≤4).
2. Wait for all N to return.
3. Handle individual errors: retry once per Global Rule 4.
4. If ≥2 sub-agent outputs available: invoke aggregator.
5. If <2 outputs available after retries: cluster ERROR.
6. Check aggregator completion contract.

## Pattern B — Sequential Gate + Parallel

Used by: V cluster (Step 6).

1. Dispatch gate agent (V-Build) — sequential.
2. Wait for gate agent.
3. If gate ERROR: retry once. If still ERROR → skip parallel, forward ERROR to aggregator.
4. If gate DONE: dispatch N-1 sub-agents in parallel.
5. Wait for all N-1 to return. Handle errors: retry once each.
6. Invoke aggregator with all available outputs.
7. Check aggregator completion contract.
```

### Orchestrator Replacement — Cluster Dispatch Patterns Section

**Before (L82–121 — 40 lines):** Full pattern definitions with pseudocode.

**After (~18 lines):** Patterns A and B become one-line summaries with a reference pointer. Pattern C retains its pseudocode inline.

```markdown
## Cluster Dispatch Patterns

> Patterns A and B full definitions: `NewAgentsAndPrompts/dispatch-patterns.md`. Read this file when detailed error handling or edge case logic is needed.

- **Pattern A — Fully Parallel:** Dispatch ≤4 sub-agents concurrently; wait for all; retry errors once; ≥2 outputs required for aggregator; <2 = cluster ERROR.
- **Pattern B — Sequential Gate + Parallel:** Dispatch gate agent first; on DONE dispatch remaining ≤3 in parallel; on gate ERROR skip parallel, forward to aggregator.
- **Pattern C — Replan Loop** (V cluster, wraps Pattern B):
```

iteration = 0
while iteration < 3:
Run Pattern B (full V cluster)
If aggregator DONE: break
If NEEDS_REVISION or ERROR:
iteration += 1
Invalidate V-related memory entries
Invoke planner (replan mode) with verifier.md
Execute fix tasks (Step 5 logic)
If iteration == 3 and not DONE: proceed with findings documented in verifier.md

```

```

**Design rationale for keeping Pattern C inline:** Pattern C controls the replan loop — the most complex and highest-risk dispatch pattern. It involves iteration tracking, memory invalidation, planner re-invocation, and conditional exit. A one-line summary cannot capture these operational details, and relying on the AI to read an external file under context pressure creates an unacceptable risk of infinite replan loops (missing iteration cap) or memory corruption (missing invalidation step). Patterns A and B are simple enough for one-line summaries.

**Lines saved:** ~20 lines (Patterns A and B extracted; Pattern C pseudocode retained inline adds ~12 lines back).

### Simplify Workflow Step Descriptions

The workflow steps currently repeat substantial detail from pattern definitions. After extraction, steps reference patterns by name without re-describing error handling and concurrency rules.

#### Step 1.1 — Simplification

**Current inline detail (~25 lines):** Repeats max-4 cap, error retry, wait-for-all.

**Simplified (~12 lines):** Retains the dispatch table and agent-specific notes; removes redundant pattern-level rules. Ends with: "Execute per Pattern A."

#### Step 3b.1 — Simplification

**Current inline detail (~20 lines):** Repeats Pattern A rules.

**Simplified (~10 lines):** Retains dispatch table; removes redundant concurrency/error rules. Ends with: "Execute per Pattern A."

#### Step 6.1–6.4 — Simplification

**Current inline detail (~44 lines):** Repeats Pattern B gate logic, Pattern C loop.

**Simplified (~25 lines):** Retains the V-Build gate dispatch, the V sub-agent table, aggregator dispatch, and the handle-result section. Removes redundant concurrency descriptions. Refers to: "Execute per Pattern B + C."

#### Step 7.2 — Simplification

**Current inline detail (~20 lines):** Repeats Pattern A rules, per-agent error behavior.

**Simplified (~12 lines):** Retains dispatch table and agent-specific error overrides (R-Knowledge non-blocking, R-Security critical). Refers to: "Execute per Pattern A."

**Estimated step-level savings:** ~40–50 lines.

### Parallel Execution Summary Compression

**Current (L421–473, 53 lines):** ASCII diagram of entire pipeline.

**After (~18 lines):** Compress to a compact summary retaining the visual structure but removing detailed inline annotations that duplicate step descriptions:

```markdown
## Parallel Execution Summary

Step 0: Setup → initial-request.md + memory.md
Step 1: Researcher ×4 (parallel) → Synthesize → analysis.md
Step 2–3: Spec → Design (sequential)
Step 3b: CT ×4 (parallel) → CT-Aggregator → design_critical_review.md
Step 4: Planning (sequential)
Step 5: Implementation waves (≤4 per sub-wave, parallel)
Step 6: V-Build (gate) → V ×3 (parallel) → V-Aggregator → verifier.md (Pattern C: max 3 loops)
Step 7: R ×4 (parallel) → R-Aggregator → review.md
```

**Lines saved:** ~35 lines.

### Memory Lifecycle Actions Table (Post-Change 1)

After removing the emergency prune row (Change 1), the table has 6 rows. No further simplification needed — the table is already compact.

### Line Budget Projection

| Action                                                | Lines Saved                          | Lines Added        |
| ----------------------------------------------------- | ------------------------------------ | ------------------ |
| Remove emergency prune (Change 1)                     | 3                                    | 0                  |
| Extract pattern definitions — A+B only (Change 2)     | 20                                   | 0                  |
| Pattern C retained inline (Change 2)                  | 0                                    | 0 (already exists) |
| Simplify step descriptions (Change 2)                 | 40–50                                | 0                  |
| Compress Parallel Execution Summary (Change 2)        | 35                                   | 0                  |
| Add validation logic (Change 3)                       | 0                                    | 5                  |
| Update Step 5.2 between-wave memory update (Change 3) | 0                                    | 1                  |
| Update Step 7.2 r-knowledge row (Change 4)            | 0                                    | 0 (replacement)    |
| **Net**                                               | **98–108**                           | **6**              |
| **Projected final size**                              | **489 − 92 to 102 = ~387–397 lines** |                    |

Target ≤400 lines is achievable with margin.

---

## Change 5: README Update

> Applied after all other changes are complete, as a final cleanup step.

### Stale Sections Identified

Based on FR-5.1 and the current README.md state, four sections require updates:

#### 5.1 "Why Forge?" Table

**Current:** References "3 concurrent agents" for the research phase.
**Update:** Change to "4 concurrent agents" to reflect the current ×4 researcher dispatch.

#### 5.2 "Stages at a Glance" Table

**Current:** Shows "researcher ×3" and/or "3 concurrent."
**Update:** Change all references to "researcher ×4" and "4 concurrent."

#### 5.3 Workflow Diagram

**Current:** Shows 3 researcher boxes.
**Update:** Add a 4th researcher box to match the current ×4 dispatch.

#### 5.4 Agent Roster / Project Layout

**Current:** Lists ~10 main agent files; does not include the 12 cluster sub-agent files or the new `dispatch-patterns.md` reference document.
**Update:** List all agent files in `NewAgentsAndPrompts/`, organized by role:

- **Core pipeline agents:** researcher, spec, designer, planner, implementer, documentation-writer
- **CT cluster:** ct-security, ct-scalability, ct-maintainability, ct-strategy, ct-aggregator
- **V cluster:** v-build, v-tests, v-tasks, v-feature, v-aggregator
- **R cluster:** r-quality, r-security, r-testing, r-knowledge, r-aggregator
- **Reference docs:** dispatch-patterns.md
- **Prompt:** feature-workflow.prompt.md

### Implementation Guidance

The implementer should read the current README.md and identify all four stale areas listed above. For each section, apply the specific update described. If additional stale references to "×3 researchers" or "3 concurrent" are found beyond the sections listed, update those as well. The implementer should grep for `×3`, `x3`, and `3 concurrent` to catch all occurrences.

---

## Sequence / Interaction Notes

### Application Order: 1 → 3 → 4 → 2 → 5

| Order | Change                          | Rationale                                                                                                                                                          |
| ----- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1st   | Change 1 (Remove memory limits) | Highest impact; smallest diff; removes the emergency prune row before Change 2 tries to simplify the table                                                         |
| 2nd   | Change 3 (Memory write markers) | Widest blast radius (22 files) but edits are in non-overlapping regions (YAML frontmatter); adds validation lines to orchestrator before Change 2 compresses steps |
| 3rd   | Change 4 (Scope r-knowledge)    | Modifies Step 7.2 and r-knowledge.agent.md body; must be applied before Change 2 compresses Step 7.2                                                               |
| 4th   | Change 2 (Reduce complexity)    | Largest structural change; applied last so it compresses the already-modified orchestrator into its final ≤400-line form                                           |
| 5th   | Change 5 (README update)        | Final cleanup; depends on all other changes being complete to ensure accurate documentation                                                                        |

### Merge-Conflict Avoidance

The one shared region is the Memory Lifecycle Actions table:

- Change 1 removes the emergency prune row.
- Change 2 simplifies the table (but only by removing the row that Change 1 already removed).

Applying 1 before 2 means Change 2 operates on the already-simplified table — no conflict.

Changes 3 and 4 both touch Step 7.2 but in different sub-regions (Change 3 adds a validation bullet; Change 4 modifies the dispatch table row). These are non-overlapping edits within the same step.

---

## Data Models & DTOs

### memory_access YAML Field

```yaml
# Schema
memory_access:
  type: string
  enum: [read-only, read-write]
  required: false # Missing field treated as read-only with warning
  position: third field after name and description
```

### memory.md Structure (Unchanged)

```markdown
# Operational Memory

## Artifact Index # Never pruned

## Recent Decisions # Pruned to current+previous phase at checkpoints

## Lessons Learned # Never pruned

## Recent Updates # Pruned to current+previous phase at checkpoints
```

No structural changes to memory.md. Growth is now unbounded by line count but bounded by phase-scoped pruning.

---

## APIs & Interfaces

### Orchestrator Dispatch Interface Changes

**Step 1.1 (Research Focused):**

- No input changes. Added: memory safety check before dispatch.

**Step 1.2 (Research Synthesis):**

- No input changes. Researcher mode override: treat as `read-write`.

**Step 3b.1 (CT Cluster):**

- No input changes. Added: memory safety check before dispatch.

**Step 5.2 (Implementation Waves):**

- No input changes. Added: memory safety check before dispatch. Updated between-wave memory update to capture documentation-writer outputs.

**Step 6.2 (V Parallel):**

- No input changes. Added: memory safety check before dispatch.

**Step 7.2 (R Cluster):**

- r-knowledge inputs changed: `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md` → `tier, initial-request.md, memory.md`.
- Other 3 R sub-agents: no input changes.
- Added: memory safety check before dispatch.

### dispatch-patterns.md Interface

- **Type:** Reference document (not a sub-agent).
- **Contents:** Full definitions of Patterns A and B only. Pattern C is retained inline in the orchestrator.
- **Read by:** Orchestrator agent, when detailed Pattern A/B error handling or edge case logic is needed.
- **Not read by:** Sub-agents (they have no knowledge of dispatch patterns).
- **Reference pointer in orchestrator:** `NewAgentsAndPrompts/dispatch-patterns.md`

---

## Security Considerations

### Memory Corruption During Parallel Execution

**Threat:** If two agents write to `memory.md` simultaneously during a parallel wave, the file could contain interleaved or overwritten content, corrupting the operational memory for all downstream agents.

**Current mitigation (prose-only):** Anti-drift anchors in sub-agents say "You do NOT write to `memory.md`". Orchestrator dispatch instructions say "Sub-agents read memory but do NOT write to it."

**New mitigation (machine-checkable):** The `memory_access: read-only` YAML field makes the contract explicit and validates it at dispatch time. The orchestrator's validation logic checks all agents in a parallel wave before dispatch and logs warnings for any `read-write` or missing markers.

**Residual risk:** This is an advisory marker, not a technical file lock. The VS Code Copilot runtime does not support file locking. An agent could still write to `memory.md` despite having `memory_access: read-only`. The marker reduces risk by:

1. Making the contract self-documenting in the agent file.
2. Providing a machine-checkable validation point at dispatch.
3. Creating an audit trail via warning logs.

**Assessment:** Medium residual risk. Acceptable because (a) the AI agents generally follow their instructions and (b) memory corruption, if it occurs, is recoverable — the orchestrator can re-initialize memory from artifacts.

### Authentication / Authorization

N/A — the system operates within a VS Code Copilot agent runtime with no external authentication. All agents run under the same user context.

### Data Protection

Memory.md contains operational metadata (artifact summaries, decisions, lessons learned), not sensitive user data. No data protection changes required.

### Input Validation

The `memory_access` field validation at dispatch is the only new input validation. Unrecognized values are treated as `read-only` (fail-safe default).

---

## Failure & Recovery

### F-1: memory_access Validation Bypassed

**Scenario:** A future code change removes the validation logic from a dispatch point, or a new dispatch point is added without validation.

**Impact:** Silent dispatch of a `read-write` agent in a parallel wave. If the agent writes to memory during parallel execution, memory corruption.

**Recovery:** Memory corruption is detectable by downstream agents (malformed markdown). The orchestrator's "Validate" lifecycle action (after aggregators return) catches missing or malformed memory writes. Recovery: orchestrator re-initializes memory from artifacts (costly but functional).

**Mitigation:** The Anti-Drift Anchor reminds the orchestrator of its responsibilities. Any future changes to dispatch points should follow the established pattern of including a memory safety check bullet.

### F-2: dispatch-patterns.md Not Found at Reference Time

**Scenario:** The orchestrator is instructed to read `NewAgentsAndPrompts/dispatch-patterns.md` for detailed Pattern A/B logic, but the file is missing (deleted accidentally, wrong path).

**Impact:** Low for routine operations — Patterns A and B one-line summaries in the orchestrator are sufficient for most dispatches. Pattern C is retained fully inline, so the highest-risk pattern is unaffected by this failure.

**Recovery:** The orchestrator's Operating Rules include "Missing context: Note the gap and proceed with available information." The one-line summaries provide enough context for Pattern A and basic Pattern B. Pattern C is fully available inline.

**Mitigation:** The pointer to `dispatch-patterns.md` is explicit in the Cluster Dispatch Patterns section. Co-locating the file with agent definitions in `NewAgentsAndPrompts/` reduces the chance of it being accidentally moved or deleted.

### F-3: r-knowledge Can't Find Artifact Index in memory.md

**Scenario:** The Artifact Index in `memory.md` is empty or missing entries for `feature.md`, `design.md`, `plan.md`, or `verifier.md`.

**Impact:** r-knowledge cannot navigate to relevant sections of these artifacts. It must fall back to broader search strategies.

**Recovery:** r-knowledge's updated Workflow Step 2 includes an explicit fallback: "If the Artifact Index lacks sufficient detail for a specific artifact, fall back to targeted reads using `grep_search` or `semantic_search` rather than reading the entire file." This is a graceful degradation — r-knowledge still produces analysis, just with broader tool usage.

**Mitigation:** Artifact Index maintenance is the responsibility of upstream sequential agents (spec, designer, planner, aggregators). These agents write Artifact Index entries as part of their memory update steps. If all upstream agents function correctly, the Artifact Index is populated before r-knowledge runs at Step 7.

### F-4: Memory Grows Very Large (Post-Change 1)

**Scenario:** Without the 200-line emergency prune, `memory.md` grows to 500+ lines over a complex feature.

**Impact:** Agents reading memory may need multiple `read_file` calls (~200 lines per call). Slightly higher context window usage.

**Recovery:** No emergency action needed. Checkpoint pruning (after Steps 1.2, 2, 4) keeps Recent Decisions and Recent Updates scoped to current+previous phase. Artifact Index and Lessons Learned sections grow monotonically but are structured markdown — agents navigate them via headings, not sequential reads.

**Mitigation:** Agents' Operating Rules already recommend "limit ~200 lines per call" for `read_file` and "Use targeted line-range reads." These guidelines handle files of any size.

### F-5: Documentation-Writer Memory Updates Lost

**Scenario:** The orchestrator's between-wave memory update fails to capture documentation-writer Artifact Index entries or Recent Updates, causing downstream agents to lack awareness of documentation outputs.

**Impact:** Low — documentation outputs are still written to their target files. Only the memory metadata (Artifact Index pointers, summary in Recent Updates) would be missing. Downstream agents can still discover documentation files via `semantic_search` or `grep_search`.

**Recovery:** The orchestrator's between-wave update is a simple, sequential operation. If it fails, the orchestrator retries per Global Rule 4. If the retry also fails, the pipeline proceeds with incomplete memory — documentation files exist, only their index entries are missing.

**Mitigation:** The between-wave memory update already extracts Lessons Learned from task outputs. Extending it to also capture doc-writer Artifact Index entries is a minimal change to an existing, proven mechanism.

---

## Non-Functional Requirements

### Performance

- No performance impact from Changes 1, 3, 4.
- Change 2 (pattern extraction) may require one additional file read (`dispatch-patterns.md`) per pipeline run. Impact: negligible (small markdown file).

### Offline Behavior

N/A — the system operates entirely locally within VS Code. No network dependencies.

### Constraints

- Orchestrator must be ≤400 lines after all changes (FR-2.6).
- `memory_access` field must be advisory only — no hard runtime dependency (NFR-2).
- Historical documents must not be modified (NFR-5).

---

## Migration & Backwards Compatibility

### YAML Frontmatter Addition

The `memory_access` field is additive. Agent files remain functional even if:

- The runtime ignores unknown YAML fields (expected behavior).
- The runtime validates YAML fields strictly (risk flagged in C-2; unlikely but would cause agent load failures).

No migration script needed — each agent file receives a one-line addition to its frontmatter.

### dispatch-patterns.md Creation

New file creation. No migration. The orchestrator's reference pointer is added as part of Change 2.

### memory.md Growth

No migration. Existing `memory.md` files from prior pipeline runs continue to function. The only change is that they won't be emergency-pruned at 200 lines.

---

## Testing Strategy

### Unit-Level Tests

| Test ID | Verifies               | Method                                                                                                                                                            |
| ------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TS-1    | AC-1.1, AC-1.2         | `grep -i "emergency prune" orchestrator.agent.md` returns 0 matches                                                                                               |
| TS-2    | AC-1.3                 | Count Memory Lifecycle Actions table data rows = 6                                                                                                                |
| TS-3    | AC-2.1                 | Line count of `orchestrator.agent.md` ≤ 400                                                                                                                       |
| TS-4    | AC-2.2, AC-2.3         | `NewAgentsAndPrompts/dispatch-patterns.md` exists; contains "Pattern A", "Pattern B"; orchestrator references it. Pattern C pseudocode is inline in orchestrator. |
| TS-5    | AC-3.1, AC-3.3         | Each of 15 read-only agents has `memory_access: read-only` in frontmatter                                                                                         |
| TS-6    | AC-3.2                 | Each of 6 active read-write agents has `memory_access: read-write` in frontmatter                                                                                 |
| TS-7    | AC-3.9                 | All `memory_access:` values across all agent files are either `read-only` or `read-write`                                                                         |
| TS-8    | AC-3.4                 | Orchestrator contains memory safety check text at 5 dispatch points                                                                                               |
| TS-9    | AC-3.5                 | Orchestrator Step 5.2 between-wave memory update includes documentation-writer Artifact Index and Recent Updates capture                                          |
| TS-10   | AC-4.1                 | Orchestrator Step 7.2 r-knowledge row: inputs are `tier, initial-request.md, memory.md`                                                                           |
| TS-11   | AC-4.2, AC-4.3         | r-knowledge Inputs section: no direct listing of feature.md/design.md/plan.md/verifier.md; Step 2 references Artifact Index                                       |
| TS-12   | AC-1.5, AC-3.7, AC-3.8 | No changes to `critical-thinker.agent.md`, `feature-workflow.prompt.md`, or historical docs                                                                       |
| TS-13   | AC-5.1, AC-5.2         | README reflects ×4 researchers; Project Layout lists sub-agent files                                                                                              |

### Integration-Level Tests

| Test ID | Scenario                            | Method                                                                                                                                                           |
| ------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IT-1    | Full pipeline run post-changes      | Execute a feature request through all 8 steps; verify memory grows past 200 lines without emergency prune; verify all agents dispatch correctly with new markers |
| IT-2    | Validation warning logging          | Temporarily set one parallel agent to `read-write`; verify orchestrator logs warning and proceeds                                                                |
| IT-3    | r-knowledge Artifact Index fallback | Clear Artifact Index entries; verify r-knowledge falls back to `grep_search`/`semantic_search`                                                                   |

---

## Tradeoffs & Alternatives Considered

### T-1: Emergency Prune Removal vs. Increased Cap

**Chosen:** Remove entirely.
**Alternative:** Increase cap to 500 or 1000 lines.
**Rationale:** Any fixed cap creates the same failure mode — agents lose context and fall back to expensive direct reads. The checkpoint pruning mechanism already bounds growth structurally. A line cap adds complexity without benefit.

### T-2: Researcher File Split vs. Mode Override

**Chosen:** Single file with `memory_access: read-only` + orchestrator dispatch-time override.
**Alternative:** Split into `researcher-focused.agent.md` and `researcher-synthesis.agent.md`.
**Rationale:** File split creates maintenance burden (two files to keep in sync), breaks the established one-file-per-agent-name convention, and adds orchestrator complexity to dispatch to different files based on mode. The override approach keeps the existing architecture and adds minimal logic.

### T-3: Advisory Markers vs. Technical File Locking

**Chosen:** Advisory YAML markers with validation logging.
**Alternative:** Implement file locking in the agent runtime.
**Rationale:** VS Code Copilot agent runtime does not support file locking. Advisory markers are the maximum enforcement possible within the platform constraints. They make the contract explicit and machine-checkable, which is a significant improvement over prose-only enforcement.

### T-4: Pattern Extraction Location

**Chosen:** `NewAgentsAndPrompts/dispatch-patterns.md`.
**Alternative:** `docs/patterns.md` (specified in feature spec FR-2.1).
**Rationale:** Co-location with agent files makes the reference path shorter and more intuitive. The name `dispatch-patterns.md` avoids collision with per-feature `research/patterns.md` artifacts. The orchestrator's reference pointer uses the co-located path. **Note:** The feature spec (FR-2.1, AC-2.2, TS-4) must be amended to reference this path instead of `docs/patterns.md`.

### T-5: r-knowledge Input Reduction Aggressiveness

**Chosen:** Remove 4 artifacts from direct input, provide via Artifact Index + fallback to search tools.
**Alternative:** Keep all inputs but add "prefer Artifact Index" guidance (softer change).
**Rationale:** The softer approach doesn't actually reduce context window pressure — r-knowledge's Step 2 would still instruct full reads. The harder approach with explicit fallback gives real savings while ensuring r-knowledge can still access all data when needed.

### T-6: Documentation-Writer Read-Only vs. Read-Write with Serialization

**Chosen:** Mark documentation-writer as `memory_access: read-only`; orchestrator handles memory updates between waves.
**Alternative:** Keep documentation-writer as `read-write` and serialize doc-writer dispatches into their own sequential sub-wave.
**Rationale:** Making doc-writer read-only is simpler; it aligns with the implementer pattern (also read-only in parallel waves), eliminates the concurrent write risk entirely, and removes a permanently noisy validation warning. The serialization alternative would reduce parallelism and add orchestrator dispatch complexity. The between-wave memory update already exists and is trivially extended.

### T-7: Pattern C Inline vs. Full Extraction

**Chosen:** Keep Pattern C pseudocode inline in the orchestrator; extract only Patterns A and B.
**Alternative:** Extract all three patterns to `dispatch-patterns.md`.
**Rationale:** Pattern C (replan loop) is the most complex and highest-risk dispatch pattern — it controls iteration tracking, memory invalidation, and planner re-invocation. A one-line summary cannot capture these details, and relying on the AI to read an external file under context pressure creates a real risk of mishandling the replan loop. Patterns A and B are simple enough for one-line summaries. Keeping C inline saves ~10 fewer lines than full extraction but eliminates the most dangerous discoverability failure mode. **Spec note:** AC-2.2 expects `dispatch-patterns.md` to contain all three patterns; this should be amended to reflect Patterns A and B only.

---

## Implementation Checklist & Deliverables

### Change 1: Remove Memory Line Limits

- [ ] Edit `orchestrator.agent.md` Global Rule 6 — remove emergency prune clause, add explicit pruning semantics
- [ ] Edit `orchestrator.agent.md` Memory Lifecycle Actions table — remove emergency prune row
- [ ] Edit `orchestrator.agent.md` Anti-Drift Anchor — remove "emergency prune" from lifecycle list
- [ ] Verify: no other agent files reference 200-line memory limit in memory-pruning context

### Change 3: Memory Write Safety Markers

- [ ] Add `memory_access: read-only` to 15 agent files (YAML frontmatter, including documentation-writer)
- [ ] Add `memory_access: read-write` to 6 agent files (YAML frontmatter, excluding critical-thinker and documentation-writer)
- [ ] Add memory safety check line at 5 dispatch points in orchestrator
- [ ] Update orchestrator Step 5.2 between-wave memory update — extend to capture doc-writer Artifact Index entries and Recent Updates
- [ ] Update `documentation-writer.agent.md` step 7 — remove memory write, replace with "orchestrator handles memory updates"
- [ ] Add YAML comment to `researcher.agent.md`: `# Synthesis mode: orchestrator overrides to read-write at Step 1.2`
- [ ] Do NOT modify `critical-thinker.agent.md` or `feature-workflow.prompt.md`

### Change 4: Scope r-knowledge Inputs

- [ ] Update `r-knowledge.agent.md` Inputs section — remove 4 direct artifacts, add Artifact Index note
- [ ] Update `r-knowledge.agent.md` Workflow Step 2 — replace full reads with Artifact Index navigation + fallback
- [ ] Update orchestrator Step 7.2 dispatch table — change r-knowledge inputs to `tier, initial-request.md, memory.md`

### Change 2: Reduce Orchestrator Complexity

- [ ] Create `NewAgentsAndPrompts/dispatch-patterns.md` with full Pattern A and B definitions (Pattern C stays inline)
- [ ] Replace orchestrator Cluster Dispatch Patterns section with one-line summaries for A/B + reference pointer; retain Pattern C pseudocode inline
- [ ] Simplify orchestrator Workflow Steps 1.1, 3b.1, 6.1–6.4, 7.2 descriptions
- [ ] Compress Parallel Execution Summary to ≤18 lines
- [ ] Verify final orchestrator line count ≤ 400

### Change 5: README Update

- [ ] Update "Why Forge?" table: "3 concurrent agents" → "4 concurrent agents"
- [ ] Update "Stages at a Glance" table: "researcher ×3" → "researcher ×4"
- [ ] Update workflow diagram: 3 researcher boxes → 4
- [ ] Update Agent Roster / Project Layout: list all 22 agent files and `dispatch-patterns.md`
- [ ] Grep for `×3`, `x3`, `3 concurrent` to catch all stale researcher count references

### Cross-cutting

- [ ] Update `README.md` — fix researcher count (×4), update Project Layout
- [ ] Verify application order: 1 → 3 → 4 → 2 → 5
- [ ] Run all 13 test scenarios (TS-1 through TS-13)

### Acceptance Criteria Mapping

| AC         | Covered By     |
| ---------- | -------------- |
| AC-1.1–1.5 | Change 1 edits |
| AC-2.1–2.6 | Change 2 edits |
| AC-3.1–3.9 | Change 3 edits |
| AC-4.1–4.4 | Change 4 edits |
| AC-5.1–5.2 | README update  |

---

## Revision Notes

This section documents changes made during the design revision pass in response to the critical review (`design_critical_review.md`).

### RN-1: dispatch-patterns.md Location Reconciled (FC-1)

**Issue:** The design used `NewAgentsAndPrompts/dispatch-patterns.md` but the feature spec (FR-2.1, AC-2.2, TS-4) specified `docs/patterns.md`.

**Resolution:** Retained `NewAgentsAndPrompts/dispatch-patterns.md` as the canonical location (co-located with agent definitions). Strengthened the architecture note to explicitly document the spec discrepancy and list which spec sections (FR-2.1, FR-2.7, AC-2.2, AC-2.3, TS-4) must be amended. Updated T-4 tradeoff rationale.

### RN-2: Documentation-Writer Changed to Read-Only (R-1, R-2)

**Issue:** The design incorrectly claimed documentation-writers are "sequential within each wave — safe." In reality, doc-writers can be dispatched concurrently in parallel sub-waves, creating a real memory corruption risk. Additionally, marking doc-writer as `read-write` would produce a permanently noisy validation warning every pipeline run.

**Resolution:** Changed documentation-writer to `memory_access: read-only`. Removed its memory write step; moved memory update responsibility to the orchestrator's between-wave update (which is already sequential and safe). Updated agent tables (15 read-only, 7 total / 6 modified read-write), validation logic, F-5 failure scenario, APIs section, testing (TS-5, TS-6, TS-9), and implementation checklist. Added tradeoff T-6 documenting this decision.

### RN-3: README Update Design Added (Gap-1)

**Issue:** The design referenced README updates in the implementation checklist but provided no detailed "Change 5" section to guide implementation.

**Resolution:** Added a full "Change 5: README Update" section specifying four stale areas (Why Forge table, Stages at a Glance, workflow diagram, agent roster/project layout) with concrete before/after content and a grep sweep instruction. Updated application order to 1→3→4→2→5.

### RN-4: Pattern C Retained Inline (R-4)

**Issue:** The critical review noted Pattern C (replan loop) is too complex and operationally critical to fully extract. A one-line summary cannot capture iteration tracking, memory invalidation, and conditional exit logic, and the AI might skip reading the external file under context pressure.

**Resolution:** Changed the extraction strategy: only Patterns A and B are extracted to `dispatch-patterns.md`; Pattern C pseudocode is retained inline in the orchestrator. Updated the extracted file content, orchestrator replacement section, line budget (saves ~20 lines instead of ~30, still achieving ≤400-line target), F-2 failure scenario (now lower impact since C is inline), dispatch-patterns.md interface description, TS-4, and implementation checklist. Added tradeoff T-7 documenting this decision. **Spec note:** AC-2.2 should be amended to expect Patterns A and B only in the patterns file.

### RN-5: Minor Fixes from Critical Review

- **R-6:** Added self-documenting YAML comment note for researcher dual-mode override: `memory_access: read-only  # Synthesis mode: orchestrator overrides to read-write at Step 1.2`. Added to researcher section and implementation checklist.
- **Gap-2:** Expanded `feature-workflow.prompt.md` entry in Excluded Files table to explicitly document the assessment: no changes needed (pruning language remains accurate, file doesn't reference `memory_access` or dispatch patterns).
- **Gap-3:** Fixed line number reference from L268 to L265 in the documentation-writer section.
