# Memory: implementer-fix-eval-errors

## Status

DONE: Fixed evaluation error handling in 6 agent files + design.md template

## Key Findings

- 6 agent files used "skip evaluation and proceed" instead of the evaluation-schema.md Rule 4 compliant "write an `evaluation_error` block" directive
- Indentation varies: spec.agent.md uses 3-space indent, designer/planner use 4-space indent, v-tests/v-tasks/v-feature use no indent
- design.md contained the same incorrect template text at ~line 340
- After fix, grep confirms 0 remaining occurrences of the incorrect phrase

## Decisions Made

- Preserved each file's existing indentation exactly as found, per task instructions.

## Artifact Index

- .github/agents/spec.agent.md — §Evaluate Upstream Artifacts Rules (line ~87)
- .github/agents/designer.agent.md — §Evaluate Upstream Artifacts Rules (line ~91)
- .github/agents/planner.agent.md — §Evaluate Upstream Artifacts Rules (line ~117)
- .github/agents/v-tests.agent.md — §Evaluate Upstream Artifacts Rules (line ~160)
- .github/agents/v-tasks.agent.md — §Evaluate Upstream Artifacts Rules (line ~112)
- .github/agents/v-feature.agent.md — §Evaluate Upstream Artifacts Rules (line ~115)
- docs/feature/self-improvement-system/design.md — §Evaluation template (line ~340)
