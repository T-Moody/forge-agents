# Task 05: Write Implementer Agent

## Task Goal

Create the complete `implementer.agent.md` file in `NewAgentsAndPrompts/`, performing a **full rewrite** — the most significantly changed agent. Replaces the "never build or test" workflow with TDD enforcement, adds security rules, code quality principles, and self-reflection.

**Output file:** `NewAgentsAndPrompts/implementer.agent.md`

## depends_on

none

## Cluster

**Cluster A (TDD) — Primary defining agent.** This task defines the TDD terminology and workflow that must be consistent with:

- Task 06 (verifier) — must agree on "unit-level TDD" vs "integration-level verification"
- Task 09 (orchestrator) — must agree on global rule "Implementers perform unit-level TDD"

**Reference design.md § Per-Agent Design → 7. `implementer.agent.md` and § Cross-Cutting Patterns for canonical terminology.**

## In-Scope

- **Full rewrite** of `implementer.agent.md` — removes all "Do NOT build/test" rules, replaces with TDD workflow
- Frontmatter with `name: implementer` and updated description (TDD-focused)
- Role preamble (3 sentences — TDD focus, NEVER read plan.md, NEVER skip TDD)
- `detailed thinking on` directive with experimental comment
- **Inputs (STRICT)** section (task file, feature.md, design.md; MUST NOT read plan.md)
- **Outputs** section (code files, test files, updated task file)
- **Operating Rules** block (implementer variant, Rule 5: multi_replace_string_in_file, get_errors, list_code_usages)
- **Code Quality Principles** (NEW — YAGNI, KISS, DRY, list_code_usages before refactoring)
- **Security Rules** (NEW — no hardcoded secrets, no PII, fix in-scope vulnerabilities)
- **TDD Workflow** (NEW — 7 steps: understand → write failing tests → write code → verify → refactor → update task → self-reflect)
- **TDD Fallback** (NEW — detection heuristics for test framework availability, non-testable task detection, fallback procedure)
- **Rules** (updated — old "no build/test" removed)
- **Completion Contract** (DONE/ERROR only — implementer does NOT use NEEDS_REVISION)
- **Anti-Drift Anchor**

## Out-of-Scope

- Modifying files in `.github/`
- Adding NEEDS_REVISION (implementer uses only DONE/ERROR)
- Comprehensive test suites (implementer writes focused unit tests, not exhaustive suites)

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/implementer.agent.md` and is non-empty
2. **Does NOT contain** "Do NOT build" or "Do NOT run tests" (old rules removed)
3. Contains TDD Workflow with 7 steps: Understand → Write Failing Tests → Write Production Code → Verify → Refactor → Update Task → Self-Reflection
4. TDD Workflow step 2 requires writing tests first and confirming they FAIL
5. TDD Workflow step 3 requires `get_errors` after **every** file edit
6. TDD Workflow step 4 requires confirming tests PASS
7. Contains TDD Fallback section with:
   - Detection heuristics per ecosystem (JavaScript, Python, .NET, Java, Rust, Go)
   - Non-testable task detection (config-only, infrastructure, static assets)
   - Fallback procedure (note in task file, use get_errors, flag untested code)
8. Contains Code Quality Principles (YAGNI, KISS, DRY, list_code_usages/grep_search fallback)
9. Contains Security Rules (no secrets, no PII, scope-aware vulnerability fixing)
10. Self-reflection step is conditional: Medium/High effort tasks only, skip for Low
11. Still prohibits reading plan.md
12. Anti-drift anchor: "You are the **Implementer**. You write code and tests for exactly one task. You never modify other tasks' files. You never skip TDD steps. You never modify plan.md. Stay as implementer."

## Test Requirements

1. **TDD enforcement test (TS-3):** File contains TDD workflow (write test, confirm fail, write code, confirm pass)
2. **Old rules removed test (TS-3):** File does NOT contain "Do NOT build" or "Do NOT run tests"
3. **get_errors test (TS-3):** File requires get_errors after file edits
4. **Security rules test (TS-6):** File contains rules about secrets, API keys, PII
5. **TDD fallback test:** File includes fallback for non-testable contexts
6. **Cluster A consistency:** Terminology says "unit-level TDD" (not just "tests") — matches verifier "integration-level" and orchestrator global rule

## Implementation Steps

1. Read design.md § Per-Agent Design → 7. `implementer.agent.md` — marked as MAJOR REWRITE with complete content specifications for every section
2. Read design.md § Cross-Cutting Patterns for shared blocks
3. **DO NOT reference the current implementer file for workflow** — the design specifies a complete rewrite
4. Create `NewAgentsAndPrompts/implementer.agent.md` with:
   - YAML frontmatter (`name: implementer`, TDD-focused description)
   - `# Implementer Agent Workflow` title
   - Role preamble (TDD focus, from design.md)
   - `detailed thinking on` directive
   - `## Inputs (STRICT)` (task file, feature.md, design.md; MUST NOT read plan.md)
   - `## Outputs` (code files, test files, updated task file)
   - `## Operating Rules` (implementer variant — multi_replace_string_in_file, get_errors, list_code_usages)
   - `## Code Quality Principles` (YAGNI, KISS, DRY, impact check)
   - `## Security Rules` (no secrets, no PII, scope-aware fixes)
   - `## TDD Workflow` (7-step from design.md — exact content)
   - `## TDD Fallback` (detection heuristics, non-testable detection, fallback procedure — exact content from design.md)
   - `## Rules` (updated — no old prohibitions)
   - `## Completion Contract` (DONE: task-id / ERROR: reason)
   - `## Anti-Drift Anchor`

## Estimated Effort

Medium

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/implementer.agent.md`
- [x] Full rewrite — NO "Do NOT build" or "Do NOT run tests" rules remain
- [x] TDD Workflow present with all 7 steps
- [x] get_errors required after every file edit
- [x] TDD Fallback with ecosystem-specific detection heuristics
- [x] Security Rules section (secrets, PII, vulnerabilities)
- [x] Code Quality Principles (YAGNI, KISS, DRY, list_code_usages)
- [x] Self-reflection conditional on effort level
- [x] plan.md still prohibited from reading
- [x] Completion contract is two-state (DONE/ERROR)
- [x] Anti-drift anchor with exact text
- [x] Cluster A terminology: "unit-level TDD" used consistently
