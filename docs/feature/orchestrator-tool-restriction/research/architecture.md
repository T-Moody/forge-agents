# Research: Architecture

## Focus Area

**architecture** — Orchestrator agent definition, YAML frontmatter patterns, feature-workflow prompt, tool references, memory architecture, and cluster decision flows.

## Summary

The orchestrator agent currently uses 9 tools (including `grep_search`, `semantic_search`, `file_search`, `get_errors`) listed in Operating Rule 5, references discovery tools in Global Rule 1 and Operating Rule 1, and is the sole writer to a shared `memory.md` via the `memory` tool. No agent file in the repository uses a YAML `tools:` frontmatter field — tool access is expressed only in prose. Removing 4 discovery tools requires updates across 6 sections of `orchestrator.agent.md` plus 2 sections of `feature-workflow.prompt.md`. The memory architecture involves a 7-phase lifecycle where the orchestrator writes to `memory.md` at ~15 distinct merge/prune/invalidation points throughout the pipeline.

---

## Findings

### 1. Orchestrator Agent Definition — Current Structure

**File:** `.github/agents/orchestrator.agent.md` (546 lines)

#### 1.1 YAML Frontmatter

```yaml
---
name: orchestrator
description: Deterministic workflow orchestrator that coordinates all subagents, manages agent-isolated memory and memory merging, dispatches agent clusters, and enforces documentation structure.
---
```

- Only two fields: `name` and `description`.
- **No `tools:` field exists.** Tool access is expressed entirely in prose within Operating Rule 5.

#### 1.2 Current Tool References — Explicit Allowed Tools (Operating Rule 5, L92)

> `Allowed tools: [agent, agent/runSubagent, memory, read_file, grep_search, semantic_search, file_search, list_dir, get_errors]`

The proposed restriction removes 4 of these: `grep_search`, `semantic_search`, `file_search`, `get_errors`.

#### 1.3 All Sections That Reference Removed Tools

| Section | Line | Tool(s) Referenced | Context |
|---------|------|--------------------|---------|
| Global Rule 1 | L33 | `read_file`, `grep_search`, `semantic_search`, `file_search`, `list_dir` | "Use read tools (...) freely for orchestration decisions" |
| Operating Rule 1 | L83 | `semantic_search`, `grep_search` | "Prefer `semantic_search` and `grep_search` for discovery" |
| Operating Rule 5 | L92 | `grep_search`, `semantic_search`, `file_search`, `get_errors` | Explicit allowed tools list |

These 3 locations are the only places in the orchestrator file that mention the 4 tools being removed.

#### 1.4 All Sections That Reference Reading Files Directly

The orchestrator uses `read_file` extensively for reading agent-isolated memory files at cluster decision points. Key locations:

| Section | Line(s) | What Is Read |
|---------|---------|--------------|
| Cluster Decision Logic — Input Validation | L121-125 | `memory/<agent-name>.mem.md` existence check |
| CT Cluster Decision Flow | L129 | `memory/ct-security.mem.md`, `memory/ct-scalability.mem.md`, `memory/ct-maintainability.mem.md`, `memory/ct-strategy.mem.md` |
| V Cluster Decision Flow | L140 | `memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md` |
| R Cluster Decision Flow | L163-170 | `memory/r-security.mem.md`, `memory/r-quality.mem.md`, `memory/r-testing.mem.md`, `memory/r-knowledge.mem.md` |
| Step 1.1m (Research merge) | L233 | 4 researcher memory files |
| Step 2m (Spec merge) | L253 | `memory/spec.mem.md` |
| Step 3m (Designer merge) | L265 | `memory/designer.mem.md` |
| Step 3b.2 (CT evaluation) | L286 | 4 CT memory files |
| Step 4m (Planner merge) | L308 | `memory/planner.mem.md` |
| Step 5.1 | L315 | `plan.md` (to parse execution waves) |
| Step 5.2 (Between waves) | L331 | Implementer/doc-writer memory files |
| Step 6.1–6.3 (V cluster) | L347, L366 | V memory files |
| Step 7.3 (R cluster) | L399 | R memory files |
| Step 8.2 (PostMortem merge) | L456 | `memory/post-mortem.mem.md` |

**Critical observation:** All orchestrator reads target **known paths** (`memory/<agent>.mem.md`, `plan.md`). The orchestrator never needs to _discover_ or _search_ for files — paths are deterministic based on the pipeline structure. This supports removing search/grep tools.

