# Memory: researcher-patterns

## Status

DONE: Completed patterns research covering memory architecture options (A/B/C), orchestrator purity, subagent input patterns, and anti-patterns.

## Key Findings

- Pattern B (eliminate shared memory.md) is lowest cost: all 19 subagents already have fallback logic to proceed without memory.md, and all explicitly list upstream isolated `.mem.md` files as primary inputs
- The orchestrator currently performs 10–20+ memory merge operations per run; Pattern A (merge subagent) would convert all of these to subagent dispatches, adding ~10K–30K tokens and significant latency
- Pattern C (subagents self-assemble) converges with Pattern B when the orchestrator provides relevance-filtered memory file paths rather than a full manifest
- The documentation-writer agent already operates in Pattern B mode — its memory-first rule skips memory.md and goes straight to upstream memories
- Prior decision in decisions.md already settled that `read_file` is acceptable for coordinators (read = observation, write = modification); the tool restriction should remove search/discovery tools, not read tools

## Highest Severity

N/A

## Decisions Made

None — research only, no design decisions.

## Artifact Index

- [research/patterns.md](../research/patterns.md)
  - §1.1 Current Memory Merge Points — quantified merge operations per pipeline run (10–20+)
  - §1.2 Pattern A — memory-merge subagent analysis with cost quantification and hypothetical prompt
  - §1.3 Pattern B — eliminate memory.md analysis with current codebase evidence (fallback logic, subagent input lists)
  - §1.4 Pattern C — self-assembly analysis with token cost comparison (~160K extra vs ~30K current)
  - §2 Orchestrator Purity — minimum tool set analysis, read_file justification, multi-agent shared state patterns
  - §3 Subagent Input Patterns — detailed analysis of spec, implementer, documentation-writer, ct-security agents
  - §4 Anti-Patterns — Pattern A waste analysis, Option A rejection rationale, over-restriction consequences
