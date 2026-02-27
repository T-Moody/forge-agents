# Initial Request: Next-Gen Multi-Agent System Architecture

## Summary

Design a next-generation, deterministic, reliable multi-agent system for GitHub Copilot in VS Code by deeply analyzing and merging two existing systems (Forge Orchestrator and Anvil Agent) along with failure-prevention principles from a GitHub engineering blog post.

## Context

- **Forge Orchestrator**: A multi-agent pipeline system at `.github/` with 23 agent definitions, 8-step pipeline, memory system, cluster dispatch patterns (CT, V, R), artifact evaluations, and telemetry
- **Anvil Agent**: A single evidence-first coding agent at `Anvil/anvil.agent.md` focused on adversarial multi-model review, SQL-tracked verification ledger, risk classification, and pushback system
- **Article**: "Multi-agent workflows often fail. Here's how to engineer ones that don't" - GitHub Blog (Feb 2026)

## Article Key Principles

1. **Typed schemas**: Natural language is messy; typed schemas make inter-agent communication reliable. Treat schema violations like contract failures: retry, repair, or escalate.
2. **Action schemas**: Vague intent breaks agents; constrain every agent's output to an explicit, finite set of allowed actions. Agents must return exactly one valid action.
3. **MCP enforcement**: Loose interfaces create errors; use enforcement layers (like MCP) to turn patterns into contracts with explicit input/output schemas validated before execution.
4. **Design principles**: Design for failure first, validate every agent boundary, constrain actions before adding more agents, log intermediate state, expect retries and partial failures, treat agents like distributed systems not chat flows.

## Requirements

- Determine optimal architecture: may merge Anvil into Forge, Forge into Anvil, replace components, remove agents, introduce new ones, or restructure from scratch
- Must be deterministic and reliable
- Must support parallel subagents where beneficial
- Must use subagents strategically, not excessively
- Must eliminate unnecessary orchestration overhead
- Remove "pending step" pattern from Forge
- Retain and extend approval-mode (fully autonomous + interactive with structured multiple-choice prompts)
- Memory system must be audited and potentially redesigned (minimize unnecessary merge cost)
- Determine: what to persist and why, transactional vs ephemeral storage, deterministic agent communication, gating and enforcement, risk-based escalation, subagent structure, when parallelization helps vs harms, appropriate data format per responsibility (SQL, JSON, Markdown, etc.)

## Specific Anvil Features to Incorporate

1. Anvil-style adversarial review as a phase (probably after design)
2. For **Large OR 🔴 files**: Three reviewers in parallel (same prompt) using suggested models
3. Evidence gating before final recommendation
4. Justification scoring for architectural decisions
5. Risk classification of proposed structures
6. Force risk classification for all proposed structures

## Constraints

- System correctness > human-readable files
- Optimize for robustness, determinism, auditability, and long-term scalability
- Do not optimize for familiarity
- Output to `X:\programProjects\OrchestratorAgents\NewAgents\.github` for agents and prompts
- Create a README with final mermaid diagram in `X:\programProjects\OrchestratorAgents\NewAgents`

## Source Files

- Forge agents: `X:\programProjects\OrchestratorAgents\.github\agents\` (23 agent files)
- Forge prompts: `X:\programProjects\OrchestratorAgents\.github\prompts\`
- Anvil agent: `X:\programProjects\OrchestratorAgents\Anvil\anvil.agent.md`
