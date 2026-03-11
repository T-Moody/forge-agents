---
name: Quick Fix Pipeline
description: Simplified pipeline for trivial fixes — skips Research and Architecture
agent: orchestrator
---

# Quick Fix Pipeline

## Feature Request

{{USER_FEATURE}}

## Pipeline Steps

Execute ONLY these 6 steps in order:

1. **Setup** (Step 1) — Initialize feature directory, git baseline tag, classify risk, generate run_id
2. **Planning** (Step 4) — Single planner produces minimal task DAG
3. **Implementation** (Step 5) — Implement changes with strict TDD, unit tests only
4. **Testing** (Step 6) — Static verification + dynamic tests based on risk level
5. **Code Review** (Step 7) — Parallel reviewers — MANDATORY, NOT SKIPPABLE
6. **Completion** (Step 8) — Knowledge extraction, evidence bundle, pre-commit validation, git commit

Steps skipped: Research (Step 2), Architecture (Step 3).

## Scope Guardrails

This pipeline is for **trivial, low-risk fixes only**. Use the full `feature-workflow.prompt.md` for any feature that:

- Requires research or external documentation lookup
- Has medium or high risk files
- Needs architectural design decisions
- Spans more than a few files

## Execution Rules

- Code Review (Step 7) is **mandatory and not skippable** for all pipeline modes.
- The orchestrator validates each step output before routing to the next step.
- Follow the orchestrator agent definition for coordination and routing logic.
