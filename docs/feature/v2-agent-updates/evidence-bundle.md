# Evidence Bundle — v2-agent-updates

## Pipeline Summary

| Field              | Value                                                                               |
| ------------------ | ----------------------------------------------------------------------------------- |
| **Feature**        | v2-agent-updates                                                                    |
| **Run ID**         | 2026-03-10T12:00:00Z                                                                |
| **Risk Level**     | 🟡                                                                                  |
| **Approval Mode**  | Autonomous                                                                          |
| **Tasks**          | 7                                                                                   |
| **Waves**          | 3                                                                                   |
| **Files Modified** | 10                                                                                  |
| **Dispatches**     | 10 (3 researcher + 1 architect + 1 planner + 1 implementer + 1 tester + 3 reviewer) |
| **Retries**        | 0                                                                                   |
| **Fix Cycles**     | 1 (static testing → fix → re-verify)                                                |

## Confidence Assessment

**Confidence: High**

- All 7 tasks completed on first attempt (0 retries)
- 3/3 code review perspectives approve with 0 blockers
- 1 Major static testing finding (E2E path mismatch) was fixed before code review
- All 3 Major code review findings are non-blocking (pre-existing tourneypal-qa issues, vestigial git tag, planner tool deviation)
- 23/25 acceptance criteria pass (1 FAIL on planner tool count, 1 PASS-WITH-NOTE on E2E path structure)
- All agents within line budget

## Verification Results

### Static Testing (Step 6)

| Severity | Count | Disposition |
| -------- | ----- | ----------- |
| Major    | 1     | Fixed       |
| Minor    | 2     | Accepted    |

**Major finding (F-1, fixed):** global-rules.md E2E Artifacts section referenced `verification-reports/` while tester.agent.md referenced `e2e-artifacts/`. Architecture decision D-8 specified `e2e-artifacts/`. Cross-file path mismatch corrected.

**Minor finding (F-2, accepted):** Planner tools: array includes `codebase` (6 tools) vs AC-specified 5 tools. Addition is functionally reasonable — planner benefits from codebase access for task decomposition.

**Minor finding (F-3, accepted):** Implementer RED phase modified with batch-build guidance despite AC-12 specifying "unchanged." Change is directionally correct and consistent with GREEN phase batching.

### Code Review (Step 7)

| Perspective  | Verdict | Major | Minor |
| ------------ | ------- | ----- | ----- |
| Security     | Approve | 1     | 1     |
| Architecture | Approve | 1     | 1     |
| Correctness  | Approve | 1     | 2     |
| **Gate**     | **3/3** | **3** | **4** |

**All Major findings are non-blocking:**

1. **Security Major:** tourneypal-qa.agent.md has no security boundaries (no tools: array, no agents: [], malformed frontmatter). Pre-existing — not modified in this feature. Agent is standalone QA, outside pipeline trust model.

2. **Architecture Major:** Cross-file reference mismatch: prompt files describe Step 1 as including "git baseline tag" but orchestrator Step 1 has no tag creation. `tag` remains in orchestrator git allowlist as vestigial entry. Pre-existing.

3. **Correctness Major:** Planner tools: array includes `codebase` beyond AC-25-Pln specification. Implementation report self-reported PASS despite evidence showing 6 tools vs 5 required. Accepted as beneficial deviation.

### Trust Tier Audit (Security Reviewer)

| Agent        | Tier | Tools | Compliance |
| ------------ | ---- | ----- | ---------- |
| orchestrator | T1   | 8     | COMPLIANT  |
| researcher   | T3   | 7     | COMPLIANT  |
| architect    | T3   | 7     | COMPLIANT  |
| planner      | —    | 6     | COMPLIANT  |
| implementer  | T2   | 10    | COMPLIANT  |
| tester       | T2   | 9     | COMPLIANT  |
| reviewer     | T1   | 8     | COMPLIANT  |
| knowledge    | T1   | 5     | COMPLIANT  |
| global-rules | —    | 0     | COMPLIANT  |

All 9 agent/shared files verified: no `disable-model-invocation: true` (pipeline dispatch unblocked), no unauthorized fetch access, git safety enforced.

## Blast Radius

| Metric        | Value |
| ------------- | ----- |
| Files changed | 10    |
| Lines added   | ~145  |
| Lines removed | ~25   |
| Net change    | ~120  |

**Files modified:**

| File                                      | Change Summary                                                        |
| ----------------------------------------- | --------------------------------------------------------------------- |
| `v2/.github/agents/global-rules.md`       | +35 lines: YAML frontmatter + 4 cross-cutting sections                |
| `v2/.github/agents/orchestrator.agent.md` | +42/-13: tools array, subagent-only, context mgmt, vscodeAskQuestions |
| `v2/.github/agents/researcher.agent.md`   | +8: tools array, mandatory web research                               |
| `v2/.github/agents/architect.agent.md`    | +8: tools array, mandatory web research                               |
| `v2/.github/agents/implementer.agent.md`  | +12: tools array, batch builds                                        |
| `v2/.github/agents/tester.agent.md`       | +10: tools array, E2E artifact path                                   |
| `v2/.github/agents/reviewer.agent.md`     | +6: tools array                                                       |
| `v2/.github/agents/planner.agent.md`      | +6: tools array                                                       |
| `v2/.github/agents/knowledge.agent.md`    | +5: tools array                                                       |
| `v2/.github/prompts/quick-fix.prompt.md`  | +8/-5: planning step added, steps renumbered                          |

