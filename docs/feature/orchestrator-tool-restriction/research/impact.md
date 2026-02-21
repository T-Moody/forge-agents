# Research: Impact Analysis

**Focus Area:** impact
**Summary:** Removing `grep_search`, `semantic_search`, `file_search`, and `get_errors` from the orchestrator has low pipeline impact since all orchestrator use cases involve known-path reads. Removing `memory.md` write access has high impact — 7 Memory Lifecycle Actions, 3 cluster self-verification logs, and 3 warning-log sites must be redesigned or delegated. All subagents already have fallback behavior for missing `memory.md`.

---

## 1. Tool Removal Impact

### 1.1 `grep_search` — References in Orchestrator Prompt

| Location | Line(s) | Context | Can `read_file`/`list_dir` replace it? |
|---|---|---|---|
| Global Rules §1 | L33 | Listed in the "read tools" set: `Use read tools (read_file, grep_search, semantic_search, file_search, list_dir) freely for orchestration decisions.` | Yes — orchestrator reads known-path files (memory files, plan.md, task files). No discovery needed. |
| Operating Rules §1 | L83 | `Prefer semantic_search and grep_search for discovery. Use targeted line-range reads with read_file.` | Yes — this is a reading efficiency guideline. The orchestrator reads structured memory files at fixed paths; `read_file` with line ranges is sufficient. |
| Operating Rules §5 | L92 | Listed in the allowed tools set: `[agent, agent/runSubagent, memory, read_file, grep_search, semantic_search, file_search, list_dir, get_errors]` | N/A — this line must be updated to the new tool set. |
| Anti-Drift Anchor | L541 | `You use read tools freely for cluster decisions and routing.` (implicit reference to all read tools) | Yes — update to reference only `read_file` and `list_dir`. |

**What breaks if removed without replacement:** Nothing in the actual pipeline logic. The orchestrator never uses `grep_search` for decision-making — all reads target known paths (e.g., `memory/ct-security.mem.md`, `plan.md`, `tasks/01-*.md`). The reference in Operating Rules §1 is an efficiency guideline that can be replaced with `read_file` line-range guidance.

### 1.2 `semantic_search` — References in Orchestrator Prompt

| Location | Line(s) | Context | Can `read_file`/`list_dir` replace it? |
|---|---|---|---|
| Global Rules §1 | L33 | Listed in the "read tools" set (same reference as grep_search) | Yes — same reasoning. |
| Operating Rules §1 | L83 | `Prefer semantic_search and grep_search for discovery.` | Yes — orchestrator has no discovery use case; all file paths are deterministic. |
| Operating Rules §5 | L92 | Listed in the allowed tools set | N/A — must be updated. |

**What breaks if removed without replacement:** Nothing. The orchestrator follows a deterministic pipeline with known file paths. `semantic_search` is a discovery tool useful for research/review agents exploring codebases, but the orchestrator never discovers — it reads pre-defined artifact paths.

### 1.3 `file_search` — References in Orchestrator Prompt

| Location | Line(s) | Context | Can `read_file`/`list_dir` replace it? |
|---|---|---|---|
| Global Rules §1 | L33 | Listed in the "read tools" set | Yes — `list_dir` serves any directory listing need. |
| Operating Rules §5 | L92 | Listed in the allowed tools set | N/A — must be updated. |

**What breaks if removed without replacement:** Nothing. `file_search` (glob-pattern file search) is never used in any orchestrator workflow step. The orchestrator reads known paths. For task discovery (Step 5.1), `list_dir` on the `tasks/` directory suffices.

### 1.4 `get_errors` — References in Orchestrator Prompt

| Location | Line(s) | Context | Can `read_file`/`list_dir` replace it? |
|---|---|---|---|
| Operating Rules §5 | L92 | Listed in the allowed tools set | N/A — must be removed. |

**What breaks if removed without replacement:** Nothing for the orchestrator itself. `get_errors` is for checking compile/lint errors — a subagent concern (V-Build, Implementer). The orchestrator never uses `get_errors` in any workflow step. All error detection is delegated to V-Build and Implementer agents.

