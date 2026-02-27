# Memory: researcher-architecture

## Status

DONE: Comprehensive architecture analysis of Forge (23-agent pipeline) and Anvil (single evidence-first agent) systems completed.

## Key Findings

- Forge is a deterministic 8-step pipeline with 23 agents (4 clusters: Research ×4, CT ×4, V ×4, R ×4 + sequential agents + PostMortem), using file-based communication, three-state completion contracts, and three dispatch patterns (A: fully parallel, B: sequential gate + parallel, C: replan loop max 3 iterations)
- Forge uses a dual-layer memory system: shared `memory.md` (orchestrator sole writer, pruned at checkpoints) + isolated `memory/<agent>.mem.md` per agent; memory-first reading pattern reduces context window usage
- Anvil is a single 419-line monolithic agent with a 12-phase loop (Boost → Git Hygiene → Understand → Recall → Survey → Plan → Baseline → Implement → Verify → Learn → Present → Commit), using SQL verification ledger (`anvil_checks`) with three phases (baseline/after/review) and defense-in-depth 3-tier verification cascade
- Anvil's adversarial review scales by risk: 1 reviewer for Medium, 3 parallel multi-model reviewers for Large/🔴 files; risk classification (🟢/🟡/🔴) auto-escalates 🔴 files to Large; pushback system halts execution for questionable requests
- Both systems enforce strict file boundaries per agent, use anti-drift anchors, and have explicit completion contracts; Forge uses Markdown files at deterministic paths while Anvil uses SQL for verification state

## Highest Severity

N/A

## Artifact Index

- research/architecture.md — §Repository Structure (full directory layout), §Forge Orchestrator Architecture (8-step pipeline, agent taxonomy, dispatch patterns, completion contracts, cluster decision logic), §Forge Memory Architecture (dual-layer memory, lifecycle, memory-first reading), §Anvil Agent Architecture (12-phase loop, task sizing, verification cascade, adversarial review, pushback, session/recall, evidence bundle), §Documentation Structure (Forge pipeline output layout), §Architectural Patterns (communication, state management, gating, parallelism, error handling, evaluation system, agent file structure)
