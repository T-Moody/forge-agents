# Memory: spec

## Status

DONE: Comprehensive feature specification produced for the self-improvement system covering all 6 parts plus orchestrator tool restriction, with 8 functional requirement groups, 7 non-functional requirements, 14 acceptance criteria, 11 edge cases, and 15 test scenarios.

## Key Findings

- 14 agents require artifact evaluation modifications; evaluations use YAML in fenced code blocks within Markdown files stored in per-feature `artifact-evaluations/` directory — first structured YAML output in the codebase
- Orchestrator tool restriction to `[agent, agent/runSubagent, memory]` conflicts with current direct file-write operations in Step 0 setup and all memory lifecycle actions; requires delegation to subagents or `memory` tool — `memory` tool scope is the highest-risk dependency
- PostMortem agent uses the non-blocking R-Knowledge pattern (two-state contract, ERROR does not block pipeline) as Step 8 after R cluster; boundary with R-Knowledge is quantitative metrics (PostMortem) vs. qualitative suggestions (R-Knowledge)
- Execution timestamp precision is unavailable in the LLM runtime; telemetry uses best-effort timestamps with graceful degradation — all other telemetry fields (retry count, failure reason, completion status) are fully observable
- Evaluation generation is secondary to primary agent output; evaluation failure must never cause agent ERROR — non-blocking evaluation pattern is critical for safety

## Highest Severity

N/A

## Decisions Made

- Evaluations stored in dedicated `artifact-evaluations/` directory as separate Markdown files (not appended to primary output), one file per evaluating agent containing YAML code blocks
- Per-feature scoping for all 3 new directories (not global) — aligns with existing documentation structure convention
- PostMortem uses two-state completion contract (DONE/ERROR only, no NEEDS_REVISION) following R-Knowledge non-blocking pattern
- Execution time fields use best-effort timestamps with explicit "unavailable" fallback, rather than omitting the fields entirely
- One evaluation file per agent (may contain multiple evaluation blocks for agents consuming multiple artifacts), named `<agent-name>.md`

## Artifact Index

- [feature.md](../feature.md)
  - §Functional Requirements — FR-1 through FR-8 covering all 6 parts + orchestrator tool restriction + safety + deliverables
  - §Non-Functional Requirements — 7 NFRs including additive-only, structured output, non-blocking, graceful degradation
  - §Acceptance Criteria — 14 testable criteria (AC-1 through AC-14) with explicit pass/fail definitions
  - §Edge Cases & Error Handling — 11 edge cases (EC-1 through EC-11) with input/condition, expected behavior, and severity
  - §Test Scenarios — 15 test scenarios (TS-1 through TS-15) mapped to acceptance criteria
  - §Dependencies & Risks — dependency table and risk matrix with mitigations
  - §Constraints & Assumptions — 6 constraints and 5 assumptions grounding the specification
