# Research: Patterns

## Focus Area

**patterns** — Memory architecture patterns, orchestrator purity patterns, current subagent input patterns, and anti-pattern analysis for the orchestrator tool restriction and memory redesign.

## Summary

The current memory architecture has the orchestrator performing 10–20+ merge operations per pipeline run into a shared `memory.md`, but all subagents already have fallback logic to proceed without it. Pattern B (eliminate shared memory.md) is the lowest-cost option because subagents already read upstream isolated memories directly. Pattern C (subagents self-assemble) is a refinement of B. Pattern A (memory-merge subagent) adds significant overhead for minimal benefit. The existing codebase already implements most of the infrastructure needed for Pattern B.

---

## Findings

### 1. Memory Architecture Pattern Analysis

#### 1.1 Current Memory Merge Points (Counting the Cost)

The orchestrator currently performs memory merges at these points in the pipeline ([orchestrator.agent.md](../../../.github/agents/orchestrator.agent.md)):

| Merge Point | Step | Files Merged | Notes |
|---|---|---|---|
| 1.1m | After research | 4 researcher memories | Batch merge |
| 2m | After spec | 1 spec memory | Sequential |
| 3m | After designer | 1 designer memory | Sequential |
| 3b.2 | After CT cluster | 4 CT memories | Batch merge |
| 4m | After planner | 1 planner memory | Sequential + prune |
| Between waves | After each implementation wave | N implementer/doc-writer memories | Variable (1–11+ per wave) |
| 6.3 | After V cluster | 4 V memories | Batch merge |
| 7.3 | After R cluster | 4 R memories | Batch merge |
| 8.2 | After post-mortem | 1 post-mortem memory | Final merge |

**Minimum merge operations per pipeline run: ~10** (no implementation tasks).
**Typical: ~15–20+** (with 7–11 implementation tasks spread across waves).

Each merge reads an isolated `.mem.md` file and writes extracted Key Findings, Decisions, Artifact Index, and Lessons Learned into shared `memory.md`.

#### 1.2 Pattern A: Dedicated Memory-Merge Subagent

**Mechanism:** After each agent/cluster completes, the orchestrator dispatches a lightweight "memory-merger" subagent that:
1. Reads the completed agent's `memory/<agent>.mem.md`
2. Reads current `memory.md`
3. Merges Key Findings, Decisions, Artifact Index into `memory.md`
4. Handles pruning at checkpoints (Steps 1.1, 2, 4)
5. Returns `DONE:` or `ERROR:`

**Hypothetical prompt shape:**
```
You are the Memory Merger agent. Merge the following isolated memory file into shared memory.md.

Inputs:
- memory/<agent>.mem.md (the source)
- memory.md (the target)

Actions:
1. Read both files
2. Append Key Findings to Artifact Index (deduplicate)
3. Append Decisions to Recent Decisions
4. Append Lessons Learned (never prune)
5. Update Recent Updates with merge timestamp
6. If PRUNE mode: remove Recent Decisions/Updates older than 2 phases

Output: Updated memory.md
Return: DONE: merged <agent> memory
```

**Quantified cost analysis:**
- 15–20 subagent dispatches per run solely for memory merging
- Each dispatch adds latency (subagent spin-up, context loading, tool execution, return)
- Token cost: each merge subagent reads ~2 files (source mem + target memory.md) and writes 1 file = ~500–1500 tokens per dispatch
- Total additional token cost: ~10K–30K tokens per pipeline run
- Total additional latency: ~15–20 sequential blocking operations (merges cannot be parallelized since they target the same file)

