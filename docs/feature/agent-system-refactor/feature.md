# Agent System Refactor — v2 Targeted Improvements (Run 3)

## Title & Summary

**Feature:** 10 Targeted Improvements to v2 Multi-Agent System

Surgical, bounded modifications to the existing v2 agent files (`v2/.github/agents/`) addressing tool name correctness, interactive mode capabilities, git safety, TDD best practices, exploratory QA, and documentation updates. No new agent files; all changes within existing 150-line budgets.

**Selected Direction:** A (Targeted In-Place Edits)

---

## Background & Context

The v2 multi-agent system was built in Run 2 as a clean-slate rebuild. It consists of 8 agents (~1,004 lines) with completion-contract routing, zero SQLite, and 150-line per-agent budgets. Run 3 addresses 10 specific improvements identified through operational experience and research.

### Research Sources

- [architecture.yaml](research/architecture.yaml) — Tool name analysis, subagent dispatch, VS Code frontmatter gaps
- [impact.yaml](research/impact.yaml) — Risk/value assessment per improvement
- [dependencies.yaml](research/dependencies.yaml) — Tool mapping, skill dependencies, MCP, hooks
- [patterns.yaml](research/patterns.yaml) — VS Code ecosystem, TourneyPal QA patterns, TDD cycles
- [tool-names-deep-dive.yaml](research/tool-names-deep-dive.yaml) — Complete tool name mapping, silent failure analysis

### Key Finding

All v2 tool names use invalid `namespace/tool` format (e.g., `read/readFile`). VS Code silently ignores unrecognized tool identifiers, meaning ALL tool restrictions are currently non-functional. This is the highest-priority fix.

---

## Functional Requirements

### FR-1: E2E Testing Skill Setup Guidance

The orchestrator scans `.github/skills/` at Step 1 (Setup) for E2E-related SKILL.md files. In interactive mode, if none found, presents a multiple-choice question (Playwright CLI / HTTP API / Skip / Custom). In autonomous mode, logs a warning. The tester references discovered skills by path rather than hardcoded names.

### FR-2: Correct All Tool Names (CRITICAL)

Replace ALL `namespace/tool` format tool names with correct flat camelCase across all 8 agent files:

| Invalid (Current)           | Correct             |
| --------------------------- | ------------------- |
| `read/readFile`             | `readFile`          |
| `edit/createFile`           | `createFile`        |
| `edit/editFiles`            | `editFiles`         |
| `search/listDirectory`      | `listDirectory`     |
| `search/textSearch`         | `textSearch`        |
| `search/fileSearch`         | `fileSearch`        |
| `search/codebase`           | `codebase`          |
| `execute/runInTerminal`     | `runInTerminal`     |
| `execute/getTerminalOutput` | `getTerminalOutput` |
| `web/fetch`                 | `fetch`             |
| `read/problems`             | `problems`          |
| `search/changes`            | `changes`           |
| `todo`                      | `todos`             |
| `vscode/memory`             | _(remove)_          |

Additionally: orchestrator retains `agent` tool set + `agents:` frontmatter, body text instructs use of `runSubagent`, all workers set `user-invocable: false`.

### FR-3: Architect Clarifying Questions

Add a clarification step between 'Read Inputs' and 'Analyze Codebase'. In interactive mode, architect presents up to 5 clarifying questions when ambiguities exist. In autonomous mode, documents assumptions instead. Requires structured question tool in frontmatter.

### FR-4: Planner Accept/Refine

Add a final step presenting plan summary with 3 options: Accept, Refine (with freeform input), Reject. Gated by interactive mode. Requires structured question tool in frontmatter.

### FR-5: Git Safety for Subagents

Remove `git add -A` and `git commit` from implementer. Update implementer allowlist to permit only `git diff` and `git status`. Add git-only-orchestrator rule to `global-rules.md`. Orchestrator stages files at Step 8 using selective pathspecs.

### FR-6: No Autocommit at Pipeline End

Orchestrator stages files at Step 8 but does NOT auto-commit. Interactive: present Commit/Review/Unstage choice. Autonomous: stage only, leave for developer review.

### FR-7: Optional Documentation Update

After knowledge agent completes in interactive mode, present Apply/Review/Skip choice. If 'Apply', re-dispatch knowledge in doc-update mode targeting only `.instructions.md`, `copilot-instructions.md`, and `AGENTS.md` files. Never modify `.agent.md` files. Skip entirely in autonomous mode.

### FR-8: Architect Web Research Step

Add explicit 'Web Research' step between 'Analyze Codebase' and 'Identify Requirements', gated by `web_research_enabled=true`. Capped at 3-5 targeted lookups. Fix tool name from `web/fetch` to `fetch`.

### FR-9: Tester Exploratory QA

