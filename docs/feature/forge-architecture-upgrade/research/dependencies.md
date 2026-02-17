# Dependencies Research

## Focus Area

Dependencies — Module interactions, data flow, API/interface contracts, external dependencies, integration points between affected areas, ordering constraints, parallelization opportunities, and critical path analysis for the proposed Forge architecture upgrade.

## Summary

The proposed upgrade has 6 major change areas with a shared infrastructure foundation (memory system + aggregator pattern) that gates all downstream work. Three cluster splits (CT, V, R) are independent of each other but all compete for orchestrator file modifications, creating a serialization bottleneck. The critical path runs: Memory Design → Aggregator Pattern → Cluster Splits (sequenced on orchestrator) → Integration. Research expansion (3→4) is fully independent and can proceed at any time.

---

## Change Dependency Graph

### Proposed Change Areas

| ID     | Change                               | Description                                                             | Files Created              | Files Modified                                       |
| ------ | ------------------------------------ | ----------------------------------------------------------------------- | -------------------------- | ---------------------------------------------------- |
| **C1** | Memory System                        | Operational memory with artifact navigation, decisions, lessons learned | 1 (memory schema/template) | All 11 existing files                                |
| **C2** | Critical Thinker Split               | 4 parallel CT sub-agents + aggregator                                   | 5 new agent files          | `orchestrator.agent.md`, `critical-thinker.agent.md` |
| **C3** | Verifier Split                       | 4 parallel V sub-agents + aggregator                                    | 5 new agent files          | `orchestrator.agent.md`, `verifier.agent.md`         |
| **C4** | Reviewer Split + Knowledge Evolution | 4 parallel R sub-agents + aggregator with knowledge evolution           | 5 new agent files          | `orchestrator.agent.md`, `reviewer.agent.md`         |
| **C5** | Research Expansion (3→4)             | Add 4th research focus area                                             | 0                          | `orchestrator.agent.md`, `researcher.agent.md`       |
| **C6** | Memory Integration (All Agents)      | Add memory read/write to every agent workflow                           | 0                          | All 11 existing files                                |

### Change-to-Change Dependencies

```
C1 (Memory System Design) ──────────────────────────────────┐
  │                                                          │
  └──→ C6 (Memory Integration - All Agents)                  │
         │                                                   │
         ├──→ Enhances C2 (CT sub-agents use memory)         │
         ├──→ Enhances C3 (V sub-agents use memory)          │
         └──→ Enhances C4 (R sub-agents use memory)          │
                                                             │
Aggregator Pattern Design ←──────────────────────────────────┘
  │ (shared infrastructure, derived from C1's phase-awareness)
  │
  ├──→ C2 (CT Cluster needs aggregator)
  ├──→ C3 (V Cluster needs aggregator)
  └──→ C4 (R Cluster needs aggregator)

C5 (Research Expansion) ── fully independent ── no dependencies on C1-C4 or C6

C2, C3, C4 ── mutually independent (different pipeline stages)
           ── but ALL modify orchestrator.agent.md (serialization point)
```

### Dependency Types

| From       | To                 | Dependency Type                                     | Strength    | Notes                                                            |
| ---------- | ------------------ | --------------------------------------------------- | ----------- | ---------------------------------------------------------------- |
| C6         | C1                 | **Hard** — cannot integrate memory without a design | Blocking    | Memory schema must exist before agents can read/write it         |
| C2         | Aggregator Pattern | **Hard** — cluster requires aggregation             | Blocking    | Aggregator template must be defined                              |
| C3         | Aggregator Pattern | **Hard** — cluster requires aggregation             | Blocking    | Same aggregator pattern, different inputs/outputs                |
| C4         | Aggregator Pattern | **Hard** — cluster requires aggregation             | Blocking    | Same aggregator pattern, different inputs/outputs                |
| C2         | C1                 | **Soft** — CT sub-agents can work without memory    | Optional    | Memory integration adds value but isn't functionally required    |
| C3         | C1                 | **Soft** — V sub-agents can work without memory     | Optional    | Memory integration adds value but isn't functionally required    |
| C4         | C1                 | **Soft** — R sub-agents can work without memory     | Optional    | R-Knowledge benefits most from memory for knowledge persistence  |
| C5         | None               | **None** — fully independent                        | Independent | Only touches orchestrator Step 1.1 + researcher focus area table |
| C2, C3, C4 | Each other         | **None** — different pipeline stages                | Independent | But all require orchestrator changes at different sections       |

