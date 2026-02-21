# Task 03: Feature-Workflow Prompt Update

## Task Goal

Add an orchestrator tool restriction note to `feature-workflow.prompt.md` to document the restricted tool set, preventing documentation drift.

## depends_on

none

## agent

implementer

## In-scope

- Add a clarification note to the Rules section of `.github/prompts/feature-workflow.prompt.md` per design §4.3:
  > "The orchestrator uses only `read_file` and `list_dir` for reading (no `grep_search`, `semantic_search`, `file_search`, or `get_errors`). All orchestrator reads target known deterministic paths."
- This is a **required** change (per CT-Maintainability planning constraint — not optional despite design §4.3 labeling it "Optional Clarification")

## Out-of-scope

- No changes to the L24 memory system rule (design §4.1 confirms it remains accurate)
- No changes to the Key Artifacts table (design §4.2)
- No changes to `orchestrator.agent.md` (Tasks 01, 02)
- No changes to `feature.md` (Task 04)

## Acceptance Criteria

1. `feature-workflow.prompt.md` contains a statement that the orchestrator uses only `read_file` and `list_dir` for reading
2. The statement explicitly lists the 4 excluded tools: `grep_search`, `semantic_search`, `file_search`, `get_errors`
3. The statement mentions "known deterministic paths"
4. No other content in `feature-workflow.prompt.md` is modified

## Test Requirements

After applying the edit, verify:
- The new note is present in the Rules section
- The existing memory system rule at L24 is unchanged
- The Key Artifacts table is unchanged
- The note contains all 4 excluded tool names

## Implementation Steps

1. Read `feature-workflow.prompt.md` to understand its current structure and identify the Rules section
2. Identify the appropriate insertion point within the Rules section (after existing rules)
3. Add the orchestrator tool restriction note per design §4.3
4. Verify no other content was modified

**Reference:** Design §4.3 in `docs/feature/orchestrator-tool-restriction/design.md`.

## Estimated Effort

Low

## Completion Checklist

- [x] Tool restriction note added to Rules section of `feature-workflow.prompt.md`
- [x] Note lists `read_file` and `list_dir` as the only read tools
- [x] Note lists all 4 excluded tools
- [x] Note mentions "known deterministic paths"
- [x] No other content in the file was modified
