# Researcher Memory — Impact Focus Area

## Status

DONE

## Key Findings

- 14 existing agent files need modification to add artifact evaluation sections (spec, designer, 4×CT, planner, implementer, documentation-writer, 3×V-sub-agents, r-quality, r-testing)
- Orchestrator (486 lines) requires the heaviest changes: telemetry tracking at every dispatch point, tool restriction removing direct file operations, new Step 8 for post-mortem, and documentation structure updates
- 1 new agent file (`.github/agents/post-mortem.agent.md`) and 3 new storage directories (`agent-metrics/`, `artifact-evaluations/`, `post-mortems/`) must be created
- 4 agents are safely excluded from artifact evaluation: researcher (no upstream pipeline artifacts), v-build (codebase only), r-security (git diff only), r-knowledge (self-referential)
- Existing completion contracts (DONE/ERROR/NEEDS_REVISION) remain unchanged — all changes are additive

## Highest Severity

N/A

## Decisions Made

_(none — impact research only, no design decisions)_

## Artifact Index

- [research/impact.md](../research/impact.md)
  - §Artifact Consumption Map — full matrix of which agents consume which upstream artifacts
  - §Existing Agent Files Requiring Modification — detailed per-file change descriptions with line references
  - §Orchestrator Changes — telemetry, tool restriction, and new pipeline step analysis
  - §New Agent File — post-mortem agent creation requirements
  - §New Storage Directories — 3 new directories with purpose and contents
  - §Files That Will NOT Be Modified — safety constraint analysis
  - §Open Questions — 6 items needing design decisions
