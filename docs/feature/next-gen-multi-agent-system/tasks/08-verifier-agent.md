# Task 08: Verifier Agent Definition

## Task Goal

Create `NewAgents/.github/agents/verifier.agent.md` — the unified verification cascade agent with SQL evidence ledger, per-task dispatch, baseline cross-check, and 4-tier cascade.

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following design.md §Decision 10 template
- Role: unified verification cascade + SQL evidence ledger for a single task
- Input: single `implementation-reports/<task-id>.yaml`, task-relevant sections of plan/spec outputs, verification ledger, codebase, `run_id`
- Output: `verification-reports/<task-id>.yaml` (typed report with evidence gate summary) + SQL INSERT entries into `anvil_checks`
- 4-tier verification cascade (v4 H11):
  - Tier 1 (always): IDE diagnostics, syntax checks
  - Tier 2 (if tooling exists): build, type check, lint, tests
  - Tier 3 (required if no runtime verification in Tiers 1–2): import/load test, smoke execution
  - Tier 4 (Large tasks only): operational readiness — observability, degradation, secrets scan
- SQL evidence recording: every check → INSERT into `anvil_checks` with `run_in_terminal`
  - phase='baseline' or phase='after' entries
  - `check_name` conventions for SQL LIKE queries
  - `CREATE TABLE IF NOT EXISTS` as safety net (primary init in Step 0)
- Baseline cross-check (v4 C5): use `git show pipeline-baseline-{run_id}:<path>` to verify Implementer baseline claims
- Per-task evidence gate: report `gate_status: passed | failed` in YAML output
- Regression detection: compare baseline vs after checks; flag regressions
- Acceptance criteria verification against spec (task-relevant criteria only)
- Tool access: `read_file`, `list_dir`, `grep_search`, `file_search`, `run_in_terminal`, `get_terminal_output`, `get_errors`, `ide-get_diagnostics`
- Tool restriction: read-only for source code; may execute build/test but MUST NOT modify source
- Completion contract: DONE | NEEDS_REVISION | ERROR
- Self-verification + anti-drift anchor

## Out-of-Scope

- Schema definitions (in schemas.md)
- Implementation of fixes (implementer's responsibility)
- Dispatch coordination (orchestrator handles per-task dispatch)
- Evidence gate SQL queries by orchestrator (orchestrator's independent verification)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/verifier.agent.md`
2. Follows agent definition template with all required sections
3. References `verification-report` schema from schemas.md
4. All 4 tiers documented with: what checks to run, conditions for execution, SQL INSERT format
5. Tier 4 conditional execution documented (Large tasks only)
6. SQL INSERT pattern for `anvil_checks` documented (fields: run_id, task_id, phase, check_name, tool, command, exit_code, output_snippet, passed, round)
7. Baseline cross-check via `git show pipeline-baseline-{run_id}:<path>` documented
8. Regression detection logic documented (baseline vs after comparison)
9. Per-task evidence gate summary in output (`total_checks`, `passed`, `failed`, `gate_status`)
10. Tool list matches design.md §Decision 2 (Verifier) — 8 tools, read-only restriction on source code
11. Completion contract: DONE | NEEDS_REVISION | ERROR (3-state)
12. `check_name` convention documented (e.g., `readiness-{type}` for Tier 4, `baseline-discrepancy` for cross-check)
13. Self-verification and anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Grep for all 4 tiers in cascade
- Grep for `anvil_checks` and SQL INSERT
- Grep for `git show pipeline-baseline`
- Grep for NEEDS_REVISION in completion contract
- Verify tool list (8 tools, read-only restriction)

## Implementation Steps

1. Read design.md §Decision 2, Verifier agent detail for inputs, outputs, tools, cascade, contract, per-task dispatch
2. Read design.md §Decision 6 for full verification architecture: tiered cascade, evidence format, SQL schema, baseline capture integration, check_name convention
3. Read design.md §Decision 3, Step 6 for verification dispatch flow, Tier 4 (H11), baseline cross-check (C5)
4. Read design.md §Data Storage for `anvil_checks` schema details (reconcile with §Decision 6 — prefer §Decision 6)
5. Reference `Anvil/anvil.agent.md` for verification cascade and SQL ledger patterns
6. Reference existing `.github/agents/v-build.agent.md` and `.github/agents/v-tests.agent.md` for build/test detection
7. Create `NewAgents/.github/agents/verifier.agent.md` with full template structure including:
   - Detailed 4-tier cascade with conditional execution rules
   - SQL INSERT templates for each tier
   - Baseline cross-check workflow
   - Regression detection logic
   - Evidence gate summary output
8. Self-verify all sections present, all 4 tiers documented

## Relevant Context from design.md

- §Decision 2 (lines ~230–265) — Verifier: inputs, outputs, tools, cascade tiers, per-task dispatch, baseline cross-check, completion contract
- §Decision 6 (lines ~640–870) — Full verification architecture: SQL schema, evidence gating, baseline capture, check_name convention, minimum signals
- §Decision 3, Step 6 (lines ~395–420) — Verification dispatch flow, Tier 4, SQL queries with run_id/round
- §Data Storage (lines ~2050–2140) — anvil_checks detailed schema and key queries

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present
- [x] 4-tier cascade documented with conditions
- [x] SQL INSERT pattern for anvil_checks documented
- [x] Baseline cross-check (git show) documented
- [x] Regression detection documented
- [x] Evidence gate summary (gate_status) in output
- [x] check_name convention documented
- [x] Tool list correct (8 tools, read-only restriction)
- [x] 3-state completion contract present
- [x] Anti-drift anchor present
