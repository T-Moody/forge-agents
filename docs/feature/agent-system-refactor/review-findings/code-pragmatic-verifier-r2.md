# Adversarial Review: code — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🔴
- **Round:** 2
- **Run ID:** 2026-03-08T12:00:00Z

## Round 1 Fix Verification

All 8 Major fixes from Round 1 have been verified as correctly applied:

| Fix                                 | Source                | Status      | Verification                                                                           |
| ----------------------------------- | --------------------- | ----------- | -------------------------------------------------------------------------------------- |
| A-1: Quick-fix inline tasks         | pragmatic-verifier    | ✅ Resolved | Orchestrator Step 1 quick-fix clause creates `plan-output.yaml` + `tasks/task-01.yaml` |
| C-1: Tester schema fields           | pragmatic-verifier    | ✅ Resolved | Tester static workflow now uses `files_modified`, `tdd_compliance`, `test_results`     |
| C-2: Research approval gate         | pragmatic-verifier    | ✅ Resolved | Orchestrator Step 2 includes interactive `vscode_askQuestions` gate                    |
| S-1: Implementer fetch_webpage      | security-sentinel     | ✅ Resolved | `fetch_webpage` removed from implementer YAML frontmatter                              |
| S-2: Tester command allowlist       | security-sentinel     | ✅ Resolved | Allowlist added to tester Constraints with specific patterns                           |
| S-3: Architect web_research_enabled | security-sentinel     | ✅ Resolved | Gate + Inputs table updated in architect.agent.md                                      |
| A-1: Verdict/completion routing     | architecture-guardian | ✅ Resolved | Orchestrator routing constraint now acknowledges verdict field                         |
| A-3: global-rules.md references     | pragmatic-verifier    | ✅ Resolved | Researcher, reviewer, knowledge all reference global-rules.md                          |

Additionally confirmed: Minor S-1 (fetch_webpage gating inconsistency) resolved — architect now has explicit `web_research_enabled` gate, implementer lost the tool entirely.

## Security Analysis

**Category Verdict:** approve

No new security findings. Round 1 security concerns are fully resolved:

- Implementer no longer has `fetch_webpage` (exfiltration vector closed)
- Architect's `fetch_webpage` correctly gated on `web_research_enabled`
- Tester has command allowlist matching implementer's pattern
- All 3 previously-ungated agents now reference `global-rules.md`

## Architecture Analysis

**Category Verdict:** approve

No new architecture findings. Round 1 architecture concerns are resolved:

- Quick-fix pipeline now has inline task creation (A-1 resolved)
- Verdict/completion routing contradiction resolved — orchestrator constraint explicitly carves out verdict field reading for Steps 6-7
- Minor A-2 (pipeline-log.yaml schema undefined) remains advisory — the field list in the orchestrator Constraints section provides sufficient implicit guidance for an LLM agent

## Correctness Analysis

**Category Verdict:** approve

Round 1 correctness Majors (C-1 tester schema, C-2 research gate) are fully resolved. One new finding:

### Finding C-1: 🟢 feature workflow routing skips Architecture — dead code and Planner input gap

- **Severity:** Major
- **Description:** Orchestrator Step 1 (line 40) routes 🟢 features with `🟢 risk → skip to Step 4`. This bypasses Step 3 (Architecture) entirely. However, Step 3 (line 54) contains explicit 🟢 handling: `For 🟢: architect receives no research inputs (handles gracefully)` — this code is dead since 🟢 never reaches Step 3. More critically, the Planner at Step 4 requires `architecture-output.yaml` as its first Input (planner.agent.md line 22) and its first Workflow step (line 29) says `Read architecture output. Parse the component design...` — but no architecture output exists because Step 3 was skipped. The Planner has no graceful-degradation clause for missing architecture (unlike the Architect, which handles missing research). The `feature-workflow.prompt.md` describes Research as "skipped for green-risk features" but does NOT describe Architecture as skipped, and the `global-rules.md` risk table shows 🟢 with "Design Review: None" (no embedded review), not "Architecture: None". The intent appears to be that 🟢 features skip Research (Step 2) but still run Architecture (Step 3) with no research inputs.
- **Affected artifacts:** [orchestrator.agent.md](v2/.github/agents/orchestrator.agent.md) line 40, [orchestrator.agent.md](v2/.github/agents/orchestrator.agent.md) line 54 (dead code), [planner.agent.md](v2/.github/agents/planner.agent.md) line 22 and line 29
- **Recommendation:** Change line 40 from `🟢 risk → skip to Step 4` to `🟢 risk → skip to Step 3`. This preserves Architecture for 🟢 features (architect handles missing research gracefully per its own instructions), provides the Planner with its required `architecture-output.yaml` input, and activates the currently-dead 🟢 handling code in Step 3. One-word fix.
- **Evidence:** Orchestrator line 40: `🟢 risk → skip to Step 4.` Orchestrator line 54: `For 🟢: architect receives no research inputs (handles gracefully)` — dead code. Planner line 22: `architecture-output.yaml — component design, decisions, schemas`. Planner line 29: `Read architecture output.` feature-workflow.prompt.md: only Research marked as skipped for green-risk.

Minor C-3 from Round 1 (pre-commit validation checks self-reported paths vs actual git diff) remains advisory — the practical exploit path is narrow given Knowledge agent's Tier 3 restrictions.

## Summary

All 8 Round 1 Major fixes verified as correctly applied across 7 files. The Tester schema now matches the Implementer's actual output fields, the quick-fix pipeline has inline task creation, the research approval gate is in place, trust boundary violations are closed, and cross-agent consistency is restored. One new Major finding discovered: the 🟢 feature workflow routing skips Architecture to Step 4, leaving the Planner without its required `architecture-output.yaml` input and creating dead 🟢 handling code in Step 3. Fix is trivial (one-word change: Step 4 → Step 3). Overall system quality is strong — 11 files, ~998 lines, zero cross-file § references, zero SQLite, clear agent boundaries.