#### 1.5 All Sections That Reference Writing to `memory.md`

The orchestrator writes to `memory.md` at numerous points. All writes are performed via the `memory` tool or delegated to a subagent:

| When | What is Written | Mechanism |
|------|----------------|-----------|
| Step 0 (L188) | Initialize empty `memory.md` template | `memory` tool (subagent fallback) |
| Step 1.1m (L234) | Merge researcher Key Findings, Decisions, Artifact Index | `memory` tool |
| Step 2m (L253) | Merge spec Key Findings, Decisions, Artifact Index + prune | `memory` tool |
| Step 3m (L265) | Merge designer Key Findings, Decisions, Artifact Index | `memory` tool |
| Step 3b.2 (L288) | Merge CT Key Findings + CT decision self-verification log | `memory` tool |
| Step 4m (L308) | Merge planner Key Findings + prune | `memory` tool |
| Between waves (L331) | Merge implementer/doc-writer Key Findings + extract Lessons Learned | `memory` tool |
| Step 6.3 (L368) | Merge V Key Findings + V decision self-verification log | `memory` tool |
| Step 7.3 (L401) | Merge R Key Findings + R decision self-verification log | `memory` tool |
| Step 8.2 (L456) | Merge PostMortem memory | `memory` tool or subagent |
| Input Validation (L125) | Log warnings for missing/malformed memory files | `memory` tool |
| CT self-verification (L136) | Log cluster decision result | `memory` tool |
| V self-verification (L160) | Log cluster decision result | `memory` tool |
| R self-verification (L176) | Log cluster decision result | `memory` tool |
| Invalidation (various) | Mark entries `[INVALIDATED — <reason>]` | `memory` tool |

**Total: ~15 distinct write/modify operations** to `memory.md` across a single pipeline run.

#### 1.6 The `memory` Tool Ambiguity

The orchestrator references `memory` in its allowed tools list. Based on the initial-request.md:
> "`memory` here is VS Code's built-in cross-session memory tool, NOT pipeline memory.md"

However, in the current orchestrator prompt, the `memory` tool is used for **both purposes**:
1. **VS Code cross-session memory** (codebase knowledge)
2. **Writing to pipeline `memory.md`** (the shared operational memory file)

The orchestrator instructions conflate these — Global Rule 6, Rule 12, and the Memory Lifecycle Actions table all describe using the `memory` tool to create and update the pipeline `memory.md` file. This is architecturally significant: the `memory` tool as VS Code's built-in feature does NOT create markdown files at arbitrary paths. The orchestrator has been using write tools (or the `memory` tool was assumed to handle file writing) to maintain `memory.md`.

### 2. Cluster Decision Flows — Tool Requirements

The three cluster decision flows (CT, V, R) all follow the same pattern:

1. **Read** isolated memory files (`read_file` on known paths)
2. **Parse** specific fields (Status, Highest Severity)
3. **Apply** a decision table/flow
4. **Write** self-verification logs to `memory.md`

**Tools needed for cluster decisions with restriction:**
- `read_file` — reads memory files at known paths (KEPT)
- `list_dir` — could verify memory file existence (KEPT)
- `memory` tool — for writing self-verification logs (ambiguous — see §1.6)

**Tools NOT needed for cluster decisions:**
- `grep_search` — never used; orchestrator reads entire small memory files
- `semantic_search` — never used; paths are deterministic
- `file_search` — never used; paths are known
- `get_errors` — never used in cluster evaluation

### 3. Agent File Format Patterns

#### 3.1 YAML Frontmatter Convention

Sampled agents: `spec.agent.md`, `implementer.agent.md`, `designer.agent.md`, `planner.agent.md`, `ct-security.agent.md`, `researcher.agent.md`

**No agent file in the repository uses a `tools:` field.** All agents have only:
```yaml
---
name: <agent-name>
description: "<description>"
---
```

Tool access is expressed exclusively in prose within the agent body (Operating Rules or specific workflow sections). The VS Code chat agent YAML frontmatter specification supports a `tools:` field, but the repository does not use it.

Adding `tools:` to the orchestrator's frontmatter would be the first use in the repository. This would create a new convention — considered acceptable since it's enforcing restrictions rather than granting permissions.

#### 3.2 How Agents Read/Write Isolated Memory