---

## Agent Inter-dependency Map

### Current Pipeline Data Flow (Artifact-Based)

```
                        initial-request.md
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
              research/   research/  research/
           architecture.md impact.md dependencies.md
                    │         │         │
                    └─────────┼─────────┘
                              ▼
                         analysis.md
                              │
                    ┌─────────┘
                    ▼
                feature.md
                    │
          ┌─────────┼──────────┐
          ▼         ▼          │
      design.md     │          │
          │         │          │
          ▼         │          │
 design_critical_   │          │
    review.md       │          │
          │         │          │
          ▼         ▼          ▼
              plan.md + tasks/*.md
                    │
                    ▼
              [implementer(s)]
                    │
                    ▼
              verifier.md
                    │
                    ▼
               review.md
                    │
                    ▼
             decisions.md (optional)
```

### Agent-to-Agent Dependency Matrix

| Producer Agent          | Output Artifact                                                              | Consumer Agents                                                                            |
| ----------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Orchestrator            | `initial-request.md`                                                         | ALL agents (grounding)                                                                     |
| Researcher (focused ×3) | `research/architecture.md`, `research/impact.md`, `research/dependencies.md` | Researcher (synthesis)                                                                     |
| Researcher (synthesis)  | `analysis.md`                                                                | Spec, Designer, Planner                                                                    |
| Spec                    | `feature.md`                                                                 | Designer, Critical Thinker, Planner, Implementer, Verifier                                 |
| Designer                | `design.md`                                                                  | Critical Thinker, Planner, Implementer, Verifier                                           |
| Critical Thinker        | `design_critical_review.md`                                                  | Designer (on NEEDS_REVISION), Orchestrator                                                 |
| Planner                 | `plan.md`, `tasks/*.md`                                                      | Orchestrator (wave parsing), Implementer, Verifier                                         |
| Implementer             | Code + test files, task file updates                                         | Verifier, Reviewer                                                                         |
| Verifier                | `verifier.md`                                                                | Planner (on NEEDS_REVISION), Orchestrator                                                  |
| Reviewer                | `review.md`, `decisions.md`, `.github/instructions`                          | Implementer (on NEEDS_REVISION), Orchestrator, Researcher (future runs via `decisions.md`) |
| Documentation Writer    | Documentation files, task file updates                                       | Verifier                                                                                   |

### Completion Contract Dependencies

The orchestrator routes work based on completion contracts. This creates implicit dependencies:

| Agent            | Contract States                     | Routing Dependency                                                                                                                                          |
| ---------------- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Critical Thinker | `DONE` / `NEEDS_REVISION` / `ERROR` | `NEEDS_REVISION` → Designer (max 1 loop). Splitting CT into 4 sub-agents means the **aggregator** must produce the consolidated contract.                   |
| Verifier         | `DONE` / `NEEDS_REVISION` / `ERROR` | `NEEDS_REVISION` → Planner replan loop (max 3). Splitting V into cluster means **aggregator** produces consolidated contract and maps failures to task IDs. |
| Reviewer         | `DONE` / `NEEDS_REVISION` / `ERROR` | `NEEDS_REVISION` → Implementer(s) (max 1 loop). Splitting R into cluster means **aggregator** produces consolidated contract and maps findings to tasks.    |

**Key Dependency:** Each aggregator must produce the same contract format as the current single agent, ensuring the orchestrator's routing logic remains compatible. This is a **contract compatibility constraint**.

### Proposed Pipeline Data Flow (With Clusters + Memory)

