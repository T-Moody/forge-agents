# Task 11: Orchestrator Agent Definition

## Task Goal

Create `NewAgents/.github/agents/orchestrator.agent.md` — the lean dispatch coordinator with routing, approval gates, evidence verification via SQL, retry orchestration, and pipeline initialization.

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following design.md §Decision 10 template
- Role: lean dispatch coordinator — THE central agent that drives the entire pipeline
- 5 core responsibilities (v4):
  1. Dispatch routing — read completion contracts, determine next step, dispatch agents
  2. Approval gate management — structured prompts in interactive, auto-proceed in autonomous
  3. Error categorization and retry — classify failures (transient vs deterministic), retry transient once
  4. Evidence gate verification (v3) — independently verify via `run_in_terminal` SQL queries on `anvil_checks`
  5. Pipeline initialization (v4) — Step 0: SQLite schema creation, WAL mode, git hygiene, pushback
- Full pipeline step sequence (Steps 0–9) reference
- Input: `initial-request.md`, all upstream agent outputs (typed YAML)
- Output: none (orchestrator writes NO files — all state in-context only)
- Tool access: `agent`, `agent/runSubagent`, `memory`, `read_file`, `list_dir`, `run_in_terminal`, `get_terminal_output`
- Tool restrictions (MUST NOT use): `create_file`, `replace_string_in_file`, `grep_search`, `semantic_search`, `file_search`, `get_errors`
- `run_in_terminal` constraint (DR-1): ONLY for:
  - SQLite read queries (SELECT) on verification ledger
  - SQLite DDL for initialization at Step 0 (CREATE TABLE, PRAGMA)
  - Git read operations (git status, git rev-parse, git show, git tag)
  - Git staging/commit at Step 9 (git add, git commit)
  - MUST NOT use for builds, tests, code execution, or file modification
- Pipeline state model: in-context only (no file writes), tracks: feature_slug, approval_mode, current_step, overall_risk, run_id, completed_steps, error_log
- Pipeline state recovery (EC-5): scan feature directory via `list_dir` + `read_file` to discover existing outputs at deterministic paths
- Complete orchestrator decision table from design.md (all routing conditions)
- Step 0 initialization: SQLite DDL (CREATE TABLE + PRAGMA + indexes), git hygiene (status, branch check), lightweight pushback evaluation, run_id generation
- Approval gate prompts: structured multiple-choice format at gate-post-research and gate-post-planning
- Evidence gate SQL queries (v4 corrected): all 7 query templates from design.md §Decision 6
- Security blocker policy: any verdict='blocker' → pipeline ERROR
- Dispatch patterns: reference dispatch-patterns.md for Pattern A/B
- Retry budgets: 1 orchestrator retry per agent, 3 verification loop iterations, 1 design revision, 2 review rounds
- Step 9 auto-commit: `git add -A && git commit -m "feat({feature_slug}): pipeline complete"` (skip if Confidence: Low)
- Completion contract: DONE | ERROR (no NEEDS_REVISION — handles revision routing internally)
- Self-verification + anti-drift anchor

## Out-of-Scope

