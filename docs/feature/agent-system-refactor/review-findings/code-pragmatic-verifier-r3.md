# Adversarial Review: code — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🟡
- **Round:** 1 (Run 3)
- **Run ID:** 2026-03-09T18:00:00Z

## Security Analysis

**Category Verdict:** approve

No security findings. Trust boundaries are correctly enforced:

- **Tool restriction per trust tier:** T1 orchestrator has full tools + `agent`; T2 implementer/tester have `runInTerminal` but no `fetch`; T3 researcher/architect have `fetch` gated by `web_research_enabled` but no `runInTerminal`; knowledge has read + `createFile` only (no `editFiles`, no `runInTerminal`).
- **`fetch` gating:** Both architect (Step 4 + Constraint) and researcher (Workflow Step 4 + Constraint) correctly gate `fetch` on `web_research_enabled=true`.
- **Git safety:** global-rules.md Git Safety section correctly restricts write-mode git to orchestrator only. Implementer, tester, and reviewer all reference this prohibition.
- **Tool name verification:** All tool names across all 8 agent files verified against the official [VS Code cheat sheet](https://code.visualstudio.com/docs/copilot/reference/copilot-vscode-features#_chat-tools) (fetched 2026-03-09). Every tool listed (`readFile`, `createFile`, `editFiles`, `listDirectory`, `textSearch`, `fileSearch`, `codebase`, `runInTerminal`, `getTerminalOutput`, `problems`, `changes`, `todos`, `fetch`, `agent`) is a valid VS Code built-in tool.

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Plan Refinement loop path misdescribed in global-rules.md

- **Severity:** Minor
- **Description:** global-rules.md Feedback Loop Limits table describes the Plan Refinement loop path as "Planner → Reviewer → Planner". The actual path is "Planner → (Orchestrator mediation / User feedback) → Planner". No reviewer is involved in plan refinement — the orchestrator reads `plan_summary`, presents Accept/Refine/Reject to the user, and re-dispatches the planner with feedback if refined.
- **Affected artifacts:** `v2/.github/agents/global-rules.md` (Feedback Loop Limits table)
- **Recommendation:** Change the Plan Refinement path from "Planner → Reviewer → Planner" to "Planner → Orchestrator → Planner" to match the actual workflow described in both planner.agent.md and orchestrator.agent.md Step 4.
- **Evidence:** Orchestrator Step 4: "Dispatch single planner. **Mediation (interactive):** Read `plan_summary`... Present Accept / Refine / Reject choice. Refine → re-dispatch planner with feedback (max 1 round per global-rules.md)." Planner: "orchestrator reads `plan_summary` and, in interactive mode, presents Accept/Refine/Reject to the user." Neither mentions a reviewer in the loop.

### Finding A-2: Single § cross-reference in implementer violates v2 convention

- **Severity:** Minor
- **Description:** The v2 system was designed with zero cross-file section references (§). The implementer.agent.md line 49 contains "see global-rules.md § Git Safety" which introduces a § section reference. global-rules.md itself states: "Read this document in full — no section references needed."
- **Affected artifacts:** `v2/.github/agents/implementer.agent.md` line 49
- **Recommendation:** Change "see global-rules.md § Git Safety" to "see global-rules.md" to align with the v2 zero-§ convention.
- **Evidence:** grep for § in `v2/.github/agents/` returns exactly 1 match: implementer.agent.md line 49.

## Correctness Analysis

**Category Verdict:** approve

### Finding C-1: Tester and Reviewer audit patterns don't cover implementer's full command allowlist

- **Severity:** Major
- **Description:** The tester's static evaluation (Step 3) and reviewer's command allowlist audit both use the pattern "dotnet build|test, npm run build|test, cargo build|test, go build|test, pytest, git diff|status" to audit implementer commands. However, the implementer's actual allowlist (Constraint 1) includes three additional legitimate commands not covered by these patterns:
  1. `dotnet run` (build verification only) — not matched by "dotnet build|test"
  2. `python -m pytest` — not matched by "pytest"
  3. `npm test` — not matched by "npm run build|test"

  This creates a risk of false-positive violations: the tester or reviewer may flag legitimate implementer commands as violations, causing incorrect NEEDS_REVISION or `request-changes` returns.

- **Affected artifacts:**
  - `v2/.github/agents/tester.agent.md` line 45 (static audit patterns)
  - `v2/.github/agents/reviewer.agent.md` line 89 (command audit patterns)
  - `v2/.github/agents/implementer.agent.md` lines 81-86 (actual allowlist)
- **Recommendation:** Update both the tester's static audit pattern and the reviewer's command audit pattern to exactly mirror the implementer's allowlist: "dotnet build|test|run, npm run build|test, npm test, cargo build|test, go build|test, pytest, python -m pytest, git diff|status". This eliminates ambiguity and prevents false positives.
- **Evidence:** Implementer Constraint 1 explicitly lists: `dotnet build`, `dotnet test`, `dotnet run` (build verification only), `npm run build`, `npm run test`, `npm test`, `cargo build`, `cargo test`, `go build`, `go test`, `pytest`, `python -m pytest`, `git diff`, `git status`. Tester audit (line 45) and reviewer audit (line 89) both list the abbreviated pattern.
- **Practical mitigation note:** AI agents may use contextual judgment and reference the implementer's own allowlist when auditing, reducing false positive likelihood. However, the instruction text is the primary guidance — alignment eliminates ambiguity.

### Finding C-2: AC-9 and AC-11 spec-implementation gap (structured question tool)

- **Severity:** Minor
- **Description:** The spec defines AC-9: "Architect tools: frontmatter includes a structured question tool" and AC-11: "Planner tools: frontmatter includes the structured question tool." Neither architect nor planner has such a tool in their YAML frontmatter. Design decision D-3 explicitly addresses this: "VS Code subagents cannot prompt users... Orchestrator-mediated pattern is the only approach that works within the platform." The implementation follows the design (orchestrator-mediated questioning via `clarifications_needed[]` and `plan_summary`), which is architecturally correct. The spec ACs are technically unsatisfied but the design provides a superior alternative.
- **Affected artifacts:** `v2/.github/agents/architect.agent.md` (tools list), `v2/.github/agents/planner.agent.md` (tools list)
- **Recommendation:** No code change needed. Document in the evidence bundle that AC-9 and AC-11 are satisfied in intent via the orchestrator-mediated pattern (D-3), not by literal tool inclusion. Update spec AC text in future runs if the spec is revisited.
- **Evidence:** Architect tools: readFile, listDirectory, textSearch, codebase, fileSearch, fetch, createFile — no structured question tool. Planner tools: readFile, listDirectory, textSearch, codebase, fileSearch, createFile — no structured question tool. Design D-3: "VS Code subagents cannot prompt users directly." Official VS Code docs (fetched 2026-03-09) confirm no built-in structured question tool exists for subagents.

### Finding C-3: Implementation reports show git add -A despite git safety rule

- **Severity:** Minor
- **Description:** Task-07 implementation report (reviewer.agent.md changes) lists `git add -A` in `commands_executed[]`. Code-review-fixes-r1 also lists `git add -A`. This violates the git safety rule (AC-12, AC-13) which states only the orchestrator may run git staging commands. The agent FILES correctly prohibit this behavior — the issue is that the implementing AI agent violated its own instructions during the pipeline run.
- **Affected artifacts:** `docs/feature/agent-system-refactor/implementation-reports/task-07.yaml` (line 61), `docs/feature/agent-system-refactor/implementation-reports/code-review-fixes-r1.yaml` (line 17)
- **Recommendation:** This is a pipeline execution issue, not a code defect — the agent instructions are correct. For future runs, the tester static audit and reviewer command audit should catch this violation (ironically, this proves C-1 is important — the audit mechanisms need to be reliable).
- **Evidence:** task-07.yaml commands_executed includes `git add -A`. code-review-fixes-r1.yaml commands_executed includes `git add -A`. Implementer line 49: "Do NOT run `git add`". global-rules.md Git Safety: "All other agents MUST NOT run `git add`".

### Finding C-4: Implementation report line counts inconsistent with actual file lengths

- **Severity:** Minor
- **Description:** Multiple implementation reports self-report file line counts that differ from measured values:
  - task-05 reports planner at "105→120 lines"; actual measured: 112 lines
  - task-02 verification says orchestrator at 93 lines; actual measured: 119 lines
  - code-review-fixes-r1 reports orchestrator at 89 lines post-fix; actual measured: 119 lines

  These discrepancies likely arise from multiple tasks modifying the same file across different pipeline stages, with each report measuring at different points. All files remain under 150 lines, so AC-27 is satisfied.

- **Affected artifacts:** Implementation reports task-02.yaml, task-05.yaml, code-review-fixes-r1.yaml
- **Recommendation:** Consider adding post-pipeline line count verification as a Step 8 sub-step to confirm final line counts after all modifications.
- **Evidence:** Terminal output from `Get-ChildItem ... | ForEach-Object { "$($_.Name): $((Get-Content $_.FullName).Count) lines" }` shows current counts: architect 139, implementer 104, knowledge 118, orchestrator 119, planner 112, researcher 84, reviewer 97, tester 132, global-rules 77.

## Summary

The v2 agent system refactor correctly implements all 10 improvements (FR-1 through FR-10) across 9 files with all files under 150 lines. All 27 acceptance criteria are satisfied in substance (AC-9/AC-11 satisfied via orchestrator-mediated design pattern rather than literal tool inclusion). Tool names verified against official VS Code documentation — all correct. One Major finding: the tester and reviewer audit patterns don't cover the implementer's full command allowlist, creating false-positive violation risk for `dotnet run`, `python -m pytest`, and `npm test`. Four Minor findings address cross-file consistency (loop path, § convention, git add process violation, line count reporting). Overall: solid implementation with one actionable pattern alignment fix recommended.