### 1.5 Tool Removal Summary

| Tool | References | Used in pipeline logic? | Breaks without replacement? | Action |
|---|---|---|---|---|
| `grep_search` | 3 mentions (Global Rules, Operating Rules ×2) | No | No | Remove references, update tool list |
| `semantic_search` | 3 mentions (Global Rules, Operating Rules ×2) | No | No | Remove references, update tool list |
| `file_search` | 2 mentions (Global Rules, Operating Rules) | No | No | Remove references, update tool list |
| `get_errors` | 1 mention (Operating Rules) | No | No | Remove from tool list |

**Conclusion:** All four tools are referenced in 2–3 prompt locations but are not used in any procedural pipeline step. Removal is safe. Only prompt text needs updating — no behavioral impact.

---

## 2. Memory Write Removal Impact

The orchestrator currently writes to `memory.md` in the following contexts:

### 2.1 Comprehensive Inventory of `memory.md` Write Operations

| Operation | Prompt Location | Line(s) | Description |
|---|---|---|---|
| **Initialize** | Step 0 §1 | L188 | Create `memory.md` with empty template using `memory` tool (or delegate to setup subagent) |
| **Merge (research)** | Step 1.1m | L232–235 | Read 4 researcher `*.mem.md` files → merge Key Findings, Decisions, Artifact Index into `memory.md` |
| **Merge (spec)** | Step 2m | L253 | Read `memory/spec.mem.md` → merge into `memory.md`. Prune. |
| **Merge (designer)** | Step 3m | L265 | Read `memory/designer.mem.md` → merge into `memory.md` |
| **Merge (CT cluster)** | Step 3b.2 §3 | L288 | Merge from all CT memories into `memory.md` |
| **Merge (planner)** | Step 4m | L308 | Read `memory/planner.mem.md` → merge into `memory.md`. Prune. |
| **Merge (between waves)** | Step 5.2 between-waves | L331 | Read implementer/documentation-writer memories → merge + extract Lessons Learned |
| **Merge (V cluster)** | Step 6.3 §3 | L368 | Merge from all V memories into `memory.md` |
| **Merge (R cluster)** | Step 7.3 §3 | L401 | Merge from all R memories into `memory.md` |
| **Merge (post-mortem)** | Step 8.2 | L456 | Read `memory/post-mortem.mem.md` → merge into `memory.md` |
| **Log warning (malformed memory)** | Cluster Decision Logic — Input Validation §3 | L125 | Log `WARNING: memory/<agent>.mem.md missing/malformed` in `memory.md` Recent Updates |
| **Self-verification (CT)** | CT Cluster Decision Flow §6 | L136 | Log CT decision in `memory.md` Recent Updates |
| **Self-verification (V)** | V Cluster Decision Flow §6 | L160 | Log V decision in `memory.md` Recent Updates |
| **Self-verification (R)** | R Cluster Decision Flow §7 | L176 | Log R decision in `memory.md` Recent Updates |
| **Log result (post-mortem)** | Step 8.2 | L458 | Log PostMortem dispatch result in `memory.md` Recent Updates |
| **Prune** | Steps 1.1m, 2m, 4m | L235, L253, L308 | Remove old Recent Decisions and Recent Updates |
| **Invalidate** | Before revision dispatch | L507 | Mark entries with `[INVALIDATED — <reason>]` |
| **Clean invalidated** | After revision agent completes | L508 | Remove `[INVALIDATED]` entries not replaced |

**Total: 18 distinct write operations across the pipeline.**

### 2.2 Impact on Memory Lifecycle Actions

