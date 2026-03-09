# Adversarial Review: code — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-09T18:00:00Z

## Security Analysis

**Category Verdict:** approve

### Finding S-1: Implementation process violated git safety rule being implemented

- **Severity:** Minor
- **Description:** Task-07 (`commands_executed`) and code-review-fixes-r1 (`commands_executed`) both log `git add -A` — the exact operation that FR-5/D-5 prohibits for non-orchestrator agents. While the resulting agent instruction files correctly prohibit `git add` for implementers, the implementation process itself did not adhere to the rule it was establishing. All 10 task reports also show `self_check.git_staged: true`, indicating staging occurred across every implementation task.
- **Affected artifacts:** `docs/feature/agent-system-refactor/implementation-reports/task-07.yaml` (line 61), `docs/feature/agent-system-refactor/implementation-reports/code-review-fixes-r1.yaml` (line 17)
- **Recommendation:** Future pipeline runs should enforce the git safety rule from the moment task-01 establishes it. The orchestrator should be the sole agent performing `git add` even during self-improvement runs. The `self_check.git_staged` field in implementation reports should be removed or set to `false` since implementers cannot stage.
- **Evidence:** `grep "git add -A" docs/feature/agent-system-refactor/implementation-reports/*.yaml` returns hits in task-07.yaml and code-review-fixes-r1.yaml. `grep "git_staged: true" docs/feature/agent-system-refactor/implementation-reports/task-*.yaml` returns 10 hits across all tasks.

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: global-rules.md auto-detected as custom agent by VS Code

- **Severity:** Minor
- **Description:** VS Code documentation (fetched 2026-03-09) states: "VS Code detects any `.md` files in the `.github/agents` folder of your workspace as custom agents." Since `global-rules.md` resides in `v2/.github/agents/` and has no YAML frontmatter (no `user-invocable: false`), VS Code will detect it as a custom agent named "global-rules" and display it in the agents dropdown with default visibility. This is an unintended user-facing artifact — selecting it would activate an agent with no tools and only global-rules instructions.
- **Affected artifacts:** `v2/.github/agents/global-rules.md`
- **Recommendation:** Add minimal YAML frontmatter to `global-rules.md`: `---\nname: global-rules\ndescription: "Shared rules — not a standalone agent"\nuser-invocable: false\ntools: []\nagents: []\n---`. This prevents it from appearing in the agents dropdown while keeping it in the same directory for easy agent-file referencing.
- **Evidence:** VS Code custom agents docs: "VS Code detects any `.md` files in the `.github/agents` folder of your workspace as custom agents." File has no YAML frontmatter — confirmed by reading lines 1-5 of `v2/.github/agents/global-rules.md` (starts with `# Global Rules`, no `---` delimiter).

### Finding A-2: Implementation report line counts do not match actual file sizes

- **Severity:** Minor
- **Description:** Actual line counts (verified via `(Get-Content).Count`) diverge from implementation report claims for 2 files: (1) planner.agent.md — task-05 claims 120 lines, actual is 112 (8-line discrepancy). (2) knowledge.agent.md — task-08 claims 117 lines, actual is 118 (1-line gap). All files remain well within the 150-line budget so this is non-blocking, but inaccurate reporting reduces audit trail trustworthiness.
- **Affected artifacts:** `docs/feature/agent-system-refactor/implementation-reports/task-05.yaml`, `docs/feature/agent-system-refactor/implementation-reports/task-08.yaml`
- **Recommendation:** Implementers should capture line count from terminal output and copy the exact number into the report rather than estimating.
- **Evidence:** Terminal verification: `architect.agent.md: 139, global-rules.md: 77, implementer.agent.md: 104, knowledge.agent.md: 118, orchestrator.agent.md: 119, planner.agent.md: 112, researcher.agent.md: 84, reviewer.agent.md: 97, tester.agent.md: 132`. All ≤ 150.

### Architecture Positive Notes

