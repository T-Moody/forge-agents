# Task 06: Planner Agent Definition

## Task Goal

Create `NewAgents/.github/agents/planner.agent.md` — the task decomposition agent with per-file risk classification and `relevant_context` pointers.

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following design.md §Decision 10 template
- Role: decompose design into implementation tasks with risk classification
- Input: `design-output.yaml`, `spec-output.yaml`, adversarial design review verdicts
- Output: `plan-output.yaml` (typed plan-output schema with `overall_risk_summary`), `plan.md` (human-readable), `tasks/*.yaml` (per-task typed schemas with `relevant_context` pointers)
- Risk classification system: per-file 🟢/🟡/🔴 with documented criteria
  - 🟢 Additive: new tests, docs, config, utilities
  - 🟡 Business Logic: modifying existing logic, signatures, queries
  - 🔴 Critical: auth/crypto, data deletion, schema migrations, concurrency, public API
- Task sizing: Standard (no 🔴) or Large (any 🔴 file in task)
- `relevant_context` mechanism: each task includes pointers to specific design/spec sections
- `overall_risk_summary` in plan output for orchestrator routing
- Completion contract: DONE | NEEDS_REVISION (replan mode) | ERROR
- Self-verification + anti-drift anchor

## Out-of-Scope

- Schema definition (in schemas.md)
- Implementation of tasks (implementer's job)
- Verification cascade (verifier's job)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/planner.agent.md`
2. Follows agent definition template with all required sections
3. References `plan-output` and `task-schema` schemas from schemas.md
4. Risk classification system fully documented with 🟢/🟡/🔴 criteria and examples
5. Task sizing rules documented (Standard vs Large escalation)
6. `relevant_context` pointer mechanism documented with example
7. `overall_risk_summary` field in plan output documented
8. Completion contract: DONE | NEEDS_REVISION | ERROR (3-state)
9. Replan mode workflow documented (receives verification findings, produces revised plan)
10. Self-verification and anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Grep for risk classification (🟢/🟡/🔴)
- Grep for relevant_context
- Grep for NEEDS_REVISION in completion contract
- Verify all template sections present

## Implementation Steps

1. Read design.md §Decision 2, Planner agent detail for inputs, outputs, tools, contract
2. Read design.md §Decision 8 for risk classification system (per-file 🟢/🟡/🔴, escalation rules)
3. Read design.md §Decision 5 for `relevant_context` pointer mechanism with example
4. Read design.md §Decision 3, Step 4 for planner dispatch flow
5. Read existing `.github/agents/planner.agent.md` for task decomposition patterns
6. Create `NewAgents/.github/agents/planner.agent.md` with full template structure
7. Self-verify all sections present

## Relevant Context from design.md

- §Decision 2 (lines ~205–215) — Planner: inputs, outputs, tools, contract
- §Decision 3, Step 4 (lines ~370–380) — Planner dispatch, risk classification output
- §Decision 5 (lines ~610–640) — relevant_context mechanism with example task schema
- §Decision 8 (lines ~980–1050) — Risk classification criteria, task sizing, escalation paths

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present (15 sections including all §Decision 10 template sections)
- [x] Risk classification (🟢/🟡/🔴) documented with criteria and examples
- [x] relevant_context mechanism explained with example and rules
- [x] Replan mode workflow present (5-step remediation process)
- [x] Schema references to schemas.md present (Schema 5 plan-output, Schema 6 task-schema)
- [x] Anti-drift anchor present

TDD skipped: configuration-only task (Markdown agent definition, no test framework applicable). Verified via grep checks for risk classification, relevant_context, NEEDS_REVISION, overall_risk_summary, and template section headings.
