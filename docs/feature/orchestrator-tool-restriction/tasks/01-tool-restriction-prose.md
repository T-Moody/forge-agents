# Task 01: Tool Restriction — YAML Frontmatter + Prose Updates + Anti-Drift

## Task Goal

Apply the primary tool restriction edits to `orchestrator.agent.md`: add YAML `tools:` field, update Global Rule 1, Operating Rule 1, Operating Rule 5, and rewrite the Anti-Drift Anchor to reflect the restricted 5-tool set.

## depends_on

none

## agent

implementer

## In-scope

- Add `tools: [agent, agent/runSubagent, memory, read_file, list_dir]` to YAML frontmatter (design §2)
- Update Global Rule 1 to remove 4 discovery tools, add `memory` tool disambiguation (design §3.1)
- Update Operating Rule 1 to remove `semantic_search`/`grep_search` references (design §3.2)
- Update Operating Rule 5 to restrict tool list, change title from "write-restricted" to "restricted", add `memory` tool disambiguation (design §3.3)
- Rewrite Anti-Drift Anchor with restricted tool set, expanded prohibited list, dual prohibition rationale, and `memory` tool disambiguation (design §3.4)

## Out-of-scope

- Memory tool disambiguation in Global Rule 12, Step 0.1, Step 8.2, Memory Lifecycle Table (Task 02)
- Merge step wording corrections (Task 02)
- feature-workflow.prompt.md changes (Task 03)
- feature.md scope annotation (Task 04)

## Acceptance Criteria

1. **AC-1:** YAML frontmatter contains exactly `tools: [agent, agent/runSubagent, memory, read_file, list_dir]`
2. **AC-2:** Operating Rule 5 allowed-tools list matches the YAML `tools:` field exactly
3. **AC-3 (partial):** Global Rule 1, Operating Rule 1, Operating Rule 5, and Anti-Drift Anchor no longer list `grep_search`, `semantic_search`, `file_search`, or `get_errors` as allowed tools
4. **AC-11 (partial):** Global Rule 1, Operating Rule 5, and Anti-Drift Anchor each contain a `memory` tool disambiguation statement
5. **AC-12:** Anti-Drift Anchor includes `read_file` and `list_dir` as the only read tools, lists all 8 prohibited tools, provides dual prohibition rationale ("all file writes are delegated to subagents; search/discovery tools are unnecessary because the orchestrator reads known deterministic paths only"), and includes `memory` tool disambiguation

## Test Requirements

After applying all edits, verify:
- **T-1 (design §12.1):** YAML frontmatter parses correctly and contains exactly 5 tools
- **T-2:** Operating Rule 5 prose tool list matches YAML `tools:` field
- **T-3:** Search the file for `grep_search`, `semantic_search`, `file_search`, `get_errors` — they should only appear in the MUST NOT / prohibited contexts, never in allowed-tool contexts
- **T-N1 (design §12.2):** The string "Use read tools" does NOT appear in the file
- **T-N2:** The string "read tools freely" does NOT appear in the file
- **T-4:** Anti-Drift Anchor contains all 8 prohibited tools in its MUST NOT list
- **T-5:** Anti-Drift Anchor contains "memory" tool disambiguation ("VS Code's cross-session knowledge store")

## Implementation Steps

1. Read `orchestrator.agent.md` YAML frontmatter section (first ~10 lines) to confirm current state
2. Add `tools: [agent, agent/runSubagent, memory, read_file, list_dir]` line after the `description:` line in the YAML frontmatter, per design §2.2
3. Read Global Rule 1 (around L33) and replace with the exact text from design §3.1 New
4. Read Operating Rule 1 (around L83) and replace with the exact text from design §3.2 New
5. Read Operating Rule 5 (around L92) and replace with the exact text from design §3.3 New
6. Read Anti-Drift Anchor (around L541) and replace with the exact text from design §3.4 New
7. Run acceptance tests: search for prohibited tools in allowed contexts, verify YAML validity, verify Anti-Drift content

**Reference:** All exact current/new text is in `docs/feature/orchestrator-tool-restriction/design.md` §2 and §3.1–§3.4.

## Estimated Effort

Medium

## Completion Checklist

- [x] YAML frontmatter contains `tools:` field with 5 tools
- [x] Global Rule 1 updated — no discovery tools in allowed list, `memory` disambiguated
- [x] Operating Rule 1 updated — no `semantic_search`/`grep_search` references
- [x] Operating Rule 5 updated — restricted tool list, title changed, `memory` disambiguated
- [x] Anti-Drift Anchor rewritten with restricted tools, expanded prohibited list, dual rationale, `memory` disambiguation
- [x] No `grep_search`/`semantic_search`/`file_search`/`get_errors` in allowed-tool contexts
- [x] All edits match design §2 and §3.1–§3.4 specifications
