---
name: Feature Development Pipeline
agent: orchestrator
description: "Entry point for the multi-agent feature development pipeline. Requires USER_FEATURE variable. Optional APPROVAL_MODE (default: autonomous)."
---

# Feature Development Pipeline

## Initial Request

{{USER_FEATURE}}

## Execution Mode

{{APPROVAL_MODE}}

## Key Artifacts Reference

| Artifact             | Path                                           |
| -------------------- | ---------------------------------------------- |
| Orchestrator         | `.github/agents/orchestrator.agent.md`         |
| Schema reference     | `.github/agents/schemas.md`                    |
| Dispatch patterns    | `.github/agents/dispatch-patterns.md`          |
| Severity taxonomy    | `.github/agents/severity-taxonomy.md`          |
| Researcher           | `.github/agents/researcher.agent.md`           |
| Spec writer          | `.github/agents/spec.agent.md`                 |
| Designer             | `.github/agents/designer.agent.md`             |
| Planner              | `.github/agents/planner.agent.md`              |
| Implementer          | `.github/agents/implementer.agent.md`          |
| Verifier             | `.github/agents/verifier.agent.md`             |
| Adversarial reviewer | `.github/agents/adversarial-reviewer.agent.md` |
| Knowledge agent      | `.github/agents/knowledge-agent.agent.md`      |

## Pipeline Execution Rules

1. Execute the pipeline defined in `orchestrator.agent.md` (Steps 0–9).
2. Follow `APPROVAL_MODE` for gate behavior — `autonomous` skips approval gates, `interactive` pauses after research and planning.
3. All agent outputs use typed YAML schemas defined in `schemas.md`.
4. The orchestrator validates every agent output before routing to the next step.
