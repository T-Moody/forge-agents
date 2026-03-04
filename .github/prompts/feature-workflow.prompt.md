---
name: Feature Development Pipeline
agent: orchestrator
---

# Feature Development Pipeline

## Initial Request

{{USER_FEATURE}}

## Pipeline Execution Rules

1. Execute the pipeline defined in `orchestrator.agent.md` All steps.
2. The orchestrator MUST immediately begin Step 0 on invocation and must NOT attempt to perform substantive work itself. It may create `initial-request.md` only as permitted in Step 0 and otherwise must restrict actions to coordination and evidence verification.
3. Follow `APPROVAL_MODE` for gate behavior — `autonomous` skips approval gates, `interactive` pauses after research and planning.
4. All agent outputs use typed YAML schemas defined in `schemas.md`.
5. The orchestrator validates every agent output before routing to the next step.
