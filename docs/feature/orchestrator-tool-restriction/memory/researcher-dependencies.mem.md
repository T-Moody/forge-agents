# Memory: researcher-dependencies

## Status

DONE: Comprehensive dependency mapping of orchestrator tool access, memory.md write chain, cross-agent memory read/write patterns, and impact analysis across all 3 memory architecture options.

## Key Findings

- All 20 sub-agents READ `memory.md` as first input; every agent has a graceful degradation fallback ("if missing, proceed with direct artifact reads")
- Removing `grep_search`, `semantic_search`, `file_search`, `get_errors` from orchestrator breaks NO critical-path operations — all cluster decisions use `read_file` on deterministic known paths
- The `memory` tool in Option B is VS Code cross-session memory (NOT a file writer), meaning ALL 9+ memory.md merge operations must become subagent dispatches under the tool restriction
- The Artifact Index in `memory.md` is the highest-value component; Lessons Learned aggregation (cross-wave) is the only section without redundant `.mem.md` sources
- Option B (remove memory.md) requires updating all 21 agent files + prompt + dispatch-patterns; Option A/C require only orchestrator + potentially new merge agent

## Highest Severity

N/A

## Decisions Made

(none — research only)

## Artifact Index

- `research/dependencies.md` — §Files That Reference Orchestrator Tools (§1), §Cross-Agent Memory Dependencies (§2), §Tool Dependency Chain (§3), §Memory Merge Dependency Chain (§4), §Existing Implementation Patterns (§5), §memory Tool vs File Write Tools (§6), §Impact Summary by Option (§7)