1. **Line budgets respected.** All 9 files are well within the 150-line limit. Architect at 139 lines is tightest (11 headroom). Orchestrator at 119 is comfortable (31 headroom). Total system: 982 lines.
2. **Tool name consistency.** All 8 agent frontmatters use flat camelCase tool names with zero namespace/tool format remaining. 14 unique tool names used consistently across agents.
3. **YAML frontmatter structure valid.** All 8 agent files have correct frontmatter per VS Code docs: `name`, `description`, `tools` (YAML array), `agents` (YAML array). 7 worker agents have `user-invocable: false`. Orchestrator correctly omits it (user-invocable by default).
4. **Step 8 decomposition well-structured.** 8a→8b→8c→8d→8e forms a clear sequential pipeline with single responsibility per sub-step. Interactive/autonomous branching properly isolated to 8b and 8e only.
5. **Agent boundaries clean.** No responsibility overlap found. Trust model intact: T1 (orchestrator), T2 (implementer/tester — terminal access), T3 (researcher/architect — fetch-gated). `fetch` correctly restricted to researcher + architect with `web_research_enabled` gate.
6. **Global rules integration complete.** All 7 worker agents reference global-rules.md in their Constraints section. New Git Safety and Plan Refinement additions in global-rules.md are consistent with agent-level instructions.
7. **No new inter-agent coupling.** New mediation patterns (architect clarifications, planner approval) route through orchestrator — no direct agent-to-agent coupling. DAG dispatch algorithm unchanged.
8. **Naming conventions consistent.** All agent files follow `<name>.agent.md` pattern. Step naming uses "Step N: Name" format. Sub-steps use letter suffixes (8a-8e). YAML keys use snake_case consistently.

## Correctness Analysis

**Category Verdict:** approve

### Finding C-1: Implementer violated git safety rule in task-07 commands_executed

- **Severity:** Major
- **Description:** Task-07's `commands_executed` array contains `"git add -A"` — an explicit violation of the git safety rule (FR-5/D-5/AC-12) that was established by task-01 (completed at 18:05:00Z, 40 minutes before task-07 started at 18:45:00Z). The tester's static verification audit (updated in task-06 to flag `git add` as Major) should have caught this violation when evaluating task-07's implementation report. No tester finding about this was produced, indicating either the tester wasn't dispatched for task-07 or it failed to apply the updated audit pattern.
- **Affected artifacts:** `docs/feature/agent-system-refactor/implementation-reports/task-07.yaml` (line 61: `- "git add -A"`)
- **Recommendation:** (1) The orchestrator should dispatch tester (Mode: static) after every implementation wave including review-fix waves. (2) During self-improvement runs, the orchestrator should be responsible for staging from the first task onward. (3) The `git_staged: true` field in `self_check` should be removed from the implementation report schema for non-orchestrator agents, since they should never stage.
- **Evidence:** task-07.yaml `commands_executed` contains `"git add -A"`. task-01.yaml (completed 18:05:00Z) established git safety rule. task-07 started at 18:45:00Z — 40 minutes after rule was active. No tester static report exists flagging this violation.

### Correctness Positive Notes

1. **All design decisions correctly implemented.** D-1 through D-11 are reflected in the code changes. D-11 (reject disable-model-invocation) correctly avoided — no worker agents have this property.
2. **Completion contracts consistent.** All agents follow the global-rules.md completion contract format (status, summary, output_paths). Planner extends it with `plan_summary` as a compatible addition.
3. **Command allowlist patterns aligned.** Tester (static audit), Reviewer (command audit), and Implementer (command allowlist constraint) all use consistent patterns: `git diff|status` allowed, `git add|commit` prohibited.
4. **Spec deviations documented.** DR-1 and DR-2 in design-output.yaml correctly document that VS Code subagents cannot prompt users, justifying the orchestrator-mediated pattern.

## Summary

The agent-system-refactor implementation is architecturally sound. All 9 files respect line budgets (982 total, tightest is architect at 139/150). Tool names are consistently flat camelCase across all agents. YAML frontmatter structure matches VS Code official docs. Step 8 decomposition into 8a-8e is clean with proper single-responsibility sub-steps. Agent boundaries remain clean with no new coupling. One Major correctness finding (git add -A in task-07 violating the established git safety rule with no tester catch) and two Minor architecture findings (global-rules.md agent detection, report line count drift). Overall assessment: approve with advisory findings.
