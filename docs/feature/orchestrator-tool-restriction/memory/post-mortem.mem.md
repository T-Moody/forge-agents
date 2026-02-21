# Memory: post-mortem

## Status

DONE: Post-mortem analysis complete — 16 evaluation files processed (39 evaluation blocks), 8 producer agents scored, 32 telemetry entries logged.

## Key Findings

- Spec agent's feature.md had lowest producer scores (avg usefulness 6.6, avg clarity 7.5) with 12 reported inaccuracies — driven primarily by missing Phase 1/Phase 2 scope annotations (reported by 6 downstream agents)
- CT cluster required 3 iterations (15 of 32 total dispatches) — the dominant pipeline cost driver; iteration 1 found scope coupling, iteration 2 found 5 pre-existing memory tool contradictions
- NFR-3.1 token estimate (100-250 tokens) was 2-5x understated vs measured 488 avg — propagated from spec to design before CT correction
- All agents downstream of CT (planner, implementers, verifiers, reviewers) completed in single passes with 0 errors and 0 retries
- Researcher cluster scored highest overall (avg usefulness 9.0, avg clarity 8.75, 0 inaccuracies across 4 agents)

## Highest Severity

N/A

## Decisions Made

- Scored only producer agents whose artifacts received evaluation blocks (8 of 32 dispatches had scorable outputs); implementers, verifiers, reviewers, and CT agents produce artifacts that are not evaluated via artifact_evaluation blocks in this pipeline run.

## Artifact Index

- agent-metrics/2025-07-17-run-log.md — §Agent Telemetry (32 entries), §Cluster Summaries (7 clusters), §Pipeline Summary
- post-mortems/2025-07-17-post-mortem.md — §post_mortem_report (recurring issues, bottlenecks, missing information, accuracy scores), §Evaluation Coverage (39 blocks across 16 files), §Inaccuracy Detail (14 reported inaccuracies)
