# Task 07: Write Planner Agent

## Task Goal

Create the complete `planner.agent.md` file in `NewAgentsAndPrompts/`, rewriting the planner to include pre-mortem analysis, task size limits, mode detection, per-task agent field, planning principles, and plan validation.

**Output file:** `NewAgentsAndPrompts/planner.agent.md`

## depends_on

none

## Cluster

**Cluster B (Agent Routing) — Primary defining agent.** This task defines the `agent` field convention in task files that must be consistent with:

- Task 09 (orchestrator) — reads agent field, dispatches to named agent
- Task 10 (prompt) — documents agent routing rule

**Reference design.md § Per-Agent Design → 6. `planner.agent.md` for exact field specification.**

## In-Scope

- Rewrite of `planner.agent.md` following canonical structure
- Frontmatter with `name: planner` and updated description (adds "pre-mortem analysis, task size limits")
- Role preamble (3 sentences)
- `detailed thinking on` directive
- **Inputs** section (updated: adds verifier.md for replan mode, existing plan.md for extension mode)
- **Outputs** section (plan.md, tasks/\*.md)
- **Operating Rules** block (planner variant)
- **Planning Principles** (NEW — YAGNI, KISS, tangible value, minimize dependencies)
- **Mode Detection** (NEW — initial/replan/extension)
- **Workflow** (updated with mode detection, pre-mortem, task size validation)
- **Task Size Limits** (NEW — max 3 files, 2 dependencies, 500 lines production code, medium effort; test files/lines excluded)
- **Plan Validation** (NEW — circular dependency check, task size validation, dependency existence check)
- **Dependency-Aware Planning** (same structure as current)
- **Task File Requirements** (updated — adds `agent` field, updates test requirements for TDD)
- **Pre-Mortem Analysis** (NEW — per-task failure scenarios, overall risk, key assumptions)
- **plan.md Contents** (updated — adds pre-mortem section, optional implementation specification)
- **Task File Contents** (updated — adds agent field)
- **Completion Contract** (DONE/ERROR only)
- **Anti-Drift Anchor**

## Out-of-Scope

- Modifying files in `.github/`
- Adding NEEDS_REVISION (planner uses only DONE/ERROR)
- Changing the dependency graph format (same as current)

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/planner.agent.md` and is non-empty
2. Contains Pre-Mortem Analysis section with per-task failure table (Task, Failure Scenario, Likelihood, Impact, Mitigation) + overall risk level + key assumptions (TS-5)
3. Contains Task Size Limits table: 3 files, 2 dependencies, 500 lines, medium effort — with clarification that test code/files are excluded (TS-5)
4. Contains Mode Detection: initial (no plan.md), replan (verifier.md with failures), extension (existing plan.md) (TS-5)
5. Contains `agent` field in Task File Requirements: optional, default `implementer`, valid values include `documentation-writer` (TS-10)
6. Task File Requirements says "tests to write as part of TDD" (NOT "tests to write — not to run")
7. Contains Planning Principles (YAGNI, KISS, tangible value, minimize dependencies)
8. Contains Plan Validation section (circular dependency check, task size validation, dependency existence)
9. Completion contract is DONE: `<N> tasks created, <M> waves` / ERROR
10. Anti-drift anchor: "You are the **Planner**. You decompose work into tasks and execution waves. You never write code, tests, or documentation. You never implement tasks. Stay as planner."

## Test Requirements

1. **Planner completeness test (TS-5):** Contains pre-mortem, task size limits, mode detection
2. **Agent routing test (TS-10):** Defines `agent` field in task file format
3. **TDD alignment test:** Test requirements wording says "as part of TDD" not "not to run"
4. **Task size limits test:** Verify specific limits (3 files, 500 lines, 2 dependencies) with test exclusion
5. **Pre-mortem format test:** Verify per-task failure table format

## Implementation Steps

1. Read design.md § Per-Agent Design → 6. `planner.agent.md` for target state and ALL content specifications (this is the longest per-agent section)
2. Read design.md § Cross-Cutting Patterns for shared blocks
3. Create `NewAgentsAndPrompts/planner.agent.md` with:
   - YAML frontmatter
   - `# Planner Agent Workflow` title
   - Role preamble
   - `detailed thinking on` directive
   - `## Inputs` (updated with replan/extension mode inputs)
   - `## Outputs` (plan.md, tasks/\*.md)
   - `## Operating Rules` (planner variant)
   - `## Planning Principles` (NEW — from design.md)
   - `## Mode Detection` (NEW — from design.md)
   - `## Workflow` (updated)
   - `## Task Size Limits` (NEW — from design.md, including clarifications)
   - `## Plan Validation` (NEW — from design.md)
   - `## Dependency-Aware Planning` (Rules for Dependencies + Dependency Graph Format)
   - `## Task File Requirements` (updated — agent field, TDD test language)
   - `## Pre-Mortem Analysis` (NEW — from design.md)
   - `## plan.md Contents` (updated — adds pre-mortem, optional implementation spec)
   - `## Task File Contents` (updated — adds agent field)
   - `## Completion Contract` (DONE/ERROR)
   - `## Anti-Drift Anchor`

## Estimated Effort

Medium

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/planner.agent.md`
- [x] Mode Detection section present (initial/replan/extension)
- [x] Task Size Limits with specific numbers and test exclusion clarification
- [x] Plan Validation section (circular deps, size validation, dep existence)
- [x] Pre-Mortem Analysis with per-task failure table format
- [x] Planning Principles (YAGNI, KISS, tangible value, minimize deps)
- [x] `agent` field in Task File Requirements (optional, default implementer)
- [x] Test requirements wording updated for TDD
- [x] Implementation Specification marked as optional in plan.md Contents
- [x] Completion contract is two-state (DONE/ERROR)
- [x] Anti-drift anchor with exact text
- [x] Cluster B: `agent` field name consistent with design.md
