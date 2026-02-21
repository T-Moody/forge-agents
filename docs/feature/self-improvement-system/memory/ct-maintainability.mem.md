# Memory: ct-maintainability

## Status

DONE: maintainability re-review — 8 findings (3 Medium residual, 2 Medium new, 3 Low new). 4 of 11 original findings fully resolved (schema duplication, telemetry DSL, memory.md bloat, memory tool scope). 1 cleanly resolved (PostMortem/R-Knowledge boundary). Primary remaining risks: context-window telemetry accumulation reliability, schema reference-by-instruction gap, YAML validation absence, and spec/design AC-5 conflict.

## Key Findings

- Context-based telemetry eliminates 4 original findings (F1, F3-F5) but introduces implicit LLM state accumulation burden — orchestrator must maintain growing telemetry table across 20+ dispatch cycles with no explicit persistence mechanism (Medium)
- Shared evaluation-schema.md resolves 14-file duplication but is referenced by instruction only, not listed in evaluating agent Inputs — agents may not read it, weakening schema compliance (Medium)
- YAML validation gap unchanged — still 14 agents producing YAML for the first time with no validation pipeline; PostMortem skip-on-corrupt is sole defense (Medium)
- Feature spec AC-5 contradicts design's read-tool retention — spec says `[agent, runSubagent, memory]` only; design keeps read tools per CT-1 Critical resolution; spec not updated (Medium)
- Feature spec FR-3.3 and US-3 still reference `improvement_recommendations` removed by CT-6 resolution — documentation inconsistency (Low)

## Highest Severity

Medium

## Decisions Made

(none — CT agents do not make decisions)

## Artifact Index

- [ct-review/ct-maintainability.md](../ct-review/ct-maintainability.md)
  - §Previous Findings Disposition — table mapping 11 original findings to resolution status (4 resolved, 1 resolved structural, 2 partially mitigated, 2 unresolved/downgraded, 1 unchanged)
  - §Residual Findings — F-R1 (telemetry accumulation as implicit LLM state), F-R2 (PostMortem coupling surface), F-R3 (YAML validation gap)
  - §New Findings — F-N1 (spec/design AC-5 conflict), F-N2 (schema doc not in agent Inputs), F-N3 (spec references removed field), F-N4 (collision avoidance instruction gap), F-N5 (step numbering ambiguity)
  - §Cross-Cutting Observations — notes for CT-Strategy (AC-5 conflict), CT-Scalability (telemetry context growth), CT-Security (soft schema compliance)
  - §Requirement Coverage — FR-1 through FR-7, NFR-1, NFR-2, NFR-4, NFR-6, NFR-7 mapped to findings
