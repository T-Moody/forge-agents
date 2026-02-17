# Feature Specification: Architecture Upgrade Remaining Concerns

**Short Summary:** Four targeted changes to the agent architecture: remove the self-defeating 200-line memory prune cap, reduce orchestrator file size to ≤400 lines via pattern extraction, add machine-checkable `memory_access` YAML markers to all agent files, and scope r-knowledge inputs to reduce context-window pressure.

---

## Background & Context

- [Initial Request](initial-request.md) — original user request describing all four changes with rationale.
- [Analysis](analysis.md) — detailed code-level analysis of current state, blast radius, risks, and interaction matrix.
- Target directory: `NewAgentsAndPrompts/`

Post-upgrade, the orchestrator is 489 lines (exceeds its 450-line target), the 200-line emergency prune rule undermines the memory system's purpose, memory write safety is enforced only by prose, and r-knowledge receives 2× the inputs of other R sub-agents.

---

## Functional Requirements

### FR-1: Remove 200-Line Emergency Memory Prune Rule

**FR-1.1:** Remove the emergency prune clause from Global Rule 6 in `orchestrator.agent.md` (currently at L34). The clause reads: `Emergency prune if memory.md exceeds 200 lines (keep only Lessons Learned + Artifact Index + current-phase entries).` After removal, Global Rule 6 retains all other memory lifecycle actions (initialize, prune at checkpoints, invalidate on failure).

**FR-1.2:** Remove the "Emergency prune" row from the Memory Lifecycle Actions table in `orchestrator.agent.md` (currently at L416). The row reads: `Emergency prune | When memory exceeds 200 lines | Remove all except Lessons Learned + Artifact Index + current-phase entries`. The remaining 6 rows are preserved unchanged.

**FR-1.3:** Remove the phrase "emergency prune" from the Anti-Drift Anchor in `orchestrator.agent.md` (currently at L485). The text `(init, prune, invalidate, emergency prune)` becomes `(init, prune, invalidate)`.

**FR-1.4:** No changes to any other agent file. The `~200 lines per call` string in all 22 agent files' Operating Rules refers to `read_file` tool usage and is unrelated. Historical documents under `docs/feature/forge-architecture-upgrade/` are not modified.

**FR-1.5:** Existing checkpoint pruning (after Steps 1.2, 2, 4) is preserved as the structural growth control. No new pruning mechanism is introduced — the existing phase-scoped pruning already limits Recent Decisions and Recent Updates to current and previous phase.

### FR-2: Reduce Orchestrator Complexity

**FR-2.1:** Extract the three Cluster Dispatch Pattern definitions (Pattern A, Pattern B, Pattern C — currently L82–121 in `orchestrator.agent.md`) into a new file `docs/patterns.md`.

**FR-2.2:** Replace the full pattern definitions in `orchestrator.agent.md` with a one-line summary per pattern plus an explicit pointer: `See docs/patterns.md for full definitions.`

**FR-2.3:** Simplify inline pattern re-descriptions in Workflow Steps 1, 3b, 6, and 7 to reference the extracted definitions rather than repeating concurrency rules, error handling, and aggregator logic.

**FR-2.4:** Compress or remove the Parallel Execution Summary section (currently L421–473, 53 lines) since it largely duplicates the workflow steps.

**FR-2.5:** Simplify the Memory Lifecycle Actions table after Change 1 removes the emergency prune row.