```
                     initial-request.md
                           │
               ┌───────────┼───────────┬───────────┐
               ▼           ▼           ▼           ▼
         research/    research/   research/   research/
       architecture  impact      dependencies [4th area]
               │           │           │           │
               └───────────┼───────────┴───────────┘
                           ▼
                      analysis.md ──→ [memory write: synthesis summary]
                           │
                           ▼
                      feature.md ──→ [memory write: requirement summary]
                           │
                           ▼
                       design.md ──→ [memory write: design decisions]
                           │
               ┌───────────┼───────────┬───────────┐
               ▼           ▼           ▼           ▼
          CT-Security  CT-Scale  CT-Maintain  CT-Strategy
               │           │           │           │
               └───────────┼───────────┴───────────┘
                           ▼
                   CT-Aggregator → design_critical_review.md
                           │
                           ▼
                  plan.md + tasks/*.md ──→ [memory write: task index]
                           │
                           ▼
                   [implementer(s) × parallel waves]
                           │
                           ▼
                       V-Build
                           │
               ┌───────────┼───────────┐
               ▼           ▼           ▼
           V-Tests     V-Tasks     V-Feature
               │           │           │
               └───────────┼───────────┘
                           ▼
                  V-Aggregator → verifier.md
                           │
               ┌───────────┼───────────┬───────────┐
               ▼           ▼           ▼           ▼
          R-Quality   R-Security  R-Testing  R-Knowledge
               │           │           │           │
               └───────────┼───────────┴───────────┘
                           ▼
                  R-Aggregator → review.md + decisions.md + knowledge suggestions
```

---

## Artifact Dependency Matrix

### Current Artifact Input/Output Map

| Agent                      | Reads (Inputs)                                                                               | Writes (Outputs)                                    | Memory Reads (Proposed)               | Memory Writes (Proposed)                     |
| -------------------------- | -------------------------------------------------------------------------------------------- | --------------------------------------------------- | ------------------------------------- | -------------------------------------------- |
| **Orchestrator**           | User request                                                                                 | `initial-request.md`                                | Phase status, error history           | Phase transitions, dispatch records          |
| **Researcher (focused)**   | `initial-request.md`                                                                         | `research/<focus>.md`                               | Artifact index, prior decisions       | Focus-area summary, key file refs            |
| **Researcher (synthesis)** | `initial-request.md`, `research/*.md`                                                        | `analysis.md`                                       | All research summaries                | Synthesized overview, key findings           |
| **Spec**                   | `initial-request.md`, `analysis.md`                                                          | `feature.md`                                        | Analysis summary, prior decisions     | Requirement summary, acceptance criteria     |
| **Designer**               | `initial-request.md`, `analysis.md`, `feature.md`, (`design_critical_review.md` on revision) | `design.md`                                         | Spec summary, critical review context | Design decisions, component map, API surface |
| **Critical Thinker**       | `initial-request.md`, `design.md`, `feature.md`                                              | `design_critical_review.md`                         | Design summary, spec requirements     | Risk findings, assumptions challenged        |
| **Planner**                | `initial-request.md`, `analysis.md`, `feature.md`, `design.md`, (`verifier.md` on replan)    | `plan.md`, `tasks/*.md`                             | All prior summaries, failure context  | Task index, wave structure, pre-mortem       |
| **Implementer**            | Task file, `feature.md`, `design.md` (NOT `plan.md`)                                         | Code, tests, task file updates                      | Design decisions, task context        | Implementation notes, issues, test results   |
| **Verifier**               | `initial-request.md`, `feature.md`, `design.md`, `plan.md`, `tasks/*.md`                     | `verifier.md`                                       | Expected state, task summaries        | Verification results, failure details        |
| **Reviewer**               | `initial-request.md`, git diff, codebase                                                     | `review.md`, `decisions.md`, `.github/instructions` | Full feature context, impl notes      | Review findings, arch decisions, knowledge   |
| **Documentation Writer**   | Task file, `feature.md`, `design.md`, codebase                                               | Doc files, task file updates                        | Feature context, design summary       | Documentation status                         |

### Artifact Chain Dependencies (Linear Flow)

The artifact chain is strictly linear with branching only at the implementation stage:

```
initial-request.md
  ↓ (read by researcher ×3/4)
research/*.md
  ↓ (read by synthesis researcher)
analysis.md
  ↓ (read by spec)
feature.md
  ↓ (read by designer)
design.md
  ↓ (read by critical thinker; may loop back to designer once)
design_critical_review.md
  ↓ (read by orchestrator for routing decision)
plan.md + tasks/*.md
  ↓ (read by implementer, per-task)
[code + tests]
  ↓ (verified by verifier)
verifier.md
  ↓ (triggers replan if NEEDS_REVISION; max 3 loops)
review.md
  ↓ (triggers fix if NEEDS_REVISION; max 1 loop)
decisions.md (optional, append-only, persists across features)
```

**Key property:** Each artifact depends on all upstream artifacts. Breaking any link in the chain blocks all downstream work. The memory system adds a **parallel read path** — agents read memory for summaries before accessing artifacts, but artifacts remain the source of truth.

