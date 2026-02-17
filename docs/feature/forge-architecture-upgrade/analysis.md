# Forge Architecture Upgrade — Synthesized Analysis

## Executive Summary

Forge is a deterministic 8-step multi-agent orchestration pipeline (10 agents + 1 prompt) that communicates exclusively through markdown artifacts on disk. The proposed upgrade introduces an operational memory system (cross-cutting all 29 future agents), parallelizes three sequential bottlenecks into clusters (Critical Thinking ×4, Verification ×4, Review ×4 with Knowledge Evolution), and expands research from 3→4 agents. The architecture is purely declarative (no runtime code — only markdown interpreted by the Copilot agent runtime), meaning all changes are markdown edits. The orchestrator (`orchestrator.agent.md`, 323 lines, estimated ~500+ after upgrade) is the single serialization bottleneck — every proposed change funnels through it. The theoretical end-to-end speedup is ~15–20%, bounded by the unchanged sequential middle pipeline (Spec → Design → Planning at ~23% of pipeline time).

## Current Architecture Overview

### System Composition

Forge comprises **10 agent definitions** (`.agent.md`) and **1 prompt file** (`.prompt.md`) in `NewAgentsAndPrompts/`. All agents follow a consistent structural template established in v2: YAML frontmatter, role statement, explicit inputs/outputs, 5 standardized operating rules, step-by-step workflow, output file contents specification, three-state completion contract (`DONE:` / `NEEDS_REVISION:` / `ERROR:`), and an anti-drift anchor.

### Pipeline Flow

The orchestrator enforces a fixed 8-step deterministic pipeline that is never skipped or reordered:

```
Step 0: Setup → initial-request.md
Step 1: Research (PARALLEL ×3) → research/*.md → Synthesis → analysis.md
Step 2: Specification (SEQUENTIAL) → feature.md
Step 3: Design (SEQUENTIAL) → design.md
Step 3b: Critical Review (SEQUENTIAL, max 1 revision loop) → design_critical_review.md
Step 4: Planning (SEQUENTIAL) → plan.md + tasks/*.md
Step 5: Implementation (PARALLEL WAVES, max 4 concurrent per sub-wave)
Step 6: Verification (SEQUENTIAL, max 3 replan loops) → verifier.md
Step 7: Review (SEQUENTIAL, max 1 fix loop) → review.md
```

### Communication Model

All inter-agent communication is file-based. There is no shared memory, no message queue, no database, and no cross-agent runtime state. Each agent reads input artifacts from disk, writes output artifacts to disk, and returns a completion contract line. The orchestrator mediates all routing. Agents never communicate directly with each other.

### Current Parallelism

Only two points have parallelism: research (×3 concurrent) and implementation waves (max 4 concurrent per sub-wave). Six pipeline stages run as sequential single-agent bottlenecks. The middle pipeline (Spec → Design → Critical Review → Planning) is a chain of 4 sequential stages with no parallelism.

### Key Architectural Properties

- **Markdown as universal format** — no YAML, JSON, or binary formats (YAML state files explicitly rejected per `optimization-from-gem-team.md`)
- **Single source of truth** — each artifact is authoritative for its domain; no duplication
- **One creator per artifact** — each artifact has exactly one writing agent
- **Read-only consumers** — consumers never modify artifacts they don't own
- **Strong agent isolation** — explicit constraints on what files each agent can read/write (the implementer is explicitly forbidden from reading `plan.md`)
- **Two-layer verification** — unit-level TDD (implementer) separate from integration-level verification (verifier), a conscious design decision documented in README and comparison docs
- **No `.github` directory** currently exists despite reviewer references to `.github/instructions/*.instructions.md`

---

## Proposed Changes Analysis

### 1. Memory System

#### What It Is

A lightweight operational memory layer that stores artifact navigation metadata (index and section mappings), recent decisions, lessons learned, and recent artifact updates — while keeping markdown artifacts as the single source of truth. Every agent reads memory FIRST before accessing artifacts to minimize redundant full-file reads.

#### Architecture Findings

Currently there is **no shared context system** between agent invocations. Each agent reads full artifacts from disk on every invocation with no indexing, caching, or partial-read optimization. The only cross-feature knowledge mechanism is the optional `decisions.md` file (reviewer writes, researcher reads if it exists). The initial request's constraint — artifacts remain the single source of truth — is critical: memory is a navigation/summary layer, not a replacement.

