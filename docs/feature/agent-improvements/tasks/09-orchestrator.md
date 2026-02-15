# Task 09: Write Orchestrator Agent

## Task Goal

Create the complete `orchestrator.agent.md` file in `NewAgentsAndPrompts/`, rewriting the orchestrator — the most complex agent — to incorporate TDD global rule update, concurrency cap, per-task agent routing, optional human approval gates, three-state completion handling, decisions.md awareness, and all cross-cutting patterns.

**Output file:** `NewAgentsAndPrompts/orchestrator.agent.md`

## depends_on

05, 06, 07, 08

## Cluster

**Touches ALL three clusters:**

- **Cluster A (TDD):** Global Rule 6 must say "Implementers perform unit-level TDD; verifier does integration-level verification" — matching implementer (Task 05) and verifier (Task 06)
- **Cluster B (Agent Routing):** Global Rule 8 + Step 5.2 must read the `agent` field from task files — matching planner (Task 07) field convention
- **Cluster C (Approval Gates):** Global Rule 9 + Steps 1.2a, 4a must implement conditional APPROVAL_MODE — matching prompt (Task 10)

**This is the LAST agent written** to ensure all other agents' contracts and patterns are finalized.

## In-Scope

- Rewrite of `orchestrator.agent.md` following canonical structure (with orchestrator-specific extensions)
- Frontmatter with `name: orchestrator` and description
- Role preamble (3 sentences)
- `detailed thinking on` directive
- **Inputs** section (user feature request via prompt)
- **Outputs** section (initial-request.md + coordination)
- **Global Rules** (10 rules — REWRITTEN):
  1. Never modify code directly
  2. Always pass explicit file paths
  3. Require DONE/NEEDS_REVISION/ERROR from every subagent (three-state)
  4. Retry ERROR once; route NEEDS_REVISION to responsible agent
  5. Always use custom agents
  6. **Implementers perform unit-level TDD; verifier does integration-level verification** (Cluster A)
  7. **Max 4 concurrent subagent invocations** with sub-wave splitting
  8. **Per-task agent routing** via `agent` field with valid_task_agents list (Cluster B)
  9. **APPROVAL_MODE** optional gates, marked experimental with fallback (Cluster C)
  10. Display subagent and step
- **Documentation Structure** (updated: + decisions.md + design_critical_review.md)
- **Operating Rules** block (orchestrator variant, Rule 5: runSubagent only)
- **Workflow Steps** (Steps 0-7 updated):
  - Step 1.2a: Human Approval Gate Post-Research (NEW, conditional)
  - Step 3b: Design Review with NEEDS_REVISION → Designer loop (max 1)
  - Step 4a: Human Approval Gate Post-Planning (NEW, conditional)
  - Step 5.2: Execute Each Wave (REWRITTEN: sub-wave splitting + agent routing + valid agent validation)
  - Step 7: Final Review with NEEDS_REVISION → Implementer lightweight fix loop
