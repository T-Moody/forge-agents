# Task 10: R Sub-Agents — Testing + Knowledge

## Agent

implementer

## Depends On

none

## Description

Create two new Review cluster sub-agent files: `r-testing.agent.md` and `r-knowledge.agent.md`. R-Testing inherits test quality review from the current reviewer. R-Knowledge is entirely new — it implements Knowledge Evolution (instruction suggestions, skill suggestions, pattern capture, decision log). R-Knowledge is non-blocking: its ERROR does not block the pipeline. Both read `memory.md` but do NOT write.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §5.1 (R Sub-Agent Definitions), §5.2 (Knowledge Evolution Agent Details), §5.3 (Safeguards KE-SAFE-1 through KE-SAFE-7), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — R-AC-1 through R-AC-11, KE-SAFE-1 through KE-SAFE-7
- `NewAgentsAndPrompts/reviewer.agent.md` — current file as reference for R-Testing

## Output File

- `NewAgentsAndPrompts/r-testing.agent.md`
- `NewAgentsAndPrompts/r-knowledge.agent.md`

## Acceptance Criteria

1. [x] Both files follow v2 template structure
2. [x] **r-testing** covers: test coverage adequacy, test quality (behavioral vs implementation testing), missing test scenarios, test data quality
3. [x] **r-testing** has completion contract: `DONE:` / `NEEDS_REVISION:` / `ERROR:`
4. [x] **r-knowledge** covers: instruction suggestions, skill suggestions, pattern capture, decision log updates
5. [x] **r-knowledge** has completion contract: `DONE:` / `ERROR:` only (no `NEEDS_REVISION` — knowledge findings are non-blocking)
6. [x] **r-knowledge** NEVER directly modifies agent definitions in `NewAgentsAndPrompts/` — all suggestions go to `knowledge-suggestions.md` (KE-SAFE-1)
7. [x] **r-knowledge** writes 3 output files: `review/r-knowledge.md`, `review/knowledge-suggestions.md`, `decisions.md` (append-only)
8. [x] **r-knowledge** includes WARNING header in `knowledge-suggestions.md` (KE-SAFE-5)
9. [x] **r-knowledge** implements safety constraint filter: MUST NOT suggest removing/weakening safety constraints, error handling, or verification steps (KE-SAFE-6)
10. [x] **r-knowledge** suggestion format includes: what to change, why, target file/section, diff-style before/after (KE-SAFE-2)
11. [x] Both do NOT write to `memory.md`
12. [x] Both output to `docs/feature/<feature-slug>/review/r-<focus>.md`

## Status

**Complete.** TDD skipped: markdown agent definition files — no behavioral code, no test framework applicable. Both files created per specification.

## Implementation Guidance

### r-testing.agent.md

```
---
name: r-testing
description: "Test quality and coverage adequacy review for the Review cluster."
---
```

**Role:** Reviews test quality, coverage, and identifies missing test scenarios. Focus on whether tests are meaningful (behavioral, not implementation-detail testing).

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/initial-request.md`
- `docs/feature/<feature-slug>/feature.md`
- Git diff
- Entire codebase

**Outputs:**

- `docs/feature/<feature-slug>/review/r-testing.md`

**Key review criteria:** test coverage adequacy, test patterns (behavioral vs implementation), missing test scenarios, test data quality, edge case coverage.

### r-knowledge.agent.md (NEW — no predecessor)

```
---
name: r-knowledge
description: "Knowledge evolution agent: captures reusable patterns, suggests instruction/skill updates, and maintains the architectural decision log. Suggestion-only — never auto-applies changes."
---
```

**Role:** Entirely new capability. Analyzes pipeline artifacts and implementation to identify improvement opportunities for Copilot instructions, skills, and workflow patterns. Writes suggestions only — NEVER directly modifies agent definitions.

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/initial-request.md`
- `docs/feature/<feature-slug>/feature.md`
- `docs/feature/<feature-slug>/design.md`
- `docs/feature/<feature-slug>/plan.md`
- `docs/feature/<feature-slug>/verifier.md`
- Git diff
- Entire codebase
- Existing `decisions.md` (if present)
- Existing `.github/instructions/` (if present)

**Outputs:**

- `docs/feature/<feature-slug>/review/r-knowledge.md` (analysis and rationale)
- `docs/feature/<feature-slug>/review/knowledge-suggestions.md` (actionable suggestions buffer)
- `docs/feature/<feature-slug>/decisions.md` (append-only; create if not exists)

**Workflow (from design.md §5.2):**

1. Read `memory.md`
2. Read all pipeline artifacts for full context
3. Examine git diff and codebase for implementation patterns
4. Read existing `decisions.md` if present
5. Read existing `.github/instructions/` if present
6. Identify instruction updates, skill updates, patterns, decisions
7. Apply safety filters (KE-SAFE-6): reject suggestions weakening safety
8. Write `review/r-knowledge.md`
9. Write `review/knowledge-suggestions.md` (with WARNING header)
10. Append to `decisions.md` if significant decisions found

**Safeguards (must be embedded in the agent definition):**

- KE-SAFE-1: File Boundaries rule explicitly lists only its output files
- KE-SAFE-2: Each suggestion requires what/why/file/diff format
- KE-SAFE-3: Categories: `instruction-update`, `skill-update`, `pattern-capture`, `workflow-improvement`
- KE-SAFE-4: No auto-apply (reinforce in anti-drift anchor)
- KE-SAFE-5: WARNING header on `knowledge-suggestions.md`
- KE-SAFE-6: Safety constraint filter in workflow step 7
- KE-SAFE-7: `decisions.md` append-only; verify existing entries unchanged

**knowledge-suggestions.md format (from design.md §5.1):**

```markdown
# Knowledge Evolution Suggestions

> **WARNING:** These are proposals only. Do NOT apply without human review.

## Suggestions

### 1. [Category] Title

- **Target file:** Path
- **Change type:** Add / Update / Remove
- **Rationale:** Why
- **Diff:** (before/after)
- **Risk assessment:** Low / Medium / High

## Rejected Suggestions

<!-- Filtered by KE-SAFE-6 -->
```

**Note:** This is the most complex sub-agent to create. R-Knowledge has no predecessor agent file — it is entirely new. Draw heavily from design.md §5.1 and §5.2.
