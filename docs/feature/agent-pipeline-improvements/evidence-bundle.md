# Evidence Bundle — agent-pipeline-improvements

**Run ID:** 2026-03-05T12:00:00Z
**Feature:** Agent Pipeline Improvements (8 user-reported issues)
**Date:** 2026-03-05
**Pipeline Mode:** Autonomous

---

## (a) Overall Confidence: **HIGH**

All 11 implementation tasks verified with 0 failures. All 3 code reviewers approve in R2 with 0 new findings. All 3 design reviewers approved in R2. 2 Critical and 6 Major code review findings from R1 all resolved. No security blockers throughout pipeline. Orchestrator at 539/550 lines (within budget).

---

## (b) Verification Summary

| Task      | Title                                                                                 | Checks | Passed | Failed | Regressions |
| --------- | ------------------------------------------------------------------------------------- | ------ | ------ | ------ | ----------- |
| task-01   | schemas.md + sql-templates.md (E2E decouple, archive)                                 | 9      | 9      | 0      | 0           |
| task-02   | tool-access-matrix.md (fetch_webpage, planner ask_questions)                          | 8      | 8      | 0      | 0           |
| task-03   | e2e-integration.md + dispatch-patterns.md (risk-independent E2E)                      | 8      | 8      | 0      | 0           |
| task-04   | orchestrator.agent.md (9 changes: approval, archive, fast-track, E2E, anti-drift)     | 12     | 12     | 0      | 0           |
| task-05   | planner.agent.md (E2E decouple, fast-track fallback)                                  | 10     | 10     | 0      | 0           |
| task-06   | verifier.agent.md (Tier 5 gate, no-redirect, Copilot skill)                           | 7      | 7      | 0      | 0           |
| task-07   | spec.agent.md (pushback restructure, fetch_webpage, approval_mode)                    | 10     | 10     | 0      | 0           |
| task-08   | implementer.agent.md (no-redirect, approval_mode)                                     | 7      | 7      | 0      | 0           |
| task-09   | researcher.agent.md + designer.agent.md (fetch_webpage, Copilot skill, approval_mode) | 8      | 8      | 0      | 0           |
| task-10   | adversarial-reviewer.agent.md + knowledge-agent.agent.md (approval_mode)              | 7      | 7      | 0      | 0           |
| task-11   | plan-and-implement.prompt.md (fast-track pipeline prompt)                             | 6      | 6      | 0      | 0           |
| **TOTAL** |                                                                                       | **92** | **92** | **0**  | **0**       |

Additional review-phase checks: 37 (design review: 18, code review R1: 18, code review R2 fix validation: 1).

**Grand total (incl. review checks):** 129 checks, 120 passed, 9 failed (all 9 are R1 review needs_revision verdicts — resolved in R2).

---

## (c) Review Summary

### Design Review (Step 3b)

| Perspective           | Round 1        | Round 2 | Findings (R1)             |
| --------------------- | -------------- | ------- | ------------------------- |
| Security Sentinel     | approve        | approve | 0B/0C/1M/2m               |
| Architecture Guardian | needs_revision | approve | 0B/0C/5M/2m               |
| Pragmatic Verifier    | needs_revision | approve | 0B/0C/3M/4m → 1 retracted |

**R1 Key Findings (all resolved in R2):**

- S-1 (Major): Fast-track skips Step 7 review → resolved via D-14 immutable minimum step set
- A-1 (Major): FR-2.4/FR-8.6 contradiction → resolved via DR-2 (relocate to orchestrator)
- PV-A1 (Major): Retracted — pragmatic-verifier measurement error (353 vs actual 539 lines)
- PV-C1 (Major): FR-2.2 approval_mode incomplete → resolved via 8 subagent file updates
- PV-C2 (Major): Dual anti-drift amendment → resolved via consistent line 29 + line 539 changes

### Code Review (Step 7)

| Perspective           | Round 1        | Round 2 | Findings (R1) |
| --------------------- | -------------- | ------- | ------------- |
| Security Sentinel     | needs_revision | approve | 0B/2C/3M/2m   |
| Architecture Guardian | needs_revision | approve | 0B/0C/4M/2m   |
| Pragmatic Verifier    | needs_revision | approve | 0B/0C/2M/5m   |

**R1 Key Findings (all resolved in R2):**

- **Critical (2):**
  - S/C-1: orchestrator §11→§1.1 broken cross-reference for archive query
  - S-1: fast-track prompt allows 🔴-risk bypass without enforcement → escalation clause added
- **Major (6):**
  - EG-10 threshold mismatch (≥4 vs ≥3 after restructure)
  - Gate evaluation order missing e2e-additive
  - Windows-invalid timestamp in archive filenames
  - Missing fetch_webpage self-verification items (3 agents)
  - Ambiguous approval mode default in fast-track prompt
  - DR-1 scope vs archive rename contradiction
- **Minor (7):** Stale pushback option names, missing self-verification updates, -corrupt suffix documentation

**R2:** All 3 reviewers approve with 0 new findings.

---

## (d) Rollback Command

```bash
git revert --no-commit pipeline-baseline-2026-03-05..HEAD
```

---

## (e) Blast Radius