- **Parallel Execution Summary** (updated with sub-wave notation + approval gates)
- **Completion Contract** (orchestrator doesn't return DONE/NEEDS_REVISION itself — handles subagent responses)
- **Anti-Drift Anchor**

## Out-of-Scope

- Modifying files in `.github/`
- Writing code, tests, or documentation directly
- Changing the fundamental pipeline steps (8-step pipeline preserved)

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/orchestrator.agent.md` and is non-empty
2. **Cluster A (TS-3):** Global Rule 6 states "Implementers perform unit-level TDD" and "verifier performs integration-level verification"
3. **Cluster A (TS-3):** Does NOT contain "Implementers never build or run tests" (old rule removed)
4. **Concurrency (TS-4):** Global Rule 7 specifies max 4 concurrent, sub-wave splitting for >4 tasks
5. **Agent Routing (TS-10):** Global Rule 8 specifies reading `agent` field, valid_task_agents list (`implementer`, `documentation-writer`), default to implementer
6. **Agent Routing (TS-10):** Step 5.2 includes agent field reading and dispatch logic
7. **Approval Gates (TS-11):** Global Rule 9 includes APPROVAL_MODE, marked experimental, with fallback behavior
8. **Approval Gates (TS-11):** Steps 1.2a and 4a exist with conditional pause logic
9. **Three-state:** Global Rule 3 expects DONE/NEEDS_REVISION/ERROR from subagents
10. **Three-state:** Global Rule 4 retries ERROR, routes NEEDS_REVISION (does NOT retry it)
11. **NEEDS_REVISION routing:** Critical-thinker → Designer (max 1 loop); Reviewer → Implementer(s) (max 1 loop); Verifier → Planner (up to 3 iterations)
12. Documentation Structure includes decisions.md and design_critical_review.md
13. Operating Rules present with orchestrator-specific Rule 5 (runSubagent only)
14. Parallel Execution Summary updated with sub-wave notation and approval gate markers
15. Anti-drift anchor: "You are the **Orchestrator**. You coordinate agents via runSubagent. You never write code, tests, or documentation directly. You never skip pipeline steps. Stay as orchestrator."

## Test Requirements

1. **TDD cluster test (TS-3):** Cross-check Global Rule 6 against implementer and verifier files
2. **Concurrency test (TS-4):** Verify max 4 concurrent + sub-wave splitting
3. **Agent routing test (TS-10):** Verify agent field reading + valid_task_agents + default behavior
4. **Approval gates test (TS-11):** Verify APPROVAL_MODE conditional logic + autonomous default
5. **Three-state test (TS-9):** Verify DONE/NEEDS_REVISION/ERROR handling
6. **Cross-agent consistency (TS-14):** Verify no contradictions with other agents' completion contracts
7. **Routing table test:** Verify NEEDS_REVISION routing matches design.md Pattern 3 routing table

## Implementation Steps

1. Read design.md § Per-Agent Design → 1. `orchestrator.agent.md` — the most detailed per-agent section
2. Read design.md § Cross-Cutting Patterns (all 5 patterns)
3. Read design.md § Agent Communication Contracts for:
   - Three-state completion protocol
   - NEEDS_REVISION routing table
   - Orchestrator expectations per agent table
4. **Cross-reference cluster consistency:**
   - Verify TDD terminology matches implementer (Task 05) and verifier (Task 06)
   - Verify `agent` field convention matches planner (Task 07)
   - Verify APPROVAL_MODE format for prompt consistency (Task 10)
5. Create `NewAgentsAndPrompts/orchestrator.agent.md` with:
   - YAML frontmatter
   - `# Orchestrator Agent Workflow` title
   - Role preamble
   - `detailed thinking on` directive
   - `## Inputs`
   - `## Outputs`
   - `## Global Rules` (10 rules — exact content from design.md)
   - `## Documentation Structure` (updated)
   - `## Operating Rules` (orchestrator variant)
   - `## Workflow Steps` with ALL sub-steps:
     - `### 0. Setup`
     - `### 1. Research (Parallel)` with 1.1, 1.2, 1.2a
     - `### 2. Specification`
     - `### 3. Design`
     - `### 3b. Design Review` (with NEEDS_REVISION → Designer loop)
     - `### 4. Planning` with 4a
     - `### 5. Implementation Loop` with 5.1, 5.2 (REWRITTEN), 5.3
     - `### 6. Verification` (with NEEDS_REVISION → Planner loop)
     - `### 7. Final Review` (with NEEDS_REVISION → Implementer loop)
   - `## Parallel Execution Summary` (updated)
   - `## Completion Contract`
   - `## Anti-Drift Anchor`

## Estimated Effort

Medium

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/orchestrator.agent.md`
- [x] Global Rule 6: TDD terminology matches implementer/verifier (Cluster A)
- [x] Global Rule 7: Max 4 concurrent with sub-wave splitting
- [x] Global Rule 8: Agent routing with valid_task_agents list (Cluster B)
- [x] Global Rule 9: APPROVAL_MODE experimental with fallback (Cluster C)
- [x] Global Rules 3-4: Three-state completion handling
- [x] Steps 1.2a and 4a: Conditional approval gates
- [x] Step 5.2: Sub-wave splitting + agent routing logic
- [x] Step 3b: NEEDS_REVISION → Designer loop (max 1)
- [x] Step 7: NEEDS_REVISION → Implementer loop (max 1, then escalate)
- [x] Documentation Structure includes decisions.md + design_critical_review.md
- [x] No old rule "Implementers never build or run tests"
- [x] Parallel Execution Summary updated
- [x] Anti-drift anchor with exact text
- [x] Operating Rules block present
