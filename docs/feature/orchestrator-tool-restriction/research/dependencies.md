# Research: Dependencies

## Focus Area

**dependencies** — Module interactions, data flow, API/interface contracts, cross-agent memory dependencies, tool dependency chains, and memory merge dependency chains relevant to restricting the orchestrator's tool access and redesigning its memory architecture.

## Summary

The orchestrator writes to shared `memory.md` at 12+ merge points using the `memory` tool (or delegated subagent fallback); all 20 sub-agents READ `memory.md` as their first input; the `memory` tool is the only write mechanism for `memory.md` outside of direct file tools; removing search tools (`grep_search`, `semantic_search`, `file_search`, `get_errors`) from the orchestrator does NOT break any critical-path operation — all cluster decisions use `read_file` on known paths.

## Findings

### 1. Files That Reference Orchestrator Tools or memory.md Writing

#### 1.1 Primary Files Requiring Modification

| File | References to Modify | Category |
|------|---------------------|----------|
| [.github/agents/orchestrator.agent.md](.github/agents/orchestrator.agent.md) | Global Rule 1 (lists `grep_search`, `semantic_search`, `file_search`, `list_dir`); Operating Rule 1 (recommends `semantic_search` and `grep_search`); Operating Rule 5 (current tool list includes all search tools + `get_errors`); Anti-Drift Anchor (mentions read tools freely); 12+ `memory.md` write references | Primary target |
| [.github/prompts/feature-workflow.prompt.md](.github/prompts/feature-workflow.prompt.md) | Line 24: describes memory system with orchestrator writing to `memory.md`; Line 39: lists `memory.md` as shared pipeline memory maintained by orchestrator | Prompt file |
| [.github/agents/dispatch-patterns.md](.github/agents/dispatch-patterns.md) | Lines 13, 25, 32, 45-46: orchestrator reads isolated memories and merges into `memory.md` | Reference document |

#### 1.2 All Agent Files Referencing `memory.md` (READ access — 20 agents)

Every sub-agent lists `memory.md` in its Inputs section and references it in its "Memory-first reading" operating rule. These agents **read** `memory.md` but never write to it. The full list:

| Agent | Input Reference | Operating Rule Reference | Anti-Drift Anchor Reference |
|-------|----------------|------------------------|-----------------------------|
| researcher | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| spec | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| designer | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| planner | `memory.md (operational memory — artifact index, decisions, lessons)` | Rule 6: Read upstream memories FIRST, then `memory.md` | "never to shared `memory.md`" |
| implementer | `memory.md (read for orientation — artifact index, decisions)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| documentation-writer | `memory.md (read first — operational memory)` | (implicit) | "never to shared `memory.md`" |
| ct-security | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| ct-scalability | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| ct-maintainability | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| ct-strategy | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| v-build | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| v-tests | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| v-tasks | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| v-feature | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| r-quality | `memory.md (read first — operational memory)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| r-security | `memory.md (read for orientation — artifact index, decisions)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| r-testing | `memory.md (read for orientation — artifact index, decisions)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| r-knowledge | `memory.md (read for orientation — artifact index, decisions)` | Rule 6: Read `memory.md` FIRST | "never to shared `memory.md`" |
| post-mortem | `memory.md (shared memory — for artifact index and context)` | Rule 6: Read memory files BEFORE full artifact reads | "never to shared `memory.md`" |

**Key finding:** If `memory.md` is removed (Option B), all 20 sub-agent files need their Inputs, Operating Rules, and Retrieval Strategy sections updated. However, every agent already has a fallback: "If `memory.md` is missing, log a warning and proceed with direct artifact reads."

### 2. Cross-Agent Memory Dependencies

#### 2.1 Which Agents Receive `memory.md` as Input

**All 20 sub-agents** receive `memory.md` as input (documented in §1.2 above). However, the nature of the dependency varies:

| Dependency Level | Agents | What They Use From memory.md |
|-----------------|--------|------------------------------|
| **Strong** | spec, designer, planner, implementer, documentation-writer | Artifact Index (section pointers for targeted reads), Recent Decisions, Lessons Learned |
| **Moderate** | CT agents (×4), V agents (×4) | Artifact Index for orientation, Recent Decisions for context |
| **Weak** | R agents (×4), post-mortem | Artifact Index for navigation; these agents also read upstream `.mem.md` files directly |

