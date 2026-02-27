# Operational Memory

## Artifact Index

| Artifact                   | Key Sections                                                                                                                                                                                                                                                                                                                                           | Last Updated By      |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------- |
| design.md (v4)             | §Architecture Selection, §Agent Inventory (9, Orchestrator 5 responsibilities), §Pipeline Structure (Steps 0–9), §Communication (typed YAML), §Memory System (zero-merge), §Verification (SQL-primary, WAL, corrected gates), §Adversarial Review (perspective-diverse), §Deviation Records (DR-1, DR-2), §Evidence Bundle (Step 8b), §v4 Revision Log | designer (v4)        |
| feature.md                 | §Architectural Directions (5), §Common Requirements (CR-1–15), §Functional Requirements (FR-1–9), §Acceptance Criteria (AC-1–15)                                                                                                                                                                                                                       | spec                 |
| plan.md                    | §Ordered Task Index (13 tasks), §Execution Waves (3), §Dependency Graph, §Pre-Mortem Analysis                                                                                                                                                                                                                                                          | planner              |
| tasks/01-13                | 13 task files with acceptance criteria, relevant_context, agent assignments                                                                                                                                                                                                                                                                            | planner              |
| schemas.md (to create)     | 10 typed YAML schemas                                                                                                                                                                                                                                                                                                                                  | —                    |
| ct-review/\*.md            | CT findings (residual planning constraints)                                                                                                                                                                                                                                                                                                            | ct-\* agents         |
| adversarial-review-1.md    | Round 1 review from gpt-5.3-codex perspective: schema gaps, WAL, COUNT gates, tool contradiction, reviewer YAML output, centralized init, task_id semantics                                                                                                                                                                                            | adversarial-reviewer |
| adversarial-review-2.md    | Round 1 review from gemini-3-pro perspective: WAL, task_id gap, baseline impossibility, FR-1.6 contradiction, Tier 4 ops readiness, session recall, git staging, diff scalability                                                                                                                                                                      | adversarial-reviewer |
| adversarial-review-3.md    | Round 1 review from claude-opus-4.6 perspective: model routing unverified, self-fix loop lost, evidence bundle missing, spec deviations, perspective diversity, pushback timing, FR-4.7 revert, dropped Anvil features                                                                                                                                 | adversarial-reviewer |
| adversarial-review-r2-1.md | Round 2 APPROVE (gpt-5.3-codex, security): all 12 R1 findings resolved; 2 new (1M git add -A sensitive files, 1L nullable verdict/severity)                                                                                                                                                                                                            | adversarial-reviewer |
| adversarial-review-r2-2.md | Round 2 APPROVE (gemini-3-pro, architecture/correctness): 12/15 resolved, 1 partial, 2 retained; 4 new (1H duplicate schema, 1M git tag collision, 1M check_name convention, 1H diff scalability)                                                                                                                                                      | adversarial-reviewer |
| adversarial-review-r2-3.md | Round 2 APPROVE (claude-opus-4.6, scalability/maintainability): 9 resolved, 4 partial, 2 unresolved; 4 new (2M evidence bundle silent fail + schema inconsistency, 2L rollback + secrets scanning gaps)                                                                                                                                                | adversarial-reviewer |

## Recent Decisions

- [designer, step-3 v2] 9 agents, zero-merge typed YAML, YAML-primary verification, orchestrator 3 responsibilities
- [planner, step-4] 13 tasks across 3 waves. schemas.md is critical path. All 9 agents independent in Wave 2.
- [planner, step-4] Wave 2 (9 agents) split into 3 sub-waves of ≤4 due to concurrency cap.
- [planner, step-4] Only Task 13 (README) uses documentation-writer; all others use implementer.
- [designer, step-3 v4] Schema expansion: verdict/severity/round/run_id added to anvil_checks + composite indexes
- [designer, step-3 v4] WAL mandate: PRAGMA journal_mode=WAL + busy_timeout=5000 in centralized Step 0 init
- [designer, step-3 v4] Deviation Records: DR-1 (run_in_terminal vs FR-1.6), DR-2 (always-3 reviewers vs FR-5.3/CR-4)
- [designer, step-3 v4] Evidence gates corrected: all queries filter on verdict/round/run_id/check_name; 6 gate types
- [designer, step-3 v4] Evidence bundle assembly: Step 8b non-blocking by Knowledge Agent
- [designer, step-3 v4] Self-fix loop: Implementer max 2 self-fix attempts + git add -A after fix + FR-4.7 revert via git checkout baseline tag
- [designer, step-3 v4] Early pushback: lightweight evaluation in Step 0 before researcher dispatches; Blocker = halt
- [designer, step-3 v4] Auto-commit as Step 9; git hygiene in Step 0; session recall intentionally omitted
- [adversarial-review-r2, step-3b] Design v4 approved: all 3 reviewers APPROVE — ready for implementation (Wave 1)

