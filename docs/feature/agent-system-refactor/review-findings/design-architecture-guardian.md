# Adversarial Review: design — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-09T18:00:00Z

## Security Analysis

**Category Verdict:** approve

### Finding S-1: Git safety rule is advisory-only, not deterministically enforced

- **Severity:** Minor
- **Description:** The git safety rule extracted to global-rules.md (FR-5.3, D-1) is purely instructional. VS Code does not enforce tool restrictions from body text — only YAML frontmatter `tools:` arrays constrain tool access. Since `runInTerminal` is granted to the implementer and tester, those agents can technically execute `git add` or `git commit` if the AI disregards global-rules.md. The design considered hooks (Direction B) and correctly rejected them for scope.
- **Affected artifacts:** `v2/.github/agents/global-rules.md`, `v2/.github/agents/implementer.agent.md`, `v2/.github/agents/tester.agent.md`
- **Recommendation:** Explicitly document this as a known limitation in the design. The defense-in-depth strategy (tester command audit + reviewer command trail) mitigates the risk. No action needed beyond documentation.
- **Evidence:** D-1 alt-B rejected for scope. Implementer and tester `tools:` arrays include `runInTerminal` which enables arbitrary terminal commands. VS Code docs confirm: "If a given tool is not available when using the custom agent, it is ignored" — this means only frontmatter tools restrict access; body text rules are advisory.

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Orchestrator Step 8 cognitive density — 6 branching sub-workflows in one step

- **Severity:** Major
- **Description:** Current orchestrator Step 8 has 3 actions: dispatch knowledge, pre-commit validate, `git add -A && git commit`. The design expands Step 8 to 6+ branching actions: (1) dispatch knowledge, (2) optional doc-update re-dispatch with [Apply/Review/Skip] choice, (3) pre-commit validation of knowledge paths, (4) selective pathspec staging from implementation reports, (5) interactive 3-way commit/review/unstage choice, (6) autonomous stage-only path. This makes Step 8 the most complex step in the orchestrator with multiple conditional branches. An implementer could misorder sub-workflows (e.g., staging before validation, committing before doc-update completes).
- **Affected artifacts:** `v2/.github/agents/orchestrator.agent.md` Step 8 section
- **Recommendation:** Decompose Step 8 into explicit numbered sub-steps in the design (e.g., 8a: Knowledge Extraction, 8b: Doc Updates, 8c: Pre-commit Validation, 8d: Selective Staging, 8e: Commit/Review Choice). This clarifies control flow sequence for the implementer without changing the architecture. The design's Implementation Checklist items 7-11 cover these but don't sequence them relative to each other.
- **Evidence:** File inventory shows 5 of 10 orchestrator changes target Step 8: "FR-5: selective pathspec staging", "FR-6: interactive Commit/Review/Unstage", "FR-6: autonomous stage only", "FR-7: doc update choice", "D-9: pre-commit validation scoped to knowledge paths". Current Step 8 is ~8 lines; new Step 8 will be ~20+ lines with branching.

### Finding A-2: Architect replaces tester as tightest line budget agent (corrected counts)

- **Severity:** Minor
- **Description:** The design identifies the tester (143/150, 7-line headroom) as "🟡 Tightest." Based on corrected line counts (see C-1), the architect at 130 actual lines with ~15-17 lines of additions would reach ~145-147 (3-5 headroom), making it the true bottleneck. The tester at 129 actual + 13 = ~142 (8 headroom) is more comfortable than claimed. The design's D-6 fallback (SKILL.md extraction for tester) is less critical than an equivalent extraction plan for the architect.
- **Affected artifacts:** Design line budget analysis, D-6 decision rationale
- **Recommendation:** After correcting line counts (C-1), reassess risk classification for the architect (🔴 Tightest, not 🟡). Consider extracting the architect's web research step (FR-8) guidance or clarification step (FR-3) into a reference doc if budget becomes too tight during implementation.
- **Evidence:** Actual architect: 130 lines. Additions: +1 (user-invocable), +0 net (tool renames), +~7 (clarification step), +~7 (web research step) = ~145-147 post-change, 3-5 headroom.

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Design line counts are materially incorrect for 5 of 9 files

