# Task 03: Orchestrator Write-Tool Restriction (Track B)

**Task Goal:** Restrict the orchestrator's tool access by removing write/execute tools while retaining read tools, and rewrite Step 0 to delegate file creation.

**depends_on:** none

**agent:** implementer

---

## In-Scope

- Modify `.github/agents/orchestrator.agent.md`:
  - Add/update Operating Rules to declare write-restricted tool set (retain read tools, remove write/execute tools)
  - Update Global Rule 1 to say "delegate all file creation and modification to subagents"
  - Rewrite Step 0 to delegate file creation to `memory` tool or subagents
  - Reword any other instructions that reference direct file-write operations to use delegation

## Out-of-Scope

- Adding Step 8 or telemetry instructions (that's Task 10)
- Modifying Documentation Structure, Expectations table, or Parallel Summary (that's Task 10)
- Any changes to other agent files
- Removing read tools (design explicitly retains them per CT-1 Critical resolution)

---

## Acceptance Criteria

1. Operating Rules include a tool-access rule listing allowed tools: `[agent, agent/runSubagent, memory, read_file, grep_search, semantic_search, file_search, list_dir, get_errors]`
2. Operating Rules explicitly state the orchestrator MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, or `run_in_terminal`
3. Global Rule 1 updated: "Never modify code, documentation, or any file directly — delegate all file creation and modification to subagents via `runSubagent`. Use read tools freely for orchestration decisions."
4. Step 0 delegates file creation: `memory.md` via `memory` tool (with subagent fallback), `initial-request.md` via subagent or context passing, lazy directory creation
5. Step 0 does NOT initialize a Telemetry section in memory.md
6. No references to using `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, or `run_in_terminal` as orchestrator actions remain in the file
7. Read tools (`read_file`, `grep_search`, etc.) are explicitly retained — NOT removed
8. All existing completion contracts unchanged
9. No existing workflow steps renumbered

---

## Estimated Effort

Medium (~150 lines changed in 1 file)

---

## Test Requirements (TDD Fallback — Configuration-Only)

1. **Verify tool restriction declared:** grep for "write-restricted" or "Allowed tools" in Operating Rules
2. **Verify removed tools absent as orchestrator actions:** grep for `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal` — should appear only in the "MUST NOT use" restriction statement, not as instructions to use them
3. **Verify read tools retained:** grep for `read_file`, `grep_search`, `semantic_search` — should appear in the allowed tools list
4. **Verify Global Rule 1 updated:** grep for "delegate all file creation and modification"
5. **Verify Step 0 delegation:** grep for "memory tool" or "subagent" in Step 0 context
6. **Verify no Telemetry section init:** grep for "Telemetry" in Step 0 — should NOT appear

---

## Implementation Steps

1. Read current `.github/agents/orchestrator.agent.md` to understand existing structure:
   - Locate Operating Rules section
   - Locate Global Rule 1
   - Locate Step 0
   - Identify any other references to direct file-write operations
2. Read design.md §Write-Tool Restriction for exact text of:
   - Operating Rules update (rule 5 text)
   - Global Rule 1 update text
   - Step 0 rewrite text
3. Add/update Operating Rule for tool access with the write-restricted tool set declaration
4. Update Global Rule 1 with delegation language
5. Rewrite Step 0 to delegate file creation per design.md
6. Search the entire file for remaining references to `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal` used as orchestrator actions (not in restriction statements) and reword to delegation
7. Verify read tools are still referenced as available/allowed
8. Verify no existing workflow steps were renumbered
9. Verify completion contract is unchanged

---

## Completion Checklist

- [x] Operating Rules declare write-restricted tool set
- [x] Removed tools listed explicitly as MUST NOT use
- [x] Read tools explicitly retained in allowed tools list
- [x] Global Rule 1 updated with delegation language
- [x] Step 0 rewritten to delegate file creation
- [x] No remaining references to orchestrator using write/execute tools directly
- [x] Existing completion contract unchanged
- [x] No workflow steps renumbered
- [x] Anti-Drift Anchor updated with tool restriction note

TDD skipped: agent definition markdown file modification (configuration-only task)