**What memory.md actually stores after all merges (from [orchestrator.agent.md#L19-L28](../../../.github/agents/orchestrator.agent.md)):**
- **Artifact Index**: table of all artifacts with key sections and last-updated-by
- **Recent Decisions**: accumulated from all agents
- **Lessons Learned**: accumulated from implementation (never pruned)
- **Recent Updates**: log of pipeline events

#### 1.3 Pattern B: Eliminate Shared memory.md Entirely

**Mechanism:** No shared `memory.md` exists. The orchestrator reads isolated `memory/<agent>.mem.md` files directly via `read_file` for all routing decisions. Subagents receive paths to relevant upstream memory files and read them directly.

**Current evidence that this already works:**

1. **All subagents have memory.md fallback logic.** Every single subagent (19 agents) contains this pattern in Operating Rule 6:
   > "If `memory.md` is missing, log a warning and proceed with direct artifact reads."
   
   File references:
   - [researcher.agent.md#L41](../../../.github/agents/researcher.agent.md): `If memory.md is missing, log a warning and proceed with direct artifact reads.`
   - [spec.agent.md#L51](../../../.github/agents/spec.agent.md): same pattern
   - [designer.agent.md#L66](../../../.github/agents/designer.agent.md): same pattern
   - [implementer.agent.md#L52](../../../.github/agents/implementer.agent.md): `If memory.md or upstream memories are missing, log a warning and proceed with direct artifact reads.`
   - [planner.agent.md#L53](../../../.github/agents/planner.agent.md): same pattern
   - All CT, V, R agents: same pattern
   - [documentation-writer.agent.md#L48](../../../.github/agents/documentation-writer.agent.md): notably uses `If upstream memory files are missing` — does NOT reference memory.md in its fallback at all

2. **Subagents already list specific upstream memory files as primary inputs.** Every subagent explicitly names the isolated memory files it needs:
   - Spec agent: reads `memory/researcher-*.mem.md` (4 files)
   - Designer: reads `memory/spec.mem.md`, `memory/researcher-*.mem.md` (5 files)
   - CT agents: read `memory/designer.mem.md`, `memory/spec.mem.md`
   - Planner: reads `memory/designer.mem.md`, `memory/spec.mem.md`
   - Implementer: reads `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md`
   - V agents: read `memory/v-build.mem.md`, `memory/planner.mem.md`
   - Documentation-writer: reads `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md`

3. **Orchestrator already reads isolated memories directly for all cluster decisions:**
   - CT decision flow: reads `memory/ct-security.mem.md`, etc. ([orchestrator.agent.md#L131-L143](../../../.github/agents/orchestrator.agent.md))
   - V decision flow: reads `memory/v-build.mem.md`, etc. ([orchestrator.agent.md#L148-L172](../../../.github/agents/orchestrator.agent.md))
   - R decision flow: reads `memory/r-security.mem.md`, etc. ([orchestrator.agent.md#L177-L187](../../../.github/agents/orchestrator.agent.md))

**What changes in dispatch prompts under Pattern B:**

Currently, dispatch includes `memory.md` as an input. Under Pattern B:
- Remove `memory.md` from all dispatch inputs
- Each subagent's dispatch prompt includes only the specific upstream memory file paths it needs
- Example: Designer dispatch currently says "Input: `memory/spec.mem.md`, `memory/researcher-*.mem.md`, `memory.md`" → becomes "Input: `memory/spec.mem.md`, `memory/researcher-*.mem.md`"
- The orchestrator's dispatch instructions already list specific memory file paths alongside `memory.md` — removing `memory.md` is a deletion, not a restructure

**What is lost:**
- **Centralized Artifact Index** — subagents currently use memory.md's Artifact Index to know which sections of upstream artifacts to read. Without it, subagents must either: (a) read the full upstream artifacts (token-wasteful), or (b) use artifact indexes from individual memory files (already present in each `.mem.md`)
- **Accumulated Lessons Learned** — pipeline lessons from earlier phases would not be visible to later agents. Could be mitigated by passing relevant implementer memory file paths to V/R clusters.
- **Recent Decisions** — later agents wouldn't see early-pipeline decisions. However, decisions are already embedded in the artifacts those agents produced (feature.md, design.md, etc.).

**What is NOT lost:**
- Cluster decision logic — orchestrator reads isolated memories directly, memory.md is not used here
- Upstream context — subagents already read specific upstream `.mem.md` files
- Artifact navigation — each `.mem.md` already contains its own Artifact Index with section-level pointers

#### 1.4 Pattern C: Subagents Self-Assemble Context

**Mechanism:** The orchestrator includes a "memory manifest" in each dispatch — a list of which `memory/*.mem.md` files exist at that point in the pipeline, organized by phase. Each subagent reads the memory files relevant to its needs.

**How it differs from Pattern B:**
- Pattern B: orchestrator tells each subagent exactly which memory files to read (curated list)
- Pattern C: orchestrator tells each subagent ALL memory files that exist, subagent decides which to read

**Current evidence that subagents can self-assemble:**
- All subagents have read tools (`read_file`, `grep_search`, `semantic_search`, etc.)
- All subagents already implement a "memory-first reading" protocol that reads upstream memories
- Each `.mem.md` file is compact (typically ≤20 lines: Status, Key Findings, Highest Severity, Decisions, Artifact Index)
- Subagents already have filtering logic — they read memory files first, then consult Artifact Index for targeted reads of larger artifacts

**Risk of over-reading:** In a full pipeline run with ~20 agents, there could be ~20 `.mem.md` files. If every subagent reads all of them, that's ~400 tokens × 20 files = ~8K tokens of memory reading per agent. The current pipeline dispatches ~20 agents × 8K = ~160K extra tokens per run, vs. the current ~30K tokens for all memory.md reads. Pattern C would be ~5× more expensive in memory-reading tokens.

**Mitigation:** The orchestrator could categorize memory files by relevance tier (previous-phase, same-cluster, all-phases) and include only the relevant tier paths. This converges with Pattern B.

### 2. Orchestrator Purity Patterns

#### 2.1 Minimum Tool Set for a Pure Coordinator

The orchestrator's core coordination functions are:
1. **Dispatch subagents** — requires `agent`, `agent/runSubagent`
2. **Read completion status** — requires parsing subagent return values (built into `runSubagent`)
3. **Read memory files for routing decisions** — requires `read_file`
4. **Discover file existence** — requires `list_dir` or `file_search`
5. **Cross-session knowledge** — requires VS Code `memory` tool (separate from pipeline memory.md)

**Minimum pure coordinator tool set:** `[agent, agent/runSubagent, memory, read_file, list_dir]`

This is exactly the proposed Option B from the initial request.

#### 2.2 Is `read_file` Acceptable for a Coordinator?

**Yes.** The existing [decisions.md](../../../docs/feature/self-improvement-system/decisions.md) entry "Tool Restriction: Keep Read Tools for Orchestrator" explicitly addresses this:

> "Read tools are essential for orchestrator function — it cannot navigate the pipeline without reading memory.md, checking artifact existence, or orienting on workspace structure. Write tools are the actual risk surface (orchestrator modifying source code or agent definitions)."

The principled distinction is:
- **Read operations** = observing state (pure, side-effect-free) → acceptable for coordinators
- **Write operations** = modifying state (impure, side-effects) → delegate to workers
- **Search/discovery operations** (`grep_search`, `semantic_search`, `file_search`) = open-ended exploration that gives the coordinator implementation-level knowledge → not needed for coordination

`read_file` with known paths is deterministic observation. `grep_search`/`semantic_search` is unbounded exploration that could lead the orchestrator to "do work" rather than coordinate.

#### 2.3 How Other Multi-Agent Systems Handle Shared State

Common patterns in multi-agent coordination:

| Pattern | Description | Used Here? |
|---|---|---|
| **Blackboard** | Shared mutable state (memory.md) where all agents read/write | Current system (orchestrator-sole-writer variant) |
| **Message passing** | Agents communicate via structured messages routed by coordinator | Partially (dispatch prompts are one-way messages) |
| **Event sourcing** | Immutable log of events, derived state computed on read | Not used (memory.md is mutable) |
| **File-per-agent state** | Each agent writes isolated state; coordinator reads all | Current system (memory/*.mem.md) — this IS the existing secondary pattern |

The current system actually implements a **dual pattern**: file-per-agent state (primary) with a blackboard overlay (memory.md merging). The overlay adds cost (merge operations) for marginal benefit (centralized view). Removing the overlay (Pattern B) collapses to the simpler file-per-agent pattern that already exists.

### 3. Current Subagent Input Patterns

#### 3.1 Spec Agent (Step 2)

From [spec.agent.md#L19-L30](../../../.github/agents/spec.agent.md):

**Inputs:**
- `memory.md` (read first — operational memory)
- `initial-request.md`
- `memory/researcher-architecture.mem.md` (primary)
- `memory/researcher-impact.mem.md` (primary)
- `memory/researcher-dependencies.mem.md` (primary)
- `memory/researcher-patterns.mem.md` (primary)
- `research/architecture.md` (selective — sections from researcher memory indexes)
- `research/impact.md` (selective)
- `research/dependencies.md` (selective)
- `research/patterns.md` (selective)

**Pattern:** Memory.md first → 4 upstream isolated memories → targeted reads of larger artifacts using artifact indexes from those memories.

**If memory.md is absent:** "log a warning and proceed with direct artifact reads" — the agent falls back to reading the 4 researcher memories and full research artifacts directly.

**Observation:** memory.md provides the spec agent with an Artifact Index accumulated from all 4 researchers. Without it, the spec agent still has 4 individual artifact indexes (one per researcher memory file), which provide the same section-level pointers.

#### 3.2 Implementer Agent (Step 5)

From [implementer.agent.md#L15-L23](../../../.github/agents/implementer.agent.md):

**Inputs (STRICT):**
- `memory.md` (read for orientation)
- `memory/planner.mem.md` (upstream — task structure)
- `memory/designer.mem.md` (upstream — design decisions)
- `memory/spec.mem.md` (upstream — requirements)
- `tasks/<task>.md`
- `feature.md`
- `design.md`
- `.github/agents/evaluation-schema.md`

**Explicitly cannot read:** `plan.md`

**Memory-first reading protocol ([implementer.agent.md#L52](../../../.github/agents/implementer.agent.md)):**
> "Read `memory.md` (maintained by orchestrator) FIRST, then read upstream memory files (`memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md`). Use the Artifact Index in these memories to navigate directly to relevant sections."

**If memory.md or upstream memories missing:** "log a warning and proceed with direct artifact reads"

**Observation:** The implementer reads memory.md for "orientation" but its actual navigation depends on upstream `.mem.md` files' Artifact Indexes. Removing memory.md means the implementer loses accumulated Lessons Learned from prior implementation waves, but gains nothing operationally — it already reads planner, designer, and spec memories directly.

#### 3.3 Documentation Writer Agent (Step 5)

From [documentation-writer.agent.md#L15-L23](../../../.github/agents/documentation-writer.agent.md):

**Inputs (STRICT):**
- `memory.md` (read first — operational memory)
- `memory/planner.mem.md` (upstream context)
- `memory/designer.mem.md` (upstream context)
- `memory/spec.mem.md` (upstream context)
- `tasks/<task>.md`
- `feature.md` (selective)
- `design.md` (selective)

**Memory-first reading protocol ([documentation-writer.agent.md#L48](../../../.github/agents/documentation-writer.agent.md)):**
> "Read upstream memory files FIRST (`memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md`) before accessing any artifact."

**Notable:** The documentation-writer's memory-first rule does NOT mention memory.md — it goes straight to upstream memories. Its fallback is "If upstream memory files are missing" (not "If memory.md is missing"). This agent is already operating in a Pattern B-like mode.

#### 3.4 CT-Security Agent (Step 3b)

From [ct-security.agent.md#L7-L14](../../../.github/agents/ct-security.agent.md):

**Inputs:**
- `memory.md` (read first)
- `memory/designer.mem.md` (upstream)
- `memory/spec.mem.md` (upstream)
- `initial-request.md`
- `design.md`
- `feature.md`

**Memory-first reading protocol ([ct-security.agent.md#L44](../../../.github/agents/ct-security.agent.md)):**
> "Read `memory.md` FIRST (operational memory) before accessing any artifact. Then read upstream memory files (`memory/designer.mem.md`, `memory/spec.mem.md`) to consult their Artifact Indexes for targeted reads of `design.md` and `feature.md`."

**Observation:** memory.md gives the CT agent accumulated context from Steps 1–3 (researcher findings, spec decisions, designer decisions). Without memory.md, the CT agent still gets designer and spec memories directly, which contain the key decisions and artifact indexes needed.

#### 3.5 Cross-Agent Summary: What memory.md Provides That Isolated Memories Don't

| Value | In memory.md | In isolated `.mem.md` files | Gap if memory.md removed |
|---|---|---|---|
| Artifact Index (per-agent) | Merged from all agents | Present in each agent's own `.mem.md` | None — subagent reads relevant upstream `.mem.md` files |
| Recent Decisions (accumulated) | Merged from all agents | Present in each agent's own `.mem.md` Decisions section | Minor — decisions already embedded in artifacts (design.md, feature.md) |
| Lessons Learned | Accumulated across waves | Present in implementer `.mem.md` files | Moderate — late-pipeline agents miss early-wave implementation lessons |
| Recent Updates (log) | Accumulated | Not present in `.mem.md` | Low — this is a debugging aid, not used by subagents for decisions |
| Cross-phase context | Provided by merged view | Requires reading multiple `.mem.md` files | Low — each agent needs only ~2-3 upstream memories, not the full history |

### 4. Anti-Pattern Analysis

#### 4.1 Why Pattern A (Memory-Merge Subagent × 20) Is Wasteful

1. **Merge count is high and grows with tasks.** A pipeline with 10 implementation tasks has ~17 merge points. With 20 tasks: ~27 merges. Each merge adds a subagent round-trip.

2. **The merged memory.md is primarily consumed by subagents that already read isolated memories.** The main consumers of memory.md are subagents (for Artifact Index and orientation), but every consuming subagent also receives specific upstream memory file paths. The merged view adds marginal value over the direct reads.

3. **Merge operations are sequential bottlenecks.** All merges target the same file (`memory.md`), so they cannot be parallelized. Each merge must read the current state, modify, and write. This serializes what should be a parallel pipeline.

4. **Token waste from redundant reads.** Each merge subagent reads `memory.md` (which grows throughout the pipeline as information accumulates) plus the source memory file. By Step 7, memory.md could be 500+ lines. Reading it 4 times for R-cluster merges alone costs ~8K tokens.

5. **Fragility.** If the merge subagent has a bug or formatting error, it corrupts memory.md for all downstream agents. The current fallback ("if memory.md is missing, proceed with direct artifact reads") already acknowledges this risk.

6. **The orchestrator can already read isolated memories directly.** The whole point of isolated memory files was to give the orchestrator compact, focused summaries for routing decisions. Adding a merge step to re-aggregate them into memory.md creates an unnecessary data hop: agent → `.mem.md` → merge subagent → `memory.md` → downstream subagent, when the shorter path is: agent → `.mem.md` → downstream subagent.

#### 4.2 Why Fully Removing Orchestrator Reads (Original Option A) Was Rejected

From [decisions.md](../../self-improvement-system/decisions.md) — "Tool Restriction: Keep Read Tools for Orchestrator":

> "Read tools are essential for orchestrator function — it cannot navigate the pipeline without reading `memory.md`, checking artifact existence, or orienting on workspace structure."

Specific functions that require `read_file`:
1. **Cluster Decision Logic** — the orchestrator reads `memory/<agent>.mem.md` files to extract `Highest Severity` and `Status` for CT/V/R cluster routing ([orchestrator.agent.md#L120-L187](../../../.github/agents/orchestrator.agent.md))
2. **Input Validation** — checking memory file existence and format before cluster evaluation ([orchestrator.agent.md#L122-L125](../../../.github/agents/orchestrator.agent.md))
3. **Wave parsing** — reading `plan.md` to extract execution wave groups ([orchestrator.agent.md#L320](../../../.github/agents/orchestrator.agent.md))
4. **Task agent detection** — reading the `agent` field from task files for dispatch routing ([orchestrator.agent.md#L326](../../../.github/agents/orchestrator.agent.md))

Without `read_file`, the orchestrator would need a subagent to read every file it needs — adding ~30+ subagent dispatches just for reads. This would be more expensive than memory merges.

#### 4.3 What Happens If We Over-Restrict the Orchestrator

| Over-restriction | Consequence |
|---|---|
| Remove `read_file` | Orchestrator cannot evaluate cluster decisions, parse waves, or detect task agents. Must delegate all reads to subagents (~30+ extra dispatches). |
| Remove `list_dir` | Orchestrator cannot check directory contents for task file enumeration or artifact existence verification. Must use subagents for file discovery. |
| Remove `memory` (VS Code cross-session) | Orchestrator loses ability to persist cross-session knowledge (build commands, project structure). Minor impact per-run, but degrades across sessions. |
| Keep `grep_search`/`semantic_search` | Orchestrator can do open-ended code exploration, which violates the coordinator-doesn't-do-work principle. Risk: orchestrator starts "helping" with implementation-level decisions instead of routing. |
| Remove all write tools (already done) | Correct restriction. Orchestrator already delegates all file writes to subagents. The anti-drift anchor enforces this. |

#### 4.4 Risk: Silent Degradation Without memory.md

If memory.md is eliminated (Pattern B), there's a risk that some subagents perform poorly due to missing cross-phase context, but this degradation is invisible because:
- Agents proceed without errors (fallback logic)
- Output quality decreases slightly but is hard to measure
- Lessons Learned from earlier waves don't propagate

**Mitigation:** Accumulated Lessons Learned could be passed as an explicit list in the dispatch prompt for later-phase agents (V cluster, R cluster). The orchestrator can read prior implementer `.mem.md` files and extract Lessons Learned bullet points to include in dispatch context — this is a read-only operation, no file write needed.

---

## File References

| File | Relevance |
|---|---|
| [.github/agents/orchestrator.agent.md](../../../.github/agents/orchestrator.agent.md) | Primary subject — memory lifecycle, merge operations, cluster decision logic, tool restrictions |
| [.github/agents/implementer.agent.md](../../../.github/agents/implementer.agent.md) | Subagent input pattern — reads memory.md + 3 upstream memories |
| [.github/agents/designer.agent.md](../../../.github/agents/designer.agent.md) | Subagent input pattern — reads memory.md + 5 upstream memories |
| [.github/agents/spec.agent.md](../../../.github/agents/spec.agent.md) | Subagent input pattern — reads memory.md + 4 researcher memories |
| [.github/agents/ct-security.agent.md](../../../.github/agents/ct-security.agent.md) | Subagent input pattern — reads memory.md + 2 upstream memories |
| [.github/agents/v-tasks.agent.md](../../../.github/agents/v-tasks.agent.md) | Subagent input pattern — reads memory.md + 2 upstream memories |
| [.github/agents/documentation-writer.agent.md](../../../.github/agents/documentation-writer.agent.md) | Already operates without memory.md dependency in its memory-first rule |
| [.github/agents/researcher.agent.md](../../../.github/agents/researcher.agent.md) | First-phase agent — no upstream memories to read |
| [.github/agents/planner.agent.md](../../../.github/agents/planner.agent.md) | Subagent input pattern — reads memory.md + 2 upstream memories + conditionals |
| [.github/prompts/feature-workflow.prompt.md](../../../.github/prompts/feature-workflow.prompt.md) | References memory.md in rules and key artifacts |
| [docs/feature/self-improvement-system/decisions.md](../../self-improvement-system/decisions.md) | Prior decision: "Keep Read Tools for Orchestrator" — directly relevant |

---

## Assumptions & Limitations

- **Assumption:** Subagent token budgets can accommodate reading 2–5 isolated `.mem.md` files (~100–250 tokens each) without significant impact.
- **Assumption:** The `memory` tool referenced in the orchestrator's tool list is VS Code's cross-session memory, NOT the pipeline's `memory.md` file operations. This distinction is stated in the initial request but not consistently clarified in the orchestrator prompt.
- **Limitation:** Could not empirically measure the token cost of memory.md reads vs. isolated memory reads — analysis is based on typical file sizes observed in the self-improvement-system feature run.
- **Limitation:** Did not analyze runtime latency of subagent dispatches because no timing data is available from the dispatch-patterns implementation.

---

## Open Questions

1. **Lessons Learned propagation:** Under Pattern B, how should Lessons Learned from early implementation waves reach later-pipeline agents (V cluster, R cluster)? Options: (a) orchestrator extracts and includes in dispatch prompt, (b) later agents read all implementer `.mem.md` files, (c) accept the gap.
2. **Pruning without memory.md:** The current pruning logic removes stale entries from memory.md at checkpoints. Without memory.md, is any pruning needed? Isolated `.mem.md` files are immutable (written once per agent) — so pruning is N/A.
3. **Invalidation on revision:** The current system invalidates memory.md entries when a revision loop starts. Without memory.md, invalidation means telling subagents "ignore memory from the previous V cluster run" — how is this communicated in dispatch prompts?
4. **Ambiguity: `memory` tool vs. `memory.md`:** The orchestrator prompt uses "memory tool" to refer to both VS Code's cross-session memory and the act of writing to `memory.md`. This needs disambiguation in the design.

---

## Research Metadata

- **confidence_level:** high — all 19 agent files examined, all memory merge points catalogued, prior decisions consulted
- **coverage_estimate:** Examined all active agent definitions (19 files), the orchestrator's full prompt (546 lines), feature-workflow.prompt.md, the decisions.md from self-improvement-system, and the initial request. Did not examine dispatch-patterns.md in detail (referenced but not critical for pattern analysis).
- **gaps:** Did not analyze the `dispatch-patterns.md` file for Pattern A/B/C error handling edge cases that might interact with memory architecture changes. Low impact — the dispatch patterns describe concurrency mechanics, not memory flow.
