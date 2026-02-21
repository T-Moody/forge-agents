# Operational Memory

## Artifact Index

| Artifact | Key Sections | Last Updated By |
|---|---|---|
| research/architecture.md | §1 Orchestrator Agent (YAML, tools, memory writes), §2 Cluster Decisions, §3 Agent Format, §5 Memory Architecture | researcher-architecture |
| research/impact.md | §1 Tool Removal Impact, §2 Memory Write Impact, §3 Cluster Flows, §5 Downstream Impact | researcher-impact |
| research/dependencies.md | §1 Files Referencing Tools, §2 Cross-Agent Memory, §3 Tool Dependency Chain, §7 Impact by Option | researcher-dependencies |
| research/patterns.md | §1 Memory Patterns (A/B/C), §2 Orchestrator Purity, §3 Subagent Input Patterns, §4 Anti-Patterns | researcher-patterns |

## Recent Decisions

<!-- Format: - [agent-name, step-N] Decision summary. Rationale: ... -->
- [orchestrator, step-1] Research phase complete. All 4 researchers converge on Pattern B (eliminate memory.md) as best approach. Key consensus: (1) removing grep/semantic/file_search/get_errors has zero behavioral impact, (2) all 19 subagents have fallback for missing memory.md, (3) Pattern A (merge subagent) adds 10-20+ dispatches for marginal benefit, (4) VS Code `memory` tool ≠ pipeline memory.md.

## Lessons Learned

<!-- Never pruned. Format: - [agent-name, step-N] Issue → Resolution. -->

## Recent Updates

<!-- Format: - [agent-name, step-N] Updated `artifact-path` — summary. -->
- [researcher-architecture, step-1] Created `research/architecture.md` — orchestrator has no YAML `tools:` field, only 3 locations reference removed tools, ~15 memory.md write points
- [researcher-impact, step-1] Created `research/impact.md` — 18 memory write operations affected, all fallbacks already in place, cluster decisions fully read-only
- [researcher-dependencies, step-1] Created `research/dependencies.md` — all 20 agents read memory.md with graceful fallback, Artifact Index is highest-value component
- [researcher-patterns, step-1] Created `research/patterns.md` — Pattern B lowest cost, Pattern A adds 10-20+ dispatches, all subagents already list upstream .mem.md as primary inputs
