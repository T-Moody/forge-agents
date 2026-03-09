# Adversarial Review: code — security-sentinel

## Review Metadata

- **Perspective:** security-sentinel
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-08T12:00:00Z

## Security Analysis

**Category Verdict:** needs_revision

### Finding S-1: Implementer has `fetch_webpage` in YAML frontmatter — Trust Boundary Violation

- **Severity:** Major
- **Description:** The implementer.agent.md YAML frontmatter (line 16) includes `fetch_webpage` in its tools list. Design decision D-14 (Trust Boundary Model) explicitly restricts `fetch_webpage` to Tier 3 agents "Researcher, Architect only, when enabled." The Implementer is Tier 2 Standard Trust. Having `fetch_webpage` AND `run_in_terminal` on the same agent creates an exfiltration vector: the Implementer can read any workspace file via `read_file` and transmit content to external endpoints via `fetch_webpage` URL parameters. In autonomous mode, no user approval gate exists for tool usage.
- **Affected artifacts:** [v2/.github/agents/implementer.agent.md](v2/.github/agents/implementer.agent.md#L16), design-output.yaml D-14 `tier_3_read_only_create.capabilities`
- **Recommendation:** Remove `fetch_webpage` from the implementer.agent.md YAML frontmatter tools list. The implementer should not need external web access — it implements code from task specifications using codebase context.
- **Evidence:** Design D-14 `tier_3_read_only_create.capabilities`: "fetch_webpage (Researcher, Architect only, when enabled)". Implementer frontmatter line 16: `- fetch_webpage`. Design D-14 `tier_2_standard_trust`: does NOT list fetch_webpage as a capability.

### Finding S-2: Tester has `run_in_terminal` without command allowlist

- **Severity:** Major
- **Description:** The tester.agent.md has `run_in_terminal` in its frontmatter (line 11) but, unlike the implementer, has NO command allowlist in its Constraints section. The Implementer's Constraints §1 defines a strict allowlist (dotnet build/test, npm run/test, cargo build/test, go build/test, pytest, git diff/add/status). The Tester's 5 constraints cover singleton mode, mode discipline, evidence, clean shutdown, and global-rules — but no terminal command restriction. Design D-14 states "Tester: singleton for dynamic mode. Terminal usage limited to test execution commands" — but this restriction exists only in design-output.yaml, not in the agent instruction file where it would be enforced.
- **Affected artifacts:** [v2/.github/agents/tester.agent.md](v2/.github/agents/tester.agent.md) (Constraints section, lines 109-128), design-output.yaml D-14 `tier_2_standard_trust`
- **Recommendation:** Add a command allowlist constraint to tester.agent.md mirroring the implementer's pattern. Expected patterns: `dotnet run`, `dotnet test`, `npm start`, `npm test`, `npm run test:*`, `cargo run`, `cargo test`, `go test`, `pytest`, and application shutdown commands. Log all commands in the test report.
- **Evidence:** Implementer Constraints §1 defines 10+ allowlisted patterns. Tester Constraints section (5 bullet points): zero mention of terminal command restriction. Design D-14: "Terminal usage limited to test execution commands" — not present in actual agent file.

### Finding S-3: Architect lacks `web_research_enabled` gate for `fetch_webpage`

- **Severity:** Major
- **Description:** The researcher.agent.md correctly gates `fetch_webpage` usage on the `web_research_enabled` parameter: "Do NOT use fetch_webpage when web_research_enabled is absent or false" (Workflow Step 4, Constraints). However, architect.agent.md has `fetch_webpage` in its tools (line 10) with only a content restriction: "fetch_webpage is for API docs and framework references only — not general browsing." There is no conditional check against the `web_research_enabled` parameter. When a user answers "no" to the web research prompt, the Architect can still use `fetch_webpage` because its instructions don't reference the toggle.
- **Affected artifacts:** [v2/.github/agents/architect.agent.md](v2/.github/agents/architect.agent.md#L10) (frontmatter), [line 124](v2/.github/agents/architect.agent.md#L124) (constraint), [v2/.github/agents/researcher.agent.md](v2/.github/agents/researcher.agent.md#L40) (correctly gated)
- **Recommendation:** Add the `web_research_enabled` parameter to the Architect's Inputs table and add a constraint: "Only use `fetch_webpage` when `web_research_enabled` is explicitly `true`." Mirror the Researcher's gating pattern.
- **Evidence:** Researcher Workflow Step 4: "Do NOT use fetch_webpage when web_research_enabled is absent or false." Architect Constraints: no mention of `web_research_enabled`. Feature-workflow.prompt.md question 2: "Allow fetch_webpage for research agents?" — ambiguous whether "research agents" includes Architect.

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: Verification line counts do not match actual file sizes

- **Severity:** Major
- **Description:** The verification reports (tasks-01-06.yaml, tasks-07-10.yaml) report line counts that are significantly lower than the actual staged files. This undermines trust in the verification pipeline's ability to enforce line budget constraints (CR-3: ≤150 per agent, CR-2: ≤1,500 total).

  | File                | Verification Report | Actual (staged) | Delta |
  | ------------------- | ------------------- | --------------- | ----- |
  | global-rules.md     | 45                  | 70              | +25   |
  | researcher.agent.md | 69                  | 82              | +13   |
  | planner.agent.md    | 79                  | 105             | +26   |
  | reviewer.agent.md   | 75                  | 95              | +20   |
  | architect.agent.md  | 98                  | 129             | +31   |

  All actual counts still fall within the 150-line hard limit, and total system size (998 lines) is within the 1,500-line budget. But the verification evidence is unreliable — the "passed" verdicts were based on incorrect data.

- **Affected artifacts:** [docs/feature/agent-system-refactor/verification-reports/tasks-01-06.yaml](docs/feature/agent-system-refactor/verification-reports/tasks-01-06.yaml), [docs/feature/agent-system-refactor/verification-reports/tasks-07-10.yaml](docs/feature/agent-system-refactor/verification-reports/tasks-07-10.yaml)
- **Recommendation:** Investigate why verification line counts don't match actual files. Likely causes: (1) files were modified after verification ran, or (2) verification used a different counting method. Re-run verification with actual staged file sizes. The correct total is ~998 lines across 11 files.
- **Evidence:** Terminal `Get-ChildItem | Measure-Object -Line` returns: global-rules 70, researcher 82, planner 105, reviewer 95, architect 129. Verification YAML reports different values (see table above).

### Finding A-2: Reviewer verdict threshold gates only on Blocker severity

- **Severity:** Minor
- **Description:** The reviewer.agent.md verdict rules state: "Zero blocker findings → approve (majors/minors are advisory)." This means Critical findings are advisory-only and do not trigger `request-changes`. The prior system had a more granular threshold where Critical findings caused `needs_revision`. The new system's simplified model accepts any number of Critical/Major findings as long as there are no Blockers — the gate only activates on Blockers.
- **Affected artifacts:** [v2/.github/agents/reviewer.agent.md](v2/.github/agents/reviewer.agent.md#L78) (Verdict rules)
- **Recommendation:** Consider whether the threshold should be tightened: "Any blocker OR ≥2 critical → request-changes." This is an accepted design simplification — no change required, but worth monitoring in early pipeline runs.
- **Evidence:** Reviewer.agent.md: "Zero blocker findings → approve (majors/minors are advisory)." Critical severity is not mentioned in verdict logic.

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Quick-fix pipeline skips Planning but Implementation requires plan-output.yaml

- **Severity:** Major
- **Description:** The quick-fix.prompt.md defines steps {1, 5, 6, 7, 8}, explicitly skipping Planning (Step 4). However, the orchestrator's Step 5 (Implementation) begins with: "Execute DAG dispatch algorithm — Read plan-output.yaml → tasks with depends_on and files." Without Step 4, no plan-output.yaml exists. The DAG dispatch algorithm has no fallback for missing plan input. Additionally, the Implementer's file ownership constraint depends on `task.files[]` from the plan — without a plan, there's no file ownership enforcement. The quick-fix pipeline is logically incomplete: it references an Implementation step that requires an input that its pipeline flow doesn't produce.
- **Affected artifacts:** [v2/.github/prompts/quick-fix.prompt.md](v2/.github/prompts/quick-fix.prompt.md) (Step 2 references Step 5), [v2/.github/agents/orchestrator.agent.md](v2/.github/agents/orchestrator.agent.md#L51) (Step 5 DAG algorithm requires plan-output.yaml)
- **Recommendation:** Either: (a) add a lightweight planning step to quick-fix (even if inline in the orchestrator for simple features), or (b) define an alternative task specification mechanism for quick-fix mode where the orchestrator creates a single-task plan inline (one task, all changed files declared), or (c) document that the orchestrator handles quick-fix differently in Step 5 (e.g., treats the entire fix as a single task with broad file scope).
- **Evidence:** quick-fix.prompt.md line 10: "Implementation (Step 5)". orchestrator.agent.md Step 5: "Read plan-output.yaml → tasks with depends_on and files". quick-fix.prompt.md line 16: "Steps skipped: Research (Step 2), Architecture (Step 3), Planning (Step 4)."

### Finding C-2: Orchestrator pre-commit validation wording is scoped correctly per KI-1

- **Severity:** (No finding — clean)
- **Description:** The orchestrator's Step 8 pre-commit validation correctly validates only Knowledge output paths (`evidence-bundle.yaml`, `knowledge-output.yaml`, `instruction-recommendations.md` in `docs/feature/<slug>/`), consistent with KI-1 from the plan. This was a known issue from design review and was correctly implemented.

## Summary

Security-sentinel code review Round 1 found **5 Major** and **1 Minor** findings across all 3 categories. The most significant security concern is the Implementer's unauthorized `fetch_webpage` access (S-1), which violates the D-14 trust boundary model by giving a Tier 2 terminal-capable agent web access — creating a potential data exfiltration path. The Tester also lacks a command allowlist (S-2) despite having terminal access, and the Architect lacks the `web_research_enabled` gate (S-3). The quick-fix pipeline has a structural gap where Implementation depends on Planning output that is skipped (C-1). Verification line counts are inaccurate across 5 files (A-1). Overall verdict: **needs_revision** — no blockers, but 4 Major findings require attention before merge.