- Schema definitions (in schemas.md)
- Individual agent behaviors (defined in their own .agent.md files)
- Schema validation (moved to agent self-validation)
- Telemetry accumulation (Knowledge Agent reads all outputs at Step 8)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/orchestrator.agent.md`
2. Follows agent definition template with all required sections
3. All 5 core responsibilities documented
4. Tool access list matches design.md exactly (7 allowed, 6 restricted)
5. `run_in_terminal` constraint (DR-1) explicitly documented with allowed/forbidden operations
6. Complete pipeline step sequence (Steps 0–9) with routing logic
7. Orchestrator decision table present with ALL conditions from design.md §Orchestrator Decision Table
8. Step 0 initialization documented: SQLite DDL, WAL, git hygiene, pushback, run_id
9. All 7 evidence gate SQL query templates present (baseline exists, verification sufficient, design review approval + blocker, code review approval + blocker, review completion)
10. Approval gate format: structured multiple-choice with example prompts
11. Retry budgets documented (1 agent, 3 verification, 1 design, 2 review)
12. Security blocker policy present
13. Pipeline state model (in-context) documented with fields
14. Pipeline recovery (EC-5) scanning logic documented
15. Auto-commit (Step 9) documented with Confidence gate
16. Completion contract: DONE | ERROR
17. Self-verification and anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Verify tool access list matches design.md §Decision 2 (7 allowed, 6 restricted)
- Grep for all 5 responsibilities
- Grep for DR-1 / run_in_terminal constraint
- Grep for all 7 SQL query templates
- Grep for decision table routing conditions
- Grep for security blocker policy
- Verify completion contract and anti-drift

## Implementation Steps

1. Read design.md §Decision 2, Orchestrator agent detail for full responsibility list, tools, restrictions, DR-1 constraint
2. Read design.md §Decision 3 for complete pipeline step sequence (Steps 0–9), gate points, dispatch count
3. Read design.md §Orchestrator Decision Table (lines ~1770–1810) for all routing conditions
4. Read design.md §Decision 6 for all 7 evidence gate SQL queries (v4 corrected)
5. Read design.md §Decision 9 for approval gate format, structured prompts, mode switching
6. Read design.md §Decision 8 for retry budgets, escalation paths
7. Read design.md §Pipeline State Model for in-context state fields and recovery scanning
8. Read design.md §Deviation Records for DR-1 (run_in_terminal) and DR-2 (always 3 reviewers)
9. Reference existing `.github/agents/orchestrator.agent.md` for dispatch patterns and decision logic
10. Create `NewAgents/.github/agents/orchestrator.agent.md` with full template structure including:
    - 5 responsibilities with detailed descriptions
    - Tool access + restrictions + DR-1 constraint
    - Full pipeline step reference (Steps 0–9)
    - Decision table (all routing conditions)
    - Step 0 initialization block (SQLite DDL, WAL, git hygiene, pushback, run_id)
    - Evidence gate SQL queries (7 templates)
    - Approval gate prompts (2 gates with multiple-choice format)
    - Retry budgets and escalation paths
    - Pipeline state model (in-context fields)
    - Recovery scanning logic (EC-5)
    - Auto-commit (Step 9) with Confidence gate
11. Self-verify: all 5 responsibilities present, all SQL queries present, decision table complete, tool restrictions match design

## Relevant Context from design.md

- §Decision 2 (lines ~120–145) — Orchestrator: 5 responsibilities, tools, restrictions, DR-1, explicitly NOT responsible for, pipeline state recovery, contract
- §Decision 3 (lines ~310–475) — Complete pipeline Steps 0–9, gate points, dispatch count, parallelism map
- §Decision 6 (lines ~760–800) — All 7 evidence gate SQL query templates
- §Decision 9 (lines ~1050–1120) — Approval gate format, prompts, mode switching
- §Decision 8 (lines ~1030–1060) — Retry budgets, escalation paths
- §Pipeline State Model (lines ~1815–1860) — In-context state fields, recovery scanning
- §Orchestrator Decision Table (lines ~1770–1810) — All routing conditions
- §Deviation Records (lines ~2160–2230) — DR-1 (run_in_terminal), DR-2 (always 3)
- §Sequence/Interaction Notes (lines ~1700–1770) — Standard + high-risk flow

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present
- [x] 5 core responsibilities documented
- [x] Tool access + restrictions + DR-1 match design exactly
- [x] Complete pipeline step sequence (Steps 0–9)
- [x] Decision table with all routing conditions
- [x] Step 0 initialization (SQLite DDL, WAL, git hygiene, pushback, run_id)
- [x] All 7 evidence gate SQL queries present
- [x] Approval gate prompts present
- [x] Retry budgets documented
- [x] Security blocker policy present
- [x] Pipeline state model documented
- [x] Recovery scanning (EC-5) documented
- [x] Auto-commit (Step 9) documented
- [x] DONE | ERROR completion contract
- [x] Anti-drift anchor present
