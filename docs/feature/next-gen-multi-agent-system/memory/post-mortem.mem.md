# Memory: post-mortem

## Status

DONE: Post-mortem analysis complete — 75 evaluation entries processed across 25 files, 8 producer agents scored. 47 total dispatches, 1 ERROR (r-knowledge), 2 fix cycles, 3 network retries.

## Key Findings

- Designer's dual schema definitions (§Decision 6 vs §Data Storage) caused the highest recurring inaccuracy — 5 agents independently flagged conflicting output_snippet and severity enum values
- feature.md scored low usefulness (avg 5.7) across implementers because design.md already contained all implementation detail; evaluators who directly needed it (ct-\*, v-feature, r-testing) averaged 8.1
- Designer was the pipeline bottleneck: 4 iterations (v1→v4), 2 adversarial rounds, 3 network retries, plus user corrections — but final output scored highest usefulness (9.3 avg) across 19 evaluators
- r-knowledge ERROR (empty output) was non-blocking but represents a silent failure of the evidence bundle assembly step
- All 4 researcher outputs scored 8.0–10.0 usefulness with zero inaccuracies — cleanest producer quality in the run

## Highest Severity

N/A

## Decisions Made

- Computed spec agent usefulness in two cohorts (direct consumers vs implementers) to avoid misleading averages from structural low scores

## Artifact Index

- agent-metrics/2025-07-18-run-log.md — §Agent Telemetry (47 dispatches), §Cluster Summaries (6 clusters), §Run Statistics
- post-mortems/2025-07-18-post-mortem.md — §post_mortem_report (7 recurring issues, 6 bottlenecks, 7 missing information items, 8 agent accuracy scores)
