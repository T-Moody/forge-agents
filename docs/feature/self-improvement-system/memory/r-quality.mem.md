# Memory: r-quality

## Status

NEEDS_REVISION: quality — 2 major issues (post-mortem code fence format, evaluation error handling split across 6 agents) and 3 minor issues (template variants, r-testing input mismatch, step ordering)

## Key Findings

- post-mortem.agent.md wraps content in ```chatagent code fence instead of bare `---` frontmatter — deviates from all 18 other agent files
- 6/14 agents use "skip evaluation" on failure instead of "write evaluation_error block" per schema Rule 4 — root cause is design.md template conflicting with evaluation-schema.md
- 5 distinct evaluation step template variants across 14 agents (5 different implementers, no cross-checking) — increases maintenance burden
- r-testing evaluates design.md but design.md is not listed in its Inputs section — internal inconsistency
- Evaluation step ordering inconsistent: CT agents place it before self-verification (correct per design), spec/designer/planner place it after

## Highest Severity

Major

## Decisions Made

- Classified post-mortem code fence as Major (not Minor) because it deviates from all 18 other agent files and may affect runtime parsing. (2 sentences)
- Classified error handling split as Major because it creates a behavioral divergence where 6 agents silently skip vs 8 agents produce traceable error records — the schema is the authoritative reference.

## Artifact Index

- review/r-quality.md — §Findings (5 findings: 2 Major, 3 Minor), §Cross-Cutting Observations (design-schema conflict for R-Knowledge, code fence for R-Security), §Summary (overall assessment)
