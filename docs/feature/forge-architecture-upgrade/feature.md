# Feature Specification: Forge Architecture Upgrade

## Overview

A major architecture upgrade to the Forge multi-agent orchestration framework that introduces an operational memory system, parallelizes three sequential bottlenecks into agent clusters (Critical Thinking ×4, Verification ×4, Review ×4 with Knowledge Evolution), expands research from 3→4 parallel agents, and evolves the orchestrator to coordinate all new capabilities. All changes are markdown-only (no runtime code). The upgrade grows the agent count from 10+1 to 25+1 and targets a ~10–18% end-to-end pipeline speedup.

## Background & Context

- **Initial Request:** [initial-request.md](initial-request.md) — defines all requirements, current problems, and deliverables.
- **Analysis:** [analysis.md](analysis.md) — synthesized research covering architecture, impact, dependencies, risks, and performance projections.
- **Existing Agents:** `NewAgentsAndPrompts/` — 10 agent definitions + 1 prompt file following the v2 structural template.
- **Key Reference Docs:** `docs/optimization-from-gem-team.md` (memory concept origin), `docs/comparison-forge-vs-gem-team.md` (two-layer verification model).

## Goals

1. **Reduce redundant artifact reads** by introducing an operational memory layer that provides navigational metadata, recent decisions, and lessons learned — agents read memory first, full artifacts only when needed.
2. **Parallelize Critical Thinking** from 1 sequential agent to 4 concurrent sub-agents with aggregation, achieving ~2–2.5× practical speedup for Step 3b.
3. **Parallelize Verification** from 1 sequential agent to a build-then-parallel cluster (1 sequential + 3 concurrent + aggregation), achieving ~1.5–2× practical speedup for Step 6.
4. **Parallelize Review** from 1 sequential agent to 4 concurrent sub-agents with aggregation, achieving ~2.5–3× practical speedup for Step 7, while adding Knowledge Evolution capabilities.
5. **Expand Research** from 3 to 4 parallel agents to increase analytical coverage.
6. **Evolve the Orchestrator** to coordinate memory lifecycle, cluster dispatch/aggregation, and knowledge evolution workflows.
7. **Achieve ~10–18% end-to-end pipeline speedup** while preserving deterministic, artifact-grounded behavior (accounting for ~20 additional agent invocations).
8. **Maintain the two-layer verification model** (unit-level TDD by implementer + integration-level by verifier) as a core design invariant.

## Non-Goals

1. **Runtime code or infrastructure** — Forge remains a purely declarative markdown system. No code, databases, message queues, or YAML state files are introduced.
2. **Parallelizing the sequential middle pipeline** — Spec → Design → Planning remains sequential; each depends on the prior stage's output.
3. **Automated testing framework** — Validation continues via full pipeline runs on real feature requests.
4. **Replacing artifacts as the source of truth** — Memory is a navigation/summary layer only; artifacts remain authoritative.
5. **Autonomous self-modification** — Knowledge Evolution produces suggestions only; it does not autonomously apply changes to agent definitions.
6. **Reducing implementation wave parallelism** — The existing max-4 concurrent subagent cap and implementation wave model are unchanged.

---

## Feature 1: Operational Memory System

### Description

A lightweight, file-based operational memory layer stored as a single markdown file (`docs/feature/<feature-slug>/memory.md`) that provides cross-agent navigational context without duplicating or summarizing full artifact content. Memory is initialized by the orchestrator at Step 0 and updated by aggregators and sequential agents after producing output. Parallel sub-agents read memory but do not write to it — they communicate findings through intermediate output files. Artifacts remain the single source of truth.

### Functional Requirements

| ID        | Requirement                                                                                                                                                                                                                                                                                                                                           |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MEM-FR-1  | Memory is stored as a single markdown file at `docs/feature/<feature-slug>/memory.md`.                                                                                                                                                                                                                                                                |
| MEM-FR-2  | Memory contains exactly four sections: **Artifact Index** (file paths + section headings), **Recent Decisions** (last N decisions with rationale and source), **Lessons Learned** (issues encountered and resolutions), **Recent Updates** (timestamped log of last N artifact changes).                                                              |
| MEM-FR-3  | The orchestrator initializes memory at Step 0 with empty section templates before any agent runs.                                                                                                                                                                                                                                                     |
| MEM-FR-4  | Every agent reads `memory.md` as the FIRST input before accessing any other artifact.                                                                                                                                                                                                                                                                 |
| MEM-FR-5  | Only aggregators and sequential agents append to the relevant memory sections after producing output. Parallel sub-agents do NOT write to `memory.md` — they communicate findings through their intermediate output files.                                                                                                                            |
| MEM-FR-6  | Memory entries include the agent name, phase/step, and a brief summary (≤2 sentences per entry).                                                                                                                                                                                                                                                      |
| MEM-FR-7  | On revision loops (design loop, replan loop, review fix loop), the orchestrator invalidates stale memory entries by marking them with `[INVALIDATED — revision in progress]` and the reason.                                                                                                                                                          |
| MEM-FR-8  | After a revision loop completes, the revising agent writes corrected memory entries to replace invalidated ones.                                                                                                                                                                                                                                      |
| MEM-FR-9  | Parallel agents (research, implementation waves, cluster sub-agents) do NOT write to `memory.md`. They communicate findings exclusively through their intermediate output files (e.g., `research/<focus>.md`, `ct-review/ct-<focus>.md`, `verification/v-<focus>.md`, `review/r-<focus>.md`). Only aggregators and sequential agents write to memory. |
| MEM-FR-10 | Memory is pruned by the orchestrator at phase boundaries: entries older than 2 completed phases are removed (except Lessons Learned, which persists for the full run).                                                                                                                                                                                |

### File Structure

```markdown
# Operational Memory

## Artifact Index

<!-- Updated by each agent after producing output -->

| Artifact | Sections | Last Updated By |
| -------- | -------- | --------------- |

## Recent Decisions

<!-- Append-only within a phase; pruned at phase boundaries -->

- [agent-name, step-N] Decision summary. Rationale: ...

## Lessons Learned

<!-- Append-only; persists for full pipeline run -->

- [agent-name, step-N] Issue encountered → Resolution applied.

## Recent Updates

<!-- Append-only within a phase; pruned at phase boundaries -->

- [agent-name, step-N] Updated `artifact-path` — summary of change.
```

### Read/Write Responsibilities

