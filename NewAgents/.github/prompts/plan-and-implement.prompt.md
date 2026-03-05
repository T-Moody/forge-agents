---
name: Plan and Implement (Fast-Track Pipeline)
agent: orchestrator
---

# Plan and Implement — Fast-Track Pipeline

## Pipeline Mode

`pipeline_mode: fast-track`

## Initial Request

{{USER_FEATURE}}

## Scope Guardrails

This fast-track prompt is for **small, low-risk (🟢) features only**. Use the full `feature-workflow.prompt.md` for any feature that:

- Requires research or external documentation lookup
- Has 🟡 or 🔴 risk files
- Needs architectural design decisions
- Spans more than ~5 files

## Pipeline Execution Rules

1. Execute the pipeline defined in `orchestrator.agent.md` with `pipeline_mode: fast-track`.
2. Execute **only** the following steps in order:
   - **Step 0** — Initialization (setup DB, detect pipeline mode)
   - **Step 4** — Planning (planner creates tasks from initial request; uses `ask_questions` in interactive mode for clarification)
   - **Step 5** — Implementation (implementer executes tasks)
   - **Step 6** — Verification (verifier validates implementation)
   - **Step 7** — Code Review (adversarial review — **mandatory, never skip**)
   - **Step 8** — Knowledge Capture
   - **Step 9** — Auto-Commit (**mandatory, never skip**)
3. Steps 1–3b (research, spec, design, design review) are **skipped**.
4. Follow `APPROVAL_MODE` for gate behavior — `autonomous` skips approval gates, `interactive` pauses after planning. If `APPROVAL_MODE` is not specified, default to `interactive`.
5. All agent outputs use typed YAML schemas defined in `schemas.md`.
6. The orchestrator validates every agent output before routing to the next step.

## Minimum Required Steps (Immutable)

Per the orchestrator anti-drift anchor, **Step 0**, **Step 7**, and **Step 9** are immutable minimum steps that cannot be skipped by any pipeline prompt. This fast-track prompt includes all three.
