# Task 07: V Sub-Agents — Build + Tests

**Status:** complete

## Agent

implementer

## Depends On

none

## Description

Create two new Verification cluster sub-agent files: `v-build.agent.md` (the sequential gate agent) and `v-tests.agent.md` (test execution agent). V-Build inherits Steps 1-2 from the current verifier. V-Tests inherits Step 3. Both are parallel with respect to memory — they read `memory.md` but do NOT write to it.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §4.1 (V Sub-Agent Definitions), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — V-AC-1 through V-AC-10
- `NewAgentsAndPrompts/verifier.agent.md` — current file as structural reference for workflow

## Output File

- `NewAgentsAndPrompts/v-build.agent.md`
- `NewAgentsAndPrompts/v-tests.agent.md`

## Acceptance Criteria

- [x] 1. Both files follow v2 template: YAML frontmatter, role statement, inputs/outputs, 5 operating rules + Rule 6 (memory-first), workflow, output spec, completion contract, anti-drift anchor
- [x] 2. **v-build** inherits Detect Build System (Step 1) and Build (Step 2) from current verifier, including the build system detection table
- [x] 3. **v-build** outputs to `docs/feature/<feature-slug>/verification/v-build.md` containing: build system detected, build command, build status, errors/warnings, build artifact paths, environment details
- [x] 4. **v-build** completion contract: `DONE:` / `ERROR:` only (no `NEEDS_REVISION` — it either builds or doesn't)
- [x] 5. **v-build** captures ALL build state in `v-build.md` on disk — no reliance on terminal state or environment variables (V-AC-9)
- [x] 6. **v-tests** reads `v-build.md` for build context, runs full test suite, analyzes results
- [x] 7. **v-tests** outputs to `docs/feature/<feature-slug>/verification/v-tests.md` containing: test command, pass/fail/skip counts, failing test details with stack traces, cross-task integration issue flags
- [x] 8. **v-tests** completion contract: `DONE:` / `NEEDS_REVISION:` / `ERROR:`
- [x] 9. Both include read-only enforcement (MUST NOT modify source code)
- [x] 10. Both do NOT write to `memory.md` (parallel agents within the V cluster)

## Implementation Guidance

### v-build.agent.md

```
---
name: v-build
description: "Build system detection and compilation gate for the Verification cluster."
---
```

**Role:** Sequential gate agent for the V cluster. Detects the build system, runs the build command, and reports results. All downstream V sub-agents (v-tests, v-tasks, v-feature) depend on V-Build succeeding.

**Inputs:**

- `memory.md` (read first)
- Entire codebase

**Outputs:**

- `docs/feature/<feature-slug>/verification/v-build.md`

**Key workflow (inherited from verifier Steps 1-2):**

1. Read `memory.md`
2. Detect build system (same detection table as current verifier Step 1 — include the full table)
3. Run build command
4. Record all build errors/warnings with file paths and line numbers
5. Write `v-build.md` with: build system name/version, build command executed, build status (pass/fail), errors/warnings, build artifact paths, environment details (language version, OS, tool versions)

**Critical constraint:** `v-build.md` must contain ALL information downstream sub-agents need. No reliance on terminal state, environment variables, or shared process context.

### v-tests.agent.md

```
---
name: v-tests
description: "Full test suite execution and analysis for the Verification cluster."
---
```

**Role:** Runs the full test suite and analyzes results. Only runs after V-Build passes.

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/verification/v-build.md` (build context)
- Entire codebase

**Outputs:**

- `docs/feature/<feature-slug>/verification/v-tests.md`

**Key workflow (inherited from verifier Step 3):**

1. Read `memory.md`
2. Read `v-build.md` for build context and test command
3. Run full test suite (unit + integration + e2e)
4. Record pass/fail/skip counts and capture stack traces for failures
5. Flag previously-passing unit tests now failing (cross-task integration issue — high severity)
6. Write `v-tests.md`

**Output format (from design.md §4.1):**

```markdown
# Verification: Tests

## Status

## Results

## Issues Found

### [Severity] Issue Title

- **What / Where / Impact / Suggested Action**

## Cross-Cutting Observations
```

**Note:** These are markdown agent definition files — TDD does not apply. Create the files directly.