| Agent                              | Reads Memory                                     | Writes Memory                     | Sections Written                                                                       |
| ---------------------------------- | ------------------------------------------------ | --------------------------------- | -------------------------------------------------------------------------------------- |
| Orchestrator                       | Yes (lifecycle management)                       | Yes (init, invalidation, pruning) | All sections (management only)                                                         |
| Researcher (×4 focused)            | Yes (low value at Step 1 — memory is near-empty) | **No**                            | — (findings go to `research/<focus>.md`)                                               |
| Researcher (synthesis)             | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Updates                                                         |
| Spec Agent                         | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Decisions, Recent Updates                                       |
| Designer                           | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Decisions, Recent Updates                                       |
| CT Sub-Agents (×4)                 | Yes                                              | **No**                            | — (findings go to `ct-review/ct-<focus>.md`)                                           |
| CT Aggregator                      | Yes                                              | Yes (sequential)                  | Recent Decisions, Recent Updates                                                       |
| Planner                            | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Decisions, Recent Updates                                       |
| Implementer (×N)                   | Yes                                              | **No**                            | — (findings go to task output; Lessons Learned captured by orchestrator between waves) |
| V-Build                            | Yes                                              | **No**                            | — (findings go to `verification/v-build.md`)                                           |
| V-Tests / V-Tasks / V-Feature      | Yes                                              | **No**                            | — (findings go to `verification/v-*.md`)                                               |
| V Aggregator                       | Yes                                              | Yes (sequential)                  | Recent Updates, Lessons Learned                                                        |
| R-Quality / R-Security / R-Testing | Yes                                              | **No**                            | — (findings go to `review/r-*.md`)                                                     |
| R-Knowledge                        | Yes                                              | **No**                            | — (findings go to `review/r-knowledge.md`)                                             |
| R Aggregator                       | Yes                                              | Yes (sequential)                  | Recent Decisions, Recent Updates, Lessons Learned                                      |
| Documentation Writer               | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Updates                                                         |

### Acceptance Criteria

| ID       | Criterion                                                        | Pass/Fail Definition                                                                                                                                                                                                                                                                         |
| -------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MEM-AC-1 | Memory file exists after Step 0                                  | `memory.md` is present with all section headers and empty templates after orchestrator setup completes.                                                                                                                                                                                      |
| MEM-AC-2 | Every agent reads memory first                                   | Each agent's Inputs section lists `memory.md` as the first input. Workflow Step 1 of every agent is "Read memory.md."                                                                                                                                                                        |
| MEM-AC-3 | Only aggregators and sequential agents write memory after output | Only aggregators and sequential agents include a memory write step (before completion contract). Parallel sub-agents do NOT write to `memory.md`.                                                                                                                                            |
| MEM-AC-4 | Stale entries are invalidated on revision loops                  | When a `NEEDS_REVISION` loop is triggered, the orchestrator marks affected memory entries with `[INVALIDATED]` before dispatching the revision agent. Verifiable by inspecting `memory.md` during a revision loop.                                                                           |
| MEM-AC-5 | Parallel agents read memory but do not write                     | Parallel sub-agents (researchers, implementers, cluster sub-agents) read `memory.md` but do NOT write to it. They communicate findings through their intermediate output files. Aggregators consolidate relevant findings into root-level memory sections after all parallel work completes. |
| MEM-AC-6 | Memory is pruned at phase boundaries                             | After completing a pipeline step, old entries (>2 phases back) are removed from Recent Decisions, Recent Updates, and Artifact Index. Lessons Learned entries are preserved.                                                                                                                 |
| MEM-AC-7 | Memory does not duplicate artifact content                       | No memory entry contains more than 2 sentences of summary. No full code snippets, full section text, or artifact reproductions appear in memory.                                                                                                                                             |
| MEM-AC-8 | Memory size stays bounded                                        | At any point during pipeline execution, `memory.md` does not exceed 200 lines.                                                                                                                                                                                                               |

### Error Handling

| Condition                                  | Expected Behavior                                                                                                                                                                                                                         | Severity |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| `memory.md` is missing when an agent reads | Agent logs a warning in its output, proceeds without memory (falls back to direct artifact reads). Pipeline continues.                                                                                                                    | High     |
| `memory.md` is corrupted or unparseable    | Orchestrator re-initializes memory with empty templates. Downstream agents start fresh. Orchestrator logs the corruption event.                                                                                                           | High     |
| Parallel agent memory isolation            | Parallel sub-agents do NOT write to `memory.md`, eliminating write conflicts entirely. They communicate findings through their intermediate output files. Aggregators consolidate findings into memory after all parallel work completes. | Low      |
| Memory exceeds size limit                  | Orchestrator triggers emergency pruning: removes all entries except Lessons Learned and current-phase entries.                                                                                                                            | Medium   |

---

## Feature 2: Critical Thinking Cluster

### Description

The single sequential critical thinker (Step 3b) is replaced by a cluster of 4 parallel sub-agents, each covering a focused subset of risk categories, followed by an aggregator that merges findings into a single `design_critical_review.md` artifact.

### Sub-Agent Responsibilities

| Sub-Agent          | File Name                     | Risk Categories                                                         |
| ------------------ | ----------------------------- | ----------------------------------------------------------------------- |
| CT-Security        | `ct-security.agent.md`        | Security vulnerabilities, backwards compatibility risks                 |
| CT-Scalability     | `ct-scalability.agent.md`     | Scalability bottlenecks, performance implications                       |
| CT-Maintainability | `ct-maintainability.agent.md` | Maintainability concerns, complexity risks, integration risks           |
| CT-Strategy        | `ct-strategy.agent.md`        | Strategic risks, scope risks, edge cases, fundamental approach validity |

**Common sub-agent contract:**

- **Inputs:** `initial-request.md`, `design.md`, `feature.md`, `memory.md`
- **Output:** `docs/feature/<feature-slug>/ct-review/ct-<focus>.md` (intermediate finding file)
- **Completion contract:** `DONE:` / `ERROR:`
- **Constraint:** Sub-agents NEVER propose solutions — they identify problems only.
- **Cross-boundary flag:** Each sub-agent must include a "Cross-Cutting Observations" section for issues that span their scope boundary.

### Aggregation Strategy

| Component               | Detail                                                                                                                                                                                 |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Aggregator file         | `ct-aggregator.agent.md`                                                                                                                                                               |
| Inputs                  | All 4 sub-agent output files + `design.md` + `memory.md`                                                                                                                               |
| Output                  | `docs/feature/<feature-slug>/design_critical_review.md`                                                                                                                                |
| Conflict handling       | Conflicting findings across sub-agents are surfaced explicitly as "Unresolved Tension" items — the aggregator does NOT resolve conflicts; it presents both sides for the designer.     |
| Deduplication           | Findings appearing in multiple sub-agent outputs are merged into a single entry with attribution to all originating sub-agents.                                                        |
| Cross-cutting synthesis | Cross-Cutting Observations from all sub-agents are synthesized into a dedicated section.                                                                                               |
| Completion contract     | `DONE:` (all sub-agents DONE, aggregation complete) / `NEEDS_REVISION:` (any critically-rated finding requires design change) / `ERROR:` (any sub-agent ERROR, or aggregation failure) |
| Severity threshold      | `NEEDS_REVISION` is triggered only if at least one finding is rated `critical` or `high` severity. Medium/low findings are reported but do not block.                                  |

### Pipeline Flow

```
Designer → [CT-Security, CT-Scalability, CT-Maintainability, CT-Strategy] (parallel ×4)
         → CT-Aggregator
         → design_critical_review.md
         → NEEDS_REVISION? → Designer (max 1 loop) → re-run full cluster
```

### Acceptance Criteria

