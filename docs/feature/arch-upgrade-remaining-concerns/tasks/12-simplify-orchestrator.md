# Task 12: Simplify Orchestrator — Extract Patterns, Compress Sections

**Task Goal:** Reduce `orchestrator.agent.md` to ≤400 lines by replacing inline Pattern A/B definitions with one-line summaries + reference pointer, simplifying workflow step descriptions, and compressing the Parallel Execution Summary.

**depends_on:** 09, 11

**agent:** implementer

---

## In-Scope

- Replace Cluster Dispatch Patterns section with compressed version (one-line summaries for A/B, retain Pattern C inline)
- Add reference pointer to `NewAgentsAndPrompts/dispatch-patterns.md`
- Simplify Workflow Steps 1.1, 3b.1, 6.1–6.4, 7.2 descriptions to reference patterns instead of repeating rules
- Compress Parallel Execution Summary to ≤18 lines
- Verify final line count ≤400

## Out-of-Scope

- Creating `dispatch-patterns.md` (already done in Task 09)
- Modifying YAML frontmatter
- Modifying validation logic added by Task 08
- Modifying the r-knowledge dispatch row (already done in Task 11)
- Removing any pipeline steps or changing behavior

---

## Acceptance Criteria

1. `orchestrator.agent.md` is ≤400 lines
2. Orchestrator contains reference to `NewAgentsAndPrompts/dispatch-patterns.md`
3. Pattern A and B have one-line summaries inline
4. Pattern C pseudocode is retained inline (NOT extracted)
5. Parallel Execution Summary is ≤20 lines
6. All pipeline steps remain functionally described
7. No behavioral changes — only compression and extraction

---

## Estimated Effort

Medium

---

## Test Requirements

- Line count: `(Get-Content NewAgentsAndPrompts/orchestrator.agent.md | Measure-Object -Line).Lines` → ≤400
- `grep "dispatch-patterns.md" NewAgentsAndPrompts/orchestrator.agent.md` → at least 1 match
- `grep "Pattern A" NewAgentsAndPrompts/orchestrator.agent.md` → match (one-line summary)
- `grep "Pattern B" NewAgentsAndPrompts/orchestrator.agent.md` → match (one-line summary)
- `grep "Pattern C" NewAgentsAndPrompts/orchestrator.agent.md` → match (inline pseudocode)

---

## Implementation Steps

### Step 1: Replace Cluster Dispatch Patterns Section (~L82–121)

**Find the current full pattern definitions (~40 lines).**

**Replace with (~18 lines):**

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

### Step 2: Simplify Step 1.1 Description

Remove redundant repetition of Pattern A rules (max-4 cap, error retry, wait-for-all). Keep the dispatch table and agent-specific notes. End with: "Execute per Pattern A."

**Target:** ~12 lines (from ~25 lines)

### Step 3: Simplify Step 3b.1 Description

Remove redundant Pattern A re-description. Keep dispatch table. End with: "Execute per Pattern A."

**Target:** ~10 lines (from ~20 lines)

### Step 4: Simplify Steps 6.1–6.4 Description

Remove redundant Pattern B gate logic and Pattern C loop re-description. Keep V-Build gate dispatch, V sub-agent table, aggregator dispatch, and handle-result section. Reference: "Execute per Pattern B + C."

**Target:** ~25 lines (from ~44 lines)

### Step 5: Simplify Step 7.2 Description

Remove redundant Pattern A re-description. Keep dispatch table and agent-specific error overrides (R-Knowledge non-blocking, R-Security critical). End with: "Execute per Pattern A."

**Target:** ~12 lines (from ~20 lines)

### Step 6: Compress Parallel Execution Summary (~L421–473)

**Replace the current 53-line section with a compact ~10-line summary:**

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

### Step 7: Verify Final Line Count

Count total lines. Must be ≤400.

If over 400, compress further:

- Additional step description simplification
- Remove any remaining redundant inline annotations
- Compress NEEDS_REVISION routing table if possible

### Step 8: Read Through for Coherence

Read the final file end-to-end to verify:

- All 8 pipeline steps are still described
- All dispatch points have memory safety checks (from Task 08)
- Reference to dispatch-patterns.md is present
- Pattern C pseudocode is inline
- Anti-drift anchor is intact

---

## Completion Checklist

- [x] Cluster Dispatch Patterns section compressed with reference pointer
- [x] Pattern A and B → one-line summaries
- [x] Pattern C → retained inline with pseudocode
- [x] Steps 1.1, 3b.1, 6.1–6.4, 7.2 simplified
- [x] Parallel Execution Summary compressed to ≤20 lines (13 lines)
- [x] Final line count ≤400 (398 lines)
- [x] Reference to `NewAgentsAndPrompts/dispatch-patterns.md` present
- [x] No behavioral changes — only compression
- [x] File reads coherently end-to-end

TDD skipped: documentation-only task (no behavioral code changes).