### How Memory Changes Artifact Dependencies

1. **Reduced direct reads:** Agents read memory summaries first, then access artifacts only if memory is insufficient. This reduces but does not eliminate artifact reads.
2. **New write dependency:** Every agent now writes to memory after producing its output. This creates a new shared resource that parallel agents may contend for.
3. **Stale memory risk:** If an artifact is revised (design loop, replan loop), memory entries written before the revision become stale. The orchestrator must invalidate relevant memory entries on revision loops.
4. **Memory does NOT replace artifacts:** Per the initial request: "keeps artifacts as the single source of truth." Memory is a navigation/summary layer, not a replacement.

---

## Ordering Constraints

### Hard Ordering Constraints (Must Be Sequential)

| Constraint                                                        | Reason                                                                                                                      | Impact                                            |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| C1 (Memory Design) **before** C6 (Memory Integration)             | Cannot add memory read/write to agents without a defined schema                                                             | All agents blocked until memory schema exists     |
| Aggregator Pattern Design **before** C2, C3, C4                   | All clusters need aggregation; designing the pattern once avoids inconsistency                                              | All cluster work blocked until pattern is defined |
| CT Sub-agent Design **before** CT Aggregator                      | Aggregator must know sub-agent output format                                                                                | Within C2 only                                    |
| V-Build Sub-agent **before** V-Tests/V-Tasks/V-Feature at runtime | Build must succeed before parallel verification                                                                             | Within C3 only; affects runtime, not development  |
| R-Knowledge safeguards **before** R-Knowledge deployment          | Self-modification risk requires guardrails before activation                                                                | Within C4 only                                    |
| Orchestrator changes for C2, C3, C4 must be **sequenced**         | All three modify `orchestrator.agent.md` at different sections; concurrent editing of the same file creates merge conflicts | Serialization bottleneck                          |

### Soft Ordering Constraints (Recommended But Not Blocking)

| Constraint                                 | Reason                                                                                                                          |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| C1 before C2/C3/C4                         | Clusters benefit from memory integration but can be designed without it                                                         |
| C2 (CT Cluster) before C3 (V Cluster)      | CT is simpler (fully parallel); validate the aggregator pattern before tackling V's sequential-then-parallel model              |
| C3 (V Cluster) before C4 (R Cluster)       | V Cluster has more complex orchestrator interaction (replan loops); R Cluster adds knowledge evolution (most novel, most risky) |
| C5 (Research Expansion) before any cluster | Low-risk; quick win; validates orchestrator dispatch updates                                                                    |

### What Can Be Parallelized

| Items                                               | Can Parallel? | Notes                                                                |
| --------------------------------------------------- | ------------- | -------------------------------------------------------------------- |
| C1 (Memory Design) and C5 (Research Expansion)      | **Yes**       | Fully independent; different files, different concerns               |
| C2, C3, C4 sub-agent file creation                  | **Yes**       | New files don't conflict; only orchestrator integration serializes   |
| Memory schema design and aggregator pattern design  | **Yes**       | Independent design activities with different outputs                 |
| CT sub-agents (4 files)                             | **Yes**       | All 4 CT sub-agent files can be written in parallel                  |
| V sub-agents (4 files)                              | **Yes**       | All 4 V sub-agent files can be written in parallel                   |
| R sub-agents (4 files)                              | **Yes**       | All 4 R sub-agent files can be written in parallel                   |
| Orchestrator C2 changes and Orchestrator C3 changes | **No**        | Same file; must be sequenced to avoid conflicts                      |
| C6 (Memory integration across agents)               | **Partially** | Each agent's memory integration is independent, but all depend on C1 |

---

## Shared Infrastructure Requirements

### 1. Memory System Infrastructure (Foundation for C1 + C6)

**What must be built:**

- Memory file schema/template (what fields, what sections, what lifecycle)
- Memory initialization protocol (orchestrator Step 0)
- Memory read contract (every agent reads memory first)
- Memory write contract (every agent writes summary after producing output)
- Memory invalidation protocol (on design revision, replan loop, review fix)
- Memory concurrency model (how parallel agents write without conflicts)
- Memory pruning strategy (prevent unbounded growth)

**Who uses it:** All 11 existing agents + all 15 new sub-agents + 3 aggregators = **29 agents** total.