| ID      | Criterion                                     | Pass/Fail Definition                                                                                                                                              |
| ------- | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CT-AC-1 | Four CT sub-agents run in parallel            | Orchestrator dispatches all 4 CT sub-agents concurrently (not sequentially). All 4 complete before the aggregator is invoked.                                     |
| CT-AC-2 | Each sub-agent covers its assigned categories | Each sub-agent's output file contains findings addressing only its assigned risk categories (plus a Cross-Cutting Observations section).                          |
| CT-AC-3 | Aggregator produces unified review            | `design_critical_review.md` contains findings from all 4 sub-agents, organized by severity, with attribution.                                                     |
| CT-AC-4 | Conflicts are surfaced, not resolved          | When two sub-agents produce contradictory findings, both appear in an "Unresolved Tensions" section with references to both originating analyses.                 |
| CT-AC-5 | Cross-cutting issues are synthesized          | The aggregated output contains a "Cross-Cutting Concerns" section drawing from all sub-agents' Cross-Cutting Observations.                                        |
| CT-AC-6 | NEEDS_REVISION triggers full cluster re-run   | On revision, the designer revises `design.md`, then all 4 CT sub-agents and the aggregator run again (not a subset).                                              |
| CT-AC-7 | Max 1 revision loop preserved                 | The design → critical thinking loop executes at most once. If NEEDS_REVISION persists after revision, the pipeline proceeds with findings documented.             |
| CT-AC-8 | Sub-agents follow v2 template                 | Each sub-agent file contains: YAML frontmatter, role statement, inputs/outputs, 5 operating rules, workflow, output spec, completion contract, anti-drift anchor. |
| CT-AC-9 | Sub-agent intermediate files are preserved    | All 4 intermediate finding files (`ct-security.md`, etc.) persist alongside `design_critical_review.md` for traceability.                                         |

### Error Handling

| Condition                       | Expected Behavior                                                                                                                                                                 | Severity |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| 1 of 4 sub-agents returns ERROR | Aggregator proceeds with 3 available outputs. Missing coverage area is noted in `design_critical_review.md`. Orchestrator retries the failing sub-agent once (per Global Rule 4). | High     |
| 2+ sub-agents return ERROR      | Aggregator returns ERROR. Orchestrator retries the entire cluster once.                                                                                                           | Critical |
| Aggregator returns ERROR        | Orchestrator retries the aggregator once with the same sub-agent outputs. If it fails again, pipeline halts with ERROR.                                                           | Critical |
| Sub-agent produces empty output | Aggregator treats as a DONE with no findings for that category. Logs a warning.                                                                                                   | Medium   |

---

## Feature 3: Verification Cluster

### Description

The single sequential verifier (Step 6) is replaced by a cluster of 4 sub-agents with a mandatory sequential-then-parallel execution model: V-Build runs first; only if it succeeds do V-Tests, V-Tasks, and V-Feature run in parallel. An aggregator merges all results into `verifier.md`.

### Sub-Agent Responsibilities

| Sub-Agent | File Name            | Responsibility                                                     | Execution Order             |
| --------- | -------------------- | ------------------------------------------------------------------ | --------------------------- |
| V-Build   | `v-build.agent.md`   | Detect build system, compile/build project, report build status    | **First** (sequential gate) |
| V-Tests   | `v-tests.agent.md`   | Run full test suite, analyze results, report pass/fail/skip counts | After V-Build passes        |
| V-Tasks   | `v-tasks.agent.md`   | Verify each task's acceptance criteria against implemented code    | After V-Build passes        |
| V-Feature | `v-feature.agent.md` | Verify feature-level acceptance criteria from `feature.md`         | After V-Build passes        |

**V-Build contract:**

- **Inputs:** Entire codebase, `memory.md`
- **Output:** `docs/feature/<feature-slug>/verification/v-build.md` (build status, build system details, build artifact paths)
- **Completion contract:** `DONE:` (build passes) / `ERROR:` (build fails — blocks remaining sub-agents)
- **Critical constraint:** V-Build must produce the build artifact paths and environment details needed by downstream sub-agents.

**V-Tests / V-Tasks / V-Feature common contract:**

- **Inputs:** Entire codebase, `memory.md`, `v-build.md` (for build context), plus `feature.md`, `plan.md`, `tasks/*.md` as relevant
- **Output:** `docs/feature/<feature-slug>/verification/v-<focus>.md`
- **Completion contract:** `DONE:` / `NEEDS_REVISION:` / `ERROR:`
- **Constraint:** Verifier sub-agents NEVER modify source code, test files, or configuration files.

### Aggregation Strategy

| Component           | Detail                                                                                                                                   |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Aggregator file     | `v-aggregator.agent.md`                                                                                                                  |
| Inputs              | All 4 sub-agent outputs + `memory.md`                                                                                                    |
| Output              | `docs/feature/<feature-slug>/verifier.md`                                                                                                |
| Completion contract | See table below                                                                                                                          |
| Replan triggering   | `NEEDS_REVISION` from the aggregator triggers the orchestrator's replan loop (planner → implementer → re-verify, max 3 total iterations) |

**Completion contract aggregation:**

| Scenario                                    | Aggregated Result                                              |
| ------------------------------------------- | -------------------------------------------------------------- |
| V-Build DONE + all parallel DONE            | DONE                                                           |
| V-Build ERROR                               | ERROR (pipeline cannot verify; blocks all downstream)          |
| V-Build DONE + any parallel NEEDS_REVISION  | NEEDS_REVISION (merge failing criteria for replan)             |
| V-Build DONE + any parallel ERROR           | ERROR (retry failing sub-agent once, then ERROR if persistent) |
| V-Build DONE + mixed NEEDS_REVISION + ERROR | ERROR takes priority                                           |

### Pipeline Flow

```
Implementation waves complete
  → V-Build (sequential)
    → Build fails? → ERROR → Replan loop
    → Build passes? → [V-Tests, V-Tasks, V-Feature] (parallel ×3)
      → V-Aggregator
        → verifier.md
        → NEEDS_REVISION? → Replan (planner re-plans → implementers fix → full cluster re-run)
        → Max 3 replan iterations
```

### Acceptance Criteria

| ID      | Criterion                                    | Pass/Fail Definition                                                                                                                                                       |
| ------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| V-AC-1  | V-Build runs before other sub-agents         | V-Tests, V-Tasks, and V-Feature are never dispatched until V-Build returns DONE.                                                                                           |
| V-AC-2  | Three sub-agents run in parallel after build | V-Tests, V-Tasks, and V-Feature are dispatched concurrently (not sequentially) after V-Build succeeds.                                                                     |
| V-AC-3  | Build failure blocks verification            | If V-Build returns ERROR, no other verification sub-agent is dispatched. The aggregator is not invoked. The orchestrator enters replan mode.                               |
| V-AC-4  | Aggregator produces unified verifier.md      | `verifier.md` contains build results, test results, per-task verification, and feature-level verification — all with clear pass/fail status per criterion.                 |
| V-AC-5  | Replan loops re-run full cluster             | On NEEDS_REVISION, after replanning and re-implementation, the entire verification cluster runs again from V-Build, not a partial subset.                                  |
| V-AC-6  | Max 3 replan iterations preserved            | The verification-replan loop executes at most 3 times. If issues persist, the pipeline proceeds with findings documented in `verifier.md`.                                 |
| V-AC-7  | Two-layer verification model preserved       | V sub-agents perform integration-level verification only. Unit-level TDD remains the implementer's responsibility. No V sub-agent duplicates unit-level checks.            |
| V-AC-8  | Sub-agents follow v2 template                | Each sub-agent file contains: YAML frontmatter, role statement, inputs/outputs, 5 operating rules, workflow, output spec, completion contract, anti-drift anchor.          |
| V-AC-9  | Build context is shared via artifact         | V-Build writes build system details and artifact paths to `v-build.md`. The 3 parallel sub-agents read this file for build context (not relying on shared terminal state). |
| V-AC-10 | Intermediate verification files preserved    | All 4 intermediate verification files persist in `docs/feature/<feature-slug>/verification/` for traceability.                                                             |