**FR-2.6:** The final `orchestrator.agent.md` file MUST be ≤400 lines (including any lines added by Change 3's validation logic).

**FR-2.7:** The new `docs/patterns.md` contains the full text of Pattern A, Pattern B, and Pattern C definitions. It is a reference document — no agent reads it automatically; the orchestrator agent is instructed to read it when needed.

### FR-3: Memory Write Safety Markers

**FR-3.1:** Add a `memory_access` field to the YAML frontmatter of every active agent file (22 files total, excluding `orchestrator.agent.md`, `critical-thinker.agent.md` (deprecated), and `feature-workflow.prompt.md` (not an agent)).

**FR-3.2:** The `memory_access` field accepts exactly two values: `read-only` or `read-write`. No other values are permitted.

**FR-3.3:** The following 14 agents receive `memory_access: read-only`:

| #   | File                          | Cluster  | Rationale                                              |
| --- | ----------------------------- | -------- | ------------------------------------------------------ |
| 1   | `ct-security.agent.md`        | CT       | Parallel sub-agent                                     |
| 2   | `ct-scalability.agent.md`     | CT       | Parallel sub-agent                                     |
| 3   | `ct-maintainability.agent.md` | CT       | Parallel sub-agent                                     |
| 4   | `ct-strategy.agent.md`        | CT       | Parallel sub-agent                                     |
| 5   | `v-tests.agent.md`            | V        | Parallel sub-agent                                     |
| 6   | `v-tasks.agent.md`            | V        | Parallel sub-agent                                     |
| 7   | `v-feature.agent.md`          | V        | Parallel sub-agent                                     |
| 8   | `v-build.agent.md`            | V        | Sequential gate, but explicitly read-only per workflow |
| 9   | `r-quality.agent.md`          | R        | Parallel sub-agent                                     |
| 10  | `r-security.agent.md`         | R        | Parallel sub-agent                                     |
| 11  | `r-testing.agent.md`          | R        | Parallel sub-agent                                     |
| 12  | `r-knowledge.agent.md`        | R        | Parallel sub-agent                                     |
| 13  | `implementer.agent.md`        | Impl     | Parallel within waves; no memory write                 |
| 14  | `researcher.agent.md`         | Research | Default marker (see FR-3.6 for dual-mode handling)     |

**FR-3.4:** The following 8 agents receive `memory_access: read-write`:

| #   | File                            | Context                      | Rationale                           |
| --- | ------------------------------- | ---------------------------- | ----------------------------------- |
| 1   | `ct-aggregator.agent.md`        | After CT parallel wave       | Only CT agent that writes to memory |
| 2   | `v-aggregator.agent.md`         | After V parallel wave        | Only V agent that writes to memory  |
| 3   | `r-aggregator.agent.md`         | After R parallel wave        | Only R agent that writes to memory  |
| 4   | `spec.agent.md`                 | Sequential Step 2            | Writes to memory at step 7          |
| 5   | `designer.agent.md`             | Sequential Step 3            | Writes to memory at step 12         |
| 6   | `planner.agent.md`              | Sequential Step 4            | Writes to memory at step 11         |
| 7   | `documentation-writer.agent.md` | Sequential Step 5 (per-task) | Writes to memory at step 7          |
| 8   | `critical-thinker.agent.md`     | Deprecated                   | Skip — do not modify                |

**FR-3.5:** YAML frontmatter format. The `memory_access` field is added as the third field after `name` and `description`:

```yaml
---
name: <agent-name>
description: <agent-description>
memory_access: read-only # or read-write
---
```

**FR-3.6: Researcher Dual-Mode Handling.** `researcher.agent.md` is a single file used in two modes:

- **Focused mode:** dispatched ×4 in parallel → requires read-only behavior.
- **Synthesis mode:** dispatched ×1 sequentially → requires read-write behavior (writes to memory at step 7).

Resolution: Set `memory_access: read-only` in the YAML frontmatter (the default/safe value). The orchestrator applies a **mode override** at dispatch time: when dispatching the researcher in synthesis mode (Step 1.2), the orchestrator treats it as read-write regardless of the YAML marker. The orchestrator's validation logic (FR-3.8) skips the `memory_access` check for researcher when dispatched in synthesis mode.

**FR-3.7: Documentation-Writer Contradiction Resolution.** The orchestrator (Step 5.2, L268) currently states "Sub-agents read memory but do NOT write to it." However, `documentation-writer.agent.md` writes to memory at step 7. Resolution: Mark `documentation-writer.agent.md` as `memory_access: read-write` and update the orchestrator's Step 5.2 text to acknowledge documentation-writer as a write exception. Specifically, change "Sub-agents read memory but do NOT write to it" to: "Sub-agents read memory but do NOT write to it, except `documentation-writer` which updates memory after completing its task (sequential within each wave — safe)."

**FR-3.8: Orchestrator Validation Logic.** Add validation at each parallel dispatch point in the orchestrator. Before dispatching agents in a parallel wave, the orchestrator MUST:

1. Read the `memory_access` field from each agent's YAML frontmatter.
2. If ALL agents in the wave have `memory_access: read-only`: proceed.
3. If ANY agent has `memory_access: read-write` (and is not covered by a known override like researcher synthesis): log a warning: `"WARNING: Agent <name> has memory_access: read-write but is dispatched in parallel. Memory corruption risk."` Proceed anyway (no hard block).
4. If the `memory_access` field is missing: log a warning: `"WARNING: Agent <name> missing memory_access field."` Treat as read-only for validation purposes.

Validation is added to these orchestrator dispatch points:

- Step 1.1 (research focused parallel dispatch)
- Step 3b.1 (CT cluster parallel dispatch)
- Step 5.2 (implementation wave parallel dispatch)
- Step 6.2 (V cluster parallel dispatch)
- Step 7.2 (R cluster parallel dispatch)

### FR-4: Scope r-knowledge Inputs

**FR-4.1:** Update the orchestrator's Step 7.2 dispatch table to change r-knowledge's inputs from `tier, initial-request.md, feature.md, design.md, plan.md, verifier.md` to `tier, initial-request.md, memory.md`.

**FR-4.2:** Update `r-knowledge.agent.md` Inputs section (currently L18–28) to list `memory.md` and `initial-request.md` as primary inputs. Remove `feature.md`, `design.md`, `plan.md`, and `verifier.md` from the direct inputs list. Add a note: "Use the Artifact Index in `memory.md` to navigate to specific sections of other artifacts as needed."

**FR-4.3:** Update `r-knowledge.agent.md` Workflow Step 2 (currently L80–82) to replace full-file reads with Artifact Index navigation. Change from: "Read `initial-request.md`, `feature.md`, `design.md`, `plan.md`, and `verifier.md` for full pipeline history and context." To: "Read `initial-request.md` for original request context. Use the Artifact Index in `memory.md` to identify and read only the relevant sections of `feature.md`, `design.md`, `plan.md`, and `verifier.md` — do not read these files in full."

**FR-4.4:** `r-knowledge.agent.md` retains access to `decisions.md`, `.github/instructions/`, `git diff`, and entire codebase as self-directed reads. Only the orchestrator-provided artifact inputs are reduced.

### FR-5: README Update

**FR-5.1:** Update `README.md` to reflect changes made by this feature. Specific stale data to correct:

- "Why Forge?" table: "3 concurrent agents" → "4 concurrent agents" for the research phase.
- "Stages at a Glance" table: "researcher ×3" → "researcher ×4".
- Workflow diagram: update from 3 researcher boxes to 4.
- "Project Layout": list all cluster sub-agent files, not just the 10 main agent files.

---

## Non-Functional Requirements

**NFR-1: No Behavioral Regression.** All existing pipeline workflows (8-step pipeline, cluster dispatch, memory lifecycle, completion contracts) continue to function identically except where explicitly changed by this specification.

**NFR-2: Backward Compatibility of YAML Frontmatter.** The `memory_access` field must be additive — agent files remain functional even if the runtime ignores unknown YAML fields. The field is advisory and machine-checkable; it does not create a hard runtime dependency.

**NFR-3: Orchestrator Readability.** After all changes, `orchestrator.agent.md` must be ≤400 lines and remain self-contained enough for the AI running as orchestrator to follow the pipeline without needing to read `docs/patterns.md` for routine dispatches. Pattern pointers must be explicit.

**NFR-4: Memory Growth Tolerance.** With the emergency prune removed, `memory.md` may grow beyond 200 lines. Existing agents' `read_file` operating rules (~200 lines per call) and Artifact Index navigation patterns are sufficient to handle larger memory files. No additional mitigation is required.

**NFR-5: Historical Document Preservation.** No files under `docs/feature/forge-architecture-upgrade/` or `docs/feature/agent-improvements/` are modified. These are historical artifacts.

---

## Constraints & Assumptions

**C-1:** All changes target files in `NewAgentsAndPrompts/` and `docs/`. No source code, tests, or configuration files outside these directories are affected.

**C-2:** The VS Code Copilot agent runtime is assumed to ignore unknown YAML frontmatter fields. If it rejects them, Change 3 would cause agent load failures. This assumption is based on standard YAML behavior.

**C-3:** The Artifact Index in `memory.md` is assumed to be maintained with sufficient detail by upstream sequential agents (spec, designer, planner). Change 4's effectiveness depends on this.

**C-4:** `critical-thinker.agent.md` is deprecated and has malformed frontmatter. It is excluded from Change 3.

**C-5:** `feature-workflow.prompt.md` is a prompt file, not an agent. It is excluded from all changes.

**C-6:** The orchestrator itself (`orchestrator.agent.md`) does not receive a `memory_access` field — it manages memory directly and is not a sub-agent.

---

## Acceptance Criteria

### AC-1: Emergency Prune Removal

- **AC-1.1:** The string `200 lines` does not appear in `orchestrator.agent.md` in any context related to memory pruning. (The string `~200 lines per call` in Operating Rules of other agent files is unrelated and must remain.)
- **AC-1.2:** The string `emergency prune` (case-insensitive) does not appear anywhere in `orchestrator.agent.md`.
- **AC-1.3:** The Memory Lifecycle Actions table in `orchestrator.agent.md` contains exactly 6 rows (Initialize, Prune, Extract Lessons, Invalidate on revision, Clean invalidated, Validate). The "Emergency prune" row is absent.
- **AC-1.4:** Global Rule 6 in `orchestrator.agent.md` still contains: `Initialize memory.md at Step 0`, `Prune memory at pipeline checkpoints`, and `Invalidate memory entries on step failure/revision`.
- **AC-1.5:** No files outside `orchestrator.agent.md` (in `NewAgentsAndPrompts/`) are modified for this change.

### AC-2: Orchestrator Line Count Reduction

- **AC-2.1:** `orchestrator.agent.md` is ≤400 lines (measured by `wc -l` or equivalent).
- **AC-2.2:** `docs/patterns.md` exists and contains the full definitions of Pattern A, Pattern B, and Pattern C.
- **AC-2.3:** `orchestrator.agent.md` contains a reference to `docs/patterns.md` (e.g., the string `docs/patterns.md` appears in the file).
- **AC-2.4:** `orchestrator.agent.md` retains one-line summaries for each of the three dispatch patterns (Pattern A, Pattern B, Pattern C) so the orchestrator agent has enough inline context for routine dispatch.
- **AC-2.5:** The Parallel Execution Summary section is either removed or compressed to ≤20 lines.
- **AC-2.6:** No sub-agent file references Pattern A, B, or C by name (verified pre-change; must remain true post-change).

### AC-3: Memory Write Safety Markers

- **AC-3.1:** All 14 read-only agents (listed in FR-3.3) contain `memory_access: read-only` in their YAML frontmatter.
- **AC-3.2:** All 7 active read-write agents (listed in FR-3.4, excluding `critical-thinker.agent.md`) contain `memory_access: read-write` in their YAML frontmatter.
- **AC-3.3:** `researcher.agent.md` has `memory_access: read-only` in its YAML frontmatter.
- **AC-3.4:** The orchestrator contains validation logic at all 5 parallel dispatch points (Steps 1.1, 3b.1, 5.2, 6.2, 7.2) that checks the `memory_access` field.
- **AC-3.5:** The orchestrator's Step 5.2 text acknowledges `documentation-writer` as a memory-write exception.
- **AC-3.6:** `orchestrator.agent.md` does NOT contain a `memory_access` field in its own YAML frontmatter.
- **AC-3.7:** `critical-thinker.agent.md` is not modified.
- **AC-3.8:** `feature-workflow.prompt.md` is not modified.
- **AC-3.9:** Every `memory_access` value across all agent files is either exactly `read-only` or exactly `read-write` — no other values.

### AC-4: r-knowledge Input Scoping

- **AC-4.1:** The orchestrator's Step 7.2 dispatch table row for r-knowledge lists inputs as `tier, initial-request.md, memory.md` (not `feature.md`, `design.md`, `plan.md`, `verifier.md`).
- **AC-4.2:** `r-knowledge.agent.md` Inputs section lists `memory.md` and `initial-request.md` as primary inputs and does NOT list `feature.md`, `design.md`, `plan.md`, or `verifier.md` as direct inputs.
- **AC-4.3:** `r-knowledge.agent.md` Workflow Step 2 instructs Artifact Index navigation rather than full-file reads of `feature.md`, `design.md`, `plan.md`, `verifier.md`.
- **AC-4.4:** `r-knowledge.agent.md` retains access to `decisions.md`, `.github/instructions/`, git diff, and entire codebase.

### AC-5: README

- **AC-5.1:** `README.md` reflects ×4 researchers (not ×3) in all relevant sections.
- **AC-5.2:** `README.md` Project Layout lists all cluster sub-agent files.

---

## Edge Cases & Error Handling

### EC-1: Memory Grows Very Large (Post-Change 1)

- **Input/Condition:** `memory.md` exceeds 500+ lines because no emergency prune exists.
- **Expected Behavior:** Existing checkpoint pruning (after Steps 1.2, 2, 4) limits growth by removing entries older than 2 completed phases. Agents use Artifact Index for navigation rather than reading memory sequentially. No additional mechanism required.
- **Severity if Missed:** Medium — agents may experience slower context loading but the pipeline still functions.

### EC-2: Agent YAML Frontmatter Missing `memory_access` Field (Post-Change 3)

- **Input/Condition:** A new agent file is created without a `memory_access` field, or the field is accidentally deleted.
- **Expected Behavior:** Orchestrator validation logic logs: `"WARNING: Agent <name> missing memory_access field."` The agent is treated as `read-only` for validation purposes. Dispatch proceeds normally.
- **Severity if Missed:** Medium — silent assumption could mask a read-write agent being dispatched in parallel.

### EC-3: Agent YAML Contains Invalid `memory_access` Value

- **Input/Condition:** An agent file contains `memory_access: append-only` or another non-standard value.
- **Expected Behavior:** Orchestrator validation logic logs: `"WARNING: Agent <name> has unrecognized memory_access value '<value>'. Treating as read-only."` Dispatch proceeds.
- **Severity if Missed:** Medium — invalid value is silently ignored, potentially masking a write agent.

### EC-4: Researcher Dispatched in Synthesis Mode

- **Input/Condition:** Orchestrator dispatches `researcher.agent.md` at Step 1.2 in synthesis mode (sequential).
- **Expected Behavior:** Orchestrator applies mode override — treats researcher as `read-write` despite `memory_access: read-only` in YAML. No warning is logged for this specific override.
- **Severity if Missed:** High — synthesis mode writes to memory as part of its core function; blocking or warning on this would disrupt the pipeline.

### EC-5: `docs/patterns.md` Not Read by Orchestrator Agent

- **Input/Condition:** The AI running as orchestrator does not read `docs/patterns.md` and lacks full pattern details during dispatch.
- **Expected Behavior:** One-line summaries in `orchestrator.agent.md` provide enough context for routine dispatch. The pointer to `docs/patterns.md` is explicit and the orchestrator's Operating Rules or workflow text instruct it to read the file when detailed pattern logic is needed (e.g., error handling, edge cases within patterns).
- **Severity if Missed:** High — orchestrator may mishandle Pattern C replan loop or Pattern B sequential gate if it doesn't recall error-handling steps.

### EC-6: r-knowledge Artifact Index Has Insufficient Detail

- **Input/Condition:** The Artifact Index in `memory.md` does not contain detailed section descriptions for `feature.md`, `design.md`, `plan.md`, or `verifier.md`.
- **Expected Behavior:** r-knowledge falls back to reading the artifacts more broadly using `semantic_search` and `grep_search` (per its Operating Rules). The change from full reads to Artifact Index navigation is a best-effort optimization, not a hard restriction — r-knowledge retains the ability to read any file in the codebase.
- **Severity if Missed:** Medium — r-knowledge may produce less thorough knowledge analysis, but r-aggregator compensates by aggregating available inputs.

### EC-7: Documentation-Writer Dispatched in Parallel with Other Implementers

- **Input/Condition:** A wave in Step 5 contains both `implementer` and `documentation-writer` tasks dispatched simultaneously.
- **Expected Behavior:** Orchestrator validation checks `memory_access` for all agents in the wave. `documentation-writer` has `memory_access: read-write`, so the validator logs a warning. However, the orchestrator's Step 5.2 text explicitly acknowledges documentation-writer as a write exception that operates sequentially within each wave, making this safe. The warning is informational.
- **Severity if Missed:** High — if documentation-writer actually writes to memory in parallel with another documentation-writer, memory corruption could occur.

### EC-8: Orchestrator Exceeds 400 Lines After All Changes

- **Input/Condition:** Adding validation logic (Change 3, ~10–15 lines) plus pattern pointers offsets the line reductions from Changes 1 and 2.
- **Expected Behavior:** The implementer must balance extraction aggressiveness with validation additions to meet the ≤400-line target. If the target cannot be met, the Parallel Execution Summary compression is the primary relief valve (up to 30+ lines recoverable).
- **Severity if Missed:** Low — the 400-line target is a quality goal, not a functional requirement. The pipeline works at 410 lines; the risk is reduced comprehension by the AI running as orchestrator.

---

## User Stories / Flows

### US-1: Normal Pipeline Run (Post-Changes)

1. User submits feature request.
2. Orchestrator initializes `memory.md` (no emergency prune threshold).
3. Orchestrator dispatches 4 focused researchers in parallel. Before dispatch, reads each agent's `memory_access` field → all `read-only` → proceeds.
4. Orchestrator dispatches researcher in synthesis mode. Applies mode override → treats as `read-write` → proceeds sequentially.
5. Checkpoint prune after Step 1.2. Memory entries older than 2 phases removed.
6. Sequential agents (spec, designer, planner) each write to memory. Each has `memory_access: read-write`.
7. CT cluster dispatched. 4 sub-agents validated as `read-only` → parallel dispatch. CT-aggregator dispatched sequentially → `read-write`.
8. Implementation waves dispatched. Implementers validated as `read-only`. Documentation-writer detected as `read-write` → warning logged → acknowledged as safe exception.
9. V cluster dispatched via Pattern B+C (orchestrator reads `docs/patterns.md` for full logic or uses inline summary for routine dispatch).
10. R cluster dispatched. r-knowledge receives only `tier, initial-request.md, memory.md`. Uses Artifact Index to navigate other artifacts.
11. Pipeline completes. `orchestrator.agent.md` is ≤400 lines throughout.

### US-2: Future Agent Addition

1. Developer creates a new parallel sub-agent file.
2. Developer adds `memory_access: read-only` to YAML frontmatter.
3. Orchestrator validation automatically checks the field at dispatch.
4. If developer forgets the field, orchestrator logs a warning and treats as read-only.

---

## Test Scenarios

### TS-1: Emergency Prune String Removal

- **Verifies:** AC-1.1, AC-1.2
- **Method:** `grep -i "emergency prune" NewAgentsAndPrompts/orchestrator.agent.md` returns no results. `grep "200 lines" NewAgentsAndPrompts/orchestrator.agent.md` returns no results related to memory (may return results about `read_file` if present, which is acceptable).

### TS-2: Memory Lifecycle Table Row Count

- **Verifies:** AC-1.3
- **Method:** Count rows in the Memory Lifecycle Actions table in `orchestrator.agent.md`. Expect exactly 6 data rows (excluding header and separator).

### TS-3: Orchestrator Line Count

- **Verifies:** AC-2.1
- **Method:** `wc -l NewAgentsAndPrompts/orchestrator.agent.md` returns ≤400.

### TS-4: Patterns File Existence and Content

- **Verifies:** AC-2.2, AC-2.3
- **Method:** `test -f docs/patterns.md` succeeds. `grep "Pattern A" docs/patterns.md` returns results. `grep "Pattern B" docs/patterns.md` returns results. `grep "Pattern C" docs/patterns.md` returns results. `grep "docs/patterns.md" NewAgentsAndPrompts/orchestrator.agent.md` returns results.

### TS-5: Memory Access Field — Read-Only Agents

- **Verifies:** AC-3.1, AC-3.3
- **Method:** For each of the 14 read-only agent files, parse YAML frontmatter and verify `memory_access: read-only` is present. Can be done with: `grep "memory_access: read-only" NewAgentsAndPrompts/<file>` for each file.

### TS-6: Memory Access Field — Read-Write Agents

- **Verifies:** AC-3.2
- **Method:** For each of the 7 active read-write agent files (excluding `critical-thinker.agent.md`), parse YAML frontmatter and verify `memory_access: read-write` is present.

### TS-7: Memory Access Values Are Valid

- **Verifies:** AC-3.9
- **Method:** `grep "memory_access:" NewAgentsAndPrompts/*.agent.md` — every match must end with either `read-only` or `read-write`. No other values permitted.

### TS-8: Orchestrator Validation Points

- **Verifies:** AC-3.4
- **Method:** `grep -c "memory_access" NewAgentsAndPrompts/orchestrator.agent.md` returns ≥5 (one per dispatch point plus possible references in rules). Verify textually that validation logic exists at Steps 1.1, 3b.1, 5.2, 6.2, 7.2.

### TS-9: Documentation-Writer Exception Acknowledged

- **Verifies:** AC-3.5
- **Method:** `grep "documentation-writer" NewAgentsAndPrompts/orchestrator.agent.md` returns a result near Step 5.2 that acknowledges it as a memory-write exception.

### TS-10: r-knowledge Inputs in Orchestrator

- **Verifies:** AC-4.1
- **Method:** In the Step 7.2 dispatch table of `orchestrator.agent.md`, the r-knowledge row lists `tier, initial-request.md, memory.md`. The strings `feature.md`, `design.md`, `plan.md`, `verifier.md` do NOT appear in the r-knowledge row specifically (they may appear elsewhere for other agents).

### TS-11: r-knowledge Agent File Inputs Section

- **Verifies:** AC-4.2, AC-4.3
- **Method:** `r-knowledge.agent.md` Inputs section does not list `feature.md`, `design.md`, `plan.md`, or `verifier.md` as direct inputs. Workflow Step 2 contains the phrase "Artifact Index" and does NOT instruct reading those files in full.

### TS-12: Excluded Files Not Modified

- **Verifies:** AC-1.5, AC-3.7, AC-3.8
- **Method:** `git diff` shows no changes to `critical-thinker.agent.md`, `feature-workflow.prompt.md`, or any file under `docs/feature/forge-architecture-upgrade/` or `docs/feature/agent-improvements/`.

### TS-13: README Accuracy

- **Verifies:** AC-5.1, AC-5.2
- **Method:** `grep "×3" README.md` returns no researcher references. `grep "×4" README.md` or `grep "4" README.md` returns researcher references. Project Layout section lists cluster sub-agent files.

---

## Files Modified

### New Files Created

| File               | Change                                            |
| ------------------ | ------------------------------------------------- |
| `docs/patterns.md` | Extracted dispatch pattern definitions (Change 2) |

### Modified Files

| File                                                | Changes Applied                                                                                                                           |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `NewAgentsAndPrompts/orchestrator.agent.md`         | All 4 changes: remove emergency prune (1), extract patterns & compress (2), add validation logic (3), update r-knowledge dispatch row (4) |
| `NewAgentsAndPrompts/r-knowledge.agent.md`          | Changes 3 & 4: add `memory_access: read-only` frontmatter, reduce Inputs section, update Workflow Step 2                                  |
| `NewAgentsAndPrompts/ct-security.agent.md`          | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/ct-scalability.agent.md`       | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/ct-maintainability.agent.md`   | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/ct-strategy.agent.md`          | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/ct-aggregator.agent.md`        | Change 3: add `memory_access: read-write`                                                                                                 |
| `NewAgentsAndPrompts/v-tests.agent.md`              | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/v-tasks.agent.md`              | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/v-feature.agent.md`            | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/v-build.agent.md`              | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/v-aggregator.agent.md`         | Change 3: add `memory_access: read-write`                                                                                                 |
| `NewAgentsAndPrompts/r-quality.agent.md`            | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/r-security.agent.md`           | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/r-testing.agent.md`            | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/r-aggregator.agent.md`         | Change 3: add `memory_access: read-write`                                                                                                 |
| `NewAgentsAndPrompts/researcher.agent.md`           | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/spec.agent.md`                 | Change 3: add `memory_access: read-write`                                                                                                 |
| `NewAgentsAndPrompts/designer.agent.md`             | Change 3: add `memory_access: read-write`                                                                                                 |
| `NewAgentsAndPrompts/planner.agent.md`              | Change 3: add `memory_access: read-write`                                                                                                 |
| `NewAgentsAndPrompts/implementer.agent.md`          | Change 3: add `memory_access: read-only`                                                                                                  |
| `NewAgentsAndPrompts/documentation-writer.agent.md` | Change 3: add `memory_access: read-write`                                                                                                 |
| `README.md`                                         | Change 5: fix stale researcher counts and project layout                                                                                  |

**Total: 1 new file, 23 modified files.**

### Files NOT Modified

| File                                             | Reason                            |
| ------------------------------------------------ | --------------------------------- |
| `NewAgentsAndPrompts/critical-thinker.agent.md`  | Deprecated, malformed frontmatter |
| `NewAgentsAndPrompts/feature-workflow.prompt.md` | Prompt file, not an agent         |
| `docs/feature/forge-architecture-upgrade/*`      | Historical artifacts              |
| `docs/feature/agent-improvements/*`              | Historical artifacts              |
| `docs/optimization-from-gem-team.md`             | Not affected                      |
| `docs/comparison-forge-vs-gem-team.md`           | Not affected                      |

---

## Out of Scope

- **File locking mechanisms:** VS Code Copilot has no file-locking API. The `memory_access` markers are advisory and machine-checkable, not enforced by runtime locks.
- **Splitting `researcher.agent.md` into two files:** The dual-mode complication is handled via orchestrator override, not file splitting. File splitting may be considered in a future change.
- **Modifying `critical-thinker.agent.md`:** Deprecated agent with malformed frontmatter; excluded from this change.
- **Memory compaction algorithms:** No new pruning or compaction mechanism is introduced. Existing checkpoint pruning is sufficient.
- **Automated YAML validation tooling:** The orchestrator validates `memory_access` at dispatch time. No CI/CD or pre-commit hook is added.
- **Changes to the 8-step pipeline structure:** The pipeline sequence, step numbering, and agent roles are unchanged.
- **Pattern D or new dispatch patterns:** Only existing Patterns A, B, C are extracted. No new patterns are defined.

---

## Success Criteria

All of the following can be verified mechanically:

1. `grep -ic "emergency prune" NewAgentsAndPrompts/orchestrator.agent.md` → `0`
2. `wc -l < NewAgentsAndPrompts/orchestrator.agent.md` → `≤400`
3. `test -f docs/patterns.md` → exits 0
4. `grep -c "memory_access: read-only" NewAgentsAndPrompts/*.agent.md` → `14`
5. `grep -c "memory_access: read-write" NewAgentsAndPrompts/*.agent.md` → `7`
6. `grep -c "memory_access:" NewAgentsAndPrompts/orchestrator.agent.md` → `0` (orchestrator has no marker on itself, but references the field in validation logic; the literal YAML field is absent from its frontmatter)
7. `grep "memory_access" NewAgentsAndPrompts/orchestrator.agent.md` → returns validation logic references (≥5 lines)
8. r-knowledge dispatch row in orchestrator contains `memory.md` and does NOT contain `feature.md, design.md, plan.md, verifier.md` as inputs
9. No changes in `git diff` for `critical-thinker.agent.md`, `feature-workflow.prompt.md`, or `docs/feature/forge-architecture-upgrade/`

---

## Dependencies & Risks

### Dependencies

| Dependency                                    | Impact                                                                                                                      |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Change 1 must be applied before Change 2      | Change 2 simplifies the Memory Lifecycle table, which depends on Change 1 having removed the emergency prune row.           |
| Change 3 adds lines that Change 2 must offset | Change 2's line reduction target must account for ~10–15 lines of validation logic added by Change 3.                       |
| Artifact Index quality (upstream agents)      | Change 4's effectiveness depends on spec, designer, and planner maintaining detailed Artifact Index entries in `memory.md`. |

### Risks

| Risk                                                      | Likelihood | Impact                                   | Mitigation                                                                                                                    |
| --------------------------------------------------------- | ---------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| YAML `memory_access` field rejected by runtime            | Low        | Critical — agents fail to load           | Verify against VS Code Copilot agent runtime before deployment. Rollback plan: remove the field from all files.               |
| Orchestrator AI doesn't read `docs/patterns.md`           | Medium     | High — dispatch errors                   | Include explicit instruction in orchestrator workflow to read `docs/patterns.md` at startup or before first cluster dispatch. |
| ≤400-line target infeasible after adding validation logic | Low        | Low — quality goal, not functional       | Compress Parallel Execution Summary more aggressively or move NEEDS_REVISION routing table to `docs/patterns.md`.             |
| r-knowledge produces weaker analysis with reduced inputs  | Medium     | Medium — less thorough knowledge capture | r-knowledge retains full codebase access and can fall back to broader reads. R-aggregator compensates.                        |
| Memory grows unbounded without emergency prune            | Low        | Medium — larger context reads            | Checkpoint pruning already constrains growth. Monitor memory size across runs.                                                |