| Lifecycle Action | Current Owner | Impact if Orchestrator Can Only Read | Can Subagent Perform? |
|---|---|---|---|
| **Initialize** | Orchestrator (via `memory` tool or subagent) | Cannot initialize. Must delegate. | Yes — setup subagent already documented as fallback (L188: "delegate to a setup subagent via `runSubagent`") |
| **Merge** | Orchestrator (10 merge points) | Cannot merge. This is the highest-frequency operation. | Yes — a dedicated memory-merge subagent could be dispatched after each agent/cluster completes. Orchestrator reads `*.mem.md` files, passes them as context to merge subagent, merge subagent writes to `memory.md`. |
| **Prune** | Orchestrator (3 prune points) | Cannot prune. | Yes — prune logic is deterministic (remove entries older than 2 phases). A subagent can perform this. Could be combined with merge subagent. |
| **Extract Lessons** | Orchestrator (between implementation waves) | Cannot extract and append. | Yes — merge subagent can include lesson extraction in its merge operation. |
| **Invalidate** | Orchestrator (before revision dispatch) | Cannot mark entries as invalidated. | Yes — a subagent can add `[INVALIDATED]` markers. More complex: orchestrator must pass invalidation context. |
| **Clean invalidated** | Orchestrator (after revision) | Cannot clean. | Yes — merge subagent can clean during next merge. |
| **Validate** | Orchestrator (check file exists) | Still possible via `read_file`. Validation is read-only. | N/A — this is a read operation, stays with orchestrator. |

**Key finding:** The "Validate" action is read-only and unaffected. All 6 write actions (Initialize, Merge, Prune, Extract, Invalidate, Clean) must either be delegated to subagents or the architecture must change.

### 2.3 What Happens If Orchestrator Can Only Read Memory Files

**Option A (delegate writes to subagent):**
- Each merge point becomes a subagent dispatch: orchestrator reads isolated memories, passes findings as context to a memory-writer subagent, subagent writes `memory.md`.
- **Cost:** 10+ additional subagent invocations per pipeline run (one per merge point). This significantly increases latency and token usage.
- **Benefit:** `memory.md` stays current; subagents that read it get the same benefit.

**Option B (eliminate `memory.md` — read isolated memories directly):**
- Orchestrator reads `memory/*.mem.md` directly using `read_file` for cluster decisions. Already does this.
- Subagents that currently read `memory.md` first would instead read upstream `*.mem.md` files directly (they already have fallback for this).
- **Cost:** Slightly more reads per agent (reading multiple `*.mem.md` instead of one merged `memory.md`). Loss of Artifact Index aggregation.
- **Benefit:** No additional subagent dispatches. Simpler architecture. Eliminates merge/prune/invalidate complexity entirely.

**Option C (hybrid — auto-assembled by subagent):**
- A dedicated memory-merge subagent is invoked at key checkpoints (e.g., after Step 1.1, after Step 3b, after Step 5 waves) to assemble `memory.md` from all current `*.mem.md` files.
- **Cost:** 3–5 additional subagent invocations. Less than Option A.
- **Benefit:** `memory.md` stays mostly current; fewer dispatches than Option A.

---

## 3. Impact on Cluster Decision Flows

### 3.1 Reading `## Highest Severity` from Memory Files

**Current behavior:** The orchestrator reads individual `memory/<agent>.mem.md` files and extracts `Highest Severity` values. This uses `read_file` on known paths.

**Impact of tool restriction:** **None.** `read_file` is retained. All cluster decision reads target known paths:
- CT: `memory/ct-security.mem.md`, `memory/ct-scalability.mem.md`, `memory/ct-maintainability.mem.md`, `memory/ct-strategy.mem.md`
- V: `memory/v-build.mem.md`, `memory/v-tests.mem.md`, `memory/v-tasks.mem.md`, `memory/v-feature.mem.md`
- R: `memory/r-security.mem.md`, `memory/r-quality.mem.md`, `memory/r-testing.mem.md`, `memory/r-knowledge.mem.md`

### 3.2 Logging Warnings to `memory.md`

**Current behavior:** Three types of writes to `memory.md`:
1. **Input Validation warnings** (L125): `[orchestrator, step-N] WARNING: memory/<agent>.mem.md missing/malformed — treating as worst-case.`
2. **Self-verification logs** (L136, L160, L176): CT/V/R cluster decision logging
3. **Post-mortem result log** (L458)

