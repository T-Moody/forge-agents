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
