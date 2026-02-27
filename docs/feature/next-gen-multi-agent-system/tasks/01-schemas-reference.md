# Task 01: Schemas Reference Document

## Task Goal

Create `NewAgents/.github/agents/schemas.md` containing all 10 typed YAML schema definitions with examples, producer/consumer table, and evolution strategy.

## depends_on

none

## agent

implementer

## In-Scope

- All 10 schemas defined in design.md §Schemas Reference: completion-contract, research-output, spec-output, design-output, plan-output, task-schema, implementation-report, verification-report, review-findings, knowledge-output
- Producer/consumer dependency table (which agent produces/consumes each schema)
- Schema evolution strategy (additive vs breaking changes, version policy)
- One complete example payload per schema
- Required fields, types, allowed values per schema
- SQLite `anvil_checks` and `pipeline_telemetry` CREATE TABLE statements (from design.md §Decision 6)
- `task_id` naming convention documentation (H6)
- `check_name` naming patterns for evidence gate SQL LIKE filters

## Out-of-Scope

- Agent definitions (separate tasks 03–11)
- Dispatch pattern definitions (Task 02)
- Severity taxonomy definitions (Task 02)
- Runtime validation logic (none exists — prompt-level only)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/schemas.md`
2. Contains all 10 schemas in pipeline-step order (research → spec → design → plan → implementation → verification → review → knowledge)
3. Each schema includes: required fields, types, constraints/allowed values, one example payload
4. Producer/consumer table matches design.md §Schemas Reference exactly
5. Schema evolution strategy section present with additive/breaking change policies
6. `completion-contract` schema includes all fields: status, summary, severity, findings_count, risk_level, output_paths, evidence_summary
7. SQLite schemas included: `anvil_checks` (with v4 columns: run_id, verdict, severity, round) and `pipeline_telemetry`
8. Both SQLite schemas include WAL + busy_timeout pragmas and required indexes
9. `task_id` convention documented: per-task = planner-assigned IDs, feature-level = `{feature_slug}-design-review` / `{feature_slug}-code-review`
10. Schema version field (`schema_version: "1.0"`) documented as required in every agent output

## Estimated Effort

Medium

## Test Requirements

- Verify all 10 schema names match design.md §Schemas Reference table
- Verify `anvil_checks` schema matches design.md §Decision 6 SQL (reconcile §Decision 6 vs §Data Storage if inconsistent — prefer §Decision 6 as primary, note `output_snippet` length: 500 chars per §Decision 6)
- Verify producer/consumer relationships are complete and accurate

## Implementation Steps

1. Read design.md §Schemas Reference for the 10-schema table, producer/consumer relationships
2. Read design.md §Decision 4 for completion-contract schema and agent output schema pattern
3. Read design.md §Decision 6 for `anvil_checks` and `pipeline_telemetry` SQL schemas (use §Decision 6 as canonical — includes v4 CHECK constraints and indexes)
4. Read design.md §Data Storage for `pipeline_telemetry` queries and operational notes
5. Read design.md §Decision 5 for task-schema `relevant_context` example
6. Read design.md §Decision 2 for each agent's Inputs/Outputs to derive per-schema field lists
7. Reference existing `.github/agents/evaluation-schema.md` for document formatting pattern
8. Create `NewAgents/.github/agents/schemas.md` with:
   - Title, purpose, schema version note
   - Producer/consumer dependency table
   - All 10 YAML schemas in pipeline-step order, each with fields + example
   - SQLite schema section with full CREATE TABLE + PRAGMA + INDEX statements
   - `task_id` convention section
   - `check_name` naming patterns section
   - Schema evolution strategy section
9. Self-verify: count schemas (must be 10), verify all fields match design.md

## Relevant Context from design.md

- §Schemas Reference (lines ~1870–1960) — 10-schema table, removed schemas, evolution strategy
- §Decision 4 (lines ~490–560) — completion-contract schema, agent output schema pattern
- §Decision 6 (lines ~640–810) — anvil_checks SQL, pipeline_telemetry SQL, evidence gate queries, task_id convention
- §Decision 5 (lines ~600–640) — task-schema relevant_context example
- §Decision 2 (lines ~100–310) — agent Inputs/Outputs per agent
- §Data Storage Strategy (lines ~2020–2140) — SQLite schema details (second copy — may differ, reconcile)

## Completion Checklist

- [x] File created at correct path (`NewAgents/.github/agents/schemas.md`)
- [x] 10 schemas defined with fields, types, examples
- [x] Producer/consumer table present (matches design.md §Schemas Reference exactly)
- [x] SQLite schemas present with WAL + indexes (§Decision 6 canonical; output_snippet ≤ 500, severity = Blocker/Critical/Major/Minor)
- [x] task_id convention documented (per-task = planner IDs, feature-level = `{feature_slug}-design-review` / `{feature_slug}-code-review`)
- [x] check_name naming patterns documented with SQL LIKE filter examples
- [x] Schema evolution strategy present (additive/breaking change policies, version policy)
- [x] All schema field names match design.md references
- [x] schema_version: "1.0" documented as required in every agent output
- [x] completion-contract includes all fields: status, summary, severity, findings_count, risk_level, output_paths, evidence_summary

## Notes

- TDD skipped: documentation-only task (no behavioral code changes, no test framework applicable)
- §Decision 6 used as canonical for `anvil_checks` schema (preferred over §Data Storage per task spec)
- Differences noted: §Data Storage has `output_snippet ≤ 2000` and lowercase severity values; §Decision 6 has `output_snippet ≤ 500` and title-case severity — §Decision 6 values used