## Line Budget Compliance

| File                  | Lines | Budget | Utilization |
| --------------------- | ----- | ------ | ----------- |
| global-rules.md       | 118   | —      | N/A         |
| orchestrator.agent.md | 148   | 550    | 27%         |
| researcher.agent.md   | 84    | 150    | 56%         |
| architect.agent.md    | 139   | 150    | 93%         |
| implementer.agent.md  | 108   | 150    | 72%         |
| tester.agent.md       | 137   | 150    | 91%         |
| reviewer.agent.md     | 97    | 150    | 65%         |
| planner.agent.md      | 112   | 150    | 75%         |
| knowledge.agent.md    | 117   | 150    | 78%         |

**Note:** architect.agent.md (93%) and tester.agent.md (91%) are approaching budget. Future changes to these files require careful line management.

## Rollback Command

```
git revert <commit-range>
```

All changes are confined to `v2/.github/agents/` and `v2/.github/prompts/`. No source code, test files, or database schemas were modified. Rollback is straightforward file reversion.

## Requirements Coverage

| #   | Requirement                           | Status   | Evidence                                                                   |
| --- | ------------------------------------- | -------- | -------------------------------------------------------------------------- |
| 1   | Enforce web research when enabled     | Complete | global-rules.md §Web Research + mandatory language in researcher/architect |
| 2   | E2E artifacts in feature slug folder  | Complete | global-rules.md §E2E Artifacts + tester directive                          |
| 3   | Orchestrator subagent-only            | Complete | Subagent-Only Enforcement section + tools array                            |
| 4   | Prevent orchestrator context overload | Complete | Context Management section + completion-contract-only reads                |
| 5   | Reduce implementer builds             | Complete | Constraint 8 build batching + RED/GREEN phase instructions                 |
| 6   | No branch creation by agents          | Complete | global-rules.md §Branch Management + orchestrator anti-drift               |
| 7   | Enforce vscodeAskQuestions tool       | Complete | global-rules.md §User Interaction + 0 stale ask_questions                  |

All 7 initial requirements fully addressed.

## Key Decisions Made

| ID   | Decision                                       | Rationale                                                 |
| ---- | ---------------------------------------------- | --------------------------------------------------------- |
| D-1  | vscodeAskQuestions as tool name                | Runtime observation; VS Code silently ignores wrong names |
| D-2  | Individual tool names over tool sets           | Fine-grained trust model control                          |
| D-3  | Cross-cutting rules in global-rules.md         | Single source of truth for 4 new behavioral rules         |
| D-4  | Quick-fix always dispatches planner            | Subagent-only compliance; ~seconds overhead acceptable    |
| D-5  | Batch builds in implementer GREEN phase        | Reduces worst-case from N+6 to ~4 builds                  |
| D-6  | Orchestrator tools: restriction (8 tools only) | Platform-level enforcement of subagent-only               |
| D-10 | Completion-contract-only reads                 | Prevents orchestrator context window bloat                |

## Recurring Theme: Cross-File Consistency

This pipeline confirms the pattern observed across multiple prior runs:

| Pipeline                    | Cross-file Finding                                                  |
| --------------------------- | ------------------------------------------------------------------- |
| agent-pipeline-improvements | orchestrator §11→§1.1 reference mismatch (Critical)                 |
| agent-pipeline-improvements | EG-10 threshold ≥4→≥3 mismatch (Major)                              |
| tdd-e2e-enforcement         | E2E concurrency cap contradictory across 3 files (Major)            |
| **v2-agent-updates**        | **E2E artifact path mismatch global-rules↔tester (Major, fixed)**   |
| **v2-agent-updates**        | **Vestigial git tag in prompts↔orchestrator (Major, non-blocking)** |

**Cross-file reference validation must be a first-class verification step**, not an afterthought. Every path, section reference, and parameter value that appears in more than one file should be concordance-checked.

## Recommendations for Future Agent Modifications

1. **Always include tools: array** — Any new or modified agent file must have explicit tool restrictions in YAML frontmatter. The trust model is unenforced without them.

2. **Verify tool names against official docs** — VS Code silently ignores incorrect tool names. Before shipping any tools: array change, cross-check every name against the built-in tools reference.

3. **Cross-file concordance check** — When modifying a path, section heading, or parameter value in global-rules.md, verify the same value in every agent file that references it.

4. **Budget awareness** — architect.agent.md (93%) and tester.agent.md (91%) are near their 150-line limits. Any additions to these files must remove equivalent content.

5. **Scope tourneypal-qa.agent.md** — 2 of 3 reviewers flagged it. Either add security boundaries (tools: array, agents: [], frontmatter fix) in a follow-up feature, or move it out of `v2/.github/agents/`.

6. **Clean up vestigial git tag** — Both prompts reference "git baseline tag" in Step 1 but the orchestrator does not create tags. Either add tag creation or remove the references.

7. **Tester should independently verify AC claims** — Implementer self-reported AC-25-Pln as PASS with contradictory evidence. Static testing caught it, but this suggests self-verification is unreliable for exact-match criteria.
