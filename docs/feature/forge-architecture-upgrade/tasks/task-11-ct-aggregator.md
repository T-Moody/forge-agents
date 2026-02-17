# Task 11: CT Aggregator

## Status

DONE

## Agent

implementer

## Depends On

05, 06

## Description

Create `ct-aggregator.agent.md` — the aggregator for the Critical Thinking cluster. It reads all 4 CT sub-agent output files, merges/deduplicates findings, surfaces unresolved tensions, and produces `design_critical_review.md`. It is the only CT cluster agent that writes to `memory.md` (sequential agent — runs after all sub-agents complete).

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §2 (Aggregator Pattern), §3.3 (CT Aggregation Details), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — CT-AC-3 through CT-AC-9
- `NewAgentsAndPrompts/ct-security.agent.md` — sub-agent output format reference (created in task 05)
- `NewAgentsAndPrompts/ct-scalability.agent.md` — sub-agent output format reference (created in task 05)
- `NewAgentsAndPrompts/ct-maintainability.agent.md` — sub-agent output format reference (created in task 06)
- `NewAgentsAndPrompts/ct-strategy.agent.md` — sub-agent output format reference (created in task 06)

## Output File

- `NewAgentsAndPrompts/ct-aggregator.agent.md`

## Acceptance Criteria

1. Follows v2 template: YAML frontmatter, role statement, inputs/outputs, 5 operating rules + Rule 6 (memory-first), workflow, output spec, completion contract, anti-drift anchor
2. Reads all 4 CT sub-agent output files from `ct-review/` directory
3. Also reads `design.md` (context reference only — NOT for re-analysis) and `memory.md`
4. Produces `design_critical_review.md` as output
5. Deduplicates findings with matching Where+What fields, attributing to all originating sub-agents
6. Sorts findings by severity: Critical → High → Medium → Low
7. Synthesizes Cross-Cutting Observations from all sub-agents into "Cross-Cutting Concerns" section
8. Surfaces contradictions as "Unresolved Tensions" — presents both sides, does NOT resolve them
9. Completion contract: `DONE:` (all findings ≤Medium severity) / `NEEDS_REVISION:` (≥1 Critical/High finding) / `ERROR:` (<2 sub-agent outputs available)
10. Writes to `memory.md`: Recent Decisions + Recent Updates (sequential agent — safe)
11. If max-1 CT revision loop exhausts and Unresolved Tensions remain, formats them as "Planning Constraints" section for the planner
12. Output format matches design.md §3.3 specification

## Implementation Guidance

```
---
name: ct-aggregator
description: "Aggregates 4 CT sub-agent findings into a unified design critical review."
---
```

**Role:** Merge-only aggregator. Consumes N sub-agent outputs, merges/deduplicates, surfaces conflicts, produces a single artifact. Does NOT re-analyze source artifacts or generate new findings.

**Inputs:**

- `memory.md` (read first)
- `docs/feature/<feature-slug>/ct-review/ct-security.md`
- `docs/feature/<feature-slug>/ct-review/ct-scalability.md`
- `docs/feature/<feature-slug>/ct-review/ct-maintainability.md`
- `docs/feature/<feature-slug>/ct-review/ct-strategy.md`
- `docs/feature/<feature-slug>/design.md` (context reference only)

**Outputs:**

- `docs/feature/<feature-slug>/design_critical_review.md`

**Aggregation workflow (from design.md §3.3):**

1. Read `memory.md`
2. Read all available CT sub-agent output files
3. Collect all findings into a unified list
4. Deduplicate: merge findings with matching Where+What, attribute to all sources
5. Sort by severity (Critical → High → Medium → Low)
6. Synthesize Cross-Cutting Observations into "Cross-Cutting Concerns" section
7. Surface contradictions as "Unresolved Tensions" (see design.md §2.4 for format)
8. Merge requirement coverage tables
9. Determine completion contract per severity threshold
10. Write `design_critical_review.md`
11. Write to `memory.md`: Recent Decisions, Recent Updates

**Input validation (from design.md §2.2):**

- 1 missing sub-agent output: proceed with gap noted
- 2+ missing: return ERROR

**Output format (from design.md §3.3):**

```markdown
# Design Critical Review

## Summary

## Overall Risk Level

## Findings by Severity

### Critical / High / Medium / Low

## Cross-Cutting Concerns

## Unresolved Tensions

## Requirement Coverage Gaps

## Recommendations

## Planning Constraints (if unresolved tensions forwarded to planner)
```

**Note:** This is a markdown agent definition file — TDD does not apply.