### Error Handling

| Condition                                                  | Expected Behavior                                                                                                                                 | Severity |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| V-Build returns ERROR                                      | Orchestrator skips V-Tests/V-Tasks/V-Feature. Enters replan loop directly.                                                                        | Critical |
| 1 of 3 parallel sub-agents returns ERROR                   | Aggregator retries the failing sub-agent once. If persistent, returns ERROR.                                                                      | High     |
| Build environment state not visible to parallel sub-agents | V-Build writes all necessary state to `v-build.md` on disk. Parallel sub-agents read this file rather than relying on terminal/environment state. | High     |
| Replan loop exceeds max 3 iterations                       | Pipeline proceeds with a partial verification report. `verifier.md` documents unresolved issues.                                                  | High     |
| Build passes but no tests exist                            | V-Tests reports "no tests found" as a finding (not an error). V-Tasks and V-Feature proceed independently.                                        | Medium   |

---

## Feature 4: Review Cluster + Knowledge Evolution

### Description

The single sequential reviewer (Step 7) is replaced by 4 parallel sub-agents covering quality, security, testing, and knowledge evolution. Knowledge Evolution is a new capability that captures reusable patterns, suggests updates to Copilot instructions and skills, and maintains the architectural decision log. All Knowledge Evolution outputs are **suggestions only** — they require human approval before application.

### Sub-Agent Responsibilities

| Sub-Agent   | File Name              | Responsibility                                                                                                   |
| ----------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------- |
| R-Quality   | `r-quality.agent.md`   | Code quality, readability, maintainability, naming conventions, architectural alignment                          |
| R-Security  | `r-security.agent.md`  | Security scanning, OWASP compliance, secrets/PII detection, dependency vulnerability check                       |
| R-Testing   | `r-testing.agent.md`   | Test quality, coverage adequacy, test patterns, missing test scenarios                                           |
| R-Knowledge | `r-knowledge.agent.md` | Knowledge evolution — instruction suggestions, skill suggestions, reusable pattern capture, decision log updates |

**R-Quality / R-Security / R-Testing common contract:**

- **Inputs:** `initial-request.md`, `memory.md`, git diff, entire codebase
- **Output:** `docs/feature/<feature-slug>/review/<focus>.md`
- **Completion contract:** `DONE:` / `NEEDS_REVISION:` (actionable findings requiring code changes) / `ERROR:`

**R-Knowledge contract:**

- **Inputs:** `initial-request.md`, `memory.md`, `feature.md`, `design.md`, `plan.md`, `verifier.md`, git diff, entire codebase, existing `decisions.md` (if present), existing `.github/instructions/` (if present)
- **Output:**
  - `docs/feature/<feature-slug>/review/r-knowledge.md` (analysis and rationale)
  - `docs/feature/<feature-slug>/review/knowledge-suggestions.md` (actionable suggestions buffer)
  - `docs/feature/<feature-slug>/decisions.md` (append-only updates)
- **Completion contract:** `DONE:` (with or without suggestions) / `ERROR:`
- **Critical constraint:** R-Knowledge NEVER directly modifies agent definitions in `NewAgentsAndPrompts/`. All modifications are written to `knowledge-suggestions.md` as proposals.

### Aggregation Strategy

| Component                   | Detail                                                                                                                                            |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Aggregator file             | `r-aggregator.agent.md`                                                                                                                           |
| Inputs                      | All 4 sub-agent outputs + `memory.md`                                                                                                             |
| Output                      | `docs/feature/<feature-slug>/review.md`                                                                                                           |
| Conflict handling           | Conflicting findings (e.g., R-Quality flags complexity vs. R-Security requires additional checks) are surfaced as "Tradeoff" items, not resolved. |
| Ownership of `decisions.md` | R-Knowledge owns `decisions.md` writes. The aggregator does not modify it.                                                                        |

**Completion contract aggregation:**

| Scenario                                  | Aggregated Result                                                      |
| ----------------------------------------- | ---------------------------------------------------------------------- |
| All DONE                                  | DONE                                                                   |
| Any NEEDS_REVISION (R-Quality, R-Testing) | NEEDS_REVISION (merge findings, route to implementers)                 |
| R-Security NEEDS_REVISION or ERROR        | ERROR override — security issues are critical path blockers            |
| R-Knowledge DONE with suggestions         | DONE (knowledge suggestions are buffered, not blocking)                |
| R-Knowledge ERROR                         | DONE with warning (knowledge failure is non-blocking; review proceeds) |

### Knowledge Evolution Safeguards

| Safeguard | Requirement                                                                                                                                                                                                        |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| KE-SAFE-1 | R-Knowledge writes suggestions to `knowledge-suggestions.md` ONLY — never directly to agent definitions, instruction files, or skill files.                                                                        |
| KE-SAFE-2 | Each suggestion includes: what to change, why, the specific file and section affected, and a diff-style before/after.                                                                                              |
| KE-SAFE-3 | Suggestions are categorized as: `instruction-update`, `skill-update`, `pattern-capture`, or `workflow-improvement`.                                                                                                |
| KE-SAFE-4 | No suggestion is applied without explicit human review and approval. The pipeline does not auto-apply suggestions.                                                                                                 |
| KE-SAFE-5 | `knowledge-suggestions.md` includes a header warning: "These are proposals only. Do NOT apply without human review."                                                                                               |
| KE-SAFE-6 | R-Knowledge must not suggest removing or weakening any safety constraint, error handling, or verification step from agent definitions. Such suggestions are filtered and logged as "rejected — safety constraint." |
| KE-SAFE-7 | If `decisions.md` does not exist, R-Knowledge creates it with a standard header. If it exists, R-Knowledge appends only — never modifies or deletes existing entries.                                              |

### Acceptance Criteria

| ID      | Criterion                             | Pass/Fail Definition                                                                                                                                                    |
| ------- | ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R-AC-1  | Four R sub-agents run in parallel     | Orchestrator dispatches all 4 R sub-agents concurrently. All 4 complete before the aggregator is invoked.                                                               |
| R-AC-2  | Aggregator produces unified review.md | `review.md` contains quality, security, and testing findings organized by severity, with attribution to originating sub-agents.                                         |
| R-AC-3  | Knowledge suggestions are buffered    | `knowledge-suggestions.md` exists after R-Knowledge completes, containing only proposals (never applied changes).                                                       |
| R-AC-4  | R-Security ERROR blocks pipeline      | If R-Security returns ERROR or NEEDS_REVISION with critical severity findings, the aggregated result is ERROR or NEEDS_REVISION regardless of other sub-agent results.  |
| R-AC-5  | R-Knowledge failure is non-blocking   | If R-Knowledge returns ERROR, the aggregator still produces `review.md` from the other 3 sub-agents and returns DONE (with a warning about missing knowledge analysis). |
| R-AC-6  | No agent definition auto-modification | No file in `NewAgentsAndPrompts/` is modified during pipeline execution by any R sub-agent or the aggregator.                                                           |
| R-AC-7  | decisions.md is append-only           | After R-Knowledge runs, `decisions.md` contains all previously existing entries unmodified, plus zero or more new appended entries.                                     |
| R-AC-8  | NEEDS_REVISION routes to implementers | When the aggregator returns NEEDS_REVISION, the orchestrator routes merged findings back to implementers for fixes (max 1 fix loop preserved).                          |
| R-AC-9  | Review tiered depth model preserved   | The review cluster supports Full, Standard, and Lightweight depth modes, communicated by the orchestrator to each sub-agent at dispatch time.                           |
| R-AC-10 | Sub-agents follow v2 template         | Each sub-agent file contains: YAML frontmatter, role statement, inputs/outputs, 5 operating rules, workflow, output spec, completion contract, anti-drift anchor.       |
| R-AC-11 | Intermediate review files preserved   | All 4 intermediate review files and `knowledge-suggestions.md` persist in `docs/feature/<feature-slug>/review/` for traceability.                                       |

