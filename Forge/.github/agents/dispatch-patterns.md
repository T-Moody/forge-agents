# Cluster Dispatch Patterns — Reference

This is a reference document for the orchestrator. Contains full definitions of reusable
dispatch patterns for cluster invocations and the replan loop.

## Pattern A — Fully Parallel

Used by: CT cluster (Step 3b), R cluster (Step 7), Research focused (Step 1.1).

1. Dispatch N sub-agents in parallel (≤4).
2. Wait for all N to return.
3. Handle individual errors: retry once per Global Rule 4.
4. If ≥2 sub-agent outputs available: orchestrator reads sub-agent isolated memories (`memory/<agent-name>.mem.md`) and applies cluster decision logic to determine result (DONE / NEEDS_REVISION / ERROR).
5. If <2 outputs available after retries: cluster ERROR.

## Pattern B — Sequential Gate + Parallel

Used by: V cluster (Step 6).

1. Dispatch gate agent (V-Build) — sequential.
2. Wait for gate agent.
3. If gate ERROR: retry once. If still ERROR → skip parallel, cluster ERROR.
4. If gate DONE: dispatch N-1 sub-agents in parallel.
5. Wait for all N-1 to return. Handle errors: retry once each.
6. Orchestrator reads sub-agent isolated memories (`memory/<agent-name>.mem.md`) and applies V decision table to determine cluster result (DONE / NEEDS_REVISION / ERROR).

## Pattern C — Replan Loop

Used by: Verification-Replan cycle (Steps 5–6 iteration).

1. Run V cluster (Pattern B).
2. Orchestrator reads V sub-agent isolated memories (`memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md`).
3. If orchestrator determines DONE from V memories → pipeline continues to next step.
4. If NEEDS_REVISION → invoke planner with `MODE: REPLAN` + V memory file paths. Re-run implementation and verification.
5. If ERROR → pipeline ERROR.
6. Maximum 3 iterations. If still NEEDS_REVISION after 3 iterations → pipeline ERROR.

---

## Memory-First Pattern

All dispatch patterns follow the **memory-first** architecture:

- Each sub-agent writes an isolated memory file (`memory/<agent-name>.mem.md`) alongside its primary artifact.
- The orchestrator reads these isolated memories for routing decisions — it does **not** read full artifacts for dispatch logic.
- After evaluating cluster results, the orchestrator merges isolated memories into shared `memory.md`.
- Downstream agents read upstream isolated memories (as an index/summary) before selectively reading referenced artifact sections.