| Metric                   | Count                                                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| Files modified           | 14                                                                                                                             |
| Files created            | 1                                                                                                                              |
| **Total files affected** | **15**                                                                                                                         |
| 🔴 Risk files            | 1 (orchestrator.agent.md)                                                                                                      |
| 🟡 Risk files            | 7 (planner, verifier, spec, e2e-integration, dispatch-patterns, schemas, sql-templates)                                        |
| 🟢 Risk files            | 7 (implementer, researcher, designer, adversarial-reviewer, knowledge-agent, tool-access-matrix, plan-and-implement.prompt.md) |
| Regressions detected     | 0                                                                                                                              |

**All changes are agent instruction files** (Markdown in `NewAgents/.github/`). No runtime source code, tests, or configuration files modified.

### Files Modified

| File                          | Risk | Changes                                                                                                         |
| ----------------------------- | ---- | --------------------------------------------------------------------------------------------------------------- |
| orchestrator.agent.md         | 🔴   | Approval mode rewrite, DB archive, fast-track detection, E2E decoupling, anti-drift, typo fix (+10 fixes in R2) |
| planner.agent.md              | 🟡   | E2E decoupled from lane, fast-track fallback, self-verification #11                                             |
| verifier.agent.md             | 🟡   | Tier 5 gate e2e_required-only, no-redirect, Copilot skill support                                               |
| spec.agent.md                 | 🟡   | Per-concern pushback, fetch_webpage (interactive), approval_mode                                                |
| schemas.md                    | 🟡   | e2e_required field update, skill_format field added                                                             |
| sql-templates.md              | 🟡   | §1.1 archive query, EG-10 e2e-additive restructure                                                              |
| e2e-integration.md            | 🟡   | §0 reading guide update for all risk levels                                                                     |
| dispatch-patterns.md          | 🟡   | E2E constraints apply on e2e_required (not risk)                                                                |
| tool-access-matrix.md         | 🟢   | fetch_webpage 🔒, planner ask_questions, orchestrator scope                                                     |
| implementer.agent.md          | 🟢   | No-redirect rule, approval_mode, self-verification                                                              |
| researcher.agent.md           | 🟢   | fetch_webpage (interactive), Copilot skill discovery, approval_mode                                             |
| designer.agent.md             | 🟢   | fetch_webpage (interactive), approval_mode                                                                      |
| adversarial-reviewer.agent.md | 🟢   | approval_mode parameter docs                                                                                    |
| knowledge-agent.agent.md      | 🟢   | approval_mode parameter docs                                                                                    |
| plan-and-implement.prompt.md  | 🟢   | **NEW** — fast-track pipeline prompt (~35 lines)                                                                |

---

## (f) Known Issues

| #   | Description                                                                             | Severity |
| --- | --------------------------------------------------------------------------------------- | -------- |
| 1   | `pushback_log` schema not formally defined in schemas.md — deferred to future iteration | Minor    |

No blocking issues. Pipeline is release-ready.

---

## (g) Telemetry Summary

| Metric                    | Value                 |
| ------------------------- | --------------------- |
| Total dispatches          | 56                    |
| Total wall-clock duration | 37,740s (~10.5 hours) |
| Error count               | 0                     |
| Retry count               | 0                     |
| Steps with retries        | None                  |

### Top 3 Slowest Steps

| Rank | Step       | Agent/Instance                       | Duration             |
| ---- | ---------- | ------------------------------------ | -------------------- |
| 1    | step-2     | spec                                 | 2,700s (45 min)      |
| 2    | step-5     | implementer wave-2 (tasks 01-03, 08) | 1,200s each (20 min) |
| 3    | step-7-fix | code-review-fixes-r2                 | 1,200s (20 min)      |

**Observations:**

- Spec agent (step-2) was the single largest bottleneck — 8 FRs with 42 sub-requirements is complex.
- Implementation wave-2 tasks ran in parallel, so actual wall-clock was 20 min not 80 min.
- Code review fix round (step-7-fix) applied 10 corrections across 7 files in one pass.
- Zero retries across 56 dispatches — excellent agent reliability.

---

## (h) Evaluation Summary

| Metric            | Value     |
| ----------------- | --------- |
| Total evaluations | 26        |
| Mean usefulness   | 8.0 / 10  |
| Mean clarity      | 8.42 / 10 |

### Worst-Rated Artifacts

| Artifact                              | Avg Usefulness | Avg Clarity |
| ------------------------------------- | -------------- | ----------- |
| tasks/task-02.yaml                    | 4.0            | 5.0         |
| implementation-reports/task-11.yaml   | 7.0            | 8.0         |
| NewAgents/.github/agents/ (aggregate) | 7.5            | 7.5         |

**Analysis:** task-02.yaml (tool-access-matrix changes) scored low likely due to high-level task description lacking specific line ranges. All verifier evaluations scored 8.0+ usefulness, indicating implementation quality is high. Adversarial reviewer evaluations averaged 8.25 usefulness across all 4 rounds, reflecting comprehensive review coverage.

---

_Generated by knowledge-agent | Run 2026-03-05T12:00:00Z | Pipeline: agent-pipeline-improvements_