**Impact of write removal:** **All 5 logging sites break.** The orchestrator cannot write these logs.

**Alternatives:**
- **Option B approach (no `memory.md`):** Self-verification stays in orchestrator context window (already captured by telemetry tracking per Global Rule 13). Warnings logged to context window only. No persistent log needed — PostMortem agent receives full telemetry at Step 8.
- **Option A/C approach:** Warning logs could be included in the next merge subagent dispatch.
- **Minimal approach:** Self-verification is already implicitly captured by the orchestrator's telemetry context tracking (Rule 13). Making it also a `memory.md` write is redundant. The cluster decision is the actual action; the log is observability.

### 3.3 Self-Verification of Decisions

**Current behavior:** After each cluster decision, the orchestrator logs the decision to `memory.md` for auditability.

**Impact:** Self-verification logic (reading memories, applying decision tables) is all **read-only** and works with just `read_file`. The **logging** component breaks. But the decision-making is unaffected.

**Finding:** The orchestrator can still make all cluster decisions correctly. Only the persistent logging of those decisions to `memory.md` is lost. This is mitigated by:
1. Telemetry Context Tracking (Rule 13) already captures all dispatch metadata
2. PostMortem agent receives full telemetry at Step 8
3. Individual `*.mem.md` files preserve the source data

---

## 4. Impact on Wave Parsing and Task Dispatch

### 4.1 Reading `plan.md`

**Current behavior:** Step 5.1 reads `plan.md` to extract execution wave groups using `read_file`.

**Impact:** **None.** `read_file` is retained. `plan.md` is at a known path (`docs/feature/<feature-slug>/plan.md`).

### 4.2 Reading Task Files for `agent` Field

**Current behavior:** Step 5.2 reads each task file to get the `agent` field. Uses `read_file` on known paths (`tasks/01-*.md`, `tasks/02-*.md`, etc.).

**Impact:** **None.** `read_file` is retained.

### 4.3 Task Discovery via `list_dir`

**Current behavior:** The orchestrator needs to know which task files exist in `tasks/`. Two approaches:
1. Parse `plan.md` which lists all task files in execution waves
2. Use `list_dir` on `tasks/` directory

**Impact:** `list_dir` is retained. However, `plan.md` already contains the full task list organized by wave, so `list_dir` is only needed as a fallback verification. The primary task discovery mechanism is reading `plan.md`.

**Finding:** `list_dir` stays in the tool set and covers all directory listing needs. No impact on task dispatch.

---

## 5. Downstream Agent Impact

### 5.1 Which Agents Reference `memory.md`?

Every subagent (19 total, excluding orchestrator) has a "memory-first reading" rule that says:
> `Read memory.md FIRST before accessing any artifact.`

**All 19 agents have explicit fallback behavior:**
> `If memory.md is missing, log a warning and proceed with direct artifact reads.`

This is documented in every agent's Operating Rules §6.

### 5.2 Agents Affected

| Agent Category | Agents | How they use `memory.md` | Fallback if missing |
|---|---|---|---|
| Researchers (×4) | researcher-{architecture,impact,dependencies,patterns} | Read Artifact Index for orientation (early pipeline — `memory.md` is mostly empty at this point anyway) | Direct artifact reads (L41) |
| Spec | spec | Read Artifact Index + Recent Decisions | Direct artifact reads from researcher memories |
| Designer | designer | Read Artifact Index + Recent Decisions | Direct artifact reads (L59) |
| CT cluster (×4) | ct-{security,scalability,maintainability,strategy} | Read Artifact Index + Decisions for design context | Read upstream `memory/designer.mem.md`, `memory/spec.mem.md` directly (L46) |
| Planner | planner | Read Artifact Index + Decisions + Lessons Learned | Read upstream memories directly (L64, L93) |
| Implementers (×N) | implementer, documentation-writer | Read Artifact Index + Decisions + Lessons Learned | Read upstream `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md` directly (L52) |
| V cluster (×4) | v-{build,tests,tasks,feature} | Read Artifact Index for orientation | Direct artifact reads (L37, L41, L48, L49) |
| R cluster (×4) | r-{quality,security,testing,knowledge} | Read Artifact Index + Decisions + Lessons Learned | Direct artifact reads + upstream memories (L42, L43, L45, L48) |
| Post-Mortem | post-mortem | Read Artifact Index for context | Direct artifact reads (L49) |

