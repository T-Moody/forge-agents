# Operational Memory

## Artifact Index

| Artifact                               | Key Sections                                                                                                                                                                                                                                                                                                     | Last Updated By         |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| research/architecture.md               | §Repository Structure, §Agent Definition Format, §Pipeline Architecture, §Memory System, §Completion Contract Pattern, §Orchestrator Coordination Model, §Documentation Structure Convention, §Tool Usage Patterns, §README vs. Codebase Divergences                                                             | researcher-architecture |
| research/impact.md                     | §Artifact Consumption Map, §Existing Agent Files Requiring Modification, §Orchestrator Changes, §New Agent File, §New Storage Directories, §Files That Will NOT Be Modified, §Open Questions                                                                                                                     | researcher-impact       |
| research/dependencies.md               | §Artifact Consumption Chain, §Memory File Relationships, §Orchestrator Dispatch Patterns and Telemetry Integration Points, §Integration Points for Artifact Evaluations, §Data Flow for Post-Mortem Agent, §Orchestrator Tool Restriction, §Open Questions                                                       | researcher-dependencies |
| research/patterns.md                   | §Agent File Format Pattern, §Isolated Memory File Format Pattern, §Self-Verification Patterns, §Knowledge-Suggestions.md Pattern, §Safety Constraint Filter Pattern, §Error Handling Patterns, §YAML/Structured Output Patterns, §Naming Conventions, §Completion Contract Patterns, §Non-Blocking Agent Pattern | researcher-patterns     |
| feature.md                             | §Functional Requirements (FR-1–FR-8), §Non-Functional Requirements (7 NFRs), §Acceptance Criteria (AC-1–AC-14), §Edge Cases & Error Handling (EC-1–EC-11), §Test Scenarios (TS-1–TS-15), §Dependencies & Risks, §Constraints & Assumptions                                                                       | spec                    |
| design.md                              | §High-level Architecture, §Data Models & DTOs (YAML schemas), §APIs & Interfaces (14 agent changes + orchestrator + PostMortem), §Storage Architecture, §Sequence/Interaction Notes, §Security, §Failure & Recovery, §Tradeoffs, §Implementation Checklist                                                       | designer                |
| plan.md                                | §Feature Overview, §Success Criteria (AC mapping), §Ordered Task Index (11 tasks), §Execution Waves (3 waves), §Dependency Graph, §Implementation Specification, §Planning Constraints, §Pre-Mortem Analysis                                                                                                     | planner                 |
| tasks/01–11                            | Wave 1: 01-eval-schema, 02-postmortem-agent, 03-orch-tool-restrict, 04-prompt-update; Wave 2: 05–09-agent-evals, 10-orch-step8; Wave 3: 11-summary-doc                                                                                                                                                           | planner                 |
| review/r-quality.md                    | §Findings (2 Major, 3 Minor), §Cross-Cutting Observations, §Summary                                                                                                                                                                                                                                              | r-quality               |
| review/r-security.md                   | §Review Tier, §Findings (2 Minor), §Secrets/PII Scan, §OWASP Results                                                                                                                                                                                                                                             | r-security              |
| review/r-testing.md                    | §Findings (4 Minor), §Coverage Assessment, §Missing Test Scenarios                                                                                                                                                                                                                                               | r-testing               |
| review/r-knowledge.md                  | §Instruction/Skill/Pattern/Decision Suggestions                                                                                                                                                                                                                                                                  | r-knowledge             |
| review/knowledge-suggestions.md        | §Suggestions (9 proposals)                                                                                                                                                                                                                                                                                       | r-knowledge             |
| decisions.md                           | §All entries (6 decisions)                                                                                                                                                                                                                                                                                       | r-knowledge             |
| post-mortems/2026-02-20-post-mortem.md | §post_mortem_report (recurring issues, bottlenecks, agent scores)                                                                                                                                                                                                                                                | post-mortem             |
| agent-metrics/2026-02-20-run-log.md    | §Agent Telemetry (36 entries), §Cluster Summaries (6 clusters)                                                                                                                                                                                                                                                   | post-mortem             |

## Recent Decisions

<!-- Format: - [agent-name, step-N] Decision summary. Rationale: ... -->

- [orchestrator, step-7] R cluster decision: R-Security=DONE/Minor, R-Quality=NEEDS_REVISION/Major, R-Testing=DONE/Minor, R-Knowledge=DONE (non-blocking), initial result: NEEDS_REVISION
- [orchestrator, step-7.4] R cluster NEEDS_REVISION fix: dispatched 2 fix implementers for Major issues (post-mortem code fence, evaluation error handling). Both DONE. R cluster resolved.
- [orchestrator, step-8] PostMortem dispatch: DONE. 22 evaluations processed, 37 dispatches, 0 errors, 2 revision loops.

## Lessons Learned

