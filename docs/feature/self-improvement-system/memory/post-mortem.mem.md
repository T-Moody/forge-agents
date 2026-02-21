# Memory: post-mortem

## Status

DONE: Post-mortem analysis complete — 22 evaluations processed across 8 files, 4 producer agents scored, 2 revision loops documented

## Key Findings

- 37 total dispatches with 0 errors and 0 retries; 2 revision loops (CT cluster at step 3b, R-Quality at step 7) added 7 additional dispatches
- Spec-design divergence (AC-5/FR-6.1 tool restriction, FR-3.3 improvement_recommendations) is the most frequently reported recurring issue (4 and 3 reports respectively) — feature.md was never reconciled after CT-driven design revisions
- Design template error handling conflict (design.md "skip evaluation" vs evaluation-schema.md Rule 4 "write evaluation_error block") propagated to 6 agent files before being caught by r-quality and fixed
- v-build produced the highest-rated artifacts (avg usefulness 9.0, clarity 9.0, 0 inaccuracies); spec produced the lowest usefulness scores (avg 7.4) due to implementers finding feature.md mostly irrelevant to their configuration tasks
- All 8 evaluation files were schema-compliant; no corrupted YAML blocks encountered

## Highest Severity

N/A

## Decisions Made

- Counted designer revision (step 3b.3) and CT re-runs as part of the CT bottleneck rather than as independent dispatches, since they form a single revision loop.

## Artifact Index

- agent-metrics/2026-02-20-run-log.md — §Agent Telemetry (36 entries), §Cluster Summaries (6 clusters)
- post-mortems/2026-02-20-post-mortem.md — §post_mortem_report (recurring issues, bottlenecks, missing information, agent accuracy scores, pipeline summary)