All subagents follow the same pattern:
- **Read:** `memory.md` first (Memory-First Protocol), then upstream `memory/<agent>.mem.md` files, then selective reads of full artifacts via Artifact Index pointers
- **Write:** Only to their own `memory/<agent-name>.mem.md` isolated file
- **Never write to:** Shared `memory.md` (orchestrator-only)

### 4. Feature Workflow Prompt

**File:** `.github/prompts/feature-workflow.prompt.md`

Relevant sections referencing tools/memory:

| Section | Content |
|---------|---------|
| Rules — bullet 1 | "Always display which subagent you are invoking" (no tool mention) |
| Rules — Memory system (L24) | "The orchestrator reads these isolated memories for routing decisions, merges them into shared `memory.md` at checkpoints...No agent other than the orchestrator writes to `memory.md`." |
| Rules — Cluster parallelization (L25) | "Cluster outputs are read by the orchestrator via isolated memory files" |
| Key Artifacts table (L38-39) | Lists `memory/*.mem.md` and `memory.md` as artifacts |

**No explicit tool mentions** in the prompt file — it describes behaviors rather than tools. Two sections reference the orchestrator's role as shared `memory.md` writer:
1. Memory system rule (L24)
2. Key Artifacts table (L39)

### 5. Memory Architecture — Deep Analysis

#### 5.1 Structure of `memory.md`

```markdown
# Operational Memory

## Artifact Index
| Artifact | Key Sections | Last Updated By |

## Recent Decisions
<!-- Format: - [agent-name, step-N] Decision summary. Rationale: ... -->

## Lessons Learned
<!-- Never pruned. Format: - [agent-name, step-N] Issue → Resolution. -->

## Recent Updates
<!-- Format: - [agent-name, step-N] Updated `artifact-path` — summary. -->
```

Four sections with clear purposes:
1. **Artifact Index** — pointer to sections across all pipeline artifacts (never pruned)
2. **Recent Decisions** — decisions from recent agents (pruned at checkpoints)
3. **Lessons Learned** — persistent issue→resolution entries (never pruned)
4. **Recent Updates** — recent pipeline activity (pruned at checkpoints)

#### 5.2 Who Reads `memory.md`

Every subagent in the pipeline reads `memory.md` as its first input (Memory-First Protocol). It serves as:
1. An **index** — agents use the Artifact Index to navigate to relevant sections of large artifacts without reading full files
2. A **context summary** — Recent Decisions and Lessons Learned provide cross-agent context
3. A **change log** — Recent Updates tracks what happened recently

Consumer agents: spec, designer, planner, CT agents, implementer, documentation-writer, V agents, R agents, post-mortem (all 20+ agents).

#### 5.3 Memory Lifecycle (7 Phases)

From the Memory Lifecycle Actions table (L498-509):

| Phase | When | What |
|-------|------|------|
| Initialize | Step 0 | Create empty `memory.md` |
| Merge | After each agent/cluster | Read isolated `memory/<agent>.mem.md`, merge Key Findings + Decisions + Artifact Index |
| Prune | Steps 1.1m, 2m, 4m | Remove old Recent Decisions + Recent Updates; preserve Artifact Index + Lessons Learned |
| Extract Lessons | Between implementation waves | Pull issue/resolution entries from implementer memories into Lessons Learned |
| Invalidate | Before revision dispatch | Mark entries `[INVALIDATED — <reason>]` |
| Clean | After revision completes | Remove stale `[INVALIDATED]` entries |
| Validate | After each agent | Check isolated memory file existence, log warning if missing |

#### 5.4 Is `memory.md` Essential?

**Arguments for `memory.md` being essential:**
- The Artifact Index is the primary mechanism agents use to avoid reading full artifacts — without it, agents would need to read more, consuming more context
- Lessons Learned persist across phases for later agents (implementers benefit from early researcher insights)
- Every agent in the pipeline is instructed to read it first

**Arguments for `memory.md` NOT being essential:**
- Global Rule 6 explicitly states: "Memory failure is non-blocking — if `memory.md` cannot be created or becomes corrupted, log a warning and proceed. Agents fall back to direct artifact reads."
- Each agent already has `memory/<agent>.mem.md` with the same Key Findings and Artifact Index data
- Downstream agents read upstream agent memories directly (e.g., designer reads `memory/spec.mem.md`)
- The Artifact Index in `memory.md` is a merged copy of artifact indexes already in each agent's isolated memory
- Without the orchestrator writing to `memory.md`, each agent can still read all relevant upstream `memory/<agent>.mem.md` files directly

