# Memory: researcher-dependencies

## Status

DONE: Mapped full artifact consumption chain, memory merge patterns, dispatch integration points, and data flow for all 6 parts of the self-improvement system.

## Key Findings

- The pipeline has a strict linear artifact chain (researcher→spec→designer→CT→planner→implementer→V→R) with 10+ agents that consume upstream artifacts and are candidates for evaluation production (PART 1).
- Orchestrator dispatches all agents via `runSubagent` at 9 distinct pipeline points (Steps 1.1, 2, 3, 3b, 4, 5, 6.1, 6.2, 7) — each is a telemetry capture point; however, execution_time_ms is not natively available in the LLM runtime (PART 2).
- Orchestrator tool restriction to `[agent, agent/runSubagent, memory]` conflicts with current file-write operations (Step 0 setup, memory merge/prune, telemetry writing) — requires delegating file writes to subagents or extending the `memory` tool scope.
- R-knowledge agent already performs backward-looking pipeline analysis (knowledge-suggestions.md, decisions.md) with significant functional overlap with the proposed PostMortemAgent — boundary must be defined.
- New storage directories (agent-metrics/, artifact-evaluations/, post-mortems/) must decide between per-feature scoping (aligns with conventions) vs. global scoping (enables cross-run analysis).

## Highest Severity

N/A

## Decisions Made

_(none — research only)_

## Artifact Index

- research/dependencies.md — §Artifact Consumption Chain (maps which agents consume which artifacts — evaluation targets)
- research/dependencies.md — §Memory File Relationships (isolated→shared merge pattern and lifecycle)
- research/dependencies.md — §Orchestrator Dispatch Patterns and Telemetry Integration Points (9 dispatch points with pattern types)
- research/dependencies.md — §Integration Points for Artifact Evaluations (agent output contract changes needed for PART 1)
- research/dependencies.md — §Data Flow for Post-Mortem Agent (post-mortem integration point and r-knowledge overlap)
- research/dependencies.md — §Orchestrator Tool Restriction (conflict between tool restriction and telemetry/memory writes)
- research/dependencies.md — §Open Questions (5 design questions for downstream agents)