Add 'Exploratory QA' phase to tester dynamic mode after automated tests. Covers: happy path, edge cases, error handling, state mutation. Uses available skills (Playwright CLI, HTTP API) when present. Findings recorded with category 'exploratory' and severity ratings. Extract detailed procedures to optional SKILL.md if needed to stay within 150-line budget.

### FR-10: Implementer TDD Best Practices

Change cycle from RED-GREEN-REPORT to RED-GREEN-REFACTOR-REPORT. REFACTOR is bounded: only task files, only >=3 duplicates, only clarity renames. Must re-run tests after refactoring; revert if tests fail. Add code quality principles: YAGNI, KISS, DRY, prefer functional, avoid over-engineering, minimal code, lint compatibility.

---

## Non-Functional Requirements

- **NFR-1:** Each agent file ≤150 lines after all changes
- **NFR-2:** Total system ≤1,500 lines (currently ~1,004)
- **NFR-3:** Interactive features degrade gracefully in autonomous mode (no prompts, sensible defaults)
- **NFR-4:** Zero new runtime dependencies (no SQLite, no npm packages, no external services)
- **NFR-5:** Tool restrictions become functional (currently silently ignored due to wrong names)

---

## Constraints & Assumptions

| Constraint                    | Detail                                                      |
| ----------------------------- | ----------------------------------------------------------- |
| No new agent files            | Only modify existing 8 agents + global-rules.md + 2 prompts |
| New skills permitted          | SKILL.md files may be created under `.github/skills/`       |
| Pipeline structure frozen     | 8 steps unchanged                                           |
| Completion contracts frozen   | DONE/NEEDS_REVISION/ERROR routing unchanged                 |
| Zero SQLite                   | File-based state only                                       |
| VS Code hooks are Preview     | Direction B (hooks enforcement) noted but not required      |
| `vscode/memory` is not a tool | Remove from all frontmatter; memory is a VS Code setting    |
| `agent` is a valid tool set   | Orchestrator keeps `agent` for subagent dispatch            |

---

## Acceptance Criteria

| ID    | Criterion                                                                          | Method     |
| ----- | ---------------------------------------------------------------------------------- | ---------- |
| AC-1  | Orchestrator Step 1 scans skill directories for E2E SKILL.md files                 | Inspection |
| AC-2  | Interactive mode: orchestrator presents E2E setup choice (≥3 options)              | Inspection |
| AC-3  | Autonomous mode: warning logged, no interactive prompt                             | Inspection |
| AC-4  | Zero `namespace/tool` format names remain across all 8 agent files                 | Inspection |
| AC-5  | Orchestrator tools: contains `agent`, agents: lists 7 workers                      | Inspection |
| AC-6  | Orchestrator body instructs use of `runSubagent` by name                           | Inspection |
| AC-7  | All 7 workers have `user-invocable: false`                                         | Inspection |
| AC-8  | Architect has clarification step gated by interactive mode                         | Inspection |
| AC-9  | Architect tools: includes structured question tool                                 | Inspection |
| AC-10 | Planner has Accept/Refine/Reject step gated by interactive mode                    | Inspection |
| AC-11 | Planner tools: includes structured question tool                                   | Inspection |
| AC-12 | Implementer has no `git add -A`, `git add .`, or `git commit`                      | Inspection |
| AC-13 | global-rules.md has git-only-orchestrator rule                                     | Inspection |
| AC-14 | Orchestrator Step 8 uses selective pathspecs, not `git add -A`                     | Inspection |
| AC-15 | Orchestrator Step 8 has no unconditional `git commit`; interactive offers choice   | Inspection |
| AC-16 | Autonomous mode: orchestrator stages but does not commit                           | Inspection |
| AC-17 | Interactive: orchestrator offers Apply/Review/Skip for doc updates after knowledge | Inspection |
| AC-18 | Knowledge agent constraints prohibit modifying .agent.md in doc-update mode        | Inspection |
| AC-19 | Architect has explicit Web Research step gated by web_research_enabled             | Inspection |
| AC-20 | Architect fetch tool name is `fetch` (not `web/fetch`)                             | Inspection |
| AC-21 | Tester dynamic mode has Exploratory QA phase with 4 categories                     | Inspection |
| AC-22 | Exploratory findings recorded with category 'exploratory' + severity               | Inspection |
| AC-23 | Tester agent file ≤150 lines                                                       | Inspection |
| AC-24 | Implementer has REFACTOR step between GREEN and REPORT                             | Inspection |
| AC-25 | Implementer constraints list YAGNI, KISS, DRY, functional, minimal                 | Inspection |
| AC-26 | REFACTOR requires re-running tests; revert on failure                              | Inspection |
| AC-27 | All 8 agent files ≤150 lines each                                                  | Inspection |

---

## Edge Cases & Error Handling

