---
name: Feature Development Pipeline
agent: orchestrator
---

# Feature Development Pipeline

## Initial Request

{{USER_FEATURE}}

YOU MUST START AT STEP 0. Do not do your own research or make assumptions about the feature. Follow the steps in order. If you need more information, use subagents for research and ask the user for clarification using the ask tool with multiple choice questions.

## Pipeline Execution Rules

1. Execute the pipeline defined in `orchestrator.agent.md` All steps.
2. Follow `APPROVAL_MODE` for gate behavior — `autonomous` skips approval gates, `interactive` waits for user approval at each gate.
3. All agent outputs use typed YAML schemas defined in `schemas.md`.
4. The orchestrator validates every agent output before routing to the next step.