### Error Handling

| Condition                              | Expected Behavior                                                                                                           | Severity |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | -------- |
| R-Security returns ERROR               | Aggregator returns ERROR. Orchestrator retries R-Security once. If persistent, pipeline halts.                              | Critical |
| R-Knowledge returns ERROR              | Aggregator proceeds with 3 available outputs. `review.md` notes missing knowledge analysis. Pipeline continues.             | Medium   |
| 1 of R-Quality/R-Testing returns ERROR | Aggregator retries the failing sub-agent once. If persistent, aggregator proceeds with available results and notes the gap. | High     |
| `decisions.md` missing                 | R-Knowledge creates it with standard header and proceeds.                                                                   | Low      |
| `.github/instructions/` missing        | R-Knowledge notes absence in its analysis. Knowledge suggestions reference the directory as "to be created."                | Low      |

---

## Feature 5: Research Phase Expansion

### Description

Expand the research phase from 3 to 4 parallel research agents by adding a 4th focus area. The additional agent runs concurrently with the existing 3, increasing analytical coverage without impacting wall-clock time (all 4 finish when the slowest finishes).

### Current Research Focus Areas

1. **Architecture** — codebase structure, patterns, flow, bottlenecks → `research/architecture.md`
2. **Impact** — per-component changes, performance, risks, migration → `research/impact.md`
3. **Dependencies** — data flow, ordering, critical path, infrastructure → `research/dependencies.md`

### Fourth Research Focus Area

4. **Patterns & Conventions** — testing strategy, code conventions, developer experience patterns, operational concerns → `research/patterns.md`

The 4th area is defined during design. The specification requires that it must be a distinct, non-overlapping analytical dimension that adds coverage not addressed by the existing 3 areas.

### Acceptance Criteria

| ID       | Criterion                                 | Pass/Fail Definition                                                                                                                                  |
| -------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| RES-AC-1 | Orchestrator dispatches 4 research agents | Step 1.1 dispatches exactly 4 research agents in parallel, each with a unique focus area.                                                             |
| RES-AC-2 | 4th focus area is defined                 | The researcher agent definition includes 4 focus areas in its focus area table, not 3.                                                                |
| RES-AC-3 | 4th researcher produces output            | `research/patterns.md` (or the designated 4th file) exists after research phase completes, following the same format as the other 3 research outputs. |
| RES-AC-4 | Synthesis includes 4th research output    | The researcher in synthesis mode reads all 4 partial research files and produces `analysis.md` incorporating findings from all 4 areas.               |
| RES-AC-5 | No increase in wall-clock time            | The 4th agent runs in parallel with the existing 3; total research time is bounded by the slowest of the 4 agents, not the sum.                       |
| RES-AC-6 | Existing research outputs unchanged       | `architecture.md`, `impact.md`, and `dependencies.md` continue to be produced with their current scope and format.                                    |

### Error Handling

| Condition                    | Expected Behavior                                                                            | Severity |
| ---------------------------- | -------------------------------------------------------------------------------------------- | -------- |
| 4th researcher returns ERROR | Synthesis proceeds with 3 available partials. `analysis.md` notes the missing coverage area. | Medium   |
| 4th research output is empty | Synthesis treats as no findings for that area. Logs a warning.                               | Low      |

---

## Feature 6: Orchestrator Evolution

### Description

The orchestrator (`orchestrator.agent.md`) is updated to coordinate the memory system lifecycle, dispatch and manage 3 cluster types (CT, V, R) with distinct execution patterns, handle the expanded research phase, and integrate knowledge evolution outputs. This is the most extensively impacted component — estimated ~200–250 lines of additions/modifications across 8+ sections, growing from ~323 lines to ~470–530 lines.

### New Coordination Responsibilities

| Responsibility              | Description                                                                                                                                                        |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Memory lifecycle            | Initialize `memory.md` at Step 0; trigger invalidation on revision loops; prune at phase boundaries.                                                               |
| CT cluster dispatch         | Dispatch 4 CT sub-agents in parallel at Step 3b; await all; invoke CT-Aggregator; check completion contract; route NEEDS_REVISION.                                 |
| V cluster dispatch          | Dispatch V-Build at Step 6; await; if DONE, dispatch V-Tests/V-Tasks/V-Feature in parallel; await all; invoke V-Aggregator; manage replan loop (max 3 iterations). |
| R cluster dispatch          | Dispatch 4 R sub-agents in parallel at Step 7; await all; invoke R-Aggregator; check completion contract; handle knowledge evolution outputs.                      |
| Research dispatch           | Dispatch 4 (not 3) research agents in parallel at Step 1.1.                                                                                                        |
| Aggregator management       | Invoke aggregators after cluster sub-agents complete; pass correct inputs; handle aggregator failures.                                                             |
| Knowledge evolution routing | After R cluster completes, ensure `knowledge-suggestions.md` is preserved but not auto-applied.                                                                    |

### Cluster Management Protocol

The orchestrator must support 3 distinct cluster execution patterns:

**Pattern A — Fully Parallel (CT, R clusters):**

```
Dispatch N sub-agents in parallel → Await all → Invoke aggregator → Check contract
```

**Pattern B — Sequential Gate + Parallel (V cluster):**

```
Dispatch gate agent → Await →
  If ERROR: handle failure (replan or halt)
  If DONE: Dispatch N-1 sub-agents in parallel → Await all → Invoke aggregator → Check contract
```

**Pattern C — Replan Loop (V cluster specific):**

```
Pattern B → If NEEDS_REVISION:
  Invoke planner (replan mode) → Invoke implementers → Re-run Pattern B
  Max 3 total iterations
```

### Memory Integration

| Orchestrator Step             | Memory Action                                    |
| ----------------------------- | ------------------------------------------------ |
| Step 0 (Setup)                | Create `memory.md` with empty templates          |
| After each step               | Verify agent wrote to memory; log warning if not |
| Before revision loop          | Invalidate stale memory entries                  |
| After revision loop completes | Verify revised entries replace invalidated ones  |
| Phase boundaries              | Prune old entries (>2 phases)                    |
| End of pipeline               | Final memory state persists with `review.md`     |

### Acceptance Criteria

