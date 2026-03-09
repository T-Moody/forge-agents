# Adversarial Review: code — security-sentinel (Run 3, Round 1)

## Review Metadata

- **Perspective:** security-sentinel
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-09T18:00:00Z

## Prior Round Resolution Status

All findings from Run 2 Round 2 were resolved. This is Run 3 — a fresh review of the current staged code.

| R2 ID | Severity | Status      | Notes                                                       |
| ----- | -------- | ----------- | ----------------------------------------------------------- |
| S-1   | Major    | ✅ Resolved | fetch removed from implementer (verified absent in current) |
| S-2   | Major    | ✅ Resolved | Tester command allowlist present                            |
| S-3   | Major    | ✅ Resolved | Architect web_research_enabled gate present                 |
| C-1   | Minor    | Carried     | Tester schema still lacks own commands_executed — see C-2   |

## Security Analysis

**Category Verdict:** approve

### Finding S-1: Git safety enforcement is advisory-only — no deterministic prevention

- **Severity:** Minor
- **Description:** Both the implementer and tester have `runInTerminal` in their tool lists, which means they can technically execute any shell command including `git add`, `git commit`, `git push`, etc. The git safety constraint is enforced solely through instruction text (implementer constraint #1 allowlist, global-rules.md Git Safety section, anti-drift anchors) and post-hoc audit (tester static mode step 3, reviewer command allowlist audit). There is no deterministic tool-level prevention of git write commands.
- **Affected artifacts:** v2/.github/agents/implementer.agent.md (tools line 7: `runInTerminal`), v2/.github/agents/tester.agent.md (tools line 9: `runInTerminal`), v2/.github/agents/global-rules.md (Git Safety section)
- **Recommendation:** This is a known accepted limitation. Design decision D-11 correctly rejected `disable-model-invocation` (would break pipeline) and Direction B (hooks, Preview feature). VS Code terminal sandboxing is only available on macOS/Linux (not Windows). The current advisory model with multi-layer enforcement (agent instruction + reviewer audit + tester audit) is the strongest practical option. No code change needed — documenting for awareness.
- **Evidence:** VS Code docs confirm: "Terminal sandboxing is currently in preview and is only supported on macOS and Linux. On Windows, the sandbox settings have no effect." VS Code `chat.tools.terminal.autoApprove` can restrict commands but only controls approval prompts, not programmatic execution from subagents. The advisory model is reinforced at 3 layers: (1) implementer body text git safety box, (2) global-rules.md shared rule, (3) reviewer/tester audit of commands_executed.

### Verification of Tool Names Against VS Code Official Documentation

All 9 agent files were verified against the VS Code Copilot cheat sheet (fetched 2026-03-09). Every tool name in every YAML frontmatter `tools:` array matches an official built-in tool name:

| Tool Name           | Used By                           | Official? |
| ------------------- | --------------------------------- | --------- |
| `readFile`          | All 8 agents                      | ✅        |
| `createFile`        | All 8 agents                      | ✅        |
| `editFiles`         | orchestrator, implementer         | ✅        |
| `listDirectory`     | All 8 agents                      | ✅        |
| `textSearch`        | All 8 agents                      | ✅        |
| `fileSearch`        | All 8 agents                      | ✅        |
| `codebase`          | All except orchestrator           | ✅        |
| `runInTerminal`     | orchestrator, implementer, tester | ✅        |
| `getTerminalOutput` | orchestrator, implementer, tester | ✅        |
| `problems`          | implementer, tester, reviewer     | ✅        |
| `changes`           | orchestrator, reviewer            | ✅        |
| `todos`             | orchestrator                      | ✅        |
| `fetch`             | architect, researcher             | ✅        |
| `agent` (tool set)  | orchestrator                      | ✅        |

Zero invalid tool names. Zero misspellings. Zero silently-ignored restrictions.

### Trust Boundary Verification

| Agent        | Tier | `agents:` | `user-invocable` | `runInTerminal` | `fetch` | `editFiles` |
| ------------ | ---- | --------- | ---------------- | --------------- | ------- | ----------- |
| orchestrator | T1   | 7 agents  | (default true)   | ✅              | ❌      | ✅          |
| implementer  | T2   | `[]`      | `false`          | ✅              | ❌      | ✅          |
| tester       | T2   | `[]`      | `false`          | ✅              | ❌      | ❌          |
| architect    | T3   | `[]`      | `false`          | ❌              | ✅†     | ❌          |
| planner      | —    | `[]`      | `false`          | ❌              | ❌      | ❌          |
| reviewer     | T2   | `[]`      | `false`          | ❌              | ❌      | ❌          |
| knowledge    | —    | `[]`      | `false`          | ❌              | ❌      | ❌          |
| researcher   | T3   | `[]`      | `false`          | ❌              | ✅†     | ❌          |

† `fetch` gated by `web_research_enabled` parameter (default: false).

No worker can dispatch subagents. No worker can bypass the orchestrator. `fetch` is properly gated on both agents that have it.

### `disable-model-invocation` Decision Validation

Design decision D-11 correctly rejected adding `disable-model-invocation: true` to workers. VS Code docs confirm: "Optional boolean flag to prevent the agent from being invoked as a subagent by other agents (default is false)." Adding it would prevent the orchestrator from dispatching workers via `runSubagent`, breaking the pipeline.

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Tester/reviewer command audit patterns narrower than implementer allowlist

- **Severity:** Minor
- **Description:** The implementer's command allowlist (constraint #1) permits `dotnet run`, `npm test`, and `python -m pytest`. However, both the tester's static audit (workflow step 3) and the reviewer's command allowlist audit check against narrower patterns: `dotnet build|test`, `npm run build|test`, `cargo build|test`, `go build|test`, `pytest`, `git diff|status`. Three implementer-allowed commands would generate false-positive violations: (1) `dotnet run` — not in `dotnet build|test`, (2) `npm test` — not in `npm run build|test`, (3) `python -m pytest` — not in `pytest`.
- **Affected artifacts:** v2/.github/agents/tester.agent.md (static workflow step 3), v2/.github/agents/reviewer.agent.md (command allowlist audit constraint), v2/.github/agents/implementer.agent.md (constraint #1)
- **Recommendation:** Align the tester and reviewer audit patterns to cover all implementer-permitted commands. Add `dotnet run`, `npm test`, and `python -m pytest` to the expected patterns. This prevents false-positive Major findings during pipeline operation.
- **Evidence:** Implementer constraint #1 lists 6 command groups. Tester step 3 and reviewer constraint check 5 compressed patterns. The gap was introduced because the tester/reviewer patterns use pipe-delimited shorthand while the implementer uses an expanded list.

### Finding A-2: Knowledge agent doc-update mode cannot modify files (no editFiles tool)

- **Severity:** Minor
- **Description:** The knowledge agent's doc-update mode (Step 8b) instructs it to "update documentation" and "Propose targeted edits as diffs for user review." However, the knowledge agent's tool list includes only `createFile` — not `editFiles`. This means the agent can create new files containing proposed diffs but cannot actually apply edits to existing documentation files (copilot-instructions.md, .instructions.md, AGENTS.md). The orchestrator Step 8b "Apply" codepath is unreachable.
- **Affected artifacts:** v2/.github/agents/knowledge.agent.md (tools list: no editFiles; doc-update mode section)
- **Recommendation:** From a security perspective, this is actually a positive constraint — the knowledge agent cannot accidentally corrupt instruction files even if doc-update mode instructions are misinterpreted. The "Apply" codepath should be clarified: either (a) add `editFiles` to knowledge tools (increases risk surface) or (b) reword doc-update mode to explicitly state it produces a recommendation file only, and the orchestrator or user applies changes manually. Option (b) is more security-conscious.
- **Evidence:** Knowledge agent YAML frontmatter tools: `readFile`, `listDirectory`, `textSearch`, `codebase`, `fileSearch`, `createFile`. No `editFiles`. Doc-update mode step 3: "Propose targeted edits as diffs for user review" — the word "propose" aligns with create-only, but the orchestrator Step 8b "Apply" option implies execution.

## Correctness Analysis

**Category Verdict:** approve

### Finding C-1: Code-review-fixes-r1 implementation report records git add -A in commands_executed

- **Severity:** Minor
- **Description:** The `code-review-fixes-r1.yaml` implementation report lists `git add -A` in its `commands_executed` array. This is a violation of the git safety rule (global-rules.md Git Safety, FR-5) that was being implemented by this very fix round. This is a bootstrap problem — the implementer added the "no git add" rule while needing to stage the fixes. Future pipeline runs with a tester static audit reading this historical report would flag this as a Major finding.
- **Affected artifacts:** docs/feature/agent-system-refactor/implementation-reports/code-review-fixes-r1.yaml (commands_executed line 2: `git add -A`)
- **Recommendation:** This is a historical artifact from the Run 2 fix cycle, not a live security risk. The v2 agent definitions themselves are correct. No code change needed for the v2 files under review — this is a pipeline artifact correctness note.
- **Evidence:** code-review-fixes-r1.yaml commands_executed: `["Get-Content ... | Measure-Object -Line (7 files — line count verification)", "git add -A"]`. Global-rules.md Git Safety: "All other agents MUST NOT run `git add`..."

### Finding C-2: Tester output schema lacks commands_executed field for own terminal activity (carried from R2)

- **Severity:** Minor
- **Description:** The tester's output schema includes `command_allowlist_audit` (which audits the implementer's commands from implementation reports) but does not include a `commands_executed[]` field for the tester's own terminal commands. In dynamic mode, the tester executes multiple terminal commands (app startup, test runs, shutdown) that have no designated schema field. The tester's constraint says "All commands must be logged in the test report" but the schema has no field for this. The reviewer's command audit (workflow step 3) only reads `commands_executed[]` from implementation reports — not test reports.
- **Affected artifacts:** v2/.github/agents/tester.agent.md (output schema payload section vs constraint section)
- **Recommendation:** Add `commands_executed: ["..."]` to the tester output schema payload. Extend reviewer workflow step 3 to also audit tester commands from test reports. This is advisory — the constraint text is correct; only the schema needs alignment.
- **Evidence:** Tester output schema payload fields: `mode`, `test_categories`, `tdd_verification`, `command_allowlist_audit`, `file_ownership_audit`, `exploratory_findings`, `verdict`. No `commands_executed` field. Tester constraint: "All commands must be logged in the test report." Carried from R2 C-1 (Minor).

## Summary

Run 3 Round 1 security-sentinel code review of all 9 v2 agent files confirms: all tool names are valid per VS Code official docs (verified via fetch), trust boundaries are correctly enforced (no worker can dispatch subagents, fetch is gated, no worker has both editFiles and runInTerminal except implementer), git safety is instruction-level only (accepted limitation — no deterministic enforcement possible on Windows), and D-11 correctly rejected disable-model-invocation. 5 Minor findings across 3 categories, zero blockers/criticals/majors. Verdict: **approve**.