**Design constraints (from initial-request.md):**

- Does NOT duplicate artifact content
- Keeps artifacts as single source of truth
- Stores only: artifact navigation metadata, recent decisions, lessons learned, recent artifact updates
- Agents read memory FIRST before accessing artifacts
- Must be small, structured, and phase-aware
- Clear read/write responsibilities per agent type

**Concurrency challenge (from impact.md, Risk 1):**
Parallel agents (research ×4, implementation waves, all clusters) may write concurrently. Options:

- Per-group memory sections (each parallel group writes to its own section)
- Deferred writes via aggregator (parallel agents write to temp; aggregator consolidates)
- Write-locking (platform-dependent; may not be feasible in Copilot runtime)

### 2. Aggregator Pattern Template (Foundation for C2 + C3 + C4)

**What must be built:**

- Reusable aggregator agent template consistent with existing agent structural pattern (YAML frontmatter, role statement, inputs/outputs, operating rules, workflow, completion contract, anti-drift anchor)
- Merge strategy: how to combine N sub-agent outputs into 1 artifact
- Conflict resolution strategy: surface conflicts vs. resolve them
- Contract consolidation: how to produce a single `DONE`/`NEEDS_REVISION`/`ERROR` from N sub-agent results
- High-severity passthrough rule: any `Critical` finding passes through verbatim
- Error handling: what happens if some sub-agents succeed and some fail (partial cluster failure)

**Aggregator contract rules (derived from current orchestrator expectations):**

| Scenario                        | Sub-Agent Results            | Aggregated Result                        |
| ------------------------------- | ---------------------------- | ---------------------------------------- |
| All succeed                     | N × DONE                     | DONE                                     |
| Any ERROR with others DONE      | 1+ ERROR, rest DONE          | ERROR (retry the failing sub-agent)      |
| Any NEEDS_REVISION (CT, R only) | 1+ NEEDS_REVISION, rest DONE | NEEDS_REVISION (merge revision findings) |
| Mixed ERROR + NEEDS_REVISION    | Both present                 | ERROR takes priority (fix errors first)  |

**Who uses it:** CT-Aggregator, V-Aggregator, R-Aggregator (3 instances).

### 3. Sub-Agent Template (Foundation for all 12 new sub-agents)

**What must be built:**

- Template that follows the established agent definition pattern (per architecture.md, Section 1)
- Consistent naming convention for sub-agent files
- Scope delimitation: each sub-agent must know its boundaries relative to siblings
- Cross-cutting concerns prompt: instruction for sub-agents to flag issues that span scopes
- Output format: structured partial output that the aggregator can consume

**Who uses it:** 4 CT sub-agents, 4 V sub-agents, 4 R sub-agents (12 new agents).

---

## External Patterns and Insights

### From `comparison-forge-vs-gem-team.md`

| Pattern                                      | Relevance to Dependencies                                                                        | Source Reference                                            |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| Gem Team's cross-agent memory with citations | Validates the need for C1 (memory system); citation requirement adds input validation dependency | Comparison doc, Section 1 (Orchestrator), Memory system row |
| Gem Team's per-task agent routing            | Already adopted in v2; validates that task files need `agent` field for sub-agent dispatch       | Comparison doc, Section 3 (Planner)                         |
| Gem Team's inline per-task verification      | Confirms Forge should keep dedicated verifier (two-layer model); V Cluster preserves this        | Comparison doc, Section 5                                   |
| Gem Team has no dedicated verifier           | Reinforces that Forge's V Cluster is an enhancement, not a regression                            | Comparison doc, Summary Matrix                              |
| Gem Team has no synthesis step               | Validates Forge's synthesis model; aggregators follow the same pattern                           | Comparison doc, Section 2                                   |

### From `optimization-from-gem-team.md`