| ID         | Criterion                                      | Pass/Fail Definition                                                                                                                         |
| ---------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| ORCH-AC-1  | Memory initialized at Step 0                   | `memory.md` exists with correct template structure after Step 0 completes.                                                                   |
| ORCH-AC-2  | 4 research agents dispatched                   | Step 1.1 dispatches exactly 4 research agents (not 3).                                                                                       |
| ORCH-AC-3  | CT cluster dispatched correctly                | Step 3b dispatches 4 CT sub-agents in parallel, awaits all, then invokes CT-Aggregator.                                                      |
| ORCH-AC-4  | V cluster dispatched with sequential gate      | Step 6 dispatches V-Build first; only dispatches V-Tests/V-Tasks/V-Feature if V-Build returns DONE.                                          |
| ORCH-AC-5  | V cluster replan loop works                    | On V-Aggregator NEEDS_REVISION, orchestrator invokes planner → implementers → full V cluster re-run, up to 3 iterations.                     |
| ORCH-AC-6  | R cluster dispatched correctly                 | Step 7 dispatches 4 R sub-agents in parallel, awaits all, then invokes R-Aggregator.                                                         |
| ORCH-AC-7  | Knowledge suggestions preserved                | After Step 7, `knowledge-suggestions.md` exists and is not auto-applied.                                                                     |
| ORCH-AC-8  | Memory invalidation on revision loops          | Before re-dispatching a revision agent, orchestrator marks affected memory entries as invalidated.                                           |
| ORCH-AC-9  | Memory pruning at phase boundaries             | Old memory entries are removed per the pruning strategy after each completed step.                                                           |
| ORCH-AC-10 | Concurrency cap respected                      | No cluster dispatch exceeds 4 concurrent sub-agents (consistent with existing Global Rule 7).                                                |
| ORCH-AC-11 | NEEDS_REVISION routing updated                 | The NEEDS_REVISION routing table includes rows for all aggregators (CT-Aggregator, V-Aggregator, R-Aggregator) with correct routing targets. |
| ORCH-AC-12 | Expectations table updated                     | The expectations table includes rows for all 15 new sub-agents and 3 aggregators with correct inputs/outputs and completion contracts.       |
| ORCH-AC-13 | Orchestrator remains a single agent definition | The orchestrator is not decomposed into sub-orchestrators. Cluster coordination logic is embedded in the orchestrator's workflow steps.      |
| ORCH-AC-14 | Orchestrator stays under complexity threshold  | The final orchestrator file size does not exceed 550 lines.                                                                                  |

### Error Handling

| Condition                                   | Expected Behavior                                                                                                           | Severity |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | -------- |
| Aggregator returns ERROR                    | Orchestrator retries the aggregator once with the same inputs. If persistent, pipeline halts with ERROR.                    | Critical |
| Partial cluster failure (1 sub-agent ERROR) | Orchestrator retries the failing sub-agent once. If persistent, aggregator is invoked with available outputs.               | High     |
| Memory initialization fails                 | Orchestrator proceeds without memory. Logs warning. All agents fall back to direct artifact reads.                          | High     |
| Orchestrator exceeds 550 lines              | Design must consider extracting reusable patterns or section compression. This is a design constraint, not a runtime error. | Medium   |

---

## Feature 7: Universal Agent Memory Integration

### Description

Every agent in the system (existing 10+1 and all 15 new sub-agents + 3 aggregators) must integrate with the operational memory system by reading `memory.md` as a first input. Only aggregators and sequential agents write memory updates after producing output. Parallel sub-agents read memory but do not write to it — they communicate findings through their intermediate output files.

### Per-Agent Memory Contract

Each agent definition must be updated to include:

1. **Input addition:** `memory.md` listed as the first input in the Inputs section.
2. **Workflow Step 1:** "Read `memory.md` to load artifact index, recent decisions, lessons learned, and recent updates."
3. **Workflow penultimate step (aggregators and sequential agents only, before completion contract):** "Update `memory.md`: append to Artifact Index (if new artifact produced), Recent Decisions (if decisions made), Lessons Learned (if issues encountered), Recent Updates (summary of output produced)." Parallel sub-agents skip this step.
4. **Operating rule addition:** "Read `memory.md` FIRST before accessing any artifact. Use the Artifact Index to navigate directly to relevant sections rather than reading full artifacts."

### Agents Requiring Updates

| Agent                   | File                            | Memory Write Sections                                                  |
| ----------------------- | ------------------------------- | ---------------------------------------------------------------------- |
| Researcher (focused)    | `researcher.agent.md`           | — (does not write to memory; findings go to `research/<focus>.md`)     |
| Researcher (synthesis)  | `researcher.agent.md`           | Artifact Index, Recent Updates                                         |
| Spec Agent              | `spec.agent.md`                 | Artifact Index, Recent Decisions, Recent Updates                       |
| Designer                | `designer.agent.md`             | Artifact Index, Recent Decisions, Recent Updates                       |
| Planner                 | `planner.agent.md`              | Artifact Index, Recent Decisions, Recent Updates                       |
| Implementer             | `implementer.agent.md`          | — (does not write to memory; findings go to task output)               |
| Documentation Writer    | `documentation-writer.agent.md` | Artifact Index, Recent Updates                                         |
| Feature Workflow Prompt | `feature-workflow.prompt.md`    | N/A (reference only — add memory awareness to prompt context)          |
| CT sub-agents (×4)      | `ct-*.agent.md`                 | — (do not write to memory; findings go to `ct-review/ct-<focus>.md`)   |
| CT Aggregator           | `ct-aggregator.agent.md`        | Recent Decisions, Recent Updates                                       |
| V sub-agents (×4)       | `v-*.agent.md`                  | — (do not write to memory; findings go to `verification/v-<focus>.md`) |
| V Aggregator            | `v-aggregator.agent.md`         | Recent Updates, Lessons Learned                                        |
| R sub-agents (×4)       | `r-*.agent.md`                  | — (do not write to memory; findings go to `review/r-<focus>.md`)       |
| R Aggregator            | `r-aggregator.agent.md`         | Recent Decisions, Recent Updates, Lessons Learned                      |

### Acceptance Criteria

| ID           | Criterion                                                   | Pass/Fail Definition                                                                                                                                                                                                              |
| ------------ | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MEM-INT-AC-1 | All agents list memory.md as input                          | Every `.agent.md` file in `NewAgentsAndPrompts/` (including new sub-agents and aggregators) lists `memory.md` in its Inputs section.                                                                                              |
| MEM-INT-AC-2 | All agents read memory first                                | Every agent's workflow begins with reading `memory.md` (Step 1 or equivalent).                                                                                                                                                    |
| MEM-INT-AC-3 | Aggregators and sequential agents write memory after output | Only aggregators and sequential agents include a memory write step after producing output. Parallel sub-agents do NOT write to `memory.md`.                                                                                       |
| MEM-INT-AC-4 | Memory operating rule present                               | Every agent includes an operating rule about reading memory first and using the Artifact Index for navigation.                                                                                                                    |
| MEM-INT-AC-5 | Parallel agents do not write to memory                      | Agents running in parallel (research, implementation, cluster sub-agents) do NOT write to `memory.md`. They communicate findings exclusively through their intermediate output files. This eliminates concurrent write conflicts. |
| MEM-INT-AC-6 | Aggregators consolidate to root sections                    | Aggregators read their cluster's intermediate output files and write consolidated summaries to root-level memory sections (sequential — safe).                                                                                    |
| MEM-INT-AC-7 | Feature workflow prompt updated                             | `feature-workflow.prompt.md` references the memory system and instructs users that `memory.md` will be created during pipeline execution.                                                                                         |

