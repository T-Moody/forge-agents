# Adversarial Review: design — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-09T18:00:00Z
- **Feature:** agent-system-refactor (Run 3 — Targeted Improvements)
- **Artifacts Reviewed:** design-output.yaml, design.md, spec-output.yaml, feature.md, 8 v2 agent files, global-rules.md, VS Code official docs (fetched 2026-03-09)

---

## Security Analysis

**Category Verdict:** approve

### Finding S-1: Silent tool name failure risk persists post-implementation

- **Severity:** Minor
- **Description:** VS Code silently ignores unrecognized tool names in frontmatter (confirmed by official docs: "If a given tool is not available when using the custom agent, it is ignored"). The design correctly identifies and fixes all 14 invalid tool names with a comprehensive mapping table verified against the VS Code cheat sheet. However, the design does not include a post-implementation verification step to confirm the corrected names are actually recognized by VS Code at runtime. If any name is misspelled during implementation, the tool restriction silently fails — exactly the same failure mode as the current broken state.
- **Affected artifacts:** design-output.yaml — tool_name_corrections section, design.md — Testing Strategy section
- **Recommendation:** Add a verification acceptance criterion or test scenario that confirms tool restrictions are functional post-implementation (e.g., verify that a restricted agent cannot invoke a tool not in its list). Alternatively, note in the implementation guidance that implementers should verify tools via the VS Code Chat Customizations diagnostics view (`Right-click Chat > Diagnostics`).
- **Evidence:** VS Code cheat sheet (fetched 2026-03-09): "If a given tool is not available when using the custom agent, it is ignored." All 14 proposed tool names verified correct against the cheat sheet.

---

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Tester line budget at 143/150 — 7 lines of headroom

- **Severity:** Minor
- **Description:** The tester agent goes from 130 to an estimated 143 lines — only 7 lines below the 150-line hard limit. This is the tightest budget of all 8 agents. The exploratory QA 4-category checklist (happy path, edge cases, error handling, state mutation) must fit within ~13 lines alongside skill reference updates. If the implementation requires even slightly more descriptive text (e.g., per-category expected behavior, skill fallback instructions), the budget could be exceeded.
- **Affected artifacts:** design-output.yaml — line_budget section (tester: current=130, estimated=143, headroom=7)
- **Recommendation:** No design change needed — D-6 documents a SKILL.md extraction fallback. The implementer should be aware of this constraint and prepare to extract to `.github/skills/exploratory-qa/SKILL.md` proactively if the inline approach approaches 148+ lines. The design's contingency plan is sound.
- **Evidence:** design-output.yaml line_budget.agents[tester]: headroom=7. D-6 alternative "alt-skill" provides fallback path. Spec CR-1 requires ≤150 lines.

---

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Architect clarification flow underspecified — interaction loop unclear

