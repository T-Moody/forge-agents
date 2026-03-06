---
name: Feature Development Pipeline
agent: orchestrator
---

# Feature Development Pipeline

## Initial Request

{{USER_FEATURE}}

## Pipeline Execution Rules

1. Execute the pipeline defined in `orchestrator.agent.md` All steps.
2. Follow `APPROVAL_MODE` for gate behavior — `autonomous` skips approval gates, `interactive` waits for user approval at each gate.
3. All agent outputs use typed YAML schemas defined in `schemas.md`.
4. The orchestrator validates every agent output before routing to the next step.