---

## Cross-Cutting Concerns

### Consistency

| Concern                    | Requirement                                                                                                                                                                                                                                 |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agent template consistency | All 18 new files (15 sub-agents + 3 aggregators) follow the v2 structural template: YAML frontmatter, role statement, inputs/outputs, 5 operating rules, workflow, output spec, completion contract, anti-drift anchor.                     |
| Naming convention          | Sub-agent files use prefix-based naming: `ct-security.agent.md`, `v-build.agent.md`, `r-knowledge.agent.md`. Aggregators: `ct-aggregator.agent.md`, `v-aggregator.agent.md`, `r-aggregator.agent.md`. All placed in `NewAgentsAndPrompts/`. |
| Aggregator pattern         | All 3 aggregators follow the same structural pattern: consume N sub-agent outputs, merge/deduplicate, surface conflicts as unresolved tensions, produce a single output artifact with a single completion contract.                         |
| Completion contract        | All agents (existing and new) use the three-state contract: `DONE:` / `NEEDS_REVISION:` / `ERROR:`. Sub-agents that can only succeed or fail use `DONE:` / `ERROR:` (no `NEEDS_REVISION` — only aggregators decide revision need).          |
| Artifact ownership         | One creator per artifact. Sub-agents own their intermediate files. Aggregators own the final merged artifact. No agent modifies an artifact it does not own.                                                                                |

### Performance

| Concern              | Requirement                                                                                                                                                                                                                                |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Concurrency cap      | Maximum 4 concurrent sub-agents per cluster dispatch (unchanged Global Rule 7).                                                                                                                                                            |
| Aggregation overhead | Aggregation adds 40–60% overhead relative to the longest sub-agent execution time, accounting for semantic deduplication and cross-cutting synthesis. The aggregator should not re-analyze source material — only merge sub-agent outputs. |
| Memory read overhead | The memory read step must not add material time per agent. Memory file should stay under 200 lines.                                                                                                                                        |
| Total invocations    | Pipeline invocation count grows from ~15 to ~35+ per run. This is acceptable given the parallelization gains.                                                                                                                              |
| End-to-end target    | ~10–18% reduction in total pipeline wall-clock time vs. current architecture (accounting for ~20 additional agent invocations), measured on a representative feature request.                                                              |

### Backward Compatibility

| Concern                 | Requirement                                                                                                                                                                                           |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Artifact format         | All existing artifact formats (`feature.md`, `design.md`, `plan.md`, `verifier.md`, `review.md`, etc.) remain structurally unchanged. New content may be added but existing sections are not removed. |
| Pipeline order          | The 8-step pipeline order is preserved. No steps are skipped, reordered, or added. Clusters are internal to existing steps.                                                                           |
| Two-layer verification  | Unit-level TDD (implementer) and integration-level verification (verifier cluster) remain distinct layers.                                                                                            |
| Revision loop limits    | Max 1 design revision loop, max 3 verification replan loops, max 1 review fix loop — all preserved.                                                                                                   |
| Existing agent behavior | Agents not being split (Spec, Designer, Planner, Implementer, Documentation Writer) retain their current behavior with only memory integration changes added.                                         |

---

## User Stories / Flows

### Story 1: Standard Pipeline Run (Happy Path)

1. User invokes the feature workflow prompt with a feature request.
2. Orchestrator creates `initial-request.md` and initializes `memory.md` (Step 0).
3. Orchestrator dispatches 4 research agents in parallel (Step 1.1). Each reads `memory.md` first, then researches, writes `research/<focus>.md`, and updates memory.
4. Orchestrator dispatches researcher in synthesis mode (Step 1.2). Synthesis reads memory, then all 4 partials, produces `analysis.md`, updates memory.
5. Spec agent reads memory, then analysis, produces `feature.md`, updates memory (Step 2).
6. Designer reads memory, then feature spec, produces `design.md`, updates memory (Step 3).
7. Orchestrator dispatches 4 CT sub-agents in parallel (Step 3b). CT-Aggregator produces `design_critical_review.md`. All DONE.
8. Planner reads memory, produces `plan.md` + `tasks/*.md`, updates memory (Step 4).
9. Implementation waves execute (Step 5). Implementers read memory, produce code + tests, update memory.
10. Orchestrator dispatches V-Build → DONE → dispatches V-Tests/V-Tasks/V-Feature in parallel → V-Aggregator produces `verifier.md` → DONE (Step 6).
11. Orchestrator dispatches 4 R sub-agents in parallel → R-Aggregator produces `review.md` + `knowledge-suggestions.md` → DONE (Step 7).
12. Pipeline completes. Memory, all intermediate files, and knowledge suggestions persist.

### Story 2: Verification Replan Loop

1. V-Build passes. V-Tests finds 3 failing tests. V-Tasks finds 1 unmet criterion.
2. V-Aggregator returns NEEDS_REVISION with merged findings.
3. Orchestrator enters replan loop: invokes planner (replan mode) → implementers fix issues → full V cluster re-run.
4. Second iteration: all pass. V-Aggregator returns DONE.

### Story 3: Critical Thinking Finds Design Flaw

1. CT-Security flags a critical authentication bypass. CT-Strategy flags scope creep.
2. CT-Aggregator returns NEEDS_REVISION (critical severity finding).
3. Designer revises `design.md`. Orchestrator invalidates stale CT memory entries.
4. Full CT cluster re-runs. All DONE on second pass.

### Story 4: Knowledge Evolution Suggestions

1. R-Knowledge identifies a reusable error-handling pattern used across 3 implemented tasks.
2. R-Knowledge writes a suggestion to `knowledge-suggestions.md`: "Add error-handling pattern to `.github/instructions/error-handling.instructions.md`."
3. Pipeline completes. Human reviews `knowledge-suggestions.md` post-pipeline and decides whether to apply.

---

## Test Scenarios

