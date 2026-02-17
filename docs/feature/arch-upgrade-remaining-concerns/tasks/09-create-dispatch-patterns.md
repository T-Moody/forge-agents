# Task 09: Create dispatch-patterns.md

**Task Goal:** Create `NewAgentsAndPrompts/dispatch-patterns.md` containing full definitions of Patterns A (Fully Parallel) and B (Sequential Gate + Parallel).

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Create new file `NewAgentsAndPrompts/dispatch-patterns.md`
- Include full definitions of Pattern A and Pattern B only
- Pattern C (Replan Loop) is NOT included — it stays inline in the orchestrator

## Out-of-Scope

- Modifying `orchestrator.agent.md` (that is Task 12)
- Including Pattern C in this file
- Adding any new patterns

---

## Acceptance Criteria

1. `NewAgentsAndPrompts/dispatch-patterns.md` exists
2. File contains "Pattern A" with full definition
3. File contains "Pattern B" with full definition
4. File does NOT contain Pattern C definition (only a reference note that C is inline in orchestrator)
5. Content matches the orchestrator's current Pattern A and B definitions (L82–103 approximately)

---

## Estimated Effort

Low

---

## Test Requirements

- `Test-Path NewAgentsAndPrompts/dispatch-patterns.md` → True
- `grep "Pattern A" NewAgentsAndPrompts/dispatch-patterns.md` → match
- `grep "Pattern B" NewAgentsAndPrompts/dispatch-patterns.md` → match

---

## Implementation Steps

### Step 1: Read Current Pattern Definitions

Read `NewAgentsAndPrompts/orchestrator.agent.md` lines ~82–121 to capture the current Pattern A and Pattern B definitions. These are the source of truth.

### Step 2: Create the File

Create `NewAgentsAndPrompts/dispatch-patterns.md` with this content (adapt the pattern text from the orchestrator's current definitions):

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

**Important:** Compare the generated text against the orchestrator's current pattern definitions and adjust wording to match if the orchestrator uses different phrasing.

---

## Completion Checklist

- [x] File `NewAgentsAndPrompts/dispatch-patterns.md` created
- [x] Pattern A fully defined with usage context
- [x] Pattern B fully defined with usage context
- [x] Pattern C NOT included (only referenced as inline in orchestrator)
- [x] Content is consistent with orchestrator's current pattern definitions