| Optimization                          | Current Status                                     | Relevance to Architecture Upgrade                     |
| ------------------------------------- | -------------------------------------------------- | ----------------------------------------------------- |
| P0: Pre-mortem analysis               | **Already adopted** in v2                          | No further dependency                                 |
| P0: TDD enforcement                   | **Already adopted** in v2                          | No further dependency                                 |
| P1: Task size limits                  | **Already adopted** in v2                          | No further dependency                                 |
| P1: Concurrency cap (max 4)           | **Already adopted** in v2                          | Clusters are designed within cap (≤4 sub-agents each) |
| P1: Security-focused review           | **Already adopted** in v2                          | R-Security sub-agent inherits this capability         |
| P1: Hybrid retrieval                  | **Already adopted** in v2                          | Research expansion (C5) builds on this                |
| P2: Per-task agent routing            | **Already adopted** in v2                          | Sub-agents dispatched via same routing mechanism      |
| P2: Cross-agent memory                | **Not yet adopted** → **C1 in this upgrade**       | This is the primary new infrastructure component      |
| P2: Human approval gates              | **Already adopted** in v2 (optional)               | No further dependency                                 |
| "What NOT to adopt": YAML state file  | Still not adopted; confirms markdown-only approach | Memory system must be a markdown file, not YAML       |
| "What NOT to adopt": Mandatory pauses | Still optional                                     | No dependency                                         |

### From `docs/feature/agent-improvements/analysis.md` (v2 Precedent)

The v2 agent-improvements feature identified 3 tightly-coupled change clusters (TDD across 3 files, per-task routing across 2 files, approval gates across 2 files). The current upgrade has a similar pattern but at larger scale:

| v2 Pattern                           | v3 Equivalent                                                    | Scale Increase             |
| ------------------------------------ | ---------------------------------------------------------------- | -------------------------- |
| Cluster A (TDD): 3 files coordinated | Memory System (C1+C6): 11+ files coordinated                     | ~4× more files             |
| Cluster B (routing): 2 files         | Each cluster split: 5 new + 2 modified                           | ~3× more files per cluster |
| Cluster C (approval): 2 files        | Orchestrator as serialization point: 1 file, 3 competing changes | Same file, more changes    |

The v2 analysis recommended "update all files simultaneously" for tightly-coupled clusters (analysis.md, line 589). This principle applies more strongly here — each cluster's orchestrator changes should be applied atomically.

---

## Critical Path Analysis

### Full Dependency Graph (Implementation Order)

```
┌─────────────────────────────────────┐
│ Phase 0: Foundation (PARALLEL)      │
│                                     │
│  ┌─────────────────────┐            │
│  │ C1: Memory System   │            │
│  │ Schema Design       │            │
│  └─────────┬───────────┘            │
│            │                        │
│  ┌─────────────────────┐  ┌──────────────────┐
│  │ Aggregator Pattern  │  │ C5: Research     │
│  │ Template Design     │  │ Expansion (3→4)  │
│  └─────────┬───────────┘  └──────────────────┘
│            │ (independent of C5)    │
└────────────┼────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Phase 1: Sub-Agent Creation (PARALLEL within, but orchestrator │
│          integration SERIALIZED)                                │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ C2: CT Sub-  │  │ C3: V Sub-   │  │ C4: R Sub-   │          │
│  │ agents (×4)  │  │ agents (×4)  │  │ agents (×4)  │          │
│  │ + Aggregator │  │ + Aggregator │  │ + Aggregator │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                  │                  │
│         ▼                 ▼                  ▼                  │
│  ┌──────────────────────────────────────────────────┐           │
│  │ Orchestrator Integration (SEQUENTIAL)            │           │
│  │                                                  │           │
│  │  Step 3b rewrite (C2) ──→ Step 6 rewrite (C3)   │           │
│  │  ──→ Step 7 rewrite (C4) ──→ Routing tables     │           │
│  └──────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ Phase 2: Cross-Cutting Integration  │
│                                     │
│  C6: Memory Integration (all 29    │
│       agents get memory read/write) │
│                                     │
│  NOTE: Can start after C1 complete; │
│  can overlap with Phase 1 for       │
│  existing agents not being split    │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ Phase 3: Integration & Validation   │
│                                     │
│  End-to-end pipeline testing        │
│  Memory pruning tuning              │
│  Knowledge Evolution safeguards     │
│  Performance validation             │
└─────────────────────────────────────┘
```

### Critical Path (Minimum Sequential Steps)

The critical path — the longest chain that determines minimum implementation time — is:

```
Memory Schema Design (C1)
  → Aggregator Pattern Design
    → CT Sub-agents + Aggregator Creation (C2)
      → Orchestrator Step 3b Rewrite (C2 integration)
        → V Sub-agents + Aggregator Creation (C3)
          → Orchestrator Step 6 Rewrite (C3 integration)
            → R Sub-agents + Aggregator Creation (C4)
              → Orchestrator Step 7 Rewrite (C4 integration)
                → Memory Integration All Agents (C6)
                  → Integration Testing (Phase 3)
```