## Lessons Learned

- [ct-strategy, step-3b] Design claims must be honest about enforcement boundaries
- [ct-security, step-3b] COUNT-based evidence gates verify quantity not truthfulness
- [ct-scalability, step-3b] Dispatch count claims only valid at stated feature size
- [adversarial-review, step-3b] COUNT-based gates need WHERE filters on verdict='approve' and round; without them gates pass silently when all reviewers found issues
- [adversarial-review, step-3b] Concurrent SQLite writes (≤4 parallel agents) require WAL mode + busy_timeout; default journal mode causes SQLITE_BUSY
- [adversarial-review, step-3b] Multi-model routing via .agent.md model directives is unverified; perspective-diverse prompts (security/architecture/correctness) provide genuine analytical diversity as fallback
- [adversarial-review, step-3b] Baseline cross-check requires pre-implementation git state (tags/commits); re-running diagnostics on modified files is physically impossible
- [adversarial-review, step-3b] anvil_checks schema must include verdict/severity/round/run_id for multi-agent multi-round pipelines; single-agent schema creates evidence gating bugs
- [adversarial-review, step-3b] Tight verify-fix loop within one agent (Anvil pattern) is structurally impossible in multi-agent pipeline; Implementer self-fix loop partially recovers this
- [adversarial-review, step-3b] Pushback placement matters — firing after researcher dispatches wastes compute on bad requests; early Step 0 pushback mitigates
- [adversarial-review-r2, step-3b] Duplicate schema definitions across design sections (e.g., `anvil_checks` in §Decision 6 vs §Data Storage) create conflicting constraints (output_snippet 500 vs 2000, severity enums mismatch); must reconcile before implementation
- [adversarial-review-r2, step-3b] `git add -A` relies entirely on `.gitignore` for sensitive file exclusion; absent or incomplete `.gitignore` risks leaking secrets into version control
- [adversarial-review-r2, step-3b] `git tag` with fixed naming (e.g., `pipeline-baseline-{run_id}`) fails on replan iteration 2+ when tag already exists; use `git tag -f` or unique suffixes
- [adversarial-review-r2, step-3b] Non-blocking evidence bundle assembly means primary user deliverable silently disappears if Knowledge Agent fails; consider fallback or at minimum a warning
- [adversarial-review-r2, step-3b] `check_name` value conventions for SQL INSERT must be documented; evidence gate LIKE patterns depend on consistent naming

## Recent Updates

- [planner, step-4] Created plan.md with 13 tasks in 3 waves; 13 task files in tasks/
- [adversarial-review, step-3b] Round 1 complete: 3-model adversarial review produced 17 findings (6 Critical, 11 High) across design.md v3
- [designer, step-3 v4] design.md revised v3→v4 addressing all 17 adversarial findings: schema expansion, WAL mandate, deviation records, corrected evidence gates, evidence bundle, self-fix loop, early pushback, auto-commit, git hygiene, Tier 4 ops readiness
- [adversarial-review-r2, step-3b] Round 2 complete: all 3 reviewers APPROVE design v4; 10 new non-blocking findings (1H, 5M, 4L); design approved for planning/implementation

## Plan Summary

**Wave 1 (3 tasks, parallel):** Task 01 schemas.md, Task 02 dispatch-patterns+severity-taxonomy, Task 03 feature-workflow prompt
**Wave 2 (9 tasks, 3 sub-waves of ≤4):** Tasks 04-12 — all 9 agent definitions
**Wave 3 (1 task):** Task 13 README with Mermaid diagram

**Critical path:** schemas.md (Task 01) → all Wave 2 tasks → README (Task 13)
**Total dispatches:** ~13 implementation + verification