| ID   | Condition                                                        | Expected Behavior                           | Severity if Missed |
| ---- | ---------------------------------------------------------------- | ------------------------------------------- | ------------------ |
| EC-1 | Skills directory exists but contains no SKILL.md files           | Treat as 'no skills found'; prompt or warn  | Medium             |
| EC-2 | User selects 'Refine' for plan approval with empty input         | Re-present the question                     | Low                |
| EC-3 | Architect finds zero ambiguities in interactive mode             | Skip clarification silently, proceed        | Low                |
| EC-4 | REFACTOR step causes test regressions                            | Auto-revert, use pre-refactor code          | High               |
| EC-5 | Parallel implementers run `git status` simultaneously            | Permitted (read-only, no conflict)          | Low                |
| EC-6 | Knowledge re-dispatched for doc-update but recommendations empty | Return DONE with 'No updates needed'        | Low                |
| EC-7 | Exploratory QA finds critical bug but no Playwright skill        | Record finding with terminal-based evidence | Medium             |

---

## User Stories / Flows

### Story 1: Developer starts a feature pipeline (interactive)

1. Developer invokes `@orchestrator` with a feature request
2. Orchestrator Step 1 scans for E2E skills, finds none, asks developer to choose setup
3. Developer selects "Playwright CLI skill"
4. Architect reads inputs, asks 2 clarifying questions, receives answers
5. Architect performs web research on relevant framework docs
6. Planner produces plan, presents summary, developer accepts
7. Implementer writes tests (RED), implementation (GREEN), refactors (REFACTOR), reports (REPORT)
8. Tester runs automated tests + exploratory QA
9. Orchestrator stages files, asks developer: "Commit now?"
10. Developer selects "Review changes first"

### Story 2: CI pipeline (autonomous)

1. Pipeline invoked with `approval_mode=autonomous`
2. Orchestrator scans for E2E skills, finds Playwright SKILL.md, proceeds
3. Architect skips questions, documents assumptions
4. Planner produces plan, skips approval
5. Implementer follows RED-GREEN-REFACTOR-REPORT
6. Orchestrator stages files, does NOT commit (autonomous safety)

---

## Test Scenarios

| #    | Scenario                                                              | Linked ACs   |
| ---- | --------------------------------------------------------------------- | ------------ |
| T-1  | Verify all tool names are flat camelCase via grep across 8 files      | AC-4, AC-20  |
| T-2  | Verify no `git add` or `git commit` in implementer.agent.md           | AC-12        |
| T-3  | Verify global-rules.md contains git-only-orchestrator rule            | AC-13        |
| T-4  | Verify all workers have `user-invocable: false`                       | AC-7         |
| T-5  | Verify orchestrator has `agent` in tools and 7 agents in agents:      | AC-5         |
| T-6  | Verify architect has clarification step with interactive gate         | AC-8, AC-9   |
| T-7  | Verify planner has approval step with 3 options                       | AC-10, AC-11 |
| T-8  | Verify tester has exploratory QA phase with 4 categories              | AC-21, AC-22 |
| T-9  | Verify implementer has REFACTOR step with revert guard                | AC-24, AC-26 |
| T-10 | Verify all 8 files ≤150 lines via `wc -l`                             | AC-23, AC-27 |
| T-11 | Verify orchestrator Step 8 has no auto-commit, has interactive choice | AC-15, AC-16 |
| T-12 | Verify knowledge agent doc-update constraints exclude .agent.md       | AC-18        |

---

## Dependencies & Risks

### Dependencies

| Component              | Dependency                               | Risk                        |
| ---------------------- | ---------------------------------------- | --------------------------- |
| FR-2 (tool names)      | VS Code built-in tool reference accuracy | Low — verified against docs |
| FR-1, FR-9 (skills)    | .github/skills/ directory convention     | Low — VS Code standard      |
| FR-3, FR-4 (questions) | Structured question tool availability    | Low — standard VS Code tool |
| FR-8 (web research)    | `fetch` tool in VS Code                  | Low — confirmed available   |
| FR-7 (doc update)      | Knowledge agent re-dispatch pattern      | Medium — new dispatch mode  |

### Risks

| Risk                                                      | Likelihood | Impact | Mitigation                                                                   |
| --------------------------------------------------------- | ---------- | ------ | ---------------------------------------------------------------------------- |
| 150-line budget exceeded for tester (FR-9)                | Medium     | Low    | Extract to SKILL.md                                                          |
| Orchestrator Step 8 becomes complex (FR-5, FR-6)          | Medium     | Low    | Keep conditional logic minimal                                               |
| `vscode/memory` removal breaks knowledge agent intent     | Low        | Medium | Memory is VS Code setting, not tool; agent uses readFile/editFiles for files |
| Hooks (Direction B) may change in future VS Code versions | N/A        | N/A    | Direction B not selected; noted as future option                             |