| ID    | Scenario                                                                        | Verifies                                   | Expected Result                                                                                                                 |
| ----- | ------------------------------------------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| TS-1  | Run full pipeline on a simple feature request                                   | MEM-AC-1 through MEM-AC-8, all cluster ACs | Pipeline completes; `memory.md` exists with entries from all phases; all artifacts produced.                                    |
| TS-2  | Run pipeline where CT-Aggregator returns NEEDS_REVISION                         | CT-AC-6, CT-AC-7, ORCH-AC-8                | Designer revises; full CT cluster re-runs; max 1 loop; stale memory invalidated.                                                |
| TS-3  | Run pipeline where V-Build fails                                                | V-AC-1, V-AC-3                             | V-Tests/V-Tasks/V-Feature never dispatched; replan loop entered immediately.                                                    |
| TS-4  | Run pipeline where V-Aggregator returns NEEDS_REVISION 3 times                  | V-AC-5, V-AC-6                             | 3 replan iterations occur; 4th time pipeline proceeds with documented issues.                                                   |
| TS-5  | Run pipeline where R-Knowledge returns ERROR                                    | R-AC-5                                     | `review.md` produced from other 3 sub-agents; pipeline continues; warning logged.                                               |
| TS-6  | Run pipeline where R-Security returns ERROR                                     | R-AC-4                                     | Pipeline halts with ERROR after aggregator retry.                                                                               |
| TS-7  | Verify `memory.md` does not exceed 200 lines during a multi-wave implementation | MEM-AC-8                                   | Pruning triggers; memory stays bounded.                                                                                         |
| TS-8  | Verify knowledge-suggestions.md is not auto-applied                             | R-AC-6, KE-SAFE-1 through KE-SAFE-4        | No file in `NewAgentsAndPrompts/` is modified during pipeline execution.                                                        |
| TS-9  | Verify all 28 agent files contain memory integration                            | MEM-INT-AC-1 through MEM-INT-AC-4          | Every `.agent.md` lists `memory.md` as input; workflow starts with memory read; ends with memory write.                         |
| TS-10 | Run pipeline with 4 researchers; verify synthesis reads all 4                   | RES-AC-1 through RES-AC-4                  | `analysis.md` references findings from all 4 research areas.                                                                    |
| TS-11 | Verify intermediate cluster files persist after aggregation                     | CT-AC-9, V-AC-10, R-AC-11                  | All sub-agent output files exist alongside aggregated outputs.                                                                  |
| TS-12 | Verify V-Build output is file-based, not terminal-state-based                   | V-AC-9                                     | `v-build.md` contains build system details and artifact paths. Parallel sub-agents read this file.                              |
| TS-13 | Verify CT sub-agents include Cross-Cutting Observations                         | CT-AC-5                                    | Each CT sub-agent output has a "Cross-Cutting Observations" section. Aggregated output has a synthesized cross-cutting section. |
| TS-14 | Verify orchestrator NEEDS_REVISION table has aggregator rows                    | ORCH-AC-11                                 | NEEDS_REVISION routing table in orchestrator contains CT-Aggregator, V-Aggregator, R-Aggregator entries.                        |
| TS-15 | Run pipeline; verify no memory entry exceeds 2-sentence limit                   | MEM-AC-7                                   | Every entry in Recent Decisions, Recent Updates, and Lessons Learned is ≤2 sentences. No code snippets or full artifact text.   |

---

## Dependencies & Risks

### Internal Dependencies

```
Memory System (F1) ──[hard]──→ Universal Agent Memory Integration (F7)
Memory System (F1) ──[soft]──→ CT Cluster (F2), V Cluster (F3), R Cluster (F4)

Aggregator Pattern (design artifact) ──[hard]──→ CT Cluster (F2)
                                     ──[hard]──→ V Cluster (F3)
                                     ──[hard]──→ R Cluster (F4)

Research Expansion (F5) ── fully independent ── no dependencies

All features (F1–F5, F7) ──[serialize on]──→ Orchestrator Evolution (F6)
```

### Implementation Ordering Constraint

The orchestrator is the single serialization bottleneck. All features funnel through it. Recommended sequencing for orchestrator changes: F5 → F2 → F3 → F4 → F1/F7.

### Risk Register

| Risk                                           | Likelihood  | Impact   | Mitigation                                                                                                                                           | Owner                |
| ---------------------------------------------- | ----------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| Knowledge Evolution self-modification          | Medium      | Critical | Suggestion-only writes; human approval gate; safety constraint filter (KE-SAFE-6)                                                                    | R-Knowledge design   |
| Stale memory / memory-artifact drift           | High        | High     | Invalidation protocol on revision loops (MEM-FR-7, MEM-FR-8); agents verify critical decisions against artifacts                                     | Memory system design |
| Orchestrator complexity explosion (>550 lines) | Medium      | High     | Strict 550-line cap (ORCH-AC-14); extract reusable cluster patterns; keep orchestrator focused on high-level flow                                    | Orchestrator design  |
| Memory write conflicts (parallel agents)       | Low         | Low      | Parallel agents do NOT write to `memory.md` (R1 fix). Only aggregators and sequential agents write, eliminating concurrent write conflicts entirely. | Memory system design |
| Aggregation quality degradation                | Medium      | High     | Conflicts surfaced not resolved; critical findings included verbatim; cross-cutting synthesis section                                                | Aggregator design    |
| Quality loss from agent splitting              | Medium-High | Medium   | Cross-Cutting Observations section in every sub-agent; aggregator cross-referencing                                                                  | Sub-agent design     |
| Migration breaks working pipeline              | Medium      | High     | Phased implementation; per-phase validation via full pipeline run                                                                                    | Implementation plan  |
| V-Cluster build state sharing                  | Medium      | High     | V-Build writes state to `v-build.md` on disk; no reliance on terminal state (V-AC-9)                                                                 | V-Build design       |
| No automated testing                           | Always      | Medium   | Full pipeline validation per migration phase (TS-1)                                                                                                  | Process              |
| Memory size growth                             | Medium      | Medium   | 200-line cap (MEM-AC-8); pruning at phase boundaries (MEM-FR-10); emergency pruning                                                                  | Memory system design |

---

## Glossary

| Term                           | Definition                                                                                                                                                                       |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Agent**                      | A markdown file (`.agent.md`) interpreted by the Copilot agent runtime that defines a role, inputs, outputs, workflow, and completion contract.                                  |
| **Aggregator**                 | A specialized agent that consumes N sub-agent outputs and produces a single merged artifact with a unified completion contract.                                                  |
| **Artifact**                   | A markdown file produced by an agent that serves as the authoritative source of truth for its domain (e.g., `design.md`, `verifier.md`).                                         |
| **Cluster**                    | A group of sub-agents that run in parallel (or partially parallel) to replace a single sequential agent, followed by an aggregator.                                              |
| **Completion Contract**        | The final line returned by an agent: `DONE:`, `NEEDS_REVISION:`, or `ERROR:` with a summary.                                                                                     |
| **Cross-Cutting Observations** | A section in each sub-agent's output that flags issues spanning beyond the sub-agent's primary focus area.                                                                       |
| **Knowledge Evolution**        | The capability of R-Knowledge to suggest improvements to Copilot instructions, skills, and workflow patterns based on pipeline learnings.                                        |
| **Memory**                     | The `memory.md` file providing navigational metadata, recent decisions, lessons learned, and recent updates — NOT a replacement for artifacts.                                   |
| **Operational Memory**         | Synonym for Memory in this specification.                                                                                                                                        |
| **Pipeline**                   | The 8-step deterministic orchestration flow from Setup through Review.                                                                                                           |
| **Replan Loop**                | The verification failure recovery mechanism: verifier finds issues → planner re-plans → implementers fix → re-verify (max 3 iterations).                                         |
| **Revision Loop**              | A broader term for any feedback loop (design revision max 1, verification replan max 3, review fix max 1).                                                                       |
| **Sub-Agent**                  | An agent that is part of a cluster, covering a focused subset of the original agent's responsibilities.                                                                          |
| **Two-Layer Verification**     | The design invariant where implementers perform unit-level TDD and the verifier cluster performs integration-level verification.                                                 |
| **Unresolved Tension**         | A documented conflict between two sub-agents' findings that the aggregator surfaces for human/downstream resolution rather than resolving itself.                                |
| **v2 Template**                | The standardized agent definition structure: YAML frontmatter, role statement, inputs/outputs, 5 operating rules, workflow, output spec, completion contract, anti-drift anchor. |
