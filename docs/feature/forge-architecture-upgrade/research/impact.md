# Impact Research

## Focus Area

Impact — Affected files, modules, and components; what needs to change and where; risks of each proposed change; migration path; performance implications.

## Summary

The proposed architecture upgrade affects all 11 files in `NewAgentsAndPrompts/`, with the orchestrator requiring the most extensive changes (~60% rewrite). Memory system integration touches every agent. Parallelization of critical thinking, verification, and review introduces 12 new sub-agent definitions plus 3 aggregator agents. The upgrade is high-complexity but achievable in phases, with the memory system being the highest-risk component due to its cross-cutting nature.

---

## Findings

### 1. Memory System Impact

The initial request (initial-request.md, Memory System Requirements section) defines an operational memory layer that stores artifact navigation metadata, recent decisions, lessons learned, and recent artifact updates — while keeping markdown artifacts as the single source of truth.

#### Per-Agent Impact Analysis

| Agent                       | File                                       | Impact Level | Changes Required                                                                                                                                                                              | Risk Level                                                                       |
| --------------------------- | ------------------------------------------ | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Orchestrator**            | `orchestrator.agent.md` (323 lines)        | **Critical** | Must initialize memory at Step 0, update memory after every subagent completion, pass memory reference to all subagents, manage memory lifecycle across replan loops                          | High — memory init/update logic adds ~50-80 lines of new orchestration rules     |
| **Researcher (focused)**    | `researcher.agent.md` (137 lines)          | **High**     | Must read memory FIRST before artifact access (per initial request requirement); write focus-area findings summary back to memory; read artifact index from memory instead of full file reads | Medium — adds ~15-20 lines to workflow; risk of stale index                      |
| **Researcher (synthesis)**  | Same file, synthesis mode                  | **High**     | Must read memory to understand what partials exist; write synthesized summary back to memory for downstream agents                                                                            | Medium — adds ~10-15 lines                                                       |
| **Spec**                    | `spec.agent.md` (105 lines)                | **Medium**   | Must read memory before reading `analysis.md`; write requirement summary + key decisions back to memory                                                                                       | Low — straightforward read/write addition                                        |
| **Designer**                | `designer.agent.md` (119 lines)            | **Medium**   | Must read memory for prior context; write design decisions + component map back to memory; read memory on revision loop to see critical review context                                        | Medium — revision loop memory interaction adds complexity                        |
| **Critical Thinker**        | `critical-thinker.agent.md` (139 lines)    | **Medium**   | Must read memory for design context summary; write risk findings summary to memory                                                                                                            | Low — primarily a consumer of memory                                             |
| **Planner**                 | `planner.agent.md` (244 lines)             | **High**     | Must read memory for accumulated context; write task index + execution wave summary to memory; in replan mode, read memory for failure context + prior plan state                             | Medium — replan mode memory interaction needs careful design                     |
| **Implementer**             | `implementer.agent.md` (198 lines)         | **Medium**   | Must read memory for relevant context (design decisions, related tasks' status); write implementation notes + issues back to memory                                                           | Medium — must not read too much memory (distraction risk); strict scoping needed |
| **Verifier**                | `verifier.agent.md` (179 lines)            | **Medium**   | Must read memory for expected implementation state; write verification results summary to memory for replan loop efficiency                                                                   | Low — mostly a memory consumer with targeted writes                              |
| **Reviewer**                | `reviewer.agent.md` (220 lines)            | **High**     | Must read memory for full feature context; write review findings + architectural decisions to memory; knowledge evolution writes (instructions, patterns) to memory                           | High — most complex memory writes; knowledge evolution adds new write targets    |
| **Documentation Writer**    | `documentation-writer.agent.md` (95 lines) | **Low**      | Must read memory for context; minimal writes                                                                                                                                                  | Low — least affected agent                                                       |
| **Feature Workflow Prompt** | `feature-workflow.prompt.md` (30 lines)    | **Low**      | May need memory config variable (e.g., `{{MEMORY_MODE}}`)                                                                                                                                     | Low — minimal change                                                             |

#### Memory System Risks

1. **Stale Memory Risk (High):** If an agent writes to memory but a later agent's output invalidates that memory (e.g., designer revises after critical review), stale entries could mislead downstream agents. The replan loop (up to 3 iterations in verification) is particularly vulnerable — each iteration changes what implementers produced, but memory may retain outdated context.
   - _Affected flow:_ Steps 3b (design revision loop), Step 6 (verification replan loop), Step 7 (review fix loop)
   - _Current safeguard:_ None — artifacts are always re-read from disk currently
2. **Memory Size Growth (Medium):** As features grow complex, memory accumulates entries from all agents. Without pruning, memory could grow large enough to consume significant context window space, defeating the purpose of reducing artifact reads.
   - _Affected agents:_ All agents that read memory at the start of their workflow
3. **Write Conflict Risk (Medium):** If parallel agents (research ×3/4, implementation waves) write to memory concurrently, entries may be lost or interleaved incorrectly. The current system avoids this because artifacts are separate files per agent.
   - _Affected flows:_ Step 1.1 (parallel research), Step 5 (parallel implementation waves)

4. **Memory-Artifact Drift (High):** If memory says "design uses pattern X" but the designer revised to "pattern Y" during the critical review loop, downstream agents reading memory first may proceed with outdated assumptions before checking the artifact.
   - _Affected agents:_ Planner (reads memory for design decisions), Implementer (reads memory for design context)

5. **Single Point of Failure (Medium):** If memory becomes corrupted or malformed, all downstream agents are affected since they read memory first. Currently, each artifact is independent — a corrupted `design.md` doesn't affect `analysis.md`.

#### Memory Read/Write Responsibility Matrix

| Agent                  | Reads From Memory                                        | Writes To Memory                                            |
| ---------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| Orchestrator           | Phase status, error history                              | Phase transitions, agent dispatch records, error logs       |
| Researcher (focused)   | Artifact index, prior decisions                          | Focus-area summary, key file references                     |
| Researcher (synthesis) | All research summaries                                   | Synthesized overview, key findings                          |
| Spec                   | Analysis summary, prior decisions                        | Requirement summary, acceptance criteria overview           |
| Designer               | Spec summary, analysis overview, critical review context | Design decisions, component map, API surface                |
| Critical Thinker       | Design summary, spec requirements                        | Risk findings, key assumptions challenged                   |
| Planner                | All prior summaries, verification failures (replan)      | Task index, wave structure, pre-mortem summary              |
| Implementer            | Design decisions, task context, related task status      | Implementation notes, issues found, test results            |
| Verifier               | Expected state, task summaries                           | Verification results, failure details                       |
| Reviewer               | Full feature context, implementation notes               | Review findings, architectural decisions, knowledge updates |
| Documentation Writer   | Feature context, design summary                          | Documentation status                                        |

---

### 2. Critical Thinking Parallelization Impact

The initial request calls for splitting the single critical-thinker into **4 parallel sub-agents** with aggregation. Currently, the critical thinker (critical-thinker.agent.md) operates at Step 3b as a single sequential agent reviewing the entire design across all risk categories.

#### Current Critical Thinker Scope (Lines 63-94 of critical-thinker.agent.md)

The current agent covers 6 formal risk categories plus 4 open-ended areas:

1. Security vulnerabilities
2. Scalability bottlenecks
3. Maintainability concerns
4. Backwards compatibility risks
5. Edge cases not covered
6. Performance implications
7. Strategic risks (open-ended)
8. Scope risks (open-ended)
9. Complexity risks (open-ended)
10. Integration risks (open-ended)

#### Proposed Split Into 4 Sub-Agents

A natural partitioning:

| Sub-Agent          | Focus                          | Current Categories Covered                                              |
| ------------------ | ------------------------------ | ----------------------------------------------------------------------- |
| CT-Security        | Security & compliance          | Security vulnerabilities, backwards compatibility                       |
| CT-Scalability     | Performance & scale            | Scalability bottlenecks, performance implications                       |
| CT-Maintainability | Code quality & architecture    | Maintainability concerns, complexity risks, integration risks           |
| CT-Strategy        | Scope, direction, fundamentals | Strategic risks, scope risks, edge cases, "is this the right approach?" |

#### Impact on Design Review Flow

**Current flow (orchestrator.agent.md, Step 3b, lines 145-156):**

```
Designer → design.md → Critical Thinker → design_critical_review.md
    ↑                                         │
    └─── NEEDS_REVISION (max 1 loop) ─────────┘
```

**Proposed flow:**

```
Designer → design.md → [CT-Security ──┐
                         CT-Scale ─────┤── parallel (max 4) → Aggregator → design_critical_review.md
                         CT-Maintain ──┤
                         CT-Strategy ──┘]
    ↑                                                                         │
    └────────── NEEDS_REVISION (max 1 loop) ──────────────────────────────────┘
```

#### New Files Required

| File                                        | Purpose                                                                     |
| ------------------------------------------- | --------------------------------------------------------------------------- |
| `critical-thinker-security.agent.md`        | Sub-agent for security-focused design review                                |
| `critical-thinker-scalability.agent.md`     | Sub-agent for scalability/performance review                                |
| `critical-thinker-maintainability.agent.md` | Sub-agent for code quality/architecture review                              |
| `critical-thinker-strategy.agent.md`        | Sub-agent for strategic/scope review                                        |
| `critical-thinker-aggregator.agent.md`      | Aggregator that merges 4 sub-agent outputs into `design_critical_review.md` |

The original `critical-thinker.agent.md` would either be replaced or retained as the aggregator.

#### Impact Assessment

1. **Conflicting Findings Risk (High):** Two sub-agents may reach contradictory conclusions. Example: CT-Security might flag "add input validation on all endpoints" while CT-Scalability flags "avoid per-request validation overhead for high-throughput endpoints." The aggregator must have conflict resolution logic.
   - _Current safeguard:_ Single agent makes internally consistent tradeoffs
   - _Mitigation needed:_ Aggregator must explicitly surface conflicts and let the designer resolve them

2. **Coverage Gap Risk (Medium):** Splitting risk categories may create blind spots at the boundaries. A risk that spans both security and maintainability might be missed if each sub-agent assumes the other will catch it.
   - _Mitigation needed:_ Overlapping scope at boundaries with deduplication in the aggregator

3. **Aggregation Bottleneck Risk (Medium):** If the aggregator itself takes significant time to merge 4 outputs, the parallelization gain is reduced. The aggregator must be lightweight — merge and flag conflicts, not re-analyze.
   - _Current analogy:_ Research synthesis (Step 1.2) takes non-trivial time to merge 3 research outputs

4. **NEEDS_REVISION Loop Complexity (Medium):** If the aggregated review returns NEEDS_REVISION, the designer must address findings from 4 different sub-agents simultaneously. If the designer's revision fixes one sub-agent's concern but breaks another's, the single retry loop (max 1 per orchestrator.agent.md line 153) may not be sufficient.
   - _Current safeguard:_ Single critical thinker produces internally coherent recommendations
   - _Impact:_ The max-1 loop constraint may need re-evaluation

5. **Output Format Coordination (Low):** All 4 sub-agents must produce outputs with compatible structure so the aggregator can merge them. Requires a shared partial output schema.

#### Affected Orchestrator Sections

- Step 3b (lines 145-156): Must be rewritten to dispatch 4 sub-agents in parallel, wait for all 4, then invoke aggregator
- NEEDS_REVISION routing table (lines 253-261): Must be updated to route from aggregator (not single critical thinker) back to designer
- Expectations table (lines 267-288): Must add rows for CT sub-agents and CT aggregator

---

### 3. Verification Cluster Impact

The initial request calls for replacing the single verifier with a **parallel Verification Cluster of 4 sub-agents** with aggregation. Currently, the verifier (verifier.agent.md) operates at Step 6 as a single sequential agent performing 6 workflow steps.

#### Current Verifier Workflow (Lines 66-115 of verifier.agent.md)

1. Detect build system
2. Build the project
3. Run full test suite
4. Per-task verification (acceptance criteria)
5. Feature-level verification
6. Report generation

#### Proposed Split Into 4 Sub-Agents

A natural partitioning:

| Sub-Agent | Focus                                           | Current Steps Covered | Parallelizable?                                             |
| --------- | ----------------------------------------------- | --------------------- | ----------------------------------------------------------- |
| V-Build   | Build system detection + build + compile errors | Steps 1-2             | **No** — must run first (others depend on successful build) |
| V-Tests   | Full test suite execution + result analysis     | Step 3                | Yes — after build passes                                    |
| V-Tasks   | Per-task acceptance criteria verification       | Step 4                | Yes — after build passes                                    |
| V-Feature | Feature-level acceptance criteria verification  | Step 5                | Yes — after build passes                                    |

**Critical Dependency:** V-Build must complete successfully before V-Tests, V-Tasks, and V-Feature can run. This means the cluster is **not fully parallel** — it's a sequential build step followed by 3 parallel verification steps.

```
V-Build → [V-Tests ──┐
            V-Tasks ──┤── parallel (max 3) → Aggregator → verifier.md
            V-Feature─┘]
```

#### Impact on Replan Loop

**Current replan flow (orchestrator.agent.md, Step 6, lines 185-197):**

```
Verifier → NEEDS_REVISION → Planner (replan) → Implementer(s) → Verifier (re-verify) → ... (max 3 loops)
```

**Proposed replan flow with cluster:**

```
V-Build → [V-Tests, V-Tasks, V-Feature] → Aggregator → NEEDS_REVISION → Planner → Implementer(s)
→ V-Build → [V-Tests, V-Tasks, V-Feature] → Aggregator → ... (max 3 loops)
```

#### New Files Required

| File                           | Purpose                                                     |
| ------------------------------ | ----------------------------------------------------------- |
| `verifier-build.agent.md`      | Sub-agent for build system detection and compilation        |
| `verifier-tests.agent.md`      | Sub-agent for test suite execution and analysis             |
| `verifier-tasks.agent.md`      | Sub-agent for per-task acceptance criteria verification     |
| `verifier-feature.agent.md`    | Sub-agent for feature-level verification                    |
| `verifier-aggregator.agent.md` | Aggregator that merges sub-agent outputs into `verifier.md` |

#### Impact Assessment

1. **Sequential Build Dependency (High):** Build must succeed before other sub-agents run. If build fails, the 3 parallel sub-agents cannot be dispatched, and the cluster degenerates to a single sequential agent (V-Build). This limits the speedup to cases where the build passes.
   - _Implication:_ In replan loops (where builds may initially fail), the cluster provides no speedup until the build is fixed

2. **Replan Loop Coordination (High):** The replan loop (max 3 iterations) now involves a cluster instead of a single agent. Each iteration requires: build → parallel verify → aggregate → decide. The orchestrator must coordinate this multi-step verification process within each replan iteration.
   - _Current orchestrator complexity:_ Step 6 is a simple invoke-and-check
   - _Proposed complexity:_ Step 6 becomes a mini-pipeline (build → parallel dispatch → aggregate → check)

3. **Targeted Re-verification Challenge (Medium):** The current verifier supports targeted re-verification (verifier.agent.md, lines 145-152): "Focus on previously failing areas first." With a cluster, "previously failing areas" may span multiple sub-agents. The aggregator must map previous failures to the correct sub-agents for targeted re-runs.
   - _Example:_ If V-Tests found 3 test failures and V-Tasks found 2 acceptance criteria gaps, the replan fix should be checked by both V-Tests and V-Tasks but not necessarily V-Feature

4. **Build Environment State (Medium):** If V-Build modifies the environment (e.g., installs dependencies, generates code), the 3 parallel sub-agents must all see the same post-build state. If run in separate terminal sessions, environment state may not be shared.
   - _Current safeguard:_ Single verifier runs all steps in sequence, so build state persists

5. **Aggregation Must Produce Actionable Items (Medium):** The current `verifier.md` contains "Actionable Items mapped to specific failing task IDs" (verifier.agent.md, line 162). The aggregator must merge actionable items from 3 sub-agents into a coherent list without duplicating or conflicting task assignments.

#### Affected Orchestrator Sections

- Step 6 (lines 185-197): Must be rewritten to dispatch build → await → dispatch parallel verify ×3 → await → aggregate → check contract
- NEEDS_REVISION routing (line 255): Must route from aggregator to planner
- Error handling: The retry loop logic (max 3 iterations) must account for the cluster's internal failure modes

---

### 4. Review Cluster + Knowledge Evolution Impact

The initial request calls for splitting the reviewer into **4 parallel sub-agents** that include **Knowledge Evolution** — updating Copilot instructions, skills, and capturing reusable workflow patterns.

#### Current Reviewer Scope (reviewer.agent.md)

The reviewer currently handles:

1. Peer-style code review (quality, correctness, readability)
2. Security review (secrets/PII scan + OWASP for Full tier)
3. Test quality and coverage assessment
4. Architectural alignment check
5. Convention compliance
6. `decisions.md` updates (append-only architectural decision log)
7. `.github/instructions/*.instructions.md` updates

#### Proposed Split Into 4 Sub-Agents

| Sub-Agent   | Focus                                                           | Current Responsibilities Covered                      |
| ----------- | --------------------------------------------------------------- | ----------------------------------------------------- |
| R-Quality   | Code quality, readability, maintainability, naming, conventions | #1, #4, #5                                            |
| R-Security  | Security scanning, OWASP, secrets/PII detection                 | #2                                                    |
| R-Testing   | Test quality, coverage, test adequacy                           | #3                                                    |
| R-Knowledge | Knowledge evolution — instructions, skills, patterns, decisions | #6, #7, plus new knowledge evolution responsibilities |

#### Knowledge Evolution (New Capability)

The initial request defines Knowledge Evolution responsibilities for R-Knowledge:

1. Suggest improvements/additions/removals to Copilot instructions
2. Suggest improvements/additions/removals to skills
3. Capture reusable workflow patterns
4. Write safe, reviewable updates

This is **new functionality** — the current reviewer only appends to `decisions.md` and updates `.github/instructions`. Knowledge Evolution expands this to:

- Modifying existing agent definitions based on learned patterns
- Capturing reusable patterns for future features
- Actively improving the Forge system itself

#### Impact on Completion Contract

**Current completion contract (reviewer.agent.md, lines 163-169):**

- `DONE:` → workflow complete
- `NEEDS_REVISION:` → route findings to affected implementer(s) (max 1 loop)
- `ERROR:` → escalate to planner for full replan

**With a review cluster, the aggregator must produce a single completion contract from 4 sub-agent results:**

| Scenario          | Sub-Agent Results                                    | Aggregated Result                      |
| ----------------- | ---------------------------------------------------- | -------------------------------------- |
| All pass          | 4× DONE                                              | DONE                                   |
| Quality issues    | R-Quality: NEEDS_REVISION, others: DONE              | NEEDS_REVISION (route to implementers) |
| Security critical | R-Security: ERROR                                    | ERROR (override other results)         |
| Mixed             | R-Quality: NEEDS_REVISION, R-Testing: NEEDS_REVISION | NEEDS_REVISION (merge findings)        |
| Knowledge only    | R-Knowledge: DONE with suggestions, others: DONE     | DONE + knowledge updates applied       |

#### Conflicting Review Findings

1. **Quality vs. Performance Conflict (Medium):** R-Quality may flag a pattern as "overly complex, simplify" while the original design chose that pattern for performance. Without cross-referencing `design.md` tradeoffs, R-Quality may produce findings that conflict with design intent.
2. **Security vs. Usability Conflict (Medium):** R-Security may flag "add rate limiting on all endpoints" while R-Quality notes "rate limiting middleware adds unnecessary complexity for internal endpoints." The aggregator must surface these as explicit tradeoffs.

3. **Knowledge Evolution Safety Risk (High):** R-Knowledge is proposed to modify Copilot instructions and agent definitions — this means it could potentially modify the very instructions that govern the Forge pipeline itself. A self-modifying system has inherent instability risks.
   - _Safeguard needed:_ Knowledge evolution updates must be "suggested" not "applied," requiring human review
   - _The initial request acknowledges this:_ "Write safe, reviewable updates"

#### New Files Required

| File                           | Purpose                                                            |
| ------------------------------ | ------------------------------------------------------------------ |
| `reviewer-quality.agent.md`    | Sub-agent for code quality, readability, conventions               |
| `reviewer-security.agent.md`   | Sub-agent for security review (OWASP, secrets, PII)                |
| `reviewer-testing.agent.md`    | Sub-agent for test quality and coverage                            |
| `reviewer-knowledge.agent.md`  | Sub-agent for knowledge evolution (instructions, skills, patterns) |
| `reviewer-aggregator.agent.md` | Aggregator that merges reviews into `review.md`                    |

#### Affected Orchestrator Sections

- Step 7 (lines 199-215): Must be rewritten to dispatch 4 review sub-agents in parallel → aggregate → check contract
- NEEDS_REVISION routing table (lines 253-261): Must route from aggregator to implementers
- Expectations table (lines 267-288): Must add rows for review sub-agents and aggregator
- New output paths: Knowledge evolution outputs (suggested instruction updates, pattern libraries) need new file locations

#### Impact on `decisions.md` Lifecycle

Currently, the reviewer creates and appends to `decisions.md` (reviewer.agent.md, lines 73-91). With the cluster, either:

- R-Knowledge owns `decisions.md` (breaking its current association with the general reviewer)
- R-Quality identifies decisions and R-Knowledge writes them (introducing a dependency between sub-agents that undermines parallelism)
- The aggregator handles `decisions.md` writes (adding responsibility to the aggregator)

---

### 5. Orchestrator Impact

The orchestrator (`orchestrator.agent.md`, 323 lines) will require the **most extensive changes** of any agent. It is the single point of control (architecture.md, Section 7.7) and must coordinate all new subsystems.

#### Change Inventory for Orchestrator

| Section                    | Current Location | Change Type                                                                                      | Estimated Impact         |
| -------------------------- | ---------------- | ------------------------------------------------------------------------------------------------ | ------------------------ |
| Step 0 (Setup)             | Lines 72-81      | **New:** Initialize memory system                                                                | +10-15 lines             |
| Step 1.1 (Research)        | Lines 93-108     | **Modify:** Dispatch 4 researchers instead of 3; pass memory reference                           | ~5 lines changed         |
| Step 1.2 (Synthesis)       | Lines 110-119    | **Minor:** Pass memory reference                                                                 | ~2 lines changed         |
| Step 3b (Critical Review)  | Lines 145-156    | **Major rewrite:** Dispatch 4 CT sub-agents → aggregate → check contract                         | ~30-40 lines new/changed |
| Step 5 (Implementation)    | Lines 158-190    | **Modify:** Pass memory reference; update memory after each wave                                 | ~10-15 lines changed     |
| Step 6 (Verification)      | Lines 192-213    | **Major rewrite:** Dispatch build → parallel verify ×3 → aggregate → replan loop coordination    | ~40-50 lines new/changed |
| Step 7 (Review)            | Lines 215-233    | **Major rewrite:** Dispatch 4 review sub-agents → aggregate → handle knowledge evolution outputs | ~30-40 lines new/changed |
| Global Rules               | Lines 27-41      | **Modify:** Add memory management rules, increase awareness of sub-agent clusters                | ~10-15 lines             |
| NEEDS_REVISION Table       | Lines 253-261    | **Rewrite:** Add rows for all aggregators; update routing                                        | ~15-20 lines             |
| Expectations Table         | Lines 267-288    | **Rewrite:** Add rows for all sub-agents + aggregators                                           | ~30-40 lines             |
| Memory Updates             | N/A (new)        | **New:** After-step memory write logic for each pipeline stage                                   | ~20-30 lines             |
| Parallel Execution Summary | Lines 291-308    | **Rewrite:** Update flow diagram to show clusters                                                | ~15-20 lines             |

#### Total Estimated Orchestrator Change

- **Current size:** 323 lines
- **Estimated additions:** ~200-250 lines
- **Estimated modifications:** ~50-70 lines
- **Estimated final size:** ~470-530 lines
- **Change ratio:** ~60% of the current content needs modification or addition

#### Complexity Assessment

1. **Cluster Coordination Logic (Critical):** The orchestrator must manage three new cluster patterns (CT cluster, V cluster, R cluster), each with different internal dependencies:
   - CT cluster: fully parallel → aggregate
   - V cluster: sequential build → parallel verify ×3 → aggregate
   - R cluster: fully parallel → aggregate + knowledge evolution

2. **Memory Management (High):** The orchestrator must initialize, update, and pass memory references at every step. Memory updates after each step must be carefully ordered to avoid stale reads.

3. **Error Handling Complexity (High):** Sub-agent clusters introduce new failure modes:
   - What if 2 of 4 CT sub-agents return ERROR? (Partial cluster failure)
   - What if V-Build succeeds but V-Tests returns ERROR? (Mixed status in replan loop)
   - What if R-Knowledge crashes but R-Quality/R-Security succeed? (Knowledge evolution is non-blocking but quality/security are critical)

4. **Orchestrator Context Window Risk (Medium):** At ~500+ lines, the orchestrator definition is large for an agent prompt. This increases the risk of the LLM losing coherence on later sections (in-context learning degradation).

---

### 6. Performance Analysis

#### Theoretical Speedup from Parallelization

**Critical Thinking (Step 3b):**

- Current: 1 agent × T_ct time = T_ct
- Proposed: 4 agents × (T_ct/4) parallel + T_aggregate = T_ct/4 + T_agg
- Theoretical speedup: ~3-3.5× (assuming aggregation is ~15-25% of single-agent time)
- Practical speedup: ~2-2.5× (sub-agents may not divide evenly; some risk categories are heavier)

**Verification (Step 6):**

- Current: 1 agent × T_verify time = T_verify
- Proposed: T_build + max(T_tests, T_tasks, T_feature) + T_aggregate
- Theoretical speedup: ~2-2.5× (build is sequential and often the longest step; only 3 steps parallelize)
- Practical speedup: ~1.5-2× (build time dominates for large projects; small projects may see no gain)
- Worst case (build fails): 1× (no speedup — only V-Build runs)

**Review (Step 7):**

- Current: 1 agent × T_review time = T_review
- Proposed: max(T_quality, T_security, T_testing, T_knowledge) + T_aggregate
- Theoretical speedup: ~3-3.5× (all 4 run in parallel; aggregation is lightweight)
- Practical speedup: ~2.5-3× (security scan has more variable runtime; quality review is typically longest)

**Research Phase (Step 1.1):**

- Current: 3 parallel agents → synthesis
- Proposed: 4 parallel agents → synthesis
- Speedup: Marginal (~0-10% improvement in overall research time; 4th agent adds coverage, not speed, since 3 already run in parallel)

#### End-to-End Pipeline Impact

```
Current Pipeline (rough relative timing):
Research(3∥) → Synth → Spec → Design → CritReview(1×) → Plan → Impl(waves) → Verify(1×) → Review(1×)
  [10%]        [5%]   [5%]   [8%]      [12%]            [10%]    [30%]         [10%]        [10%]

Proposed Pipeline:
Research(4∥) → Synth → Spec → Design → CT-Cluster(4∥) → Plan → Impl(waves) → V-Cluster(1+3∥) → R-Cluster(4∥)
  [10%]        [5%]   [5%]   [8%]      [5% + 2% agg]     [10%]    [30%]        [6% + 2% agg]     [4% + 2% agg]

Estimated time reduction for critical path: ~15-20% overall pipeline speedup
```

**Key insight:** The sequential middle pipeline (Spec → Design → Planning) remains unchanged and represents ~23% of total pipeline time. This creates a floor on maximum possible speedup. Implementation waves (~30%) are already parallelized. The parallelizable stages (CT, V, R) represent ~32% of total time — even perfect parallelization of these stages yields at most ~25% end-to-end improvement.

#### Bottleneck Analysis

1. **Sequential Middle Pipeline (Unchanged):** Spec → Design → Planning cannot be parallelized because each depends on the previous stage's output. This remains the largest sequential bottleneck.

2. **Aggregation Overhead:** Each cluster adds an aggregation step. Three clusters × T_aggregate adds ~5-8% overhead that partially offsets parallelization gains.

3. **Concurrency Cap (Max 4):** The existing max-4 concurrency cap (orchestrator.agent.md, line 34) already accommodates the proposed clusters (each has ≤4 sub-agents). However, the implementation waves already use the full cap (4 concurrent tasks per sub-wave). Clusters and implementation waves don't overlap temporally, so there's no conflict.

4. **Memory Read Overhead:** Every agent now reads memory before starting. If memory is stored as a file, this adds a file read to every agent invocation. For a pipeline with 20+ agent invocations (including all sub-agents), this adds 20+ extra file reads — potentially negating some of the speedup from reduced artifact reads.

---

### 7. Risk Assessment

#### Risk 1: Race Conditions in Memory Writes

- **What:** Parallel agents (research ×4, implementation waves, CT cluster, V cluster, R cluster) may write to memory concurrently, causing lost writes or corrupted memory state.
- **Likelihood:** High (any parallel dispatch creates this possibility)
- **Impact:** Medium-High (corrupted memory misleads downstream agents, but artifacts remain correct as source of truth)
- **Mitigation:** Write-locking mechanism OR separate memory sections per parallel agent group OR deferred writes (only write to memory after parallel group completes, via aggregator)
- **Residual risk:** Deferred writes mean parallel agents can't benefit from each other's memory writes during execution

#### Risk 2: Aggregation Quality Degradation

- **What:** Aggregator agents may lose important nuances when merging 4 sub-agent outputs. Subtle findings that depend on cross-category reasoning may be flattened or dropped.
- **Likelihood:** Medium (aggregation is a lossy process)
- **Impact:** High (missed critical finding could lead to design/implementation issues)
- **Mitigation:** Aggregators should flag conflicts rather than resolve them; include all sub-agent outputs in full as appendices; implement a "high-severity passthrough" rule where any finding rated Critical is always included verbatim
- **Residual risk:** Aggregator context window may be insufficient to process 4 full sub-agent outputs

#### Risk 3: Quality Degradation from Agent Splitting

- **What:** A single holistic agent can make connections across risk categories (e.g., "this security choice has scalability implications"). Split agents lose this cross-cutting insight.
- **Likelihood:** Medium-High (well-documented phenomenon in task decomposition)
- **Impact:** Medium (some cross-cutting insights are lost, but most risks are category-specific)
- **Mitigation:** Allow small scope overlap between sub-agents; aggregator performs basic cross-referencing; add a "cross-cutting concerns" prompt to each sub-agent
- **Residual risk:** Fundamentally, parallelization trades depth of holistic reasoning for breadth of focused analysis

#### Risk 4: Orchestrator Complexity Explosion

- **What:** The orchestrator grows from 323 lines to ~500+ lines with cluster coordination, memory management, and expanded routing tables. This increases the chance of LLM coherence loss.
- **Likelihood:** Medium (LLMs degrade on very long prompts)
- **Impact:** High (orchestrator errors cascade to the entire pipeline)
- **Mitigation:** Extract cluster coordination into reusable patterns; consider sub-orchestrators for cluster management; keep orchestrator focused on high-level flow
- **Residual risk:** Even with patterns, the orchestrator remains the most complex and longest agent definition

#### Risk 5: Knowledge Evolution Self-Modification Risk

- **What:** The R-Knowledge sub-agent can suggest modifications to Copilot instructions and agent definitions — potentially modifying the rules that govern the Forge pipeline itself.
- **Likelihood:** Medium (intended behavior, but dangerous if not constrained)
- **Impact:** Critical (self-modifying agent instructions could break the entire pipeline)
- **Mitigation:** Knowledge evolution writes must be suggestions only, stored in a separate review buffer; human approval required before applying; versioning of all agent definitions; rollback mechanism
- **Residual risk:** If the knowledge system is misconfigured to auto-apply, damage could be irreversible within a run

#### Risk 6: Verification Cluster Build State Sharing

- **What:** V-Build runs and modifies environment state. V-Tests, V-Tasks, and V-Feature run in parallel and must all see the post-build state. If they run in separate shells/contexts, they may not inherit the build artifacts.
- **Likelihood:** Medium (depends on runtime environment)
- **Impact:** High (verification may fail spuriously if sub-agents can't find build output)
- **Mitigation:** V-Build must produce build artifacts on disk (not just in-memory); verify that all sub-agents can access the same filesystem; provide the build output path to all sub-agents
- **Residual risk:** Platform-dependent behavior of `runSubagent` terminal isolation

#### Risk 7: Replan Loop Complexity with Clusters

- **What:** The verification replan loop (max 3 iterations) becomes significantly more complex with a cluster. Each iteration involves: build → parallel verify ×3 → aggregate → planner → implementer(s) → build → ... The orchestrator must track which sub-agents reported failures and map fixes to specific sub-agents for targeted re-verification.
- **Likelihood:** Medium (replan loops are triggered by failures, which are common)
- **Impact:** Medium (increased complexity may cause orchestrator errors in the replan loop)
- **Mitigation:** Simplify re-verification: always re-run the full cluster on replan (not targeted sub-agents); accept the slight performance penalty for simplicity
- **Residual risk:** Full re-runs waste time when only one sub-agent's concerns were addressed

#### Risk 8: Migration Breaks Existing Working Pipelines

- **What:** The current pipeline works and has been validated (per the v2 agent-improvements feature). The upgrade modifies all 11 files simultaneously. If any modification introduces a bug, the entire pipeline may fail.
- **Likelihood:** Medium (large-scale refactor of working system)
- **Impact:** High (complete pipeline failure)
- **Mitigation:** Phased migration (see Migration Complexity section); parallel pipeline existence during transition; comprehensive testing of each phase before proceeding
- **Residual risk:** Even phased migration may have integration issues between phases

---

### 8. Migration Complexity

#### Overall Assessment: **High Complexity**

The upgrade touches all 11 files, introduces 15 new agent definition files (5 per cluster × 3 clusters), adds a memory system that crosscuts every agent, and fundamentally changes the orchestrator's coordination logic.

#### Recommended Migration Phases

| Phase                                             | Scope                                                                  | Agents Affected                                                                  | Risk Level                        | Dependencies                          |
| ------------------------------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------- | --------------------------------- | ------------------------------------- |
| **Phase 1: Memory Foundation**                    | Implement memory file structure and basic read/write operations        | orchestrator.agent.md (memory init), all agents (add memory read/write sections) | High (cross-cutting)              | None — foundational change            |
| **Phase 2: Research Expansion**                   | Add 4th research focus area                                            | orchestrator.agent.md (dispatch 4), researcher.agent.md (add 4th focus)          | Low                               | Phase 1 (memory integration optional) |
| **Phase 3: Critical Thinking Cluster**            | Split critical thinker into 4 sub-agents + aggregator                  | critical-thinker.agent.md (split), orchestrator.agent.md (Step 3b rewrite)       | Medium                            | Phase 1 desired                       |
| **Phase 4: Verification Cluster**                 | Split verifier into 4 sub-agents + aggregator                          | verifier.agent.md (split), orchestrator.agent.md (Step 6 rewrite)                | High (replan loop complexity)     | Phase 1 desired                       |
| **Phase 5: Review Cluster + Knowledge Evolution** | Split reviewer into 4 sub-agents + aggregator with knowledge evolution | reviewer.agent.md (split), orchestrator.agent.md (Step 7 rewrite)                | High (knowledge evolution safety) | Phase 1 desired                       |
| **Phase 6: Integration & Optimization**           | End-to-end testing, performance tuning, memory pruning                 | All agents                                                                       | Medium                            | Phases 1-5 complete                   |

#### Phase Dependencies

```
Phase 1 (Memory) ──→ Phase 2 (Research) ──→ Phase 6 (Integration)
       │                                          ↑
       ├──→ Phase 3 (CT Cluster) ─────────────────┤
       ├──→ Phase 4 (V Cluster) ──────────────────┤
       └──→ Phase 5 (R Cluster + Knowledge) ──────┘
```

Phases 3, 4, and 5 can be developed in parallel (they're independent clusters), but all depend on Phase 1 (memory) being complete for full integration. Phase 2 is independent and low-risk.

#### File Change Summary

| Change Type               | Files | Details                                                                            |
| ------------------------- | ----- | ---------------------------------------------------------------------------------- |
| **Major rewrite**         | 1     | `orchestrator.agent.md` (~60% rewrite)                                             |
| **Moderate modification** | 10    | All other agent files (add memory read/write sections, ~15-25% change each)        |
| **New files**             | 15    | 5 CT sub-agents/aggregator + 5 V sub-agents/aggregator + 5 R sub-agents/aggregator |
| **New file**              | 1     | Memory system definition/template (e.g., `memory.md` or `memory-schema.md`)        |
| **Total files affected**  | 27    | 11 modified + 16 new                                                               |

#### Migration Risk Factors

1. **All-or-Nothing Risk for Memory:** Memory integration touches every agent. If the memory system design is wrong, every agent must be updated again. This makes Phase 1 the highest-risk phase.

2. **Orchestrator as Bottleneck:** All cluster phases (3, 4, 5) modify the orchestrator. If developed in parallel, they'll create merge conflicts in the orchestrator definition. Development should sequence orchestrator changes carefully.

3. **No Automated Testing:** Forge is a declarative markdown system with no automated tests. Validation requires running the full pipeline on a real feature request. There's no way to unit-test individual agent changes.

4. **Backward Compatibility:** During migration, the pipeline should continue to work with the existing (non-cluster) agents. This means maintaining backward compatibility in the orchestrator — it should gracefully handle both single-agent and cluster modes.

---

## Key Observations

1. **The orchestrator is the critical path of the migration.** Every proposed change (memory, clusters, knowledge evolution) requires orchestrator modifications. The orchestrator's current 323-line definition is already at the edge of manageable complexity for an LLM agent prompt. Growing it to 500+ lines risks coherence degradation. Consider splitting orchestrator responsibilities (e.g., a sub-orchestrator for each cluster).

2. **Memory is the highest-risk, highest-reward change.** It touches every agent and has no precedent in the current system. If done well, it significantly reduces context window waste from redundant artifact reads. If done poorly, it introduces a single point of failure that can corrupt downstream reasoning. The initial request's constraint — "keeps artifacts as the single source of truth" — is critical and must be strictly enforced.

3. **Parallelization gains are bounded by the sequential middle pipeline.** Spec → Design → Planning cannot be parallelized and represents ~23% of pipeline time. The theoretical maximum end-to-end speedup from parallelizing CT, V, and R is ~20-25%, not 3-4× as the cluster naming might suggest.

4. **Aggregation is the hidden cost of parallelization.** Each cluster needs a dedicated aggregator, adding 3 new sequential steps to the pipeline. If aggregators are poorly designed (too complex, trying to resolve conflicts rather than surface them), they become new bottlenecks.

5. **Knowledge Evolution is the most architecturally novel and risky component.** It introduces self-modification into a system designed for deterministic, repeatable execution. Strict guardrails (suggestion-only, human review, versioning) are essential. It should be the last component implemented and the most carefully designed.

6. **The verification cluster's build dependency limits its parallelism.** Unlike the CT and R clusters (fully parallel), the V cluster has a hard sequential dependency (build must succeed first). The effective parallelism is 1+3, not 4, and in build-failure scenarios, it's 1 (no gain).

7. **Agent count grows from 10 to 25.** The system goes from 10 agent definitions + 1 prompt to 25 agent definitions + 1 prompt. This increases the maintenance surface area by 2.5×. Consistent templating (per the existing agent definition pattern in architecture.md, Section 1) is essential for manageability.

8. **Conflicting findings are the primary quality risk of parallelization.** In all three clusters, parallel sub-agents may produce contradictory results. The aggregator's conflict resolution strategy is critical to maintaining the quality level of the current single-agent approach.

9. **The 4th research focus area adds coverage but not speed.** Adding a 4th researcher (currently 3) runs in parallel with the existing 3 — it adds depth of analysis without reducing wall-clock time (all 4 finish when the slowest finishes). The value is analytical completeness, not performance.

10. **No `.github` directory exists yet.** The reviewer references `.github/instructions/*.instructions.md` as an output path, but this directory doesn't exist (per architecture.md, Section 7.8). Knowledge Evolution (R-Knowledge) would need to establish this directory structure as part of its initial run.

---

## File References

| File                                                               | Relevance to Impact Analysis                                                                                                       |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| `NewAgentsAndPrompts/orchestrator.agent.md`                        | Most impacted file; all cluster + memory changes require orchestrator modifications (Steps 3b, 6, 7, routing tables, expectations) |
| `NewAgentsAndPrompts/critical-thinker.agent.md`                    | Split candidate; risk categories define sub-agent boundaries; NEEDS_REVISION contract affects design loop                          |
| `NewAgentsAndPrompts/verifier.agent.md`                            | Split candidate; sequential workflow steps define sub-agent boundaries; replan loop creates cluster coordination complexity        |
| `NewAgentsAndPrompts/reviewer.agent.md`                            | Split candidate + knowledge evolution host; tiered depth model, decisions.md lifecycle, .github/instructions updates               |
| `NewAgentsAndPrompts/researcher.agent.md`                          | Memory integration + 4th focus area; focused/synthesis modes both need memory                                                      |
| `NewAgentsAndPrompts/spec.agent.md`                                | Memory integration (medium impact)                                                                                                 |
| `NewAgentsAndPrompts/designer.agent.md`                            | Memory integration (medium impact); receives CT cluster output instead of single CT output                                         |
| `NewAgentsAndPrompts/planner.agent.md`                             | Memory integration (high impact); replan mode must work with cluster verification output                                           |
| `NewAgentsAndPrompts/implementer.agent.md`                         | Memory integration (medium impact); receives review cluster output instead of single reviewer output                               |
| `NewAgentsAndPrompts/documentation-writer.agent.md`                | Memory integration (low impact)                                                                                                    |
| `NewAgentsAndPrompts/feature-workflow.prompt.md`                   | May need memory config variable                                                                                                    |
| `docs/feature/forge-architecture-upgrade/initial-request.md`       | Defines all proposed changes and requirements                                                                                      |
| `docs/feature/forge-architecture-upgrade/research/architecture.md` | Provides structural context for impact analysis                                                                                    |
| `docs/comparison-forge-vs-gem-team.md`                             | Validates two-layer verification model that must be preserved                                                                      |
| `docs/optimization-from-gem-team.md`                               | Shows which optimizations were already adopted in v2; P2 items (memory, routing) now being promoted to core                        |

---

## Assumptions & Limitations

1. **Assumption:** The `runSubagent` mechanism in the VS Code Copilot runtime supports dispatching the increased number of sub-agents (from ~15 per pipeline run to ~35+ with clusters). This is platform-dependent and unverified.
2. **Assumption:** Parallel sub-agents within a cluster share filesystem state (critical for V-Build output being visible to V-Tests, V-Tasks, V-Feature).
3. **Assumption:** The memory system will be file-based (consistent with the markdown artifact pattern) rather than requiring external infrastructure.
4. **Assumption:** Sub-agent context windows are independent — each sub-agent gets a fresh context. This means splitting a single agent into 4 provides 4× the effective context budget.
5. **Limitation:** No runtime testing was performed. All impact assessments are based on static analysis of agent definitions and architectural reasoning.
6. **Limitation:** The actual performance of aggregation agents is unknown — aggregation quality and speed depend heavily on LLM capabilities with multi-document synthesis.
7. **Assumption:** The max-4 concurrency cap is sufficient for all proposed clusters (each cluster has ≤4 sub-agents).

---

## Open Questions

1. **Memory concurrency model:** How should parallel agents write to memory without conflicts? Options: write-locking, per-group memory sections, deferred writes via aggregator.
2. **Aggregator placement:** Should aggregators be standalone agents invoked by the orchestrator, or should aggregation logic be embedded in the orchestrator itself?
3. **Knowledge Evolution scope:** Should R-Knowledge be able to modify agent definitions in `NewAgentsAndPrompts/`, or only suggest changes to a separate review buffer?
4. **Orchestrator size management:** How to keep the orchestrator under ~350 lines despite the additional cluster coordination logic? Sub-orchestrators? Extracted patterns?
5. **Sub-agent naming convention:** What naming convention for the 15 new sub-agent files? Prefix-based (e.g., `ct-security.agent.md`) or directory-based (e.g., `agents/critical-thinking/security.agent.md`)?
6. **Build state sharing:** How does the V-Build sub-agent ensure its build artifacts are visible to the 3 parallel verification sub-agents?
7. **Backward compatibility during migration:** Should the orchestrator support both single-agent and cluster modes during the transition, or is a big-bang switch acceptable?
8. **4th research focus area:** What should the 4th research focus area be? (The initial request says "should use 4 parallel agents" but current areas are architecture, impact, dependencies.)

---

## Research Metadata

- **confidence_level:** high — All 11 agent/prompt files were read in full; architecture research was incorporated; supporting documentation (comparison, optimization docs) was examined for context.
- **coverage_estimate:** Complete coverage of all agent definitions and their interaction patterns. Impact analysis covers all 8 proposed changes (memory, 3 clusters, research expansion, knowledge evolution, orchestrator changes, performance). Migration analysis includes phasing and risk assessment.
- **gaps:** No runtime testing or validation was performed. The actual behavior of `runSubagent` with increased parallelism is unknown. Aggregation agent quality is speculative — no precedent exists for aggregation agents in this system. The specific design of the memory file structure was not assessed (deferred to design phase).
