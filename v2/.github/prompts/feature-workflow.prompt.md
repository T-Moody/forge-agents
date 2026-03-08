---
name: Feature Development Pipeline
description: Full 8-step multi-agent pipeline for feature development
agent: orchestrator
---

# Feature Development Pipeline

## Feature Request

{{USER_FEATURE}}

## Pipeline Steps

Execute ALL 8 steps in order:

1. **Setup** — Initialize feature directory, git baseline tag, classify risk, generate run_id
2. **Research** — Dispatch up to 4 parallel researchers (skipped for green-risk features)
3. **Architecture** — Single architect produces combined spec + design output
4. **Planning** — Single planner decomposes into DAG task graph with file ownership
5. **Implementation** — Parallel implementers in DAG-ordered waves, strict TDD, unit tests only
6. **Testing** — Single tester runs static verification + dynamic tests based on risk level
7. **Code Review** — Parallel reviewers (security, architecture, correctness) — MANDATORY
8. **Completion** — Knowledge extraction, evidence bundle, pre-commit validation, git commit

## Startup Questions

Before beginning Step 1, ask the user:

1. **Approval mode** — `interactive` (pause at gates for approval) or `autonomous` (no pauses)?
2. **Web research** — Allow `fetch_webpage` for research agents? (yes/no)

## Execution Rules

- Follow the orchestrator agent definition for coordination and routing logic.
- Each step produces typed YAML output in the feature directory.
- The orchestrator validates each output before routing to the next step.
- Risk-based scaling applies: green skips Research, red adds embedded design review.
- Code Review (Step 7) is mandatory for ALL risk levels.
