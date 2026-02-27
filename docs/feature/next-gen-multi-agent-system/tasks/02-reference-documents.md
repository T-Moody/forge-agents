# Task 02: Reference Documents (Dispatch Patterns + Severity Taxonomy)

## Task Goal

Create `NewAgents/.github/agents/dispatch-patterns.md` and `NewAgents/.github/agents/severity-taxonomy.md` — two shared reference documents used by all agents.

## depends_on

none

## agent

implementer

## In-Scope

### dispatch-patterns.md

- Pattern A (Fully Parallel): definition, gate condition (≥N of M DONE), retry policy, examples (researchers ×4, reviewers ×3)
- Pattern B (Sequential with Replan Loop): definition, max iterations, NEEDS_REVISION routing, examples (Steps 5–6 implementation-verification loop)
- Concurrency cap: max 4 concurrent subagents per wave
- Sub-wave partitioning: how to split >4 tasks into sub-waves of ≤4
- Pattern usage table: which pipeline steps use which pattern

### severity-taxonomy.md

- Unified severity levels: Blocker, Critical, Major, Minor — with precise definitions
- Severity-driven behavior: what each severity level triggers in the pipeline
- Security blocker policy: any Blocker from any model → immediate pipeline ERROR
- Severity field usage: which agents produce severity values, where they appear in schemas

## Out-of-Scope

- Schema definitions (Task 01)
- Agent definitions (Tasks 03–11)
- Risk classification system (🟢/🟡/🔴) — defined in planner agent, not severity taxonomy

## Acceptance Criteria

1. `dispatch-patterns.md` exists at `NewAgents/.github/agents/dispatch-patterns.md`
2. `severity-taxonomy.md` exists at `NewAgents/.github/agents/severity-taxonomy.md`
3. Pattern A and Pattern B formally defined with gate conditions, retry budgets, concurrency limits
4. Concurrency cap of 4 documented with sub-wave partitioning rules
5. All 4 severity levels (Blocker/Critical/Major/Minor) defined with precise criteria and pipeline impact
6. Security blocker policy explicitly stated: any Blocker → pipeline halt (no retry, no deferral)
7. Both documents follow consistent Markdown structure with clear section headers

## Estimated Effort

Low

## Test Requirements

- Verify Pattern A matches design.md §Decision 3 parallelism map
- Verify severity definitions match design.md §Decision 8 + feature.md CR-7
- Verify security blocker policy matches design.md §Decision 7

## Implementation Steps

1. Read design.md §Decision 3 (Pipeline Structure) for Pattern A/B definitions, parallelism map, concurrency cap
2. Read existing `.github/agents/dispatch-patterns.md` for formatting pattern and Pattern A/B/C definitions
3. Read design.md §Decision 8 (Risk & Escalation) for severity levels, escalation paths
4. Read design.md §Decision 7 (Adversarial Review) for security blocker policy, disagreement resolution
5. Read feature.md CR-7 (unified severity taxonomy requirement)
6. Create `NewAgents/.github/agents/dispatch-patterns.md` with:
   - Pattern A: Fully Parallel (definition, gate, retry, examples)
   - Pattern B: Sequential with Replan Loop (definition, max iterations, routing)
   - Concurrency rules and sub-wave partitioning
   - Pattern usage by pipeline step
7. Create `NewAgents/.github/agents/severity-taxonomy.md` with:
   - 4 severity levels with definitions
   - Pipeline impact per severity level
   - Security blocker policy
   - Severity usage by agent
8. Self-verify both documents for completeness

## Relevant Context from design.md

- §Decision 3 (lines ~310–475) — Pipeline steps, parallelism map, gate points, dispatch count
- §Decision 7 (lines ~880–980) — Security blocker policy, review cycling, disagreement resolution
- §Decision 8 (lines ~980–1050) — Risk classification, severity, escalation paths, retry budgets

## Completion Checklist

- [x] dispatch-patterns.md created at correct path
- [x] severity-taxonomy.md created at correct path
- [x] Pattern A and B formally defined
- [x] Concurrency cap (4) documented
- [x] All 4 severity levels defined
- [x] Security blocker policy present

## Status

**DONE** — Implemented by implementer agent. Both reference documents created at `NewAgents/.github/agents/`. TDD skipped: configuration/documentation-only task (no behavioral code).