- **Severity:** Critical
- **Description:** The design's stated "current" line counts are wrong for the majority of v2 agent files. Verified actual counts (via PowerShell `(Get-Content ...).Count`) vs design claims:

  | Agent        | Design Claims | Actual  | Delta   |
  | ------------ | ------------- | ------- | ------- |
  | orchestrator | 105           | 117     | +12     |
  | researcher   | 78            | 83      | +5      |
  | architect    | **108**       | **130** | **+22** |
  | planner      | 97            | 105     | +8      |
  | implementer  | 94            | 99      | +5      |
  | tester       | 130           | 129     | -1      |
  | reviewer     | **106**       | **96**  | **-10** |
  | knowledge    | **84**        | **107** | **+23** |
  | global-rules | 78            | 70      | -8      |

  The architect is off by **22 lines** (130 actual vs 108 claimed). The knowledge agent is off by **23 lines** (107 actual vs 84 claimed). These errors propagate through the entire line budget analysis. The design misidentifies the tightest agent (claims tester at 7 headroom; actual tightest is architect at 3-5 headroom). The design's "System total: 1,004" is also wrong (actual: 966).

  **Corrected line budget analysis:**

  | Agent        | Actual | Est. Additions | Est. Post | Headroom | Risk            |
  | ------------ | ------ | -------------- | --------- | -------- | --------------- |
  | orchestrator | 117    | +25            | ~142      | 8        | 🟡              |
  | researcher   | 83     | +1             | ~84       | 66       | 🟢              |
  | architect    | 130    | +17            | **~147**  | **3**    | **🔴 Tightest** |
  | planner      | 105    | +11            | ~116      | 34       | 🟢              |
  | implementer  | 99     | +16            | ~115      | 35       | 🟢              |
  | tester       | 129    | +13            | ~142      | 8        | 🟡              |
  | reviewer     | 96     | +1             | ~97       | 53       | 🟢              |
  | knowledge    | 107    | +11            | ~118      | 32       | 🟢              |

- **Affected artifacts:** `design-output.yaml` line_budget section, `design.md` Line Budget table, file_inventory current_lines and estimated_post_lines fields
- **Recommendation:** Update all line count data in both design-output.yaml and design.md using verified actual counts. Reassess risk classification for the architect (should be 🔴 Tightest). Consider whether the architect needs content reduction or extraction for FR-8 (web research step) or FR-3 (clarification step) to maintain safe headroom (>5 lines).
- **Evidence:** Line counts verified via PowerShell `(Get-Content "v2/.github/agents/<file>").Count` on 2026-03-09. Design likely used stale counts from the original v2 system metrics ("11 files, 1,004 lines" per repo memory) which diverged after prior pipeline runs.

### Finding C-2: Selective staging scope excludes non-implementation pipeline artifacts

- **Severity:** Major
- **Description:** The selective pathspec staging (D-5, FR-5.4) reads file paths exclusively from "implementation reports' files_modified lists." However, several agents produce output files NOT in implementation reports:
  - **Knowledge agent:** evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md
  - **Tester:** verification-reports/test-report.yaml
  - **Reviewer:** review-findings/_.yaml, review-verdicts/_.yaml
  - **Planner:** plan-output.yaml, tasks/\*.yaml
  - **Researcher:** research/\*.yaml
  - **Architect:** architecture-output.yaml

  The current `git add -A` stages all of these. The new selective staging would stage ONLY implementation code changes, leaving all docs/feature/ pipeline artifacts unstaged. Meanwhile, the design includes "pre-commit validation" for knowledge output paths — "pre-commit" implies these files should be committed, creating a contradiction.

- **Affected artifacts:** D-5 decision, FR-5.4 spec requirement, orchestrator Step 8 staging design
- **Recommendation:** Specify a two-source staging strategy: (1) implementation code from reports' `files_modified`, (2) pipeline artifacts via `git add docs/feature/<slug>/`. Alternatively, define explicitly that pipeline artifacts are ephemeral and NOT committed — but then remove "pre-commit" terminology from D-9.
- **Evidence:** FR-5.4: "staging only files declared in implementation reports' files_modified lists." Knowledge output schema: writes to `docs/feature/<slug>/` paths. These paths do not appear in any implementation report.

### Finding C-3: Question-answer data flow for orchestrator-mediated clarifications not specified

- **Severity:** Minor
- **Description:** D-3 specifies that the architect outputs `clarifications_needed` in YAML and the orchestrator presents them to the user. The design does not describe the return path: How are user answers passed back to the architect? Is the architect re-dispatched with answers prepended to task context? Is there a file-based handoff? The same gap applies to the planner approval flow (FR-4). Without this, different implementers could create incompatible handoff patterns.
- **Affected artifacts:** D-3 decision rationale, architect clarification workflow, planner plan_summary workflow
- **Recommendation:** Add a data flow statement to D-3: "After collecting user answers, the orchestrator re-dispatches the architect with original inputs plus a `clarification_responses` parameter. The architect resumes at Step 2 with clarifications resolved." Apply same pattern for planner.
- **Evidence:** D-3 rejects NEEDS_REVISION loop because "Full re-dispatch wastes tokens" — but the orchestrator-mediated approach also requires re-dispatch. The distinction is that the architect structures the questions, but the design doesn't describe the return path.

## Summary

The design is architecturally sound in its choice of Direction C (hybrid inline + shared extraction), individual tool names for trust model enforcement (verified against VS Code cheat sheet 2026-03-09), and orchestrator-mediated questioning (confirmed correct by VS Code subagent isolation docs). All 9 decisions are well-reasoned with clear tradeoffs. Tool name corrections are verified correct against the official VS Code built-in tools reference. However, the line budget analysis is based on materially incorrect current line counts (architect off by 22, knowledge off by 23), misidentifying the tightest agent and understating the architect's budget risk at ~3 headroom post-change. Additionally, the selective staging scope creates a functional gap for non-implementation pipeline artifacts. These correctness issues require design revision before implementation can proceed safely.
