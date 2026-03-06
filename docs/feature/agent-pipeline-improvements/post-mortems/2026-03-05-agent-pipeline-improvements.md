# Post-Mortem — agent-pipeline-improvements

**Run ID:** 2026-03-05T12:00:00Z
**Date:** 2026-03-05
**Pipeline Outcome:** SUCCESS

---

## Summary

The agent-pipeline-improvements feature addressed 8 user-reported issues across 15 files (14 agent instruction files + 1 new prompt). The pipeline completed with 100% agent reliability (56/56 dispatches successful, 0 retries), all 11 implementation tasks verified (92/92 checks passed), and all 3 code reviewers approving in R2 after a single fix round resolved 2 Critical + 6 Major + 7 Minor findings.

---

## What Worked

1. **Wave decomposition**: 3-wave plan with correct dependency ordering. Wave-1 (foundation schemas + reference docs) enabled Wave-2 (core agent files) cleanly.
2. **Design review effectiveness**: R1 caught 5 Major gaps (minimum step set, FR contradiction, approval_mode coverage, dual anti-drift). All resolved in R2 before implementation started.
3. **Zero-retry implementation**: All 11 tasks completed on first attempt. Implementer self-checks caught issues pre-verification.
4. **100% verification pass rate**: 92 checks, 0 failures across all 11 tasks. Verifier correctly applied TDD fallback (EC-2) for documentation-only tasks.
5. **Single code review fix round**: 10 corrections across 7 files addressed all 15 R1 findings. R2 produced 0 new findings.
6. **Hybrid modification strategy (Direction C)**: Successfully kept orchestrator at 539/550 lines by extracting complex logic to reference documents while inlining simple changes.

---

## What Could Be Improved

1. **Task description specificity**: task-02.yaml scored 4.0/10 usefulness — the lowest of all artifacts. Task descriptions for tool-access-matrix changes should include specific line ranges and section references.
2. **Cross-reference validation**: Both Critical code review findings were broken cross-references (§11→§1.1, EG-10 threshold mismatch). Implementer self-checks should include cross-file reference validation.
3. **Spec agent bottleneck**: Spec step took 2,700s (longest single step) processing 8 FRs with 42 sub-requirements. Consider splitting complex features into sub-specifications.

---

## Bottleneck Analysis (Top 3)

| Rank | Step                 | Duration | Cause                                                         | Mitigation                                                 |
| ---- | -------------------- | -------- | ------------------------------------------------------------- | ---------------------------------------------------------- |
| 1    | step-2 (spec)        | 2,700s   | 8 FRs × 42 sub-requirements from 8 user issues                | Split large features into smaller specs                    |
| 2    | step-5 wave-2 (impl) | 1,200s   | 5 parallel tasks, task-04 (🔴 orchestrator) was critical path | Orchestrator changes inherently slow — acceptable          |
| 3    | step-7-fix           | 1,200s   | 10 corrections across 7 files in single pass                  | Multi-file fix rounds are efficient — no mitigation needed |

---

## Agent Reliability Metrics

| Agent                | Dispatches | Success Rate | Avg Duration (s) |
| -------------------- | ---------- | ------------ | ---------------- |
| researcher           | 4          | 100%         | 300              |
| spec                 | 2          | 100%         | 1,500            |
| designer             | 2          | 100%         | 300              |
| planner              | 1          | 100%         | 300              |
| implementer          | 24         | 100%         | 688              |
| verifier             | 11         | 100%         | 709              |
| adversarial-reviewer | 12         | 100%         | 550              |

**No agents required retries.** This is the highest reliability observed in a 🔴-risk pipeline run.

---

## Evaluation Quality Analysis

- **Mean usefulness:** 8.0/10 — strong overall quality
- **Mean clarity:** 8.42/10 — artifacts are well-structured
- **Worst artifact:** tasks/task-02.yaml (4.0 usefulness, 5.0 clarity) — task description lacked specificity
- **Best artifacts:** Adversarial review verdicts (8.25 avg) — consistent quality across all 4 review rounds
- **Gap pattern:** No common missing information patterns detected across evaluations

---

## Root Cause Analysis — Code Review Findings

### Critical Finding #1: Broken §11 cross-reference

- **Root cause**: Orchestrator referenced sql-templates.md §11 for archive query, but the section was created as §1.1. The implementer used the section number from the design document rather than verifying the actual heading in the modified file.
- **Fix**: Corrected to §1.1 across all references.
- **Prevention**: Add cross-file reference check to implementer self-verification.

### Critical Finding #2: Fast-track 🔴 risk bypass

- **Root cause**: plan-and-implement.prompt.md defined reduced step set but had no enforcement preventing 🔴-risk tasks from skipping design review. The design specified D-14 (minimum steps) but the implementation didn't add the escalation clause.
- **Fix**: Added 🔴 risk escalation clause to fast-track prompt.
- **Prevention**: Security-sensitive constraints in anti-drift anchors should be cross-referenced in all entry points (prompts, agents).

---

## Key Decisions & Their Outcomes

| Decision                 | Outcome                                                                    |
| ------------------------ | -------------------------------------------------------------------------- |
| D-2 (E2E decouple)       | Clean implementation, 0 verification failures                              |
| D-4 (Approval mode fix)  | Required touching all 8 subagent files — correctly identified in design R2 |
| D-10 (Fast-track prompt) | Required 🔴 escalation clause (caught in code review R1)                   |
| D-14 (Minimum step set)  | Successfully prevented security bypass in fast-track                       |
| Direction C (Hybrid)     | Net +0 lines on orchestrator (539→539) — excellent budget management       |

---

## Improvement Recommendations

1. **Cross-file reference validation**: Add to implementer self-check — verify all §section references, EG threshold values, and check_names match across referencing files.
2. **Task description quality gate**: Require specific line ranges in task descriptions for modification tasks (prevents low-usefulness evaluations).
3. **Spec decomposition heuristic**: If FR count > 5 or sub-requirement count > 30, consider splitting into multiple spec rounds.
4. **pushback_log schema formalization**: Deferred from this pipeline — should be added to schemas.md in next iteration.

---

_Generated by knowledge-agent | Run 2026-03-05T12:00:00Z_
