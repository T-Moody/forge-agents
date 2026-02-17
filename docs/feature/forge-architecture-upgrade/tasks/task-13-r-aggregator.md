# Task 13: R Aggregator

## Agent

implementer

## Depends On

09, 10

## Description

Create `r-aggregator.agent.md` — the aggregator for the Review cluster. It reads all 4 R sub-agent output files, merges review findings, handles R-Security's override rule (security blocks pipeline), treats R-Knowledge as non-blocking, and produces `review.md`. It also surfaces high-value knowledge suggestions separately. Writes to `memory.md`.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §2 (Aggregator Pattern), §5.5 (R Aggregation Details), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — R-AC-1 through R-AC-11
- `NewAgentsAndPrompts/r-quality.agent.md` — output format reference (created in task 09)
- `NewAgentsAndPrompts/r-security.agent.md` — output format reference (created in task 09)
- `NewAgentsAndPrompts/r-testing.agent.md` — output format reference (created in task 10)
- `NewAgentsAndPrompts/r-knowledge.agent.md` — output format reference (created in task 10)

## Output File

- `NewAgentsAndPrompts/r-aggregator.agent.md`

## Acceptance Criteria

1. [x] Follows v2 template structure
2. [x] Reads all 4 R sub-agent outputs from `review/` directory + `memory.md`
3. [x] Produces `review.md` as primary output
4. [x] R-Security override: if R-Security returns ERROR or NEEDS_REVISION with Critical severity, aggregated result is ERROR — security blocks pipeline (R-AC-4)
5. [x] R-Knowledge non-blocking: R-Knowledge ERROR does NOT affect aggregated result (R-AC-9). Log the error but do not block.
6. [x] Includes "High-Value Suggestions" summary section extracted from `knowledge-suggestions.md` — only suggestions with Risk ≤ Low and clear rationale (R-AC-10)
7. [x] Preserves `knowledge-suggestions.md` as a separate artifact (does not merge into `review.md` body)
8. [x] Completion contract: `DONE:` (no Blocker/Major) / `NEEDS_REVISION:` (≥1 Major, no Blocker) / `ERROR:` (R-Security override or 2+ sub-agent ERROR)
9. [x] Writes to `memory.md`: Recent Decisions + Lessons Learned (sequential agent — safe)
10. [x] Output format matches design.md §5.5 specification

## Status

Complete — TDD skipped: markdown agent definition file, no behavioral code.

## Implementation Notes

- Created `NewAgentsAndPrompts/r-aggregator.agent.md` following v2 template structure
- R-Security pipeline override enforced at Step 3 (checked before any other processing)
- R-Security missing returns ERROR immediately (security review is mandatory)
- R-Knowledge handled as non-blocking at Step 5 (ERROR logged, does not affect result)
- High-Value Suggestions extracted at Step 6 (Risk ≤ Low, clear rationale only)
- `knowledge-suggestions.md` preserved as separate artifact (not merged into review.md body)
- Memory write at Step 11 (sequential agent — safe to write)

## Implementation Guidance

```
---
name: r-aggregator
description: "Aggregates Review cluster findings, enforces security overrides, and surfaces knowledge suggestions."
---
```

**Role:** Merge-only aggregator for R cluster. Has unique handling for two sub-agents: R-Security (override to ERROR on critical findings) and R-Knowledge (non-blocking, knowledge suggestions surfaced separately).

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/review/r-quality.md`
- `docs/feature/<feature-slug>/review/r-security.md`
- `docs/feature/<feature-slug>/review/r-testing.md`
- `docs/feature/<feature-slug>/review/r-knowledge.md`
- `docs/feature/<feature-slug>/review/knowledge-suggestions.md` (if exists)

**Outputs:**

- `docs/feature/<feature-slug>/review.md`

**Aggregation workflow (from design.md §5.5):**

1. Read `memory.md`
2. Read all available R sub-agent outputs
3. Check R-Security first — if ERROR or Critical NEEDS_REVISION, set result to ERROR immediately
4. Merge findings from R-Quality, R-Security, R-Testing (sorted by severity)
5. Log R-Knowledge status (DONE or ERROR) without affecting result
6. Extract high-value suggestions from `knowledge-suggestions.md` (Risk ≤ Low, clear rationale)
7. Compile "High-Value Suggestions" summary section
8. Determine completion contract per severity + override rules
9. Write `review.md`
10. Write to `memory.md`: Recent Decisions, Lessons Learned

**Input validation:**

- R-Knowledge missing/ERROR: proceed without knowledge section, log gap
- R-Security missing: return ERROR (security review is mandatory)
- Other sub-agents: 1 missing → proceed with gap noted; 2+ missing → ERROR

**Output format (from design.md §5.5):**

```markdown
# Code Review

## Summary

## Review Tier Applied

## Security Review (from R-Security)

## Quality Review (from R-Quality)

## Test Review (from R-Testing)

## Consolidated Findings

### Blocker / Major / Minor

## High-Value Knowledge Suggestions

## Knowledge Evolution Status

## Recommendations
```

**Note:** This is a markdown agent definition file — TDD does not apply.
