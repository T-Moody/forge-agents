# Memory: ct-security

## Status

DONE: Re-review (Revision 1) complete — 4 previous findings RESOLVED (both High findings eliminated), 8 findings remain (0 Critical, 0 High, 4 Medium, 4 Low). Key improvements: read tools retained in orchestrator, telemetry moved out of memory.md, shared evaluation schema. Remaining issues: spec-design divergences on FR-6/AC-5 and FR-3.3/AC-8, YAML enforcement gap, cross-run evaluation data contamination.

## Key Findings

- Both previous High findings resolved: orchestrator retains read tools (CT-1), telemetry no longer depends on memory tool (CT-3/CT-7)
- Two spec-design divergences (FR-6.1/AC-5 tool restriction, FR-3.3 improvement_recommendations) create conflicting implementation/acceptance targets — feature.md not updated to reflect CT-driven design revisions
- YAML schema enforcement still relies entirely on LLM self-compliance — shared schema doc improves consistency but not correctness; data loss remains the degradation path
- Cross-run evaluation data contamination: PostMortem globs all evaluations including prior-run files; no run-level metadata distinguishes current from stale data
- Prompt injection risk downgraded from Medium to Low — PostMortem's quantitative-only output and File Boundaries limit exploitation surface

## Highest Severity

Medium

## Decisions Made

(none — CT agents do not make decisions)

## Artifact Index

- [ct-review/ct-security.md](../ct-review/ct-security.md)
  - §Previous Finding Resolution Summary — table mapping 9 v0 findings to resolution status (4 resolved, 5 remain)
  - §Findings — 8 findings (4 Medium, 4 Low) with severity, likelihood, impact, and assumption-at-risk
  - §Cross-Cutting Observations — 3 observations for ct-strategy, ct-maintainability, ct-scalability scopes
  - §Requirement Coverage — FR-1 through FR-8 and NFR coverage with spec divergence flags on FR-3 and FR-6
