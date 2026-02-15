# Initial Request: Improve Orchestrator and All Related Agents

## User Request

Improve the Forge orchestrator and all of its related agents and prompts by incorporating the best practices and strengths identified from the Gem Team multi-agent framework.

## Context

- The Forge framework currently has 9 agents: orchestrator, researcher, spec, designer, critical-thinker, planner, implementer, verifier, reviewer
- All agent definitions live in `.github/agents/` as `.agent.md` files
- The workflow prompt lives in `.github/prompts/feature-workflow.prompt.md`
- Prior research has been done comparing Forge to Gem Team (see `docs/comparison-forge-vs-gem-team.md` and `docs/optimization-from-gem-team.md`)
- The prior research should NOT be taken at face value — researchers must independently analyze each Gem agent for additional improvements or things missed

## External Resources (Gem Team)

- Repository: https://github.com/mubaidr/gem-team
- Agent files:
  - https://github.com/mubaidr/gem-team/blob/main/gem-chrome-tester.agent.md
  - https://github.com/mubaidr/gem-team/blob/main/gem-devops.agent.md
  - https://github.com/mubaidr/gem-team/blob/main/gem-documentation-writer.agent.md
  - https://github.com/mubaidr/gem-team/blob/main/gem-implementer.agent.md
  - https://github.com/mubaidr/gem-team/blob/main/gem-orchestrator.agent.md
  - https://github.com/mubaidr/gem-team/blob/main/gem-planner.agent.md
  - https://github.com/mubaidr/gem-team/blob/main/gem-researcher.agent.md
  - https://github.com/mubaidr/gem-team/blob/main/gem-reviewer.agent.md

## Constraints

- Output all new/improved agents into `X:\programProjects\OrchestratorAgents\NewAgentsAndPrompts\`
- Do NOT modify files in `.github/` — those are the active agents orchestrating this work
- The orchestrator workflow is primarily used for software development projects
- **Full permission to alter any Forge agent in any way that makes it better** — no sacred cows, no architecture constraints. If Gem's approach is superior in an area, adopt it fully. If Forge's approach needs a complete rewrite, do it. The goal is the best possible agent package, period.
- Do whatever produces the highest-quality, most robust, most effective agent system — even if that means fundamental changes to how agents are structured, how they communicate, what they enforce, or what stages exist in the pipeline

## Scope of Improvements

Based on prior analysis, key areas to investigate and potentially implement:

1. **Pre-mortem analysis** in the planner (P0)
2. **TDD enforcement** in the implementer (P0)
3. **Task size limits** in the planner (P1)
4. **Concurrency cap** in the orchestrator (P1)
5. **Security-focused review** in the reviewer (P1)
6. **Hybrid retrieval strategy** in the researcher (P1)
7. **Per-task agent routing** in the planner/orchestrator (P2)
8. **Cross-agent memory / decision log** convention (P2)
9. **Optional human approval gates** (P2)
10. **Self-reflection steps** for agents (P3)

Researchers should verify these priorities, identify anything missed, and potentially discover additional improvements from deep analysis of each Gem agent file.

## Deliverables

All improved agent files and prompts output to `NewAgentsAndPrompts/`:

- `orchestrator.agent.md`
- `researcher.agent.md`
- `spec.agent.md`
- `designer.agent.md`
- `critical-thinker.agent.md`
- `planner.agent.md`
- `implementer.agent.md`
- `verifier.agent.md`
- `reviewer.agent.md`
- `feature-workflow.prompt.md`
- Any new agent files if warranted (e.g., documentation-writer)
