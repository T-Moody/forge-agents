# Task 09: R Sub-Agents — Quality + Security

## Status

DONE

## Agent

implementer

## Depends On

none

## Description

Create two new Review cluster sub-agent files: `r-quality.agent.md` and `r-security.agent.md`. R-Quality inherits the code review workflow from the current reviewer. R-Security inherits the security review capabilities. Both are parallel agents that read `memory.md` but do NOT write to it. R-Security has a special override: its ERROR or critical NEEDS_REVISION blocks the entire pipeline.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §5.1 (R Sub-Agent Definitions), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — R-AC-1 through R-AC-11
- `NewAgentsAndPrompts/reviewer.agent.md` — current file as structural reference

## Output File

- `NewAgentsAndPrompts/r-quality.agent.md`
- `NewAgentsAndPrompts/r-security.agent.md`

## Acceptance Criteria

1. [x] Both files follow v2 template structure
2. [x] **r-quality** covers: code quality, readability, maintainability, naming conventions, architectural alignment with `design.md`, DRY/KISS/YAGNI compliance
3. [x] **r-quality** inherits the Review Depth Tiers model (Full/Standard/Lightweight) — receives tier designation from orchestrator
4. [x] **r-quality** inherits the Quality Standard ("Would a staff engineer approve this code?")
5. [x] **r-security** covers: security scanning, OWASP Top 10 (for Full tier), secrets/PII detection, dependency vulnerability checks
6. [x] **r-security** inherits the full Security Review section from the current reviewer (both Standard and Full-tier checks)
7. [x] **r-security** has override rule: ERROR or NEEDS_REVISION with critical severity overrides aggregated result to ERROR — security is a pipeline blocker (R-AC-4)
8. [x] Both output to `docs/feature/<feature-slug>/review/r-<focus>.md`
9. [x] Both have completion contract: `DONE:` / `NEEDS_REVISION:` / `ERROR:`
10. [x] Both do NOT write to `memory.md`
11. [x] Both include Cross-Cutting Observations section in output

## Implementation Guidance

### r-quality.agent.md

```
---
name: r-quality
description: "Code quality, readability, and maintainability review for the Review cluster."
---
```

**Role:** Reviews code for quality, readability, maintainability, naming conventions, and architectural alignment. Inherits the reviewer's depth tier model and quality standard.

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/initial-request.md`
- `docs/feature/<feature-slug>/design.md`
- Git diff
- Entire codebase

**Outputs:**

- `docs/feature/<feature-slug>/review/r-quality.md`

**Key content to inherit from reviewer.agent.md:**

- Review Depth Tiers section (Full/Standard/Lightweight triggers and scope)
- Quality Standard section ("Would a staff engineer approve this code?")
- Workflow Step 4 (code review: maintainability, readability, naming, conventions, DRY/KISS/YAGNI)
- Self-reflection step (verify all changed files reviewed, comments actionable)

### r-security.agent.md

```
---
name: r-security
description: "Security scanning and OWASP compliance review for the Review cluster. Security issues are pipeline blockers."
---
```

**Role:** Performs security review. Its findings can block the entire pipeline.

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/initial-request.md`
- Git diff
- Entire codebase

**Outputs:**

- `docs/feature/<feature-slug>/review/r-security.md`

**Key content to inherit from reviewer.agent.md:**

- Full Security Review section (both All Reviews and Full Tier Only — OWASP Top 10)
- Heuristic pre-scan for large diffs
- Secret/PII scanning patterns
- Review Depth Tiers (to determine Standard vs Full security scope)

**Critical rule (design.md §5.1):** R-Security ERROR or NEEDS_REVISION with Critical severity findings overrides the aggregated pipeline result. Security is non-negotiable.

**Output format (from design.md §5.1):**

```markdown
# Review: <Focus Area>

## Review Tier

## Findings

### [Severity: Blocker/Major/Minor] Finding Title

- **What / Where / Why / Suggested Fix / Affects Tasks**

## Cross-Cutting Observations

## Summary
```

**Note:** These are markdown agent definition files — TDD does not apply.

## Completion Notes

- TDD skipped: markdown agent definition files — no behavioral code to test.
- Both files created following v2 template structure consistent with existing sub-agents (ct-security.agent.md, v-build.agent.md).
- r-quality inherits Review Depth Tiers, Quality Standard, and code review workflow from reviewer.agent.md.
- r-security inherits full Security Review section (secrets/PII scan + OWASP Top 10) from reviewer.agent.md.
- r-security includes explicit Pipeline Blocker Override Rule section per design.md §5.1.
- Both include memory-first protocol (read only, Rule 6), Read-Only Enforcement, Cross-Cutting Observations, and Self-Reflection steps.
