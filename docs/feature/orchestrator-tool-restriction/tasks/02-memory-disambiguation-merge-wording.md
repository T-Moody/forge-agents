# Task 02: Memory Tool Disambiguation + Merge Step Wording Corrections

## Task Goal

Fix all pre-existing "memory tool" contradictions and correct merge step wording throughout `orchestrator.agent.md` to accurately reflect subagent delegation as the mechanism for all file writes.

## depends_on

01

## agent

implementer

## In-scope

- Fix Global Rule 12 (2 lines) — replace "via `memory` tool merging" and "via `memory` tool" with subagent delegation language (design §3.5)
- Fix Step 0.1 — replace "Use the `memory` tool to create" with subagent delegation (design §3.6)
- Fix Step 8.2 — replace "using `memory` tool or delegating to subagent" with subagent delegation only (design §3.7)
- Fix Memory Lifecycle Table Initialize row — replace "Use `memory` tool" with subagent delegation (design §3.8)
- Correct all 8 merge steps: 1.1m, 2m, 3m, 3b.2, 4m, between-waves, 6.3, 7.3 — replace "merges" with "dispatches a subagent to merge" (design §3.9)
- Remove the line in Step 1.1m that says "This is an orchestrator merge operation — no subagent invocation." (design §3.9)

## Out-of-scope

- YAML frontmatter, Global Rule 1, Operating Rules, Anti-Drift Anchor (completed in Task 01)
- feature-workflow.prompt.md changes (Task 03)
- feature.md scope annotation (Task 04)
- Merge step headings — these are preserved, only body text changes

## Acceptance Criteria

1. **AC-11 (completion):** Zero occurrences of "via `memory` tool" or "using `memory` tool" in the orchestrator prompt — all replaced with subagent delegation language
2. **Merge step wording:** All 8 merge steps use "dispatches a subagent to merge" (or "Dispatch a subagent to merge") instead of direct "merges" language
3. **Step 1.1m cleanup:** The line "This is an orchestrator merge operation — no subagent invocation." is removed
4. **Memory Lifecycle Table:** Initialize row says "Delegate to a setup subagent" not "Use `memory` tool"
5. **Step 0.1:** Primary mechanism is subagent delegation, not `memory` tool with subagent fallback

## Test Requirements

After applying all edits, verify:
- **T-8 (design §12.1):** Search for `via \x60memory\x60 tool` — zero matches
- **T-9:** Search for `using \x60memory\x60 tool` — zero matches
- **T-10:** Search for `Use the \x60memory\x60 tool` — zero matches
- **T-11:** Search for `Use \x60memory\x60 tool` — zero matches
- **T-12:** Search for `no subagent invocation` — zero matches (Step 1.1m line removed)
- **T-13:** Search for `subagent fallback` — zero matches in Step 0.1 / Memory Lifecycle Table context
- **Merge verification:** Each of the 8 merge steps contains "dispatches a subagent" or "Dispatch a subagent"

## Implementation Steps

1. Read design §3.5 for Global Rule 12 exact current/new text. Locate the two lines (around L44-46) in `orchestrator.agent.md` and apply replacements.
2. Read design §3.6 for Step 0.1 exact text. Locate Step 0.1 (around L188) and apply replacement.
3. Read design §3.7 for Step 8.2 exact text. Locate Step 8.2 (around L456) and apply replacement.
4. Read design §3.8 for Memory Lifecycle Table Initialize row exact text. Locate (around L503) and apply replacement.
5. Read design §3.9 merge step table. For each of the 8 merge steps, locate current wording and replace with new wording. Note: Step 1.1m has 2 edits (text replacement + line removal).
6. Run acceptance tests: search for all prohibited patterns, verify merge step wording.

**Reference:** All exact current/new text is in `docs/feature/orchestrator-tool-restriction/design.md` §3.5–§3.9. The merge step table in §3.9 lists all 8 steps with exact current and new wording.

## Estimated Effort

Medium

## Completion Checklist

- [x] Global Rule 12 — both lines corrected to subagent delegation
- [x] Step 0.1 — subagent delegation is primary (not fallback)
- [x] Step 8.2 — "memory tool" option removed, subagent delegation only
- [x] Memory Lifecycle Table — Initialize row corrected
- [x] Step 1.1m — text corrected + "no subagent invocation" line removed
- [x] Step 2m — merge wording corrected
- [x] Step 3m — merge wording corrected
- [x] Step 3b.2 — merge wording corrected
- [x] Step 4m — merge wording corrected
- [x] Between-waves — merge wording corrected (including Lessons Learned subagent note)
- [x] Step 6.3 — merge wording corrected
- [x] Step 7.3 — merge wording corrected
- [x] Zero matches for prohibited patterns (T-8 through T-13)

## Findings

- T-12 ("no subagent invocation" zero matches) has 3 remaining matches at Steps 3b.2, 6.3, 7.3 — these are cluster evaluation headers ("No subagent invocation. The orchestrator evaluates the [X] cluster result directly:") describing the evaluation decision, not merge operations. These are out-of-scope per design §3.9 which only targets merge step body text.