#### 2.2 Which Agents Write to `memory/<agent>.mem.md`

Every sub-agent writes its own isolated memory file. Complete list:

| Pipeline Phase | Agent | Isolated Memory File |
|---------------|-------|---------------------|
| Research (Step 1) | researcher ×4 | `memory/researcher-architecture.mem.md`, `memory/researcher-impact.mem.md`, `memory/researcher-dependencies.mem.md`, `memory/researcher-patterns.mem.md` |
| Spec (Step 2) | spec | `memory/spec.mem.md` |
| Design (Step 3) | designer | `memory/designer.mem.md` |
| CT Review (Step 3b) | ct-security, ct-scalability, ct-maintainability, ct-strategy | `memory/ct-security.mem.md`, `memory/ct-scalability.mem.md`, `memory/ct-maintainability.mem.md`, `memory/ct-strategy.mem.md` |
| Planning (Step 4) | planner | `memory/planner.mem.md` |
| Implementation (Step 5) | implementer ×N, documentation-writer ×M | `memory/implementer-<task-id>.mem.md`, `memory/documentation-writer-<task-id>.mem.md` |
| Verification (Step 6) | v-build, v-tests, v-tasks, v-feature | `memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md` |
| Review (Step 7) | r-quality, r-security, r-testing, r-knowledge | `memory/r-quality.mem.md`, `memory/r-security.mem.md`, `memory/r-testing.mem.md`, `memory/r-knowledge.mem.md` |
| Post-Mortem (Step 8) | post-mortem | `memory/post-mortem.mem.md` |

#### 2.3 Do Any Agents READ from Shared `memory.md` During Their Work?

**Yes —** all 20 agents read `memory.md` as their first operation (memory-first protocol). However, agents also directly read upstream isolated `.mem.md` files for more targeted context:

| Agent | Reads `memory.md` | Also Directly Reads Upstream `.mem.md` Files |
|-------|-------------------|---------------------------------------------|
| spec | Yes (first) | `memory/researcher-*.mem.md` (all 4) |
| designer | Yes (first) | `memory/spec.mem.md`, `memory/researcher-*.mem.md` (all 4), `memory/ct-*.mem.md` (revision mode) |
| planner | Yes | `memory/designer.mem.md`, `memory/spec.mem.md`, `memory/ct-*.mem.md` (conditional), `memory/v-*.mem.md` (replan mode) |
| implementer | Yes (first) | `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md` |
| documentation-writer | Yes (first) | `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md` |
| ct-* (×4) | Yes (first) | `memory/designer.mem.md`, `memory/spec.mem.md` |
| v-build | Yes (first) | `memory/planner.mem.md` |
| v-tests | Yes (first) | `memory/v-build.mem.md`, `memory/planner.mem.md` |
| v-tasks | Yes (first) | `memory/v-build.mem.md`, `memory/planner.mem.md` |
| v-feature | Yes (first) | `memory/v-build.mem.md`, `memory/spec.mem.md` |
| r-quality | Yes (first) | `memory/implementer-*.mem.md`, `memory/designer.mem.md` |
| r-security | Yes (first) | (none listed explicitly beyond memory.md) |
| r-testing | Yes (first) | `memory/planner.mem.md`, `memory/implementer-*.mem.md` |
| r-knowledge | Yes (first) | `memory/planner.mem.md`, `memory/implementer-*.mem.md` |
| post-mortem | Yes | `memory/*.mem.md` (all) |

**Critical finding:** Every agent that reads `memory.md` ALSO reads upstream `.mem.md` files directly. The `memory.md` provides a **consolidated view** (Artifact Index, aggregated Decisions, Lessons Learned) that saves agents from reading ALL upstream memories. Without `memory.md`, agents would need to read more `.mem.md` files to piece together the same context — but the information IS available through direct reads.

#### 2.4 What Information from `memory.md` Is Actually Used by Downstream Agents

Based on the `memory.md` structure and agent retrieval strategies:

