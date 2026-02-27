# Task 07: Implementer Agent Definition

## Task Goal

Create `NewAgents/.github/agents/implementer.agent.md` — the TDD implementation agent with baseline capture, self-fix loop, git staging, and revert mode.

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following design.md §Decision 10 template
- Role: TDD implementation + documentation + baseline capture
- Input: assigned task file (`tasks/<task-id>.yaml` with `relevant_context` pointers), targeted sections of `design-output.yaml` and `spec-output.yaml`
- Output: code/test/doc files + `implementation-reports/<task-id>.yaml` (typed report with baseline records)
- Baseline capture: before changes, capture IDE diagnostics + build + test state, record as `phase: baseline`
- Self-fix loop (v4 H3): run diagnostics + build + tests; if fail, fix immediately (max 2 self-fix attempts)
- Git staging (v4 H8): `git add -A` after self-fix loop completion
- Revert mode (v4 H9): when dispatched with `{mode: 'revert', files_to_revert, baseline_tag}`, execute `git checkout pipeline-baseline-{run_id} -- {files}` and record revert in `anvil_checks`
- Documentation capability: handle `task_type: documentation` for doc-only tasks
- Tool access: full toolset (`read_file`, `list_dir`, `grep_search`, `semantic_search`, `file_search`, `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`, `get_terminal_output`, `get_errors`, `list_code_usages`)
- Completion contract: DONE | ERROR
- Self-verification + anti-drift anchor

## Out-of-Scope

- Schema definition (in schemas.md)
- Verification cascade (verifier's job)
- Dispatch coordination (orchestrator's job)
- Git tag creation (orchestrator creates baseline tag before dispatching implementers)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/implementer.agent.md`
2. Follows agent definition template with all required sections
3. References `implementation-report` schema from schemas.md
4. Baseline capture workflow documented: IDE diagnostics, build, test state before changes
5. Self-fix loop documented: run checks → fix → retry (max 2 attempts)
6. Git staging (`git add -A`) documented as final step after self-fix
7. Revert mode documented: parameters (mode, files_to_revert, baseline_tag), execution (git checkout), recording in anvil_checks
8. TDD workflow: write test → implement → verify pass
9. `relevant_context` pointer consumption documented (read only what's listed in task)
10. Documentation mode (`task_type: documentation`) documented
11. Completion contract: DONE | ERROR (no NEEDS_REVISION — orchestrator handles revision routing)
12. Self-verification and anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Grep for baseline capture section
- Grep for self-fix loop (max 2 attempts)
- Grep for `git add -A`
- Grep for revert mode
- Verify tool list is complete (12 tools)

## Implementation Steps

1. Read design.md §Decision 2, Implementer agent detail for inputs, outputs, tools, baseline responsibility, contract
2. Read design.md §Decision 3, Step 5 for self-fix loop (H3), git staging (H8)
3. Read design.md §Decision 6 for revert responsibility (H9/FR-4.7), baseline capture integration
4. Read design.md §Decision 5 for relevant_context consumption pattern
5. Reference existing `.github/agents/implementer.agent.md` for TDD workflow patterns
6. Reference `Anvil/anvil.agent.md` for baseline capture pattern
7. Create `NewAgents/.github/agents/implementer.agent.md` with full template structure
8. Self-verify all sections present, including all v4 additions (self-fix, git staging, revert)

## Relevant Context from design.md

- §Decision 2 (lines ~215–230) — Implementer: inputs, outputs, tools, baseline, contract
- §Decision 3, Step 5 (lines ~380–400) — Self-fix loop (H3), git staging (H8), sub-wave partitioning
- §Decision 6, Baseline Capture (lines ~810–850) — Git baseline tag, implementer baseline responsibility, revert mode (H9)
- §Decision 5, Targeted Context (lines ~610–640) — relevant_context consumption

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present
- [x] Baseline capture workflow documented
- [x] Self-fix loop (max 2) documented
- [x] Git staging (git add -A) documented
- [x] Revert mode documented with parameters
- [x] TDD workflow documented
- [x] Schema reference to schemas.md present
- [x] Anti-drift anchor present