**Critical path length:** 10 sequential steps.

### Optimized Critical Path (Maximum Parallelism)

By parallelizing independent work and overlapping where possible:

```
Step 1 (PARALLEL):
  - C1: Memory Schema Design
  - Aggregator Pattern Template Design
  - C5: Research Expansion (3→4) ← independent, ship anytime

Step 2 (PARALLEL):
  - C2: Create 4 CT sub-agents + CT-Aggregator (5 new files)
  - C3: Create 4 V sub-agents + V-Aggregator (5 new files)
  - C4: Create 4 R sub-agents + R-Aggregator (5 new files)
  - C6 partial: Add memory to non-split agents (spec, designer, planner,
                 implementer, documentation-writer, feature-workflow.prompt)
                 — 6 agents that aren't being split

Step 3 (SEQUENTIAL — orchestrator bottleneck):
  - Orchestrator Step 3b rewrite (C2)
  - Orchestrator Step 6 rewrite (C3)
  - Orchestrator Step 7 rewrite (C4)
  - Orchestrator memory init + routing tables + expectations tables
  - Orchestrator Step 1.1 update (C5)
  (All changes to orchestrator.agent.md applied in sequence to avoid conflicts)

Step 4 (PARALLEL):
  - C6 remaining: Add memory to all 15 new sub-agents + 3 aggregators
  - Integration testing

Step 5:
  - End-to-end validation
  - Knowledge Evolution safeguard verification
```

**Optimized critical path length:** 5 steps (reduced from 10 by parallelizing sub-agent creation and non-orchestrator memory integration).

### Orchestrator as Serialization Bottleneck

The orchestrator file (`orchestrator.agent.md`, currently 323 lines, estimated 500+ lines after upgrade) is modified by:

- C2: Step 3b rewrite (~30-40 lines new/changed)
- C3: Step 6 rewrite (~40-50 lines new/changed)
- C4: Step 7 rewrite (~30-40 lines new/changed)
- C5: Step 1.1 update (~5 lines changed)
- C6: Memory init at Step 0, memory rules in Global Rules (~30 lines)
- Routing table rewrite (~15-20 lines)
- Expectations table rewrite (~30-40 lines)
- Parallel execution summary rewrite (~15-20 lines)

**Total orchestrator changes:** ~200-250 lines of additions/modifications across 8+ sections.

This makes the orchestrator the **single bottleneck** in the implementation plan. All cluster splits and memory integration funnel through orchestrator modifications. If orchestrator changes are not carefully sequenced, they will conflict.

**Mitigation options identified in impact research:**

1. Apply all orchestrator changes in a single atomic update (requires all cluster designs to be finalized first)
2. Sequence orchestrator changes: C5 → C2 integration → C3 integration → C4 integration → C6 integration
3. Consider breaking the orchestrator into a master orchestrator + sub-orchestrators (reduces per-file change scope but increases file count)

---

## Key Observations

1. **Memory system (C1) is the foundational dependency.** All other changes either require it (C6) or benefit from it (C2-C4). It must be designed first. However, it is also the highest-risk change (cross-cutting, affects all agents, no precedent in current system). Starting with a minimal memory design and iterating is safer than designing the full system upfront.

2. **The aggregator pattern is the second foundational dependency.** All three clusters (C2, C3, C4) need aggregators. Designing one reusable pattern and instantiating it three times is more efficient than designing three bespoke aggregators. The research synthesis mode in `researcher.agent.md` is the closest existing precedent for aggregation.

3. **Cluster splits (C2, C3, C4) are mutually independent but serialize on the orchestrator.** The sub-agent files for each cluster can be created in parallel. But integrating each cluster into the orchestrator must be sequenced because they all modify different sections of the same file.

4. **Research expansion (C5) is a free win.** It has zero dependencies on any other change, is low-risk, touches only 2 files (orchestrator Step 1.1 + researcher focus area table), and can be shipped independently at any time.

5. **Knowledge Evolution (part of C4) is the highest-risk component and should be last.** It introduces self-modification into a deterministic system. It depends on the memory system for persistence, on the review cluster for context, and requires safety guardrails before activation. Implementing it last allows validating all other changes first.