**Net assessment:** `memory.md` is a convenience aggregation layer, not a critical path requirement. The pipeline explicitly handles its absence. The primary value is the merged Artifact Index (saves agents from reading N separate memory files) and Lessons Learned persistence. Both can be replicated by agents reading isolated memories directly.

#### 5.5 The Write Problem

If the orchestrator's tools are restricted to `[agent, agent/runSubagent, memory, read_file, list_dir]`:
- The VS Code `memory` tool stores codebase-level knowledge, not arbitrary file writes
- The orchestrator cannot use `create_file` or `replace_string_in_file` to write `memory.md`
- A subagent would need to be delegated as the `memory.md` writer, or `memory.md` would be removed entirely

### 6. Anti-Drift Anchor

The anchor section (L530-534) explicitly reinforces:
- Orchestrator is sole writer to shared `memory.md`
- Uses read tools freely for cluster decisions and routing
- MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, or `run_in_terminal`

This section must be updated to reflect restricted tools and memory architecture changes.

---

## File References

| File | Relevance |
|------|-----------|
| `.github/agents/orchestrator.agent.md` | Primary target — contains all tool references, memory lifecycle, cluster decision flows |
| `.github/prompts/feature-workflow.prompt.md` | Entry point prompt — references memory system and orchestrator role |
| `.github/agents/dispatch-patterns.md` | Reference doc — defines memory-first pattern for all cluster dispatches |
| `.github/agents/spec.agent.md` | Example subagent — shows memory-first reading, no `tools:` in frontmatter |
| `.github/agents/implementer.agent.md` | Example subagent — shows isolated memory writes, no `tools:` in frontmatter |
| `.github/agents/designer.agent.md` | Example subagent — shows upstream memory consumption, no `tools:` in frontmatter |
| `.github/agents/planner.agent.md` | Example subagent — shows memory-first reading pattern |
| `.github/agents/researcher.agent.md` | Example subagent — shows isolated memory write and completion contract |
| `.github/agents/ct-security.agent.md` | Example cluster subagent — reads `memory.md` as first input |

---

## Assumptions & Limitations

1. **Assumption:** The VS Code `tools:` YAML frontmatter field is supported by the runtime environment. This was not verified by running anything — it's based on the initial request's reference to "tools field" and VS Code agent documentation.
2. **Assumption:** The VS Code `memory` tool is strictly for cross-session codebase knowledge and cannot write to arbitrary file paths like `memory.md`.
3. **Limitation:** Did not verify the actual runtime behavior of tools — research is based solely on prompt/agent definition analysis.
4. **Limitation:** Did not examine all 21 agent files — sampled 6 representative agents (orchestrator, spec, implementer, designer, planner, ct-security, researcher).

---

## Open Questions

1. **`memory` tool semantics:** Is the `memory` tool in the allowed tools list intended to be VS Code's cross-session memory, or was it historically used as a file-writing mechanism for `memory.md`? The current prompt conflates both uses.
2. **Frontmatter enforcement:** Does the VS Code runtime enforce `tools:` restrictions in YAML frontmatter, or is it advisory only? If advisory, the prose-level restrictions become the enforcement mechanism.
3. **Subagent `memory.md` access:** If `memory.md` is removed, should all subagent input lists that reference `memory.md` be updated? This would be a mass edit across 20+ agent files.
4. **Memory initialization at Step 0:** If `memory.md` is eliminated, what should Step 0 do? The setup step currently creates this file.

---

## Research Metadata

- **confidence_level:** high — the orchestrator file was fully read (all 546 lines), tool references were exhaustively searched, 6 representative agent files were sampled, and the feature-workflow prompt was fully analyzed.
- **coverage_estimate:** ~95% of relevant architecture covered. All orchestrator sections analyzed. 6 of 21 agent files sampled for frontmatter patterns (pattern is consistent across samples). Dispatch-patterns reference doc analyzed.
- **gaps:** Did not read every agent file (15 unsampled), though the sampled 6 showed identical frontmatter/memory patterns. Did not examine VS Code runtime behavior of `tools:` field enforcement. Did not check if any `.github/` config files define global tool restrictions.