The memory system must be file-based (markdown), consistent with Forge's artifact-only communication model and the explicit rejection of YAML state files. It requires: a file schema/template, initialization protocol (orchestrator Step 0), read contract (every agent reads first), write contract (every agent writes summary after output), invalidation protocol (on revision/replan loops), concurrency model, and pruning strategy.

#### Impact Findings

Memory integration is **cross-cutting** — it affects all 11 existing files at varying levels:

| Impact Level | Agents                                                                                                                   |
| ------------ | ------------------------------------------------------------------------------------------------------------------------ |
| **Critical** | Orchestrator (must init, update after every step, manage lifecycle — adds ~50–80 lines)                                  |
| **High**     | Researcher (focused + synthesis), Planner (replan mode complexity), Reviewer (most complex writes + knowledge evolution) |
| **Medium**   | Spec, Designer, Critical Thinker, Implementer, Verifier                                                                  |
| **Low**      | Documentation Writer, Feature Workflow Prompt                                                                            |

#### Key Risks

1. **Stale Memory (High):** Revision loops (design loop, replan loop ×3, review fix loop) change artifacts but memory entries written before the revision become stale. Downstream agents reading memory first may proceed with outdated assumptions. Currently there are no safeguards — artifacts are always re-read from disk.
2. **Memory-Artifact Drift (High):** If memory says "design uses pattern X" but the designer revised to "pattern Y," agents reading memory first get wrong context.
3. **Write Conflict (Medium):** Parallel agents (research ×4, implementation waves, all clusters) writing concurrently may cause lost writes or interleaved corruption. Options: per-group sections, deferred writes via aggregator, or write-locking.
4. **Size Growth (Medium):** Without pruning, memory consumes context window space, defeating the purpose of reducing artifact reads.
5. **Single Point of Failure (Medium):** Corrupted memory affects all downstream agents since they read it first.

#### Dependency Findings

Memory (C1) is the **foundational dependency** — all other changes either require it (C6: integration across agents) or benefit from it (C2–C4: clusters). The memory schema must exist before any agent can read/write it. However, the dependency from clusters to memory is **soft** — clusters can be designed and function without memory, though they benefit from it. The 29 total agents (11 existing + 15 new sub-agents + 3 aggregators) will all use memory.

Memory concurrency is an **unsolved design question** that gates the critical path: the chosen model (per-group sections, deferred writes, or write-locking) affects the schema design.

---

### 2. Critical Thinking Cluster (4 Parallel Sub-Agents)

#### What It Is

Split the single sequential critical thinker into 4 parallel sub-agents with a dedicated aggregator, covering the current 10 risk categories:

| Sub-Agent          | Focus Areas                                                             |
| ------------------ | ----------------------------------------------------------------------- |
| CT-Security        | Security vulnerabilities, backwards compatibility                       |
| CT-Scalability     | Scalability bottlenecks, performance implications                       |
| CT-Maintainability | Maintainability concerns, complexity risks, integration risks           |
| CT-Strategy        | Strategic risks, scope risks, edge cases, "is this the right approach?" |

#### Architecture Findings

The current critical thinker (139 lines) reviews the entire design across all risk categories in a single sequential invocation at Step 3b. It has a feedback loop: `NEEDS_REVISION` routes back to the designer (max 1 loop). This is a fully parallel cluster — all 4 sub-agents can run concurrently.

#### Impact Findings

**New files required:** 5 (4 sub-agents + 1 aggregator, or 4 sub-agents with the original repurposed as aggregator).

**Changed flow:**

```
Current:  Designer → Critical Thinker → design_critical_review.md
Proposed: Designer → [CT-Security, CT-Scale, CT-Maintain, CT-Strategy] (parallel ×4) → CT-Aggregator → design_critical_review.md
```

**Key risks:**

- **Conflicting findings (High):** Sub-agents may reach contradictory conclusions (e.g., CT-Security: "add validation everywhere" vs. CT-Scalability: "avoid per-request validation overhead"). The aggregator must surface conflicts rather than resolve them.
- **Coverage gaps (Medium):** Splitting risk categories may create blind spots at boundaries. Needs overlapping scope with deduplication.
- **NEEDS_REVISION loop complexity (Medium):** Designer must address findings from 4 different sub-agents simultaneously. The max-1 retry loop may be insufficient.
- **Aggregation bottleneck (Medium):** If the aggregator is complex, it partially offsets parallelization gains.