6. **The v2 precedent shows tightly-coupled changes must be coordinated.** The v2 agent-improvements analysis identified that the TDD cluster (3 files) required simultaneous update. The current upgrade has the same pattern at 4× the scale (memory touches 11+ files; each cluster touches 7 files).

7. **Agent count growth from 10 to 25+ increases maintenance surface.** Each new sub-agent and aggregator must follow the established template pattern. Consistent naming conventions and a clear file organization strategy are prerequisites for manageable maintenance.

8. **Concurrency cap (max 4) is already designed to accommodate clusters.** Each cluster has ≤4 sub-agents. Clusters and implementation waves don't overlap temporally (they're at different pipeline stages). No cap adjustment is needed.

9. **The verification cluster (C3) has an internal hard dependency** that limits its parallelism: V-Build must succeed before V-Tests, V-Tasks, and V-Feature can run. This makes C3's effective structure 1+3 (sequential + parallel), not 4 parallel. In build-failure scenarios, the cluster provides zero speedup.

10. **Memory concurrency is an unsolved design question.** Parallel agents (research ×4, implementation waves, all cluster sub-agents) writing to memory simultaneously is a race condition risk. The chosen concurrency model (per-group sections, deferred writes, or write-locking) affects the memory schema design and therefore gates all downstream work.

---

## Assumptions & Limitations

1. **Assumption:** The `runSubagent` mechanism supports the increased agent count (from ~15 invocations per run to ~35+). This is platform-dependent and unverified.
2. **Assumption:** Parallel sub-agents in a cluster share filesystem state (critical for V-Build output being accessible to V-Tests/V-Tasks/V-Feature).
3. **Assumption:** Memory will be file-based (markdown), consistent with Forge's artifact-only communication model and the explicit rejection of YAML state files.
4. **Assumption:** The max-4 concurrency cap applies per-stage (not globally), so dispatching 4 CT sub-agents at Step 3b does not conflict with implementation wave capacity.
5. **Limitation:** No runtime testing was performed. All dependency analysis is based on static analysis of agent definitions and their documented contracts.
6. **Limitation:** The 4th research focus area (for C5) is not yet defined. The initial request says "should use 4 parallel agents" but the current system defines only 3 focus areas. The dependency analysis assumes a 4th area will be defined during design.
7. **Limitation:** Aggregation quality is speculative — no aggregator agents exist in the current system. The researcher synthesis mode is the closest analogy.

---

## Open Questions

1. **Memory concurrency model:** How should parallel agents write to memory without conflicts? This architectural decision must precede memory schema design (C1) and therefore gates everything.
2. **Orchestrator size management:** At 500+ lines, can the orchestrator remain a single agent definition, or should it be decomposed? This affects the serialization bottleneck analysis.
3. **Aggregator placement:** Should aggregators be standalone agents (dispatched by orchestrator) or sub-orchestrators (embedded in orchestrator logic)? Standalone agents are more consistent with current architecture but add invocation overhead.
4. **4th research focus area:** What is it? This must be defined before C5 can be implemented.
5. **Knowledge Evolution scope:** Should R-Knowledge modify files in `NewAgentsAndPrompts/` directly or write to a "suggestions" buffer? The safety model depends on this decision.
6. **Sub-agent naming convention:** Prefix-based (`ct-security.agent.md`) or directory-based (`agents/critical-thinking/security.agent.md`)? Affects file organization and orchestrator dispatch.
7. **Backward compatibility during migration:** Should the orchestrator support both single-agent and cluster modes during transition, or is a big-bang switch acceptable?

---

## Research Metadata

- **confidence_level:** high — All 11 agent/prompt files, both comparison documents, the optimization roadmap, the prior v2 analysis, and siblings architecture.md + impact.md were read in full. Dependency chains are fully traced through artifact inputs/outputs and completion contracts.
- **coverage_estimate:** Complete coverage of all inter-agent data flows, artifact dependencies, completion contract routing, proposed change interactions, and implementation ordering constraints. External pattern analysis covers all referenced comparison documents.
- **gaps:** (1) No runtime verification of `runSubagent` concurrency behavior. (2) The 4th research focus area is undefined, preventing full C5 dependency analysis. (3) Memory concurrency model is an open design question that affects the critical path but cannot be resolved through research alone. (4) Aggregator quality/performance is speculative since no aggregator agents exist yet.
