# Adversarial Review: code — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-08T12:00:00Z

## Security Analysis

**Category Verdict:** approve

### Finding S-1: `fetch_webpage` gating inconsistency across agents

- **Severity:** Minor
- **Description:** The Researcher agent explicitly gates `fetch_webpage` on the `web_research_enabled` parameter (line 40: "Do NOT use fetch_webpage when web_research_enabled is absent or false"). However, the Architect and Implementer agents both include `fetch_webpage` in their YAML frontmatter tools list but their instruction bodies contain no mention of `web_research_enabled`. The Architect's only constraint is "fetch_webpage is for API docs and framework references only — not general browsing." The Implementer has no `fetch_webpage` usage guidance at all. This means in autonomous mode (where VS Code does not prompt for approval), these agents could use web research regardless of the user's preference set at Setup. The design (D-10) acknowledges this as accepted residual risk, and the Reviewer audits URL trails as mitigation.
- **Affected artifacts:** [architect.agent.md](v2/.github/agents/architect.agent.md), [implementer.agent.md](v2/.github/agents/implementer.agent.md)
- **Recommendation:** Add a constraint to both Architect and Implementer: "Only use `fetch_webpage` when `web_research_enabled` is `true` (passed by orchestrator). Do not use `fetch_webpage` when web research is disabled." This matches the Researcher's pattern and costs ~2 lines per agent.
- **Evidence:** Researcher line 40 gates explicitly; Architect line 125 mentions API docs only; Implementer has zero `fetch_webpage` usage guidance in body.

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: Quick-fix pipeline fundamentally broken — no task definition mechanism