### 5.3 Impact if Merges Are Delayed or Skipped

**Low impact** for all agents. Reasons:

1. **Fallback is universal:** Every agent already handles `memory.md` missing gracefully. They fall back to reading upstream `*.mem.md` files directly and primary artifacts directly.

2. **`memory.md` is a convenience, not a necessity:** It aggregates the Artifact Index from multiple agents into one file. Without it, agents read 2–5 upstream `*.mem.md` files instead of 1 `memory.md` file — this is marginally more reads but functionally equivalent.

3. **Critical context is in `*.mem.md` files:** Each agent's `*.mem.md` file contains:
   - Status (DONE/NEEDS_REVISION/ERROR)
   - Key Findings (≤5 bullets)
   - Highest Severity
   - Artifact Index (for that agent's outputs)
   - Decisions Made

4. **Orchestrator's cluster decisions don't depend on `memory.md`:** The CT, V, and R decision flows read individual `*.mem.md` files, not `memory.md`.

5. **The most impactful loss is the aggregated Artifact Index:** `memory.md` provides a single-file Artifact Index spanning all completed agents. Without it, agents must check individual memory files to find section references. This is a navigation efficiency loss, not a correctness issue.

### 5.4 R-Knowledge Special Case

R-Knowledge (L27) specifically says:
> `feature.md`, `design.md`, and `plan.md` are accessed via Artifact Index navigation — not read in full. Use the Artifact Index in `memory.md` or upstream memory files.`

This agent has the strongest dependency on the aggregated Artifact Index. However, it also accepts "upstream memory files" as an alternative source, and is non-blocking — its failure doesn't affect the pipeline.

### 5.5 Subagent Input Specifications Referencing `memory.md`

The orchestrator prompt specifies `memory.md` as an input to subagents in these steps:

| Step | Agent(s) | Input reference | Impact if `memory.md` absent |
|---|---|---|---|
| 1.1 | Researchers ×4 | "Each agent receives: `initial-request.md`, `memory.md`, its assigned focus area" (L227) | Minimal — `memory.md` is nearly empty at Step 1 |
| 2 | Spec | Input includes `memory.md` (L246) | Low — spec already reads researcher `*.mem.md` files |
| 3 | Designer | Input includes `memory.md` (L258) | Low — designer reads upstream memories |
| 3b | CT ×4 | "Each receives: `initial-request.md`, `design.md`, `feature.md`, `memory.md`" (L280) | Low — CT agents read `memory/designer.mem.md`, `memory/spec.mem.md` |
| 4 | Planner | Input includes `memory.md` (L299) | Low — planner reads upstream memories |
| 5.2 | Implementers | "Each agent receives: ... `memory.md`, upstream memories" (L326) | Low — implementers read upstream memories |
| 6.1 | V-Build | Inputs include `memory.md` (L344) | Low — v-build reads `memory/planner.mem.md` |
| 6.2 | V ×3 | "Inputs (additional to memory.md)" (L354) | Low — V agents read `memory/v-build.mem.md` |
| 7.2 | R ×4 | "Inputs (additional to memory.md)" (L386) | Low — R agents read upstream memories |
| 7.2 | R-Knowledge | Specifically: "tier, initial-request.md, memory.md" (L391) | Medium — R-Knowledge most relies on aggregated Artifact Index |

---

## File References

| File/Folder | Relevance |
|---|---|
| [.github/agents/orchestrator.agent.md](.github/agents/orchestrator.agent.md) | Primary target — all tool references, memory lifecycle, cluster decision flows |
| [.github/prompts/feature-workflow.prompt.md](.github/prompts/feature-workflow.prompt.md) | References memory system, `memory.md` in Key Artifacts |
| [.github/agents/implementer.agent.md](.github/agents/implementer.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/designer.agent.md](.github/agents/designer.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/planner.agent.md](.github/agents/planner.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/spec.agent.md](.github/agents/spec.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/researcher.agent.md](.github/agents/researcher.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/r-knowledge.agent.md](.github/agents/r-knowledge.agent.md) | Strongest dependency on aggregated Artifact Index from `memory.md` |
| [.github/agents/r-quality.agent.md](.github/agents/r-quality.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/r-security.agent.md](.github/agents/r-security.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/r-testing.agent.md](.github/agents/r-testing.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/post-mortem.agent.md](.github/agents/post-mortem.agent.md) | Reads `memory.md` for context; has fallback |
| [.github/agents/ct-security.agent.md](.github/agents/ct-security.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/ct-scalability.agent.md](.github/agents/ct-scalability.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/ct-maintainability.agent.md](.github/agents/ct-maintainability.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/ct-strategy.agent.md](.github/agents/ct-strategy.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/v-build.agent.md](.github/agents/v-build.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/v-tests.agent.md](.github/agents/v-tests.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/v-tasks.agent.md](.github/agents/v-tasks.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/v-feature.agent.md](.github/agents/v-feature.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/documentation-writer.agent.md](.github/agents/documentation-writer.agent.md) | Reads `memory.md` for orientation; has fallback |
| [.github/agents/dispatch-patterns.md](.github/agents/dispatch-patterns.md) | Defines Pattern A/B/C dispatch logic |

---

## Assumptions & Limitations

1. **Assumption:** The `memory` tool in the orchestrator's current tool list refers to VS Code's built-in cross-session memory tool, not a custom file-writing tool. The orchestrator appears to use it as a file-writing mechanism for `memory.md`, but this is an overloaded use.
2. **Assumption:** All subagent `*.mem.md` files use a consistent format with `## Highest Severity`, `## Key Findings`, `## Artifact Index` sections, as documented in agent prompts.
3. **Limitation:** Did not analyze the actual execution behavior of `memory` tool usage; analysis is prompt-based only.
4. **Limitation:** Did not examine whether the orchestrator has ever used `grep_search`, `semantic_search`, `file_search`, or `get_errors` in practice — analysis is based on prompt declarations only.

---

## Open Questions

1. **Memory architecture decision:** Should `memory.md` be eliminated entirely (Option B), delegated to a subagent (Option A), or maintained via checkpoint subagent (Option C)? The impact analysis shows all three are viable; Option B has the lowest complexity and relies on already-existing fallback behavior.
2. **Self-verification persistence:** Is losing persistent self-verification logs acceptable, given telemetry context tracking (Rule 13) already captures this and PostMortem receives it at Step 8?
3. **Aggregated Artifact Index value:** How much do agents actually benefit from the merged Artifact Index vs. reading upstream `*.mem.md` files directly? R-Knowledge is the strongest dependent.
4. **Subagent prompt updates:** If `memory.md` is eliminated, should all 19 subagent prompts be updated to remove `memory.md` references, or is the existing fallback behavior sufficient?

---

## Research Metadata

- **confidence_level:** high — the orchestrator prompt and all 20 agent prompts were exhaustively searched for tool references and `memory.md` usage patterns
- **coverage_estimate:** ~95% — all agent definition files examined; all prompt references to removed tools and `memory.md` catalogued; dispatch-patterns.md existence verified
- **gaps:** Did not examine runtime execution logs to verify whether the orchestrator actually uses `grep_search`/`semantic_search`/`file_search`/`get_errors` in practice. Based solely on prompt declarations. Did not deeply analyze `dispatch-patterns.md` content (only verified its existence).
