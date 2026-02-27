# Memory: implementer-06

## Status

DONE: Created `NewAgents/.github/agents/planner.agent.md` — planner agent definition with per-file risk classification, relevant_context pointers, replan mode, and all template sections.

## Key Findings

- Planner is a Pipeline Agent at Step 4 with 3-state completion contract (DONE/NEEDS_REVISION/ERROR); NEEDS_REVISION is exclusive to replan mode
- Risk classification system has 3 levels (🟢 Additive, 🟡 Business Logic, 🔴 Critical) with per-file assignment and per-task escalation (any 🔴 → Large)
- `relevant_context` pointers use YAML fragment notation (`design-output.yaml#payload.decisions[id='D-8']`) to bound downstream agent reads
- `overall_risk_summary` is computed as the highest risk level across all tasks, used by orchestrator for routing
- Planner tool set is read+write only (no `run_in_terminal`, no code execution) — 7 tools total

## Highest Severity

N/A

## Decisions Made

- Included Plan Validation section with 4 checks (circular deps, task size, dep existence, risk consistency) — the 4th check (risk consistency) was added beyond what the existing Forge planner had, to enforce the 🔴→Large invariant.
- Structured Replan Mode as a separate workflow subsection (5 steps) rather than inline conditions, for clarity.

## Artifact Index

- [NewAgents/.github/agents/planner.agent.md](../../../NewAgents/.github/agents/planner.agent.md) — Full planner agent definition
  - §Role & Purpose — Agent identity and scope
  - §Input Schema — Primary, conditional, and reference inputs
  - §Output Schema — plan-output.yaml (Schema 5), tasks/\*.yaml (Schema 6), plan.md
  - §Risk Classification System — 🟢/🟡/🔴 criteria, classification rules, task sizing, overall_risk_summary
  - §Relevant Context Mechanism — Pointer format, rules, purpose
  - §Mode Detection — Initial, replan, extension modes
  - §Workflow — Initial mode (8 steps) and replan mode (5 steps)
  - §Completion Contract — DONE / NEEDS_REVISION / ERROR
  - §Self-Verification — 10-item checklist
  - §Anti-Drift Anchor — Identity reinforcement