<!-- Never pruned. Format: - [agent-name, step-N] Issue → Resolution. -->

- [r-quality, step-7] Design template conflict with schema → 6 agents had wrong error handling. Resolution: Always treat shared schema as authoritative over design templates; implementers should cross-reference schema directly.
- [orchestrator, step-3b] CT cluster first pass returned Critical/High → needed designer revision. Resolution: Design revision + CT re-run reduced all to Medium. CT review is effective at catching issues before planning.
- [post-mortem, step-8] Spec-design divergence most frequent recurring issue (4 reports) — feature.md not reconciled after CT revisions. Resolution: Add reconciliation step after CT revisions in future pipelines.

## Recent Updates

- [researcher-architecture, step-1] Updated `research/architecture.md` — 19-agent pipeline with dual-layer memory, 3 dispatch patterns, and standardized conventions
- [researcher-impact, step-1] Updated `research/impact.md` — 14 agents need evaluation sections, orchestrator needs telemetry/tool-restriction/step-8, 1 new agent + 3 storage dirs
- [researcher-dependencies, step-1] Updated `research/dependencies.md` — Full artifact consumption chain, 9 telemetry capture points, tool restriction conflicts, r-knowledge/post-mortem overlap
- [researcher-patterns, step-1] Updated `research/patterns.md` — 16 pattern categories, Markdown-only outputs (no YAML exists), KE-SAFE-\* model, non-blocking R-Knowledge as PostMortem template
- [spec, step-2] Updated `feature.md` — 8 FR groups, 7 NFRs, 14 ACs, 11 edge cases, 15 test scenarios covering all 6 parts + orchestrator tool restriction
- [spec, step-2] Decisions: evaluations in dedicated `artifact-evaluations/` dir as separate Markdown files with YAML blocks; per-feature scoping; PostMortem uses two-state non-blocking contract; best-effort timestamps
- [designer, step-3] Updated `design.md` — Full technical design: telemetry in memory.md Telemetry section, evaluations as separate .md files, PostMortem agent full chatagent spec, orchestrator tool restriction with Step 0 fallbacks
- [designer, step-3] Decisions: telemetry accumulates in memory.md (never pruned); lazy directory creation; PostMortem produces both run-log and report; evaluation files use bare agent names with date-append collision avoidance
- [orchestrator, step-3b] CT cluster decision: severity values [High, High, High, Critical], result: NEEDS_REVISION. Routing to designer for revision with CT artifact paths.
- [orchestrator, step-3b] CT cluster re-run decision: severity values [Medium, Medium, Medium, Medium], result: DONE. Proceeding to planning.
- [planner, step-4] Updated `plan.md` + 11 task files — 3 waves, 2 tracks (A=self-improvement, B=tool restriction), orchestrator file sequenced (03→10)
- [implementer-01, step-5] Created `.github/agents/evaluation-schema.md` — shared YAML schema reference doc
- [implementer-02, step-5] Created `.github/agents/post-mortem.agent.md` — full chatagent PostMortem agent
- [implementer-03, step-5] Modified `.github/agents/orchestrator.agent.md` — Track B tool restriction applied (8 sections changed)
- [implementer-04, step-5] Modified `.github/prompts/feature-workflow.prompt.md` — 3 new Key Artifacts + 2 new Rules
- [implementer-05, step-5] Modified spec.agent.md, designer.agent.md, planner.agent.md — added evaluation steps
- [implementer-06, step-5] Modified ct-security, ct-scalability, ct-maintainability — added evaluation steps
- [implementer-07, step-5] Modified ct-strategy, implementer, documentation-writer — added evaluation steps
- [implementer-08, step-5] Modified v-tests, v-tasks, v-feature — added evaluation steps
- [implementer-09, step-5] Modified r-quality, r-testing — added evaluation steps
- [implementer-10, step-5] Modified orchestrator.agent.md — Step 8 PostMortem, telemetry tracking, doc structure, expectations table
- [documentation-writer-11, step-5] Created `docs/feature/self-improvement-system/summary.md` — FR-8.1 deliverables documentation
- [orchestrator, step-6] V cluster decision: V-Build=DONE, V-Tests=DONE, V-Tasks=DONE, V-Feature=DONE, result: DONE. All 14 ACs met, 11/11 tasks verified, 22 structural checks passing. 1 low-severity warning (post-mortem.agent.md code fence format).
- [orchestrator, step-7] R cluster complete. 4 sub-agents dispatched: r-security (Minor), r-quality (Major), r-testing (Minor), r-knowledge (DONE).
- [orchestrator, step-7.4] Fix pass: post-mortem.agent.md code fence removed, 6 agents + design.md "skip evaluation" updated to "evaluation_error block".
- [orchestrator, step-8] PostMortem complete. Run log at agent-metrics/2026-02-20-run-log.md. Post-mortem at post-mortems/2026-02-20-post-mortem.md.