- **Severity:** Major
- **Description:** The quick-fix pipeline (`quick-fix.prompt.md`) explicitly skips Steps 2 (Research), 3 (Architecture), and 4 (Planning). However, the Implementer agent's workflow begins with "Read task definition. Parse `tasks/task-XX.yaml`" (line 36) — this file is produced exclusively by the Planner at Step 4. The Orchestrator's Step 5 (Implementation) says "Execute DAG dispatch algorithm: Read plan-output.yaml → tasks" (line 86). Without Planning, neither `plan-output.yaml` nor any `tasks/task-XX.yaml` files exist. The quick-fix pipeline would fail at Step 5 because the Orchestrator cannot find tasks to dispatch and the Implementer has no input to work from. Neither the quick-fix prompt, the Orchestrator, nor the Implementer defines a "direct mode" or fallback for operating without plan artifacts. Design decision D-13 specifies steps {1, 5, 6, 7, 8} but does not address how task definitions are created without a Planner.
- **Affected artifacts:** [quick-fix.prompt.md](v2/.github/prompts/quick-fix.prompt.md), [orchestrator.agent.md](v2/.github/agents/orchestrator.agent.md#L80), [implementer.agent.md](v2/.github/agents/implementer.agent.md#L30)
- **Recommendation:** One of: (a) Add a "Quick-Fix Setup" responsibility to the Orchestrator — at Step 1, if quick-fix mode, create a minimal `plan-output.yaml` and single `tasks/task-01.yaml` from the initial request (~10 lines of orchestrator text); (b) Add a simplified Planning step to quick-fix (making it steps {1, 4, 5, 6, 7, 8}); or (c) Give the Implementer a "direct mode" input path where it receives the feature request directly instead of a task file.
- **Evidence:** quick-fix.prompt.md lines 11-12: "Steps skipped: Research (Step 2), Architecture (Step 3), Planning (Step 4)." Implementer line 30: "tasks/task-XX.yaml | Planner". Orchestrator line 86: "Read plan-output.yaml → tasks."

### Finding A-2: `pipeline-log.yaml` schema undefined

- **Severity:** Minor
- **Description:** The Orchestrator references `pipeline-log.yaml` 5 times (role description, Step 1 initialization, DAG algorithm step 6, and Logging constraint) and specifies what fields to log ("step, agent, started_at, completed_at, status, output_paths"). The Knowledge agent reads this file at Step 8 (line 33). However, neither the Orchestrator, Knowledge, nor `global-rules.md` defines the actual YAML structure of this file. An LLM orchestrator would need to invent the structure, which may not match what the Knowledge agent expects to parse. D-15 lists the fields but this decision document is not read by agents at runtime.
- **Affected artifacts:** [orchestrator.agent.md](v2/.github/agents/orchestrator.agent.md#L99), [knowledge.agent.md](v2/.github/agents/knowledge.agent.md#L33)
- **Recommendation:** Add a minimal schema example (5-8 lines) to either the Orchestrator's Constraints section or `global-rules.md` showing the expected YAML structure of `pipeline-log.yaml` entries.
- **Evidence:** Orchestrator line 99: "append to pipeline-log.yaml after EVERY dispatch: step, agent, started_at, completed_at, status, output_paths." Knowledge line 33: "Read docs/feature/<slug>/pipeline-log.yaml to get dispatch entries, timings, and statuses." No schema definition in any of the 11 system files.

### Finding A-3: Three agents missing `global-rules.md` reference

- **Severity:** Minor
- **Description:** Of 8 agents, 5 explicitly reference `global-rules.md` (Orchestrator, Architect, Planner, Implementer, Tester). Three agents do not reference it at all: Researcher, Reviewer, and Knowledge. While key cross-cutting rules (completion contract, anti-hallucination) are partially inlined in these agents' own constraints, the retry policy ("Transient: retry once, Deterministic: fail immediately") is absent from all three. Per D-12, the intent was that global-rules.md is referenced "by filename only" from all agents.
- **Affected artifacts:** [researcher.agent.md](v2/.github/agents/researcher.agent.md), [reviewer.agent.md](v2/.github/agents/reviewer.agent.md), [knowledge.agent.md](v2/.github/agents/knowledge.agent.md)
- **Recommendation:** Add one line to each agent's Constraints section: "Read `global-rules.md` for completion contract format and retry policy." This is 1 line × 3 agents = 3 total lines added.
- **Evidence:** grep for "global-rules" in v2/.github/ returns 7 matches across 5 files; researcher.agent.md, reviewer.agent.md, and knowledge.agent.md return 0 matches.

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Tester static workflow references non-existent Implementer schema fields

- **Severity:** Major
- **Description:** The Tester's static evaluation workflow (lines 37-46) references field names from a different system version that don't exist in the Implementer's actual output schema. Specifically:
  - Tester says: "Extract: changes, commands_executed, tdd_red_green, self_check" — Implementer has: `files_modified`, `commands_executed`, `tdd_compliance`, `test_results`
  - Tester says: "`tdd_red_green.tests_written_first` is true" — Implementer has: `tdd_compliance: true | false`
  - Tester says: "`self_check.test_summary` is present" / "`self_check.test_summary.failed` is 0" — Implementer has: `test_results: { passed: N, failed: N }`
  - Tester says: "Cross-reference each report's `changes[].path`" — Implementer has: `files_modified: [list of paths]`
  - Tester says: "For each task with `task_type: code`" — Implementer schema has no `task_type` field

  An LLM Tester dispatched with these instructions would look for fields that don't exist, likely producing incorrect verification verdicts or hallucinated results. The field names appear to be carried over from the old system's verification schema (the current NewAgents verifier used these exact field names).
- **Affected artifacts:** [tester.agent.md](v2/.github/agents/tester.agent.md#L37-L46), [implementer.agent.md](v2/.github/agents/implementer.agent.md#L53-L67)
- **Recommendation:** Update Tester static workflow to use the Implementer's actual field names: (1) "Extract: `files_modified`, `commands_executed`, `tdd_compliance`, `test_results`" (2) "`tdd_compliance` is `true`" (3) "`test_results.failed` is 0" (4) "Cross-reference `files_modified` against task's `files[]`" (5) Remove `task_type` filter — all implementation tasks in this system are code tasks.
- **Evidence:** Tester lines 37-46 use `tdd_red_green`, `self_check`, `changes[]`, `task_type`. Implementer lines 53-67 define `files_modified`, `test_results`, `tdd_compliance`, `commands_executed` — no `task_type`, no `tdd_red_green`, no `self_check`, no `changes[]`.

### Finding C-2: Missing interactive approval gate after Research (Step 2)

- **Severity:** Major
- **Description:** Spec requirement FR-3.4 states: "The Orchestrator MUST provide approval gates in interactive mode after Research (Step 2) and after Planning (Step 4)." The feature.md User Story 1 (step 4) confirms: "Approval gate: Developer reviews research summary, approves." The Orchestrator's Step 4 (Planning) correctly includes an approval gate: "Interactive mode: present plan to user for approval via vscode_askQuestions." However, Step 2 (Research) only has a technical gate: "Gate: ≥2 of N return DONE → proceed." There is no mention of an interactive approval gate between Research and Architecture. In interactive mode, the developer should have the opportunity to review research findings and course-correct before the Architect begins work.
- **Affected artifacts:** [orchestrator.agent.md](v2/.github/agents/orchestrator.agent.md#L46-L51)
- **Recommendation:** Add to Orchestrator Step 2, after the ≥2 gate: "Interactive mode: present research summary to user for approval via `vscode_askQuestions` before proceeding to Step 3." This is ~2 lines.
- **Evidence:** FR-3.4 in spec-output.yaml: "The Orchestrator MUST provide approval gates in interactive mode after Research (Step 2) and after Planning (Step 4)." Orchestrator Step 2 (lines 46-51): no approval gate mention. Orchestrator Step 4 (line 56): "Interactive mode: present plan to user for approval." feature.md Story 1 step 4: "Approval gate: Developer reviews research summary, approves."

### Finding C-3: Pre-commit validation checks reported paths, not actual git diff

- **Severity:** Minor
- **Description:** The Orchestrator's Step 8 pre-commit validation says: "verify Knowledge output paths against allowlist." This checks the Knowledge agent's self-reported `completion.output_paths` against an allowlist of expected files. However, a misbehaving Knowledge agent could create files it doesn't report in `output_paths`. The design's trust boundary model (D-14) describes "pre-commit: orchestrator validates git diff against Knowledge output allowlist" — i.e., checking the actual filesystem changes, not the self-reported paths. The implementation checks the weaker property (self-reported) rather than the stronger one (actual diff). This is a low-severity gap because Knowledge is Tier 3 (create-only, no terminal) and the practical exploit path is narrow.
- **Affected artifacts:** [orchestrator.agent.md](v2/.github/agents/orchestrator.agent.md#L77-L79)
- **Recommendation:** Strengthen to: "Run `git diff --staged --name-only` after Knowledge dispatch. Verify all new files are in the allowlist. Reject commit if unexpected files are found." This checks actual changes rather than self-reported paths.
- **Evidence:** Orchestrator lines 77-79: "verify Knowledge output paths against allowlist." Design D-14 trust_boundary_model: "orchestrator validates git diff against Knowledge output allowlist before committing."

## Summary

The v2 agent system is a strong clean-slate implementation at 770 lines (51% of budget) with clear agent boundaries, co-located schemas, zero SQLite, and zero § references. Three Major findings need attention before merging: (1) the quick-fix pipeline has no task definition mechanism since Planning is skipped, (2) the Tester references old-system field names that don't match the Implementer's actual output schema, and (3) the Orchestrator is missing the Research approval gate required by FR-3.4. All three are straightforward fixes requiring ~20-30 lines total. Four Minor findings are advisory.