- **Severity:** Major
- **Description:** D-3 selects "orchestrator-mediated" interaction for architect clarifying questions (FR-3) and explicitly rejects the NEEDS_REVISION re-dispatch approach ("Full re-dispatch wastes tokens, re-dispatch resets context window"). However, the design does not specify what happens AFTER the user answers the orchestrator-presented questions.

  The VS Code subagent model is synchronous — the architect runs to completion before the orchestrator can read its output. This creates a sequencing problem:
  1. Orchestrator dispatches architect
  2. Architect runs Step 1 (Read Inputs), identifies ambiguities at Step 1.5
  3. **Problem:** The architect cannot pause here. It must either (a) complete all steps with assumptions and include `clarifications_needed` in the output, or (b) return NEEDS_REVISION with just the questions.
  4. If (a): The architect completes the full design with assumptions. The orchestrator reads `clarifications_needed`, presents to user, gets answers. But the answers may invalidate the assumptions the architect already made. How does the orchestrator apply the answers? Re-dispatch architect (contradicts D-3's rejection of re-dispatch)?
  5. If (b): This IS the NEEDS_REVISION approach that D-3 rejected.

  The design says the architect has a "clarification step between 'Read Inputs' and 'Analyze Codebase'" (AC-8/FR-3.1), but this implies a pause point that's impossible in the synchronous subagent model.

- **Affected artifacts:** design-output.yaml — D-3 decision, DR-1 deviation record; design.md — D-3 section; spec-output.yaml — FR-3.1, FR-3.2, AC-8, AC-9
- **Recommendation:** Clarify the exact interaction flow. Two viable options:
  - **Option A (Two-phase dispatch):** Architect first dispatch: read inputs → identify ambiguities → return DONE with `clarifications_needed` (not full architecture). Orchestrator mediates questions. Architect second dispatch: receive answers in context → complete full architecture. This is functionally a re-dispatch but NOT the rejected NEEDS_REVISION pattern — it's a structured two-phase workflow.
  - **Option B (Assumptions-first):** Architect completes fully with documented assumptions. Orchestrator validates assumptions against user answers. If conflict, re-dispatch architect with answer context. Document that re-dispatch only occurs when user answers contradict assumptions (low probability, acceptable token cost).
    Either option resolves the ambiguity. Update D-3 rationale to describe the chosen flow explicitly.
- **Evidence:** VS Code custom agents docs (fetched 2026-03-09): subagents run in "isolated subagent context" — synchronous, no user interaction. D-3 alt-needs-revision rejected with rationale "Full re-dispatch wastes tokens." AC-8 says "clarification step between 'Read Inputs' and 'Analyze Codebase' gated by interactive mode" — implies mid-execution pause.

### Finding C-2: Reviewer and Tester audit patterns not updated for implementer git add removal

- **Severity:** Major
- **Description:** FR-5 removes `git add` from the implementer's command allowlist (design correctly updates implementer to permit only `git diff` and `git status`). However, both the reviewer and tester currently audit implementer commands against patterns that include `git add` as acceptable:
  - **Reviewer** (reviewer.agent.md line ~62): "expected patterns: `dotnet build|test`, `npm run build|test`, `cargo build|test`, `go build|test`, `pytest`, `git diff|add|status`"
  - **Tester** (tester.agent.md lines ~107-108): "Terminal commands restricted to: ... `git diff`, `git status`. All commands must be logged." (tester's own allowlist is correct, but its AUDIT of implementer in static mode also checks against patterns that include `git add`)

  After the design's changes, if an implementer erroneously runs `git add`, the reviewer's audit would NOT flag it as a violation because `git add` is still in the reviewer's expected pattern list. This defeats the purpose of FR-5 — the safety net (reviewer audit) doesn't enforce the new restriction.

- **Affected artifacts:** design-output.yaml — file_inventory for reviewer (only lists "FR-2: Fix all tool names" and "FR-2.4: user-invocable: false"); reviewer.agent.md (command audit patterns); tester.agent.md (static mode audit patterns)
- **Recommendation:** Add to the reviewer's file_inventory changes: "FR-5: Update command audit expected patterns — remove `git add` from implementer-acceptable patterns." Similarly update the tester's static audit patterns if they reference `git add` for implementer validation. The reviewer should flag any `git add` in implementer `commands_executed[]` as a Major finding.
- **Evidence:** Current reviewer.agent.md constraints section: "git diff|add|status" in audit patterns. Current tester.agent.md constraints section references "git diff, git status" for its own allowlist but static mode audits against broader patterns. Design file_inventory for reviewer only lists tool name + user-invocable changes.

### Finding C-3: Plan refinement iteration cap unspecified

- **Severity:** Minor
- **Description:** FR-4.1 adds Accept/Refine/Reject options for the planner in interactive mode. FR-4.2 says refinement applies "within a single re-planning iteration (consistent with global-rules.md limits)." However, global-rules.md defines iteration caps for Implementation-Testing (3), Code Review (2), and Design Review (1) — but has NO explicit cap for plan refinement loops. If the user repeatedly selects "Reject and re-plan," the pipeline could loop unboundedly.

  The "consistent with global-rules.md limits" reference is currently ungrounded — no such limit exists.

- **Affected artifacts:** design-output.yaml — D-3 (planner aspect); spec-output.yaml — FR-4.2; v2/.github/agents/global-rules.md — Feedback Loop Limits table
- **Recommendation:** Either (a) add a "Plan Refinement" row to global-rules.md Feedback Loop Limits table (suggest max 1 refinement iteration, matching Design Review), or (b) update the design to specify the cap explicitly in the orchestrator Step 4 instructions. The spec already says "single re-planning iteration," so a cap of 1 refinement seems intended.
- **Evidence:** global-rules.md Feedback Loop Limits table defines 3 loops: Implementation-Testing (3), Code Review Fix (2), Design Review (1). No "Plan Refinement" entry. FR-4.2 references "global-rules.md limits" without specifying which limit.

### Finding C-4: Relationship between existing tester "live-qa" and new "Exploratory QA" unclear

- **Severity:** Minor
- **Description:** The current tester dynamic mode already has a "live-qa" test category: "Exploratory checks: verify endpoints respond, simulate user interactions, validate UI flows." The design adds a new "Exploratory QA" phase with 4 categories (happy path, edge cases, error handling, state mutation) AFTER automated tests.

  The overlap is confusing: "live-qa" is described as "exploratory checks" and the new phase is "exploratory QA." The design doesn't clarify whether:
  - (a) "live-qa" is REPLACED by the new Exploratory QA phase
  - (b) "live-qa" REMAINS as an automated skill category AND Exploratory QA runs afterward as a separate non-automated phase
  - (c) "live-qa" is RENAMED to Exploratory QA

- **Affected artifacts:** design-output.yaml — file_inventory tester changes; tester.agent.md — dynamic mode Step 3 skill categories (live-qa); spec-output.yaml — FR-9.1 ("after automated test suites")
- **Recommendation:** Clarify in the design that the existing "live-qa" category in Step 3 (Execute test skills) is subsumed by the new "Exploratory QA" phase, OR explicitly state the two are distinct (live-qa = automated skill execution, Exploratory QA = developer-style manual testing). Update the tester file_inventory to note whether Step 3's "live-qa" category is removed, renamed, or preserved.
- **Evidence:** Current tester.agent.md Step 3 categories: "playwright-ui, http-api, cli, integration-tests, live-qa." FR-9.1: "exploratory QA phase after automated test suites."

---

## Summary

Design review found 6 findings (0 Blocker, 0 Critical, 2 Major, 4 Minor) across 3 categories. All 27 acceptance criteria are addressed by the design, all 10 improvements are covered, and all proposed tool names are verified correct against the VS Code cheat sheet (fetched 2026-03-09). Two deviation records (DR-1, DR-2) correctly document the absence of a "structured question tool." The two Major findings concern: (1) the underspecified architect clarification interaction flow — the design doesn't resolve the tension between rejecting NEEDS_REVISION re-dispatch and needing post-question context, and (2) the reviewer/tester audit patterns not being updated to match the implementer's git add removal from FR-5. Both are implementability gaps that would cause confusion or missed enforcement during implementation.