**Orchestrator impact:** Step 3b (lines 145–156) must be rewritten to dispatch 4 sub-agents in parallel, await all, invoke aggregator, then check the completion contract. NEEDS_REVISION routing and expectations tables must be updated.

#### Dependency Findings

CT cluster depends on the **aggregator pattern template** (hard dependency — cluster requires aggregation). It has a soft dependency on the memory system (benefits from but doesn't require it). It is **fully independent** of the V and R clusters. However, its orchestrator integration must be sequenced with other cluster integrations since all modify the same file.

**Performance projection:** Theoretical speedup ~3–3.5× for this stage; practical speedup ~2–2.5× (sub-agents may not divide evenly; aggregation adds ~15–25% overhead).

---

### 3. Verification Cluster (4 Parallel Sub-Agents)

#### What It Is

Replace the single sequential verifier with a cluster of 4 sub-agents and an aggregator:

| Sub-Agent | Focus                                          | Parallelizable?          |
| --------- | ---------------------------------------------- | ------------------------ |
| V-Build   | Build system detection + compilation           | **No** — must run first  |
| V-Tests   | Full test suite execution + analysis           | Yes — after build passes |
| V-Tasks   | Per-task acceptance criteria verification      | Yes — after build passes |
| V-Feature | Feature-level acceptance criteria verification | Yes — after build passes |

#### Architecture Findings

The current verifier (179 lines) performs 6 sequential workflow steps at Step 6: detect build system, build, run tests, per-task verification, feature-level verification, report generation. It has the most complex feedback loop: `NEEDS_REVISION` triggers replan → re-implement → re-verify (max 3 iterations). The two-layer verification model (unit-level TDD in implementer + integration-level in verifier) is a core design decision that must be preserved.

#### Impact Findings

**Critical constraint:** V-Build must complete first. The cluster is **not fully parallel** — it's sequential build + 3 parallel verification steps. In build-failure scenarios, the cluster degenerates to a single sequential agent (zero speedup).

```
V-Build → [V-Tests, V-Tasks, V-Feature] (parallel ×3) → V-Aggregator → verifier.md
```

**New files required:** 5 (4 sub-agents + 1 aggregator).

**Key risks:**

- **Sequential build dependency (High):** Limits speedup to cases where build passes. In replan loops (where builds may initially fail), zero gain until build is fixed.
- **Replan loop coordination (High):** The max-3 replan loop now involves a mini-pipeline per iteration (build → parallel verify → aggregate → decide). The orchestrator must coordinate this multi-step process within each iteration.
- **Build environment state sharing (Medium):** If V-Build modifies environment state, the 3 parallel sub-agents must all see the same post-build state. Platform-dependent `runSubagent` terminal isolation may prevent this.
- **Targeted re-verification challenge (Medium):** Previous failures may span multiple sub-agents. Full cluster re-runs are simpler but waste time when only one sub-agent's concerns were addressed.

**Orchestrator impact:** Step 6 (lines 192–213) requires major rewrite — dispatch build → await → dispatch parallel verify ×3 → await → aggregate → check contract. Replan loop logic (max 3 iterations) must account for the cluster's internal failure modes.

#### Dependency Findings

V cluster depends on the **aggregator pattern template** (hard dependency). The internal V-Build → V-Tests/V-Tasks/V-Feature dependency is a **runtime ordering constraint** (affects execution, not development). Soft dependency on memory. Independent of CT and R clusters but serializes on orchestrator integration.

**Performance projection:** Theoretical speedup ~2–2.5× for this stage; practical speedup ~1.5–2× (build time dominates for large projects). Worst case (build fails): 1× (no speedup).

---

### 4. Review Cluster + Knowledge Evolution (4 Parallel Sub-Agents)

#### What It Is

Split the reviewer into 4 parallel sub-agents with Knowledge Evolution as a new capability:

| Sub-Agent   | Focus                                                                                    |
| ----------- | ---------------------------------------------------------------------------------------- |
| R-Quality   | Code quality, readability, maintainability, naming, conventions, architectural alignment |
| R-Security  | Security scanning, OWASP, secrets/PII detection                                          |
| R-Testing   | Test quality, coverage, test adequacy                                                    |
| R-Knowledge | Knowledge evolution — instructions, skills, patterns, decisions                          |

#### Architecture Findings

The current reviewer (220 lines) handles 7 distinct responsibilities at Step 7: peer review, security review, test quality, architectural alignment, convention compliance, `decisions.md` updates, and `.github/instructions` updates. It uses a tiered depth model (Full/Standard/Lightweight). The Review cluster is **fully parallel** — all 4 sub-agents can run concurrently.

#### Impact Findings

**New files required:** 5 (4 sub-agents + 1 aggregator).

**Knowledge Evolution is new functionality** — it goes beyond the current reviewer's `decisions.md`/instruction updates to include:

1. Suggesting improvements/additions/removals to Copilot instructions
2. Suggesting improvements/additions/removals to skills
3. Capturing reusable workflow patterns
4. Writing safe, reviewable updates

**Key risks:**

- **Knowledge Evolution self-modification risk (Critical):** R-Knowledge can suggest modifications to the agent definitions that govern Forge itself. A self-modifying system has inherent instability risks. Updates must be suggestions only, stored in a review buffer, requiring human approval before application. Versioning and rollback mechanisms are essential.
- **Conflicting review findings (Medium):** R-Quality may flag a pattern as "overly complex" while the design chose it for performance. R-Security may demand rate limiting while R-Quality calls it unnecessary complexity. The aggregator must surface these as explicit tradeoffs.
- **`decisions.md` ownership ambiguity (Medium):** Currently the reviewer owns `decisions.md`. With the split, either R-Knowledge owns it (breaking current association), R-Quality identifies decisions and R-Knowledge writes them (introducing intra-cluster dependency), or the aggregator handles it.

**Completion contract aggregation:**

| Scenario                          | Result                                                 |
| --------------------------------- | ------------------------------------------------------ |
| All DONE                          | DONE                                                   |
| Any NEEDS_REVISION                | NEEDS_REVISION (merge findings, route to implementers) |
| R-Security ERROR                  | ERROR (override — security is critical)                |
| R-Knowledge DONE with suggestions | DONE + knowledge updates applied/buffered              |

**Orchestrator impact:** Step 7 (lines 215–233) requires major rewrite. NEEDS_REVISION routing must route from aggregator to implementers. Knowledge evolution outputs need new file locations.

#### Dependency Findings

R cluster depends on the **aggregator pattern template** (hard dependency). Knowledge Evolution has the strongest dependency on the memory system (for knowledge persistence). R-Knowledge safeguards must be designed before deployment. This is the **most architecturally novel and risky component** and should be implemented last.

**Performance projection:** Theoretical speedup ~3–3.5× for this stage; practical speedup ~2.5–3× (all 4 run in parallel; aggregation is lightweight).

---

### 5. Research Phase Expansion (3→4 Agents)

#### What It Is

Add a 4th parallel research focus area to the existing architecture, impact, and dependencies.

#### Architecture Findings

Research is currently hardcoded to exactly 3 focus areas in both the orchestrator (lines 93–103) and the researcher (lines 37–42). The 4th area is **not yet defined** — the initial request says "should use 4 parallel agents" without specifying the additional scope.

#### Impact Findings

**Files affected:** 2 — `orchestrator.agent.md` (dispatch table) and `researcher.agent.md` (focus area table). Approximately 5–10 lines changed total.

**Risk level: Low.** The change is additive, self-contained, and follows the existing parallel dispatch pattern. Adding a 4th researcher runs in parallel with the existing 3, so it adds analytical coverage without reducing wall-clock time (all 4 finish when the slowest finishes).

#### Dependency Findings

**Fully independent** — zero dependencies on C1 (memory), C2 (CT cluster), C3 (V cluster), C4 (R cluster), or C6 (memory integration). Can be shipped at any time. This is a **free win**.

---

### 6. Orchestrator Evolution

The orchestrator is the **most extensively impacted** component and the **single serialization bottleneck** for the entire upgrade.

#### Scope of Changes

| Section                    | Change Type                                                               | Estimated Impact |
| -------------------------- | ------------------------------------------------------------------------- | ---------------- |
| Step 0 (Setup)             | New: Initialize memory                                                    | +10–15 lines     |
| Step 1.1 (Research)        | Modify: Dispatch 4 instead of 3                                           | ~5 lines         |
| Step 3b (Critical Review)  | **Major rewrite:** 4 CT sub-agents → aggregate                            | ~30–40 lines     |
| Step 5 (Implementation)    | Modify: Memory reference passing                                          | ~10–15 lines     |
| Step 6 (Verification)      | **Major rewrite:** V-Build → parallel verify ×3 → aggregate → replan loop | ~40–50 lines     |
| Step 7 (Review)            | **Major rewrite:** 4 R sub-agents → aggregate → knowledge evolution       | ~30–40 lines     |
| Global Rules               | Modify: Memory management, cluster awareness                              | ~10–15 lines     |
| NEEDS_REVISION Table       | Rewrite: Add all aggregator rows                                          | ~15–20 lines     |
| Expectations Table         | Rewrite: Add all sub-agent + aggregator rows                              | ~30–40 lines     |
| Memory Updates             | New: After-step memory writes                                             | ~20–30 lines     |
| Parallel Execution Summary | Rewrite: Show cluster flows                                               | ~15–20 lines     |

**Total estimated change:** ~200–250 lines of additions/modifications. Current size: 323 lines → estimated final size: ~470–530 lines (~60% rewrite).

#### Key Concerns

1. **Context window risk:** At 500+ lines, the orchestrator is at the edge of manageable complexity for an LLM agent prompt. Coherence degradation on later sections is a real risk.
2. **Three cluster coordination patterns:** The orchestrator must manage CT (fully parallel → aggregate), V (sequential build → parallel verify ×3 → aggregate), and R (fully parallel → aggregate + knowledge evolution) — each with different internal dependencies.
3. **New failure modes:** Partial cluster failures (2 of 4 sub-agents ERROR), mixed status in replan loops, non-blocking knowledge evolution vs. critical quality/security checks.
4. **Serialization bottleneck:** All cluster phases (C2, C3, C4) plus memory (C1/C6) and research expansion (C5) all require orchestrator modifications. These must be carefully sequenced.

**Mitigation options:**

- Apply all orchestrator changes in one atomic update (requires all designs finalized first)
- Sequence changes: C5 → C2 → C3 → C4 → C6
- Consider sub-orchestrators for cluster management (reduces per-file scope but increases file count)

---

## Cross-Cutting Concerns

### 1. Aggregator Pattern Reuse

All three clusters (CT, V, R) need aggregators that follow the same fundamental pattern: consume N sub-agent outputs, merge them, resolve or surface conflicts, and produce a single output artifact with a single completion contract. The researcher synthesis mode is the closest existing precedent. Designing a reusable aggregator template (consistent with the established agent definition pattern) avoids bespoke designs and ensures consistency. The aggregator contract rules are:

| Scenario                       | Aggregated Result                        |
| ------------------------------ | ---------------------------------------- |
| All DONE                       | DONE                                     |
| Any ERROR + rest DONE          | ERROR (retry failing sub-agent)          |
| Any NEEDS_REVISION + rest DONE | NEEDS_REVISION (merge revision findings) |
| Mixed ERROR + NEEDS_REVISION   | ERROR takes priority                     |

### 2. Agent Template Consistency

All 15 new sub-agents + 3 aggregators must follow the established v2 agent structural pattern (YAML frontmatter, role statement, inputs/outputs, 5 operating rules, workflow, output spec, completion contract, anti-drift anchor). The agent count grows from 10+1 to 25+1, increasing the maintenance surface by 2.5×. Consistent templating is essential.

### 3. Naming and File Organization

15 new agent files need a clear naming convention. Options:

- **Prefix-based:** `ct-security.agent.md`, `v-build.agent.md`, `r-knowledge.agent.md`
- **Directory-based:** `agents/critical-thinking/security.agent.md`

This affects orchestrator dispatch and maintenance ergonomics.

### 4. Concurrency Cap Compatibility

The existing max-4 concurrent subagent cap (Global Rule 7) already accommodates all proposed clusters (each has ≤4 sub-agents). Clusters and implementation waves don't overlap temporally (different pipeline stages), so no cap adjustment is needed. Total pipeline invocations grow from ~15 to ~35+ per run.

### 5. No Automated Testing

Forge is a purely declarative markdown system with no automated tests. Validation requires running the full pipeline on a real feature request. There is no way to unit-test individual agent changes. This makes phased migration with per-phase validation essential.

### 6. Quality vs. Speed Tradeoff

Parallelization fundamentally trades depth of holistic reasoning for breadth of focused analysis. A single critical thinker makes internally consistent tradeoffs across all risk categories. Split sub-agents lose cross-cutting insight. All aggregators should include a "cross-cutting concerns" consideration and sub-agents should flag issues that span their scope boundaries.

### 7. Error Recovery Composition

Error recovery operates at three layers: tool-level (agent retries ×2), agent-level (orchestrator retries ×1), pipeline-level (verification replan loop ×3). The worst case is 3 × 2 = 6 tool calls per operation. Clusters add a fourth layer: cluster-level failure (partial sub-agent failures). The aggregator must handle partial failures and the orchestrator must handle aggregator failures.

---

## Dependency Graph and Critical Path

### Change Areas and Dependencies

```
C1 (Memory System Design) ──────[hard]──→ C6 (Memory Integration — All Agents)
         │
         └──[soft]──→ C2, C3, C4 (clusters benefit from memory)

Aggregator Pattern Template ──[hard]──→ C2 (CT Cluster)
                             ──[hard]──→ C3 (V Cluster)
                             ──[hard]──→ C4 (R Cluster)

C5 (Research Expansion) ── fully independent ── no dependencies

C2, C3, C4 ── mutually independent (different pipeline stages)
           ── all serialize on orchestrator.agent.md modifications
```

### Optimized Implementation Plan (5 Steps)

```
Step 1 (PARALLEL):
  ├─ C1: Memory Schema Design
  ├─ Aggregator Pattern Template Design
  └─ C5: Research Expansion (3→4) — independent, ship anytime

Step 2 (PARALLEL):
  ├─ C2: Create 4 CT sub-agents + CT-Aggregator (5 new files)
  ├─ C3: Create 4 V sub-agents + V-Aggregator (5 new files)
  ├─ C4: Create 4 R sub-agents + R-Aggregator (5 new files)
  └─ C6 partial: Add memory to non-split agents (spec, designer, planner,
                  implementer, documentation-writer, feature-workflow.prompt — 6 files)

Step 3 (SEQUENTIAL — orchestrator bottleneck):
  ├─ Orchestrator memory init + global rules
  ├─ Orchestrator Step 1.1 update (C5)
  ├─ Orchestrator Step 3b rewrite (C2)
  ├─ Orchestrator Step 6 rewrite (C3)
  ├─ Orchestrator Step 7 rewrite (C4)
  └─ Orchestrator routing/expectations tables

Step 4 (PARALLEL):
  ├─ C6 remaining: Add memory to all 15 new sub-agents + 3 aggregators
  └─ Knowledge Evolution safeguard design

Step 5:
  └─ End-to-end integration testing + performance validation
```

**Critical path length:** 5 steps (reduced from 10 sequential by parallelizing sub-agent creation and non-orchestrator memory integration).

### Orchestrator as Serialization Bottleneck

All cluster splits (C2, C3, C4), research expansion (C5), and memory integration (C6) funnel through `orchestrator.agent.md`. The ~200–250 lines of orchestrator changes across 8+ sections make it the single bottleneck. Mitigation: either apply all orchestrator changes in one atomic update (after all designs are finalized) or sequence them carefully (C5 → C2 → C3 → C4 → C6).

---

## Risk Assessment

### Critical Risks

| Risk                                               | Likelihood | Impact   | Mitigation                                                                                  |
| -------------------------------------------------- | ---------- | -------- | ------------------------------------------------------------------------------------------- |
| **Knowledge Evolution self-modification**          | Medium     | Critical | Suggestion-only writes; human approval; versioning; rollback; implement last                |
| **Stale memory / memory-artifact drift**           | High       | High     | Invalidation protocol on revision loops; agents verify critical decisions against artifacts |
| **Orchestrator complexity explosion** (500+ lines) | Medium     | High     | Sub-orchestrators; extracted patterns; keep orchestrator focused on high-level flow         |

### High Risks

| Risk                                                           | Likelihood  | Impact      | Mitigation                                                                                         |
| -------------------------------------------------------------- | ----------- | ----------- | -------------------------------------------------------------------------------------------------- |
| **Memory write conflicts** (parallel agents)                   | High        | Medium-High | Per-group sections or deferred writes via aggregator                                               |
| **Aggregation quality degradation** (nuances lost)             | Medium      | High        | Flag conflicts, don't resolve; include critical findings verbatim; cross-referencing               |
| **Quality loss from agent splitting** (cross-cutting insights) | Medium-High | Medium      | Overlapping sub-agent scope; cross-cutting concerns prompt; aggregator cross-referencing           |
| **Migration breaks working pipeline**                          | Medium      | High        | Phased migration; per-phase validation; backward compatibility during transition                   |
| **V-Cluster build state sharing**                              | Medium      | High        | V-Build produces artifacts on disk; verify filesystem access; provide build path to all sub-agents |
| **Replan loop complexity with clusters**                       | Medium      | Medium      | Simplify: always re-run full cluster on replan; accept slight performance penalty                  |

### Medium Risks

| Risk                                        | Likelihood | Impact | Mitigation                                                |
| ------------------------------------------- | ---------- | ------ | --------------------------------------------------------- |
| **Memory size growth**                      | Medium     | Medium | Pruning strategy; phase-aware memory; strict size budgets |
| **Conflicting findings in CT cluster**      | High       | Medium | Aggregator surfaces conflicts; designer resolves          |
| **Orchestrator context window degradation** | Medium     | Medium | Target <400 lines; consider decomposition                 |
| **No automated testing**                    | Always     | Medium | Full pipeline validation per migration phase              |

---

## Performance Projections

### Per-Stage Speedup Estimates

| Stage                     | Current     | Proposed                       | Theoretical Speedup            | Practical Speedup |
| ------------------------- | ----------- | ------------------------------ | ------------------------------ | ----------------- |
| Critical Review (Step 3b) | 1 agent × T | 4 parallel + aggregate         | ~3–3.5×                        | ~2–2.5×           |
| Verification (Step 6)     | 1 agent × T | Build + 3 parallel + aggregate | ~2–2.5×                        | ~1.5–2×           |
| Review (Step 7)           | 1 agent × T | 4 parallel + aggregate         | ~3–3.5×                        | ~2.5–3×           |
| Research (Step 1.1)       | 3 parallel  | 4 parallel                     | ~1× (adds coverage, not speed) | ~1×               |

### End-to-End Pipeline Impact

```
Current (rough relative timing):
Research(3∥) → Synth → Spec → Design → CritReview(1×) → Plan → Impl(waves) → Verify(1×) → Review(1×)
  [10%]        [5%]   [5%]   [8%]      [12%]            [10%]    [30%]         [10%]        [10%]

Proposed:
Research(4∥) → Synth → Spec → Design → CT-Cluster(4∥) → Plan → Impl(waves) → V-Cluster → R-Cluster(4∥)
  [10%]        [5%]   [5%]   [8%]      [5%+2%agg]       [10%]    [30%]        [6%+2%agg]   [4%+2%agg]
```

**Estimated end-to-end speedup: ~15–20%.** The sequential middle pipeline (Spec → Design → Planning at ~23% of total time) and implementation waves (~30%, already parallelized) create a floor on maximum possible improvement. Aggregation overhead (~5–8% added across 3 clusters) partially offsets gains.

### Bottleneck Analysis

1. **Sequential middle pipeline (unchanged, ~23%):** Spec → Design → Planning cannot be parallelized — each depends on the previous stage's output.
2. **Aggregation overhead (~5–8%):** Three clusters × aggregation time partially offsets parallelization gains.
3. **Memory read overhead:** 20+ extra file reads across the pipeline (every agent reads memory). May negate some speedup from reduced artifact reads.
4. **V-Cluster build dependency:** In build-failure scenarios (common in replan loops), verification cluster provides zero speedup.

---

## Key Recommendations

1. **Start with Memory Schema Design (C1) and Aggregator Pattern Template in parallel.** These are the two foundational infrastructure pieces that gate all downstream work. Starting with a minimal memory design and iterating is safer than designing the full system upfront.

2. **Ship Research Expansion (C5) independently and early.** Zero dependencies, low risk, 2 files changed, validates orchestrator dispatch updates. A quick win that can proceed alongside any other work.

3. **Design the Aggregator Pattern once, instantiate three times.** Researcher synthesis mode is the closest existing precedent. A reusable template avoids bespoke designs and ensures consistency across CT, V, and R clusters.

4. **Sequence cluster orchestrator integrations: C2 → C3 → C4.** CT cluster is simplest (fully parallel), validating the aggregator pattern. V cluster is more complex (sequential-then-parallel + replan loops). R cluster is most novel (knowledge evolution). Each validates patterns for the next.

5. **Implement Knowledge Evolution (R-Knowledge) last with strict guardrails.** It introduces self-modification into a deterministic system. Suggestion-only writes, human approval, versioning, and rollback mechanisms are non-negotiable prerequisites.

6. **Address orchestrator size proactively.** At 500+ lines, coherence degradation is a real risk. Consider sub-orchestrators for cluster coordination, or extract cluster patterns into reusable templates referenced by the orchestrator.

7. **Design memory invalidation before memory itself.** Stale memory is the highest risk of the memory system. The invalidation protocol for revision loops (design, replan, review fix) must be part of the initial memory schema design, not an afterthought.

8. **Accept simplicity over optimization in replan loops.** Always re-run the full verification cluster on replan rather than targeting specific sub-agents. The slight performance penalty is worth the reduced orchestrator complexity.

9. **Establish the 4th research focus area during the design phase.** The initial request calls for it but doesn't define it. Candidates might include: testing strategy, developer experience, operational concerns, or security landscape.

10. **Maintain backward compatibility during migration.** Consider orchestrator support for both single-agent and cluster modes during transition, enabling incremental rollout and rollback if needed.

---

## Open Questions

1. **Memory concurrency model:** How should parallel agents write to memory without conflicts? (Per-group sections, deferred writes via aggregator, or write-locking?) This architectural decision must precede schema design.

2. **4th research focus area:** What should it cover? The initial request says "4 parallel agents" but doesn't define the new area.

3. **Orchestrator decomposition:** Should the orchestrator remain a single agent definition (500+ lines) or be decomposed into a master orchestrator + sub-orchestrators for cluster management?

4. **Aggregator placement:** Should aggregators be standalone agents dispatched by the orchestrator (consistent with current architecture) or embedded in orchestrator logic (simpler but violates agent isolation)?

5. **Knowledge Evolution scope:** Should R-Knowledge be able to modify agent definitions in `NewAgentsAndPrompts/` directly, or only write to a suggestions buffer requiring human approval?

6. **Sub-agent naming convention:** Prefix-based (`ct-security.agent.md`) or directory-based (`agents/critical-thinking/security.agent.md`)? Affects organization, dispatch, and maintenance.

7. **Backward compatibility during migration:** Big-bang switch or gradual rollout with dual-mode orchestrator?

8. **Build state sharing:** How does V-Build ensure its artifacts are visible to V-Tests, V-Tasks, and V-Feature if they run in separate `runSubagent` contexts?

9. **`decisions.md` ownership in the review cluster:** R-Knowledge, R-Quality, or the aggregator?

10. **`.github` directory:** The reviewer references `.github/instructions/` but it doesn't exist. Should it be created as part of this upgrade or the reference updated?

---

## Appendix / Sources

### Partial Research Files Synthesized

| File                                                 | Focus Area                                                        | Confidence | Coverage                                               |
| ---------------------------------------------------- | ----------------------------------------------------------------- | ---------- | ------------------------------------------------------ |
| [research/architecture.md](research/architecture.md) | Architecture — structure, patterns, flow, bottlenecks             | High       | Complete — all 11 agent files, README, supporting docs |
| [research/impact.md](research/impact.md)             | Impact — per-agent changes, performance, risks, migration         | High       | Complete — all proposed changes, migration phasing     |
| [research/dependencies.md](research/dependencies.md) | Dependencies — data flow, ordering, critical path, infrastructure | High       | Complete — all inter-agent flows, change dependencies  |

### Conflict Notes

- **No contradictions** between partial analyses. All three research areas reached consistent conclusions about the orchestrator being the critical bottleneck, memory being the highest-risk cross-cutting concern, and knowledge evolution being the most novel/risky component.
- **Overlapping coverage** in risk assessment (all three partials identified memory stale/drift risks and aggregation quality concerns) — deduplicated in this synthesis.
- **Consistent file references** across all partials — all reference the same agent files and line numbers.

### Key Reference Files

| File                                                         | Role in Analysis                                         |
| ------------------------------------------------------------ | -------------------------------------------------------- |
| `NewAgentsAndPrompts/orchestrator.agent.md` (323 lines)      | Central pipeline definition; most impacted file          |
| `NewAgentsAndPrompts/critical-thinker.agent.md` (139 lines)  | Split candidate for CT cluster                           |
| `NewAgentsAndPrompts/verifier.agent.md` (179 lines)          | Split candidate for V cluster                            |
| `NewAgentsAndPrompts/reviewer.agent.md` (220 lines)          | Split candidate for R cluster + knowledge evolution      |
| `NewAgentsAndPrompts/researcher.agent.md` (137 lines)        | Research expansion + synthesis precedent for aggregators |
| `docs/optimization-from-gem-team.md`                         | P2 cross-agent memory promoted to core in this upgrade   |
| `docs/comparison-forge-vs-gem-team.md`                       | Validates two-layer verification model preservation      |
| `docs/feature/forge-architecture-upgrade/initial-request.md` | Grounding document defining all requirements             |
