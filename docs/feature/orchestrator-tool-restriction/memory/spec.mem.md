# Memory: spec

## Status

DONE: Feature specification complete for orchestrator tool restriction (5-tool set) and memory architecture redesign (Pattern B — eliminate shared memory.md).

## Key Findings

- Tool restriction requires updates to only 3 locations in orchestrator.agent.md (Global Rule 1, Operating Rule 1, Operating Rule 5) plus the Anti-Drift Anchor — zero behavioral impact on pipeline
- Pattern B (eliminate memory.md) is viable because all 19 subagents have fallback logic AND already list specific upstream `.mem.md` files as primary inputs
- Lessons Learned propagation is the only gap requiring a new mechanism: orchestrator reads implementer `.mem.md` files and passes lessons as dispatch prompt context to later-wave and V/R agents
- Total file modification scope: 21 agent files + `feature-workflow.prompt.md` + `dispatch-patterns.md` = 23 files
- YAML `tools:` frontmatter field specified as new convention for the orchestrator (first use in repo) for formal tool restriction declaration alongside existing prose enforcement

## Highest Severity

N/A

## Decisions Made

- YAML `tools:` field: include in orchestrator frontmatter as formal declaration, even though no other agent uses it — prose restriction remains primary enforcement
- Memory Lifecycle Actions table: remove entirely rather than partially preserve — the 7-phase lifecycle is not applicable without shared memory.md
- Self-verification logging: retain decision logic but log to context window only (captured by telemetry tracking Rule 13) rather than persisting to any file
- Invalidation on revision: communicate via dispatch prompt instructions instead of file markers — "Prior V/R results are superseded"
- Subagent updates: update all 20 files (not just rely on fallback) to ensure clean architecture with no dangling references

## Artifact Index

- [feature.md](../feature.md)
  - §Functional Requirements — FR-1 (tool restriction, 6 sub-requirements), FR-2 (eliminate memory.md, 7 sub-requirements), FR-3 (dispatch prompts, 4 sub-requirements), FR-4 (subagent definitions, 4 sub-requirements), FR-5 (supporting documents, 3 sub-requirements), FR-6 (memory tool disambiguation, 2 sub-requirements)
  - §Non-Functional Requirements — NFR-1 through NFR-5 covering backward compatibility, no new subagents, token efficiency, consistency, cluster decision integrity
  - §Acceptance Criteria — AC-1 through AC-15 with explicit pass/fail definitions
  - §Edge Cases — EC-1 through EC-8 covering missing files, malformed memory, revision loops, R-Knowledge, Lessons Learned, concurrency, tool conflation, PostMortem context
  - §Test Scenarios — TS-1 through TS-11 mapping to acceptance criteria
  - §Dependencies & Risks — 5 dependencies, 6 risks with likelihood/impact/mitigation