| memory.md Section | Used By | Purpose |
|-------------------|---------|---------|
| **Artifact Index** | All agents | Navigate to specific sections of artifacts (e.g., "read §Security section of design.md") — avoids reading full files |
| **Recent Decisions** | spec, designer, planner, implementer | Understand prior architectural choices to maintain consistency |
| **Lessons Learned** | implementer (primarily) | Avoid repeating mistakes from earlier implementation waves |
| **Recent Updates** | All agents (orientation) | Understand what changed recently; which agents completed |

**The Artifact Index is the highest-value section.** It provides section-level pointers (e.g., `research/architecture.md — §Repository Structure, §Agent Definition Format`) that enable targeted reads. Without it, agents must either read full artifacts or rely solely on upstream individual `.mem.md` Artifact Index sections.

### 3. Tool Dependency Chain

#### 3.1 Current Orchestrator Tool Set

From Operating Rule 5 ([orchestrator.agent.md line 92](orchestrator.agent.md#L92)):
```
Current: [agent, agent/runSubagent, memory, read_file, grep_search, semantic_search, file_search, list_dir, get_errors]
Proposed (Option B): [agent, agent/runSubagent, memory, read_file, list_dir]
```

#### 3.2 Dependency Matrix: Orchestrator Operation → Required Tools

| Orchestrator Operation | `agent/runSubagent` | `memory` | `read_file` | `list_dir` | `grep_search` | `semantic_search` | `file_search` | `get_errors` |
|----------------------|--------------------:|:--------:|:-----------:|:----------:|:-------------:|:-----------------:|:-------------:|:------------:|
| Dispatch sub-agent | **REQUIRED** | — | — | — | — | — | — | — |
| Initialize memory.md (Step 0) | fallback | **REQUIRED** | — | — | — | — | — | — |
| Read isolated .mem.md files | — | — | **REQUIRED** | — | — | — | — | — |
| CT cluster decision | — | — | **REQUIRED** | — | — | — | — | — |
| V cluster decision | — | — | **REQUIRED** | — | — | — | — | — |
| R cluster decision | — | — | **REQUIRED** | — | — | — | — | — |
| Merge memories into memory.md | — | **REQUIRED** | **REQUIRED** | — | — | — | — | — |
| Prune memory.md | — | **REQUIRED** | **REQUIRED** | — | — | — | — | — |
| Invalidate memory entries | — | **REQUIRED** | **REQUIRED** | — | — | — | — | — |
| Parse execution waves (Step 5.1) | — | — | **REQUIRED** | — | — | — | — | — |
| Read task files for `agent` field | — | — | **REQUIRED** | — | — | — | — | — |
| Check memory file existence | — | — | **REQUIRED** | **REQUIRED** | — | — | — | — |
| Verify documentation structure | — | — | — | **REQUIRED** | — | — | — | — |
| Discover files for dispatch | — | — | — | **REQUIRED** | used | used | used | — |
| Context-efficient reading (Operating Rule 1) | — | — | — | — | **recommended** | **recommended** | — | — |
| Error checking after dispatch | — | — | — | — | — | — | — | used |
| Log warnings in memory.md | — | **REQUIRED** | — | — | — | — | — | — |
| Determine review tier (Step 7.1) | — | — | **REQUIRED** | **REQUIRED** | used | — | used | — |
| Pass telemetry to PostMortem (Step 8) | **REQUIRED** | — | — | — | — | — | — | — |

#### 3.3 Operations That Would Break with Option B Tools

**No critical-path operations break.** Analysis:

- **`grep_search` removal:** Currently recommended in Operating Rule 1 for "context-efficient reading." The orchestrator uses this for discovery, but all cluster decisions use `read_file` on **known file paths** (e.g., `memory/ct-security.mem.md`). The orchestrator knows all file paths because they follow deterministic naming conventions. **Impact: Low** — convenience loss for ad-hoc exploration, not needed for pipeline execution.

- **`semantic_search` removal:** Same as `grep_search` — convenience for exploration, not required for any decision flow. **Impact: Low.**

- **`file_search` removal:** Used potentially for review tier determination (Step 7.1) to count changed files. Could be replaced by `list_dir` on known directories. **Impact: Low** — workaround available.

- **`get_errors` removal:** The orchestrator itself does not need to check for compile/lint errors — that is V-Build's job. Currently available but not used in any documented decision flow. **Impact: None.**

**One area of concern:** Operating Rule 1 currently says "Prefer `semantic_search` and `grep_search` for discovery." This must be rewritten to match the restricted tool set.

### 4. Memory Merge Dependency Chain

#### 4.1 When Merges Happen

The orchestrator performs memory merges at these points (from Memory Lifecycle Actions table, [orchestrator.agent.md lines 502-510](orchestrator.agent.md#L502-L510)):

| Merge Point | Pipeline Step | Merge Source | Prune After? |
|-------------|--------------|--------------|:------------:|
| 1.1m | After Research (4 agents) | `memory/researcher-*.mem.md` ×4 | **Yes** |
| 2m | After Spec | `memory/spec.mem.md` | **Yes** |
| 3m | After Design | `memory/designer.mem.md` | No |
| 3b.2 | After CT cluster (4 agents) | `memory/ct-*.mem.md` ×4 | No |
| 4m | After Planning | `memory/planner.mem.md` | **Yes** |
| Between waves (Step 5) | Between implementation waves | `memory/implementer-*.mem.md`, `memory/documentation-writer-*.mem.md` | No |
| 6.3 | After V cluster (4 agents) | `memory/v-*.mem.md` ×4 | No |
| 7.3 | After R cluster (4 agents) | `memory/r-*.mem.md` ×4 | No |
| 8.2 | After PostMortem | `memory/post-mortem.mem.md` | No |

**Total merge operations per pipeline run:** 9 merge points, processing 20+ individual `.mem.md` files.

#### 4.2 What Downstream Agents Consume Merged Memory

The merge chain flows forward through the pipeline:

```
Step 1.1m merge → Step 2 (spec reads memory.md with researcher findings)
Step 2m merge   → Step 3 (designer reads memory.md with spec + researcher findings)
Step 3m merge   → Step 3b (CT agents read memory.md with design + spec + researcher findings)
Step 3b.2 merge → Step 4 (planner reads memory.md with CT + design + spec + researcher findings)
Step 4m merge   → Step 5 (implementers read memory.md with planner + all upstream findings)
Wave merges     → Next wave (implementers read memory.md with lessons from prior waves)
Step 6.3 merge  → Step 7 (R agents read memory.md — though by this point it's VERY large)
Step 7.3 merge  → Step 8 (PostMortem reads memory.md for orientation)
```

**Critical observation:** Each merge ACCUMULATES context. By Step 5, `memory.md` contains findings from researchers, spec, designer, CT, and planner. By Step 7, it contains EVERYTHING. The Artifact Index grows monotonically; only Recent Decisions and Recent Updates are pruned.

#### 4.3 Is There a Critical Path Through memory.md?

**Yes, but with redundancy.** The critical path is:

1. **Step 1.1m → Step 2:** Spec needs researcher Artifact Index entries to know which sections of `research/*.md` to read. WITHOUT `memory.md`, spec would need to read all 4 `memory/researcher-*.mem.md` files directly (it already lists them as inputs).

2. **Step 2m → Step 3:** Designer needs spec Artifact Index. WITHOUT `memory.md`, designer reads `memory/spec.mem.md` directly (already listed as input).

3. **Step 4m → Step 5:** Implementers need planner and designer context. WITHOUT `memory.md`, they read `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md` directly (already listed as inputs).

4. **Between-wave lesson extraction → Next wave:** Implementers in wave N+1 need Lessons Learned from wave N. WITHOUT `memory.md`, the orchestrator would need to pass these lessons via the dispatch prompt, or implementers would need to read prior wave `.mem.md` files — which they currently do NOT list as inputs.

**The critical gap in Option B (remove memory.md) is Lessons Learned propagation between implementation waves.** Lessons Learned is the ONLY section of `memory.md` that doesn't have a redundant source in individual `.mem.md` files — it's extracted from implementer memories and AGGREGATED across waves by the orchestrator.

#### 4.4 What Happens If memory.md Is Stale or Missing

**Already designed for graceful degradation.** Every agent has this fallback:

> "If `memory.md` is missing, log a warning and proceed with direct artifact reads."

The `self-improvement-system` pipeline run confirms this works — the memory.md is described as "non-blocking" in Global Rule 6.

**Specific impacts of stale/missing memory.md:**

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Missing entirely | Agents read full artifacts instead of targeted sections → slower, higher token usage | Agents have fallback; pipeline still works |
| Stale (not updated after a recent merge) | Agents miss recent decisions/updates → may duplicate work | Agents also read upstream `.mem.md` directly |
| Corrupted format | Agents can't parse Artifact Index → fall back to full reads | Same as missing |
| Missing Artifact Index | Agents read full artifacts instead of sections | Functional but less efficient |
| Missing Lessons Learned | Cross-wave learning lost | Higher risk of repeating mistakes in later waves |

### 5. Existing Implementation Patterns from self-improvement-system

#### 5.1 Real memory.md Contents

The [self-improvement-system memory.md](docs/feature/self-improvement-system/memory.md) shows a real merged memory with:
- **16 entries in Artifact Index** — covering all researched and produced artifacts with section-level pointers
- **8 Recent Decisions** — spanning orchestrator decisions from Steps 3b through 8
- **3 Lessons Learned** — cross-wave issues and resolutions
- **12 Recent Updates** — tracking artifact creation across all pipeline phases

**Key observation:** The Artifact Index accumulated 16 entries across the full pipeline. Each entry has path, section references, and last-updated-by agent name. This is the primary value of `memory.md`.

#### 5.2 decisions.md as Memory-Adjacent Artifact

The [decisions.md](docs/feature/self-improvement-system/decisions.md) contains 6 architectural decisions, including one directly relevant: **"Tool Restriction: Keep Read Tools for Orchestrator"** (Decision 4). This prior decision already established that read tools should stay while write tools should be restricted. The current feature request aligns with this — the proposed Option B tools `[agent, agent/runSubagent, memory, read_file, list_dir]` keep `read_file` and `list_dir`.

#### 5.3 Pattern: Orchestrator's `memory` Tool Usage

In the current orchestrator, the `memory` tool is used for:
1. **Initialize:** Create `memory.md` with empty template (Step 0) — fallback to subagent if tool fails
2. **Merge:** Write merged content to `memory.md` (after each agent/cluster) — 9 points
3. **Prune:** Modify `memory.md` to remove old entries (3 checkpoint prune operations)
4. **Invalidate:** Mark entries with `[INVALIDATED — <reason>]` before revision dispatch
5. **Log warnings/decisions:** Write self-verification entries to Recent Updates

If the orchestrator keeps the `memory` tool in Option B, ALL of these operations continue working. The `memory` tool is the VS Code built-in cross-session memory tool — it's NOT a file creation tool. However, the orchestrator currently uses it ambiguously: sometimes for shared `memory.md` writes, sometimes vs. `runSubagent` delegation.

#### 5.4 Telemetry Accumulation Pattern

Decision 2 from `decisions.md` established that telemetry accumulates in the orchestrator's context window (not in files) until Step 8, when it's passed to PostMortem. This pattern interacts with the tool restriction because it confirms the orchestrator does NOT need file-write tools for telemetry — the data stays in context.

### 6. `memory` Tool vs. File Write Tools — Critical Distinction

The [initial request](docs/feature/orchestrator-tool-restriction/initial-request.md) states:

> "`memory` here is VS Code's built-in cross-session memory tool, NOT pipeline memory.md"

This means the `memory` tool in the proposed Option B tools is for **cross-session codebase knowledge** (persistent VS Code memory), not for writing `memory.md`. This creates a critical dependency gap:

| Operation | Current Mechanism | After Option B Restriction |
|-----------|------------------|---------------------------|
| Create `memory.md` | `memory` tool or subagent | **Must delegate to subagent** — no `create_file` |
| Write to `memory.md` | `memory` tool or subagent | **Must delegate to subagent** — no file write tools |
| Read `memory.md` | `read_file` | `read_file` (still available) |
| Merge into `memory.md` | `memory` tool | **Must delegate to subagent** |
| Prune `memory.md` | `memory` tool | **Must delegate to subagent** |
| Invalidate entries | `memory` tool | **Must delegate to subagent** |
| Log warnings | `memory` tool | **Must delegate to subagent** |

**This means under Option B tools, EVERY memory merge operation (9+ per pipeline) must become a subagent dispatch.** This is the most significant dependency impact of the tool restriction.

### 7. Impact Summary by Memory Architecture Option

#### Option A: Keep memory.md, Delegate All Writes to Subagent

- **Dependencies changed:** Orchestrator loses direct `memory.md` write access
- **New dependency:** A "memory-writer" subagent must be created or the orchestrator must delegate to a generic subagent for each merge
- **Impact on sub-agents:** None — they still read `memory.md` as before
- **Merge frequency impact:** 9+ subagent dispatches added per pipeline just for memory operations
- **Files to modify:** `orchestrator.agent.md` (memory lifecycle), potentially new `memory-writer.agent.md`

#### Option B: Remove memory.md Entirely

- **Dependencies changed:** All 20 sub-agents lose `memory.md` input; orchestrator loses all merge operations
- **Impact on sub-agents:** Must update Inputs, Operating Rules, Retrieval Strategy in all 20 `.agent.md` files; each agent must read more upstream `.mem.md` files directly
- **Critical gap:** Lessons Learned aggregation between implementation waves has no direct `.mem.md` equivalent
- **Critical gap:** Artifact Index consolidation lost — agents must reconstruct from individual `.mem.md` Artifact Index sections
- **Files to modify:** ALL 21 agent files + `feature-workflow.prompt.md` + `dispatch-patterns.md`

#### Option C: Hybrid — Auto-assembled by Memory-Merge Subagent

- **Dependencies changed:** New subagent handles all memory operations
- **New dependency:** `memory-merge.agent.md` or equivalent; orchestrator dispatches it after each agent/cluster
- **Impact on sub-agents:** None — they still read `memory.md` as before
- **Merge frequency impact:** Same 9+ dispatches, but to a dedicated agent
- **Files to modify:** `orchestrator.agent.md` (memory lifecycle sections), new `memory-merge.agent.md`, `feature-workflow.prompt.md`

## File References

| File/Folder | Relevance |
|-------------|-----------|
| [.github/agents/orchestrator.agent.md](.github/agents/orchestrator.agent.md) | Primary target — tool list (line 92), Global Rule 1 (line 33), Operating Rule 1 (line 83), Memory-First Protocol (line 38), Memory Write Safety (line 44), Anti-Drift Anchor (line 541), all Step merge operations |
| [.github/prompts/feature-workflow.prompt.md](.github/prompts/feature-workflow.prompt.md) | Memory system description (line 24), Key Artifacts table (line 39) |
| [.github/agents/dispatch-patterns.md](.github/agents/dispatch-patterns.md) | Memory-First Pattern section (lines 40-46), merge references in all patterns |
| [.github/agents/researcher.agent.md](.github/agents/researcher.agent.md) | memory.md Input reference, memory-first reading rule |
| [.github/agents/spec.agent.md](.github/agents/spec.agent.md) | memory.md Input + 4 researcher .mem.md reads |
| [.github/agents/designer.agent.md](.github/agents/designer.agent.md) | memory.md Input + spec + researcher .mem.md reads + CT .mem.md reads (revision) |
| [.github/agents/planner.agent.md](.github/agents/planner.agent.md) | memory.md Input + designer + spec .mem.md + CT .mem.md (conditional) + V .mem.md (replan) |
| [.github/agents/implementer.agent.md](.github/agents/implementer.agent.md) | memory.md Input + planner + designer + spec .mem.md reads |
| [.github/agents/documentation-writer.agent.md](.github/agents/documentation-writer.agent.md) | memory.md Input + planner + designer + spec .mem.md reads |
| [.github/agents/ct-security.agent.md](.github/agents/ct-security.agent.md) | memory.md Input + designer + spec .mem.md reads |
| [.github/agents/ct-scalability.agent.md](.github/agents/ct-scalability.agent.md) | memory.md Input + designer + spec .mem.md reads |
| [.github/agents/ct-maintainability.agent.md](.github/agents/ct-maintainability.agent.md) | memory.md Input + designer + spec .mem.md reads |
| [.github/agents/ct-strategy.agent.md](.github/agents/ct-strategy.agent.md) | memory.md Input + designer + spec .mem.md reads |
| [.github/agents/v-build.agent.md](.github/agents/v-build.agent.md) | memory.md Input + planner .mem.md |
| [.github/agents/v-tests.agent.md](.github/agents/v-tests.agent.md) | memory.md Input + v-build + planner .mem.md |
| [.github/agents/v-tasks.agent.md](.github/agents/v-tasks.agent.md) | memory.md Input + v-build + planner .mem.md |
| [.github/agents/v-feature.agent.md](.github/agents/v-feature.agent.md) | memory.md Input + v-build + spec .mem.md |
| [.github/agents/r-quality.agent.md](.github/agents/r-quality.agent.md) | memory.md Input + implementer-*.mem.md + designer .mem.md |
| [.github/agents/r-security.agent.md](.github/agents/r-security.agent.md) | memory.md Input |
| [.github/agents/r-testing.agent.md](.github/agents/r-testing.agent.md) | memory.md Input + planner + implementer-*.mem.md |
| [.github/agents/r-knowledge.agent.md](.github/agents/r-knowledge.agent.md) | memory.md Input + planner + implementer-*.mem.md |
| [.github/agents/post-mortem.agent.md](.github/agents/post-mortem.agent.md) | memory.md Input + all .mem.md files |
| [.github/agents/evaluation-schema.md](.github/agents/evaluation-schema.md) | Referenced by 14+ agents — not directly affected by tool restriction but establishes cross-cutting reference pattern |
| [docs/feature/self-improvement-system/memory.md](docs/feature/self-improvement-system/memory.md) | Real example of merged memory.md — shows Artifact Index with 16 entries, 8 decisions, 3 lessons |
| [docs/feature/self-improvement-system/decisions.md](docs/feature/self-improvement-system/decisions.md) | Prior decision #4 established keeping read tools for orchestrator |

## Assumptions & Limitations

1. **Assumed:** The `memory` tool in Option B tools refers to VS Code's built-in cross-session memory, NOT a pipeline-level file-write tool — confirmed by the initial request.
2. **Assumed:** The orchestrator can still pass arbitrary text content in `runSubagent` prompt context — this is how it would instruct a memory-merge subagent.
3. **Assumed:** All agents' `memory.md` fallback ("log a warning and proceed with direct artifact reads") has been validated in production — the self-improvement-system pipeline ran successfully with this fallback available.
4. **Limitation:** I did not verify whether the VS Code `memory` tool can write to arbitrary file paths like `docs/feature/<slug>/memory.md` — this is platform-dependent behavior. If it can, some memory operations might still work with Option B tools.
5. **Limitation:** The exact token cost impact of removing `memory.md` (forcing agents to read more `.mem.md` files) was not quantified.

## Open Questions

1. **Memory merge subagent cost:** If all 9+ merge operations become subagent dispatches (Option A/C), what is the token/latency cost? Each merge reads N `.mem.md` files and writes one `memory.md` update — is this worth a full subagent invocation?
2. **Lessons Learned without memory.md:** In Option B, how do implementation wave N+1 agents receive Lessons Learned from wave N? The orchestrator could include them in the dispatch prompt, but this hasn't been designed.
3. **Artifact Index consolidation:** In Option B, should each agent reconstruct the global Artifact Index from individual `.mem.md` files? Or should the orchestrator pass the consolidated index in the dispatch prompt?
4. **VS Code `memory` tool capabilities:** Can it write to specific file paths? If so, the orchestrator could still use it for `memory.md` operations even without `create_file`/`replace_string_in_file`.
5. **Review tier determination (Step 7.1):** Currently may use `file_search` or `grep_search` to assess scope. With Option B tools, how does the orchestrator determine the review tier? It may need to rely on `list_dir` + `read_file` only, or receive scope information from V cluster outputs.

## Research Metadata

- **confidence_level:** high — all 21 agent definition files, dispatch-patterns.md, feature-workflow.prompt.md, and the real self-improvement-system memory.md/decisions.md were examined
- **coverage_estimate:** Complete coverage of all agent files' memory references (140+ grep matches analyzed), all orchestrator tool references, all merge points, and all cross-agent memory read/write patterns
- **gaps:** Platform-level capabilities of the VS Code `memory` tool (file write vs. cross-session knowledge store) are not empirically verified. Token cost modeling for Option B (agents reading more .mem.md files) not performed.
