# Technical Design: Orchestrator Tool Restriction (Phase 1)

**Summary:** Restrict the orchestrator agent's tool access to `[agent, agent/runSubagent, memory, read_file, list_dir]` via YAML frontmatter and prose enforcement. Remove 4 discovery/analysis tools (`grep_search`, `semantic_search`, `file_search`, `get_errors`) that are never used in deterministic pipeline logic. Fix pre-existing contradictions where the orchestrator prompt incorrectly instructs writing pipeline files via the VS Code `memory` tool — replace with subagent delegation. Keep all existing `memory.md` architecture intact. This is a focused, low-risk change to 2 files.

**Revision Note (Iteration 1):** This design was revised after CT cluster review (all 4 agents rated High). The primary CT finding — scope coupling between tool restriction and memory.md elimination — led to phasing the work. Phase 1 (this design) covers tool restriction only. Phase 2 (future feature) would cover memory architecture redesign if still desired.

**Revision Note (Iteration 2):** CT cluster re-run found 1 High and 2 Medium remaining issues. High: the 3 `memory` tool disambiguation statements directly contradict 5 preserved references that tell the orchestrator to write pipeline files via the `memory` tool (Global Rule 12, Step 0.1, Step 8.2, Memory Lifecycle Table). Fix: Phase 1 now also corrects these pre-existing contradictions by replacing "via `memory` tool" with subagent delegation. Medium: Anti-Drift prohibition rationale gap (read tools justified), feature.md scope mismatch (noted).

---

## Context & Inputs

| Source | Key Sections Used |
|--------|------------------|
| [feature.md](feature.md) | §FR-1 (Tool Restriction), §FR-6 (Memory Disambiguation), §AC-1 through AC-3, AC-11, AC-12 |
| [initial-request.md](initial-request.md) | Goals §1 (tool restriction), §3 (prompt section updates) |
| [research/architecture.md](research/architecture.md) | §1.1–§1.3 (YAML, tool refs), §6 (Anti-Drift) |
| [research/impact.md](research/impact.md) | §1 (tool removal — zero pipeline impact) |
| [research/dependencies.md](research/dependencies.md) | §1 (files referencing tools) |
| [research/patterns.md](research/patterns.md) | §2 (orchestrator purity) |
| [ct-review/ct-strategy.md](ct-review/ct-strategy.md) | §Finding 1 (scope coupling — High), §Finding 8 (no phasing strategy) |
| [ct-review/ct-scalability.md](ct-review/ct-scalability.md) | §Finding 5 (Lessons Learned non-functional), §Finding 1 (token underestimate) |
| [ct-review/ct-maintainability.md](ct-review/ct-maintainability.md) | §Finding 1 ("3 universal changes" variation), §Finding 2 (undocumented edit points) |
| [ct-review/ct-security.md](ct-review/ct-security.md) | §Finding 1 (prompt injection via Lessons Learned), §Finding 2 (advisory enforcement) |
| [ct-review/ct-maintainability.md](ct-review/ct-maintainability.md) (iter 2) | §New Findings — memory tool disambiguation contradiction (High), Anti-Drift rationale gap (Medium) |
| [ct-review/ct-security.md](ct-review/ct-security.md) (iter 2) | §Finding 1 — memory tool disambiguation contradiction (Medium, same root cause as ct-maintainability High) |
| [ct-review/ct-strategy.md](ct-review/ct-strategy.md) (iter 2) | §Findings — feature.md/design.md scope mismatch (Medium) |
| [ct-review/ct-scalability.md](ct-review/ct-scalability.md) (iter 2) | All prior findings resolved. Phase 2 process gating (Low) |
| [.github/agents/orchestrator.agent.md](../../.github/agents/orchestrator.agent.md) | Full 546-line orchestrator definition — primary modification target |
| [.github/prompts/feature-workflow.prompt.md](../../.github/prompts/feature-workflow.prompt.md) | Memory system rule (L24) — secondary modification target |

---

## 1. High-Level Architecture

### 1.1 Before vs. After

| Aspect | Before | After |
|--------|--------|-------|
| Orchestrator tools | 9 tools (incl. `grep_search`, `semantic_search`, `file_search`, `get_errors`) | 5 tools: `[agent, agent/runSubagent, memory, read_file, list_dir]` |
| Shared `memory.md` | Exists — orchestrator manages lifecycle | **Unchanged** — exists, lifecycle preserved |
| Memory architecture | Dual: file-per-agent + blackboard overlay (`memory.md`) | **Unchanged** |
| Merge operations | 10+ merge points per pipeline run | **Unchanged** (operations preserved; mechanism wording corrected from "via `memory` tool" to "via subagent delegation") |
| Merge mechanism wording | References "via `memory` tool" in Global Rule 12, Step 0.1, Step 8.2, Memory Lifecycle Table | Corrected to "via subagent delegation" — fixes pre-existing bug (memory tool cannot write files) |
| Subagent definitions | 20 agent files reference `memory.md` | **Unchanged** — not modified in Phase 1 |
| `memory` tool disambiguation | Not explicit; 5 references conflate `memory` tool with pipeline file writes | Explicit disambiguation in 3 locations + 5 contradicting references corrected |

### 1.2 Component Responsibilities (Unchanged)

| Component | Responsibility |
|-----------|---------------|
| **Orchestrator** | Dispatches subagents, manages memory.md lifecycle (init, merge, prune, invalidate), evaluates cluster decisions via `.mem.md` reads, accumulates telemetry |
| **Subagents** | Read `memory.md` + upstream `memory/*.mem.md` files, produce primary artifact + own `memory/<agent>.mem.md` |
| **VS Code `memory` tool** | Cross-session codebase knowledge store (e.g., build commands, project structure). NOT for pipeline file operations |

### 1.3 What Changes

The orchestrator loses 4 discovery tools it never uses in deterministic pipeline logic:

- `grep_search` — all orchestrator reads target known paths
- `semantic_search` — orchestrator doesn't do semantic discovery
- `file_search` — orchestrator doesn't search by filename pattern
- `get_errors` — orchestrator doesn't check compilation errors

All orchestrator reads already use `read_file` on deterministic paths (e.g., `memory/ct-security.mem.md`). The tool restriction formalizes what was already true in practice.

**Additionally (Iteration 2 fix):** Phase 1 now corrects 5 pre-existing references that incorrectly tell the orchestrator to write pipeline files via the VS Code `memory` tool. The `memory` tool is a cross-session knowledge store — it cannot create or modify files like `memory.md`. The existing prompt was broken: it instructed an operation that was impossible. Phase 1 fixes this by replacing all "via `memory` tool" merge/init language with explicit subagent delegation. The affected locations are:

- **Global Rule 12 (L44-46):** "via `memory` tool merging" and "via `memory` tool" → "by dispatching a subagent"
- **Step 0.1 (L188):** "Use the `memory` tool to create memory.md" → "Delegate to a setup subagent"
- **Step 8.2 (L456):** "using `memory` tool or delegating to subagent" → "by delegating to a subagent"
- **Memory Lifecycle Table, Initialize row (L503):** "Use `memory` tool to create" → "Delegate to a setup subagent to create"
- **All merge steps (1.1m, 2m, 3m, 3b.2, 4m, between-waves, 6.3, 7.3):** "orchestrator merges" → "orchestrator dispatches a subagent to merge" (and Step 1.1m removes "This is an orchestrator merge operation — no subagent invocation")

### 1.4 What Does NOT Change

- `memory.md` initialization, merge, prune, invalidation — all operations preserved (mechanism wording corrected; see §1.3)
- Memory Lifecycle Actions table — preserved (Initialize row wording corrected; see §3.8)
- Merge steps (1.1m, 2m, 3m, etc.) — all preserved (wording corrected from "orchestrator merges" to "orchestrator dispatches subagent to merge"; see §3.9)
- All 20 subagent definitions — untouched
- `dispatch-patterns.md` — untouched
- Lessons Learned mechanism — the existing `memory.md`-based accumulation is preserved (the broken keyword extraction heuristic from the pre-revision design is dropped; see §12.4)
- Self-verification logging to `memory.md` — preserved
- Invalidation via `[INVALIDATED]` file markers — preserved

---

## 2. Orchestrator YAML Frontmatter Change

### 2.1 Current Frontmatter

```yaml
---
name: orchestrator
description: Deterministic workflow orchestrator that coordinates all subagents, manages agent-isolated memory and memory merging, dispatches agent clusters, and enforces documentation structure.
---
```

### 2.2 New Frontmatter

```yaml
---
name: orchestrator
description: Deterministic workflow orchestrator that coordinates all subagents, manages agent-isolated memory and memory merging, dispatches agent clusters, and enforces documentation structure.
tools: [agent, agent/runSubagent, memory, read_file, list_dir]
---
```

### 2.3 Changes

1. **`description` unchanged:** The orchestrator still manages memory merging. No description change needed in Phase 1.
2. **`tools:` field added:** New YAML field listing the 5 allowed tools. This is the **first agent in the repository** to use a `tools:` YAML frontmatter field.

### 2.4 Convention: YAML `tools:` Field

- **Purpose:** Formal declaration of the agent's allowed tools. Serves as documentation and potentially advisory enforcement by VS Code runtime.
- **Enforcement:** The YAML `tools:` field is advisory — VS Code may not enforce it at runtime (ct-security Finding 2, ct-strategy Finding 7). The **prose restriction** in Operating Rule 5 is the primary enforcement mechanism. Both MUST list identical tools.
- **Scope:** Only the orchestrator uses this field. Other agents' tool access is managed via their prose Operating Rules. Future agents may adopt `tools:` as a convention.
- **Rationale:** The orchestrator is the highest-risk agent for tool misuse (coordinator doing worker tasks). Explicit YAML declaration makes the restriction visible without reading the full prompt.

---

## 3. Orchestrator Prompt Sections to Modify

### 3.1 Global Rule 1 (L33)

**Current:** "Never modify code, documentation, or any file directly — delegate all file creation and modification to subagents via `runSubagent`. Use read tools (`read_file`, `grep_search`, `semantic_search`, `file_search`, `list_dir`) freely for orchestration decisions."

**New:** "Never modify code, documentation, or any file directly — delegate all file creation and modification to subagents via `runSubagent`. Use `read_file` and `list_dir` for orchestration decisions on known paths. The `memory` tool is VS Code's cross-session knowledge store for codebase facts — it does NOT write to pipeline files."

**Rationale:** Removes 4 discovery tools from the allowed read-tools list. Adds `memory` tool disambiguation (FR-6.1). All other Global Rules (2–11, 13) are unchanged. Global Rule 12 changes separately (see §3.5).

### 3.2 Operating Rule 1 (L83)

**Current:** "**Context-efficient reading:** Prefer `semantic_search` and `grep_search` for discovery. Use targeted line-range reads with `read_file` (limit ~200 lines per call). Avoid reading entire files unless necessary."

**New:** "**Context-efficient reading:** Use `read_file` with known paths for all orchestration reads (memory files, plan.md, task files). Use `list_dir` for directory listing. Limit reads to ~200 lines per call. All orchestrator reads target deterministic known paths — no discovery tools are needed."

**Rationale:** Removes `semantic_search` and `grep_search` references. Aligns with the restricted tool set. All other Operating Rules (2–4) are unchanged. Rule 5 changes below.

### 3.3 Operating Rule 5 (L92)

**Current:** "**Tool access (write-restricted):** Allowed tools: `[agent, agent/runSubagent, memory, read_file, grep_search, semantic_search, file_search, list_dir, get_errors]`. The orchestrator MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, or `run_in_terminal`. All file creation and modification MUST be delegated to subagents via `runSubagent`."

**New:** "**Tool access (restricted):** Allowed tools: `[agent, agent/runSubagent, memory, read_file, list_dir]`. The orchestrator MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`, `grep_search`, `semantic_search`, `file_search`, or `get_errors`. All file creation and modification MUST be delegated to subagents via `runSubagent`. The `memory` tool is VS Code's cross-session knowledge store — it does NOT create or modify pipeline files."

**Rationale:** 4 tools moved from allowed to prohibited. `memory` tool disambiguated. Title changed from "write-restricted" to "restricted" since the restriction now covers read tools too. Matches YAML `tools:` field.

### 3.4 Anti-Drift Anchor (L541)

**Current:** "**REMEMBER:** You are the **Orchestrator**. You coordinate agents via runSubagent. You are the sole writer to shared `memory.md` — you merge agent-isolated memory files into shared `memory.md`. You manage the memory lifecycle (init, merge, prune, invalidate). You evaluate cluster results directly from sub-agent memories (CT, V, R decision flows). You dispatch the PostMortem agent at Step 8 (non-blocking) with accumulated telemetry data. You delegate all file creation and modification to subagents — you never create or edit files directly. You use read tools freely for cluster decisions and routing. **You MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, or `run_in_terminal` — all file writes are delegated to subagents.** You never skip pipeline steps. You never auto-apply knowledge suggestions. Stay as orchestrator."

**New:** "**REMEMBER:** You are the **Orchestrator**. You coordinate agents via `runSubagent`. You are the sole writer to shared `memory.md` — you dispatch subagents to merge agent-isolated memory files into shared `memory.md`. You manage the memory lifecycle (init, merge, prune, invalidate) via subagent delegation. You evaluate cluster results directly from sub-agent memories (CT, V, R decision flows). You dispatch the PostMortem agent at Step 8 (non-blocking) with accumulated telemetry data. You delegate all file creation and modification to subagents — you never create or edit files directly. You use `read_file` and `list_dir` for cluster decisions and routing. **You MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`, `grep_search`, `semantic_search`, `file_search`, or `get_errors` — all file writes are delegated to subagents; search/discovery tools are unnecessary because the orchestrator reads known deterministic paths only.** The `memory` tool is VS Code's cross-session knowledge store — not for pipeline files. You never skip pipeline steps. You never auto-apply knowledge suggestions. Stay as orchestrator."

**Changes:**
1. "use read tools freely" → "use `read_file` and `list_dir`" — specifies the exact allowed read tools
2. Prohibited tool list expanded to include the 4 removed tools. Justification updated: "all file writes are delegated to subagents; search/discovery tools are unnecessary because the orchestrator reads known deterministic paths only"
3. `memory` tool disambiguation added
4. Memory.md management language **preserved** — orchestrator remains "sole writer" (via subagent delegation, as corrected in §3.5)

---

### 3.5 Global Rule 12 — Memory Write Safety (L44-46)

**Current (L44):** "Only the orchestrator writes to shared `memory.md` (via `memory` tool merging). No concurrent writes possible."

**New:** "Only the orchestrator writes to shared `memory.md` (by dispatching a subagent to perform the merge). No concurrent writes possible."

**Current (L46):** "The orchestrator reads isolated memories and merges into `memory.md` (via `memory` tool) after each cluster/agent completes."

**New:** "The orchestrator reads isolated memories and dispatches a subagent to merge them into `memory.md` after each cluster/agent completes."

**Rationale:** The VS Code `memory` tool cannot write to pipeline files. The existing text was a pre-existing bug — it told the orchestrator to use a tool incapable of the described operation. Subagent delegation is how all orchestrator file writes actually work.

### 3.6 Step 0.1 — Initialize Memory (L188)

**Current:** "Use the `memory` tool to create `docs/feature/<feature-slug>/memory.md` with the empty template below. If the `memory` tool cannot create this file, delegate to a setup subagent via `runSubagent`."

**New:** "Delegate to a setup subagent via `runSubagent` to create `docs/feature/<feature-slug>/memory.md` with the empty template below. If the subagent fails, log a warning and proceed — memory is beneficial but not required."

**Rationale:** Removes the `memory` tool as the primary mechanism and the fallback framing. Subagent delegation was always the only functional path.

### 3.7 Step 8.2 — PostMortem Memory Merge (L456)

**Current:** "Orchestrator reads `memory/post-mortem.mem.md` (using `read_file`) and merges Key Findings, Decisions, and Artifact Index into `memory.md` (using `memory` tool or delegating to subagent)."

**New:** "Orchestrator reads `memory/post-mortem.mem.md` (using `read_file`) and dispatches a subagent to merge Key Findings, Decisions, and Artifact Index into `memory.md`."

**Rationale:** Removes the "using `memory` tool" option. Only subagent delegation is viable.

### 3.8 Memory Lifecycle Table — Initialize Row (L503)

**Current:** "Use `memory` tool to create `memory.md` with empty template (subagent fallback). Directories created lazily by first writing agent."

**New:** "Delegate to a setup subagent to create `memory.md` with empty template. Directories created lazily by first writing agent."

**Rationale:** Consistent with §3.6. Removes "subagent fallback" framing since subagent delegation is the primary (and only viable) mechanism.

### 3.9 Merge Steps — Wording Correction (1.1m, 2m, 3m, 3b.2, 4m, between-waves, 6.3, 7.3)

All merge steps currently say the orchestrator directly "merges" content into `memory.md`. Since the orchestrator has no file-write tools, this must be via subagent delegation. The following wording corrections apply:

| Step | Current Wording | New Wording |
|------|----------------|-------------|
| **1.1m** (L234) | "Merges Key Findings, Decisions, and Artifact Index from each into `memory.md`." | "Dispatches a subagent to merge Key Findings, Decisions, and Artifact Index from each into `memory.md`." |
| **1.1m** (L237) | "This is an orchestrator merge operation — no subagent invocation." | _(Remove this line entirely.)_ |
| **2m** (L253) | "Orchestrator reads `memory/spec.mem.md` and merges Key Findings, Decisions, and Artifact Index into `memory.md`." | "Orchestrator reads `memory/spec.mem.md` and dispatches a subagent to merge Key Findings, Decisions, and Artifact Index into `memory.md`." |
| **3m** (L265) | "Orchestrator reads `memory/designer.mem.md` and merges Key Findings, Decisions, and Artifact Index into `memory.md`." | "Orchestrator reads `memory/designer.mem.md` and dispatches a subagent to merge Key Findings, Decisions, and Artifact Index into `memory.md`." |
| **3b.2** (L288) | "Merge Key Findings, Decisions, and Artifact Index from all CT memories into `memory.md`." | "Dispatch a subagent to merge Key Findings, Decisions, and Artifact Index from all CT memories into `memory.md`." |
| **4m** (L308) | "Orchestrator reads `memory/planner.mem.md` and merges Key Findings, Decisions, and Artifact Index into `memory.md`." | "Orchestrator reads `memory/planner.mem.md` and dispatches a subagent to merge Key Findings, Decisions, and Artifact Index into `memory.md`." |
| **between-waves** (L331) | "Orchestrator reads implementer/documentation-writer memory files (...) and merges Key Findings, Decisions, and Artifact Index into `memory.md`. Extract Lessons Learned from memory files and append to `memory.md` Lessons Learned section." | "Orchestrator reads implementer/documentation-writer memory files (...) and dispatches a subagent to merge Key Findings, Decisions, and Artifact Index into `memory.md`. The subagent also extracts Lessons Learned from memory files and appends to `memory.md` Lessons Learned section." |
| **6.3** (L368) | "Merge Key Findings, Decisions, and Artifact Index from all V memories into `memory.md`." | "Dispatch a subagent to merge Key Findings, Decisions, and Artifact Index from all V memories into `memory.md`." |
| **7.3** (L401) | "Merge Key Findings, Decisions, and Artifact Index from all R memories into `memory.md`." | "Dispatch a subagent to merge Key Findings, Decisions, and Artifact Index from all R memories into `memory.md`." |

**Rationale:** The orchestrator has no file-write tools (`create_file`, `replace_string_in_file` are prohibited). All writes to `memory.md` must go through subagent delegation. The prior wording implied the orchestrator wrote directly, which was never possible. Step 1.1m explicitly said "no subagent invocation" which was incorrect. These are wording corrections to match reality, not behavioral changes — the merge operations continue to happen at the same points in the pipeline.

**Note:** The merge step headings ("1.1m Merge Research Memories", "2m Merge Spec Memory", etc.) are preserved. Only the body text changes.

---

## 4. Feature-Workflow Prompt Change

### 4.1 No Structural Changes to Memory System Rule

The `feature-workflow.prompt.md` L24 memory system rule currently says:

> "Each agent produces a compact isolated memory file (`memory/<agent-name>.mem.md`) alongside its primary artifact. The orchestrator reads these isolated memories for routing decisions, merges them into shared `memory.md` at checkpoints, and prunes to keep context manageable. No agent other than the orchestrator writes to `memory.md`."

This remains **accurate** in Phase 1 — memory.md is preserved. No change needed to this line.

### 4.2 Key Artifacts Table

The Key Artifacts table at L39 remains unchanged — `memory.md` is still a pipeline artifact.

### 4.3 Optional Clarification

Add a note to the Rules section of `feature-workflow.prompt.md` to clarify the orchestrator's restricted tool set:

> "The orchestrator uses only `read_file` and `list_dir` for reading (no `grep_search`, `semantic_search`, `file_search`, or `get_errors`). All orchestrator reads target known deterministic paths."

This is the only change to `feature-workflow.prompt.md`.

---

## 5. Data Models & DTOs

### 5.1 No Changes to Data Models

All existing data models are preserved:

- **`memory.md` format** — unchanged (Artifact Index, Recent Decisions, Lessons Learned, Recent Updates)
- **`memory/<agent>.mem.md` format** — unchanged (Status, Key Findings, Highest Severity, Decisions Made, Artifact Index)
- **Memory Lifecycle Actions table** — preserved (Initialize row wording corrected: "Use `memory` tool" → "Delegate to a setup subagent"; see §3.8)
- **Dispatch prompt structure** — unchanged (agents still receive `memory.md` as input)

---

## 6. APIs & Interfaces

### 6.1 No Changes to Dispatch Interface

The `runSubagent` dispatch interface is unchanged. Agents still receive `memory.md` as an input file. The orchestrator's tool restriction does not affect what it passes to subagents — it only affects what tools the orchestrator itself can use.

---

## 7. Sequence / Interaction Notes

### 7.1 Pipeline Flow — Unchanged

The pipeline flow remains identical. The only behavioral difference is that the orchestrator cannot use discovery tools for ad-hoc codebase exploration. Since all orchestrator reads already target known deterministic paths, this has zero functional impact on the pipeline.

Merge step wording is corrected ("orchestrator merges" → "orchestrator dispatches a subagent to merge") but this is a documentation fix, not a behavioral change. The orchestrator was always dispatching subagents for file writes.

```
Orchestrator
  ├─ Step 0: Setup → memory.md + initial-request.md (via subagent delegation)
  ├─ Step 1: Researcher ×4 (parallel) → orchestrator dispatches subagent to merge memories
  ├─ Step 2–3: Spec → Design (sequential, orchestrator dispatches subagent to merge after each)
  ├─ Step 3b: CT ×4 (parallel) → orchestrator CT evaluation → subagent merge
  ├─ Step 4: Planning (sequential, orchestrator dispatches subagent to merge)
  ├─ Step 5: Implementation waves → orchestrator dispatches subagent to merge memories between waves
  ├─ Step 6: V-Build (gate) → V ×3 (parallel) → orchestrator V evaluation → subagent merge (Pattern C)
  ├─ Step 7: R ×4 (parallel) → orchestrator R evaluation → subagent merge
  └─ Step 8: Post-Mortem (non-blocking) → subagent merge
```

All merge, prune, and lifecycle operations proceed exactly as before (via subagent delegation).

---

## 8. Security Considerations

### 8.1 Authentication/Authorization

**N/A.** This change modifies agent prompt definitions (markdown files). No authentication, authorization, or access control is involved.

### 8.2 Data Protection

**N/A.** No sensitive data is introduced or exposed.

### 8.3 Threat Model

| Threat | Likelihood | Impact | Mitigation |
|--------|-----------|--------|-----------|
| LLM interprets `memory` tool as file writer | Medium | High | Explicit disambiguation in 3 prompt locations (Global Rule 1, Operating Rule 5, Anti-Drift Anchor) **plus** removal of all 5 conflating references that previously told the orchestrator to use `memory` tool for file writes (Global Rule 12, Step 0.1, Step 8.2, Memory Lifecycle Table). Phase 1 both adds disambiguation AND removes contradictions — a net-positive over the pre-existing state. |
| Orchestrator uses prohibited tools (grep_search, etc.) | Low | Low | YAML `tools:` field (advisory) + prose restriction in Operating Rule 5 + prohibited tool list + Anti-Drift Anchor. The removed tools are read-only and non-destructive — worst case is the orchestrator does unnecessary exploration. |
| LLM ignores tool restriction entirely | Low | Medium | Triple-layered enforcement (YAML + Operating Rule + Anti-Drift). No runtime enforcement exists (ct-security Finding 2). This is a known limitation of all LLM-based tool restrictions across all agents. |

### 8.4 Input Validation

No changes. The orchestrator's existing Input Validation for cluster decisions is preserved.

---

## 9. Failure & Recovery

### 9.1 Expected Failure Modes

| Failure Mode | Probability | Impact | Recovery Strategy |
|-------------|------------|--------|-------------------|
| **LLM ignores tool restriction** | Low | Low | Advisory enforcement only. The removed tools are read-only, so unauthorized use causes no data corruption. Anti-Drift Anchor reinforces. |
| **LLM misuses `memory` tool for file ops** | Low | Medium | Triple disambiguation in prompt, plus all 5 conflating references corrected. Pre-existing risk that Phase 1 resolves by both adding disambiguation and removing contradictions. |

### 9.2 Graceful Degradation

If the LLM ignores the tool restriction and uses `grep_search` or `semantic_search`, the pipeline continues to function correctly — these are read-only tools. The restriction is about architectural purity (coordinator should not do worker-level exploration), not about preventing damage.

### 9.3 Retry/Fallback

No changes to any retry or fallback logic. All existing mechanisms are preserved.

---

## 10. Non-Functional Requirements

### 10.1 Performance

**Zero performance impact.** The tool restriction removes tools the orchestrator never uses in pipeline logic. No token cost change. No latency change.

### 10.2 Token Cost — Corrected Estimates from CT Review

The original design's token estimates for the proposed (now-deferred) Pattern B memory elimination were inaccurate. For the record, the CT scalability review (ct-scalability Finding 1) found:

| Metric | Original Design Estimate | CT-Measured Reality |
|--------|-------------------------|---------------------|
| `.mem.md` file size | ~100–250 tokens each | 150–1,180 tokens (avg 488) |
| PostMortem input with Pattern B | "equivalent" to memory.md | 5.2× increase (~16K vs ~3K) |
| Per-agent upstream read cost (3 files) | ~300–750 tokens | ~1,464 tokens |
| Net token effect of Pattern B | "approximately neutral" | Token-negative (higher cost) |

**These numbers are irrelevant to Phase 1** (which preserves memory.md), but are documented here for Phase 2 planning.

### 10.3 Constraints

- YAML `tools:` field enforcement is advisory (platform-dependent).
- Prose restriction in Operating Rule 5 is the primary enforcement mechanism.
- The `memory` tool (VS Code cross-session) must be clearly distinguished from pipeline memory files.

---

## 11. Migration & Backwards Compatibility

### 11.1 No Migration Required

The tool restriction is purely additive — it removes capabilities the orchestrator never used. No data migration, no schema changes, no backward compatibility concerns.

### 11.2 Backwards Compatibility

- Existing pipelines in progress are unaffected.
- Memory.md format and lifecycle are unchanged.
- All subagent definitions are unchanged.
- No partial-update risk — only 2 files change.

---

## 12. Testing Strategy

### 12.1 Static Verification Tests

| Test ID | What to Check | Maps to AC | Method |
|---------|--------------|------------|--------|
| T-1 | `orchestrator.agent.md` YAML frontmatter contains `tools: [agent, agent/runSubagent, memory, read_file, list_dir]` | AC-1 | Inspect YAML frontmatter |
| T-2 | Operating Rule 5 lists exactly the same 5 tools | AC-2 | Read Operating Rule 5 |
| T-3 | Zero matches for `grep_search`, `semantic_search`, `file_search`, `get_errors` in `orchestrator.agent.md` | AC-3 | Text search |
| T-4 | Anti-Drift Anchor references `read_file` and `list_dir` (not old tool list), lists 4 removed tools in prohibited list | AC-12 | Read Anti-Drift Anchor |
| T-5 | `memory` tool disambiguation exists in Global Rule 1, Operating Rule 5, and Anti-Drift Anchor | AC-11 | Text search for "cross-session" |
| T-6 | `memory.md` references are **preserved** (not removed) throughout orchestrator prompt | N/A | Confirm no regression to memory lifecycle |
| T-7 | `feature-workflow.prompt.md` contains orchestrator tool restriction note | N/A | Read feature-workflow.prompt.md |
| T-8 | Global Rule 12 no longer says "via `memory` tool" | AC-11 | Text search in Global Rule 12 section |
| T-9 | Step 0.1 no longer says "Use the `memory` tool to create" | AC-11 | Read Step 0.1 |
| T-10 | Step 8.2 no longer says "using `memory` tool" | AC-11 | Read Step 8.2 |
| T-11 | Memory Lifecycle Table Initialize row no longer says "Use `memory` tool" | AC-11 | Read Memory Lifecycle table |
| T-12 | All merge steps use "dispatches a subagent to merge" wording | N/A | Text search for "dispatches a subagent" in merge steps |
| T-13 | Zero matches for "via `memory` tool" in `orchestrator.agent.md` | AC-11 | Text search |

### 12.2 Negative Tests

| Test ID | What to Check | Expected Result |
|---------|--------------|-------------------|
| T-N1 | Memory Lifecycle Actions table still exists | Present (Initialize row wording updated, all other rows unmodified) |
| T-N2 | Merge steps (1.1m, 2m, 3m, etc.) still exist | Present (wording corrected to subagent delegation, headings preserved) |
| T-N3 | All 20 subagent files are unmodified | No changes to any subagent |
| T-N4 | `dispatch-patterns.md` is unmodified | No changes |

### 12.3 Integration Test

After implementation, a full pipeline run should verify:
1. Pipeline produces all expected outputs.
2. `memory.md` is created and managed normally.
3. The orchestrator does not use `grep_search`, `semantic_search`, `file_search`, or `get_errors`.

---

## 13. Tradeoffs & Alternatives Considered

### 13.1 Full Scope (Tool Restriction + Memory Elimination) — Deferred

The original design bundled tool restriction with memory.md elimination (Pattern B). CT review identified this as scope coupling (ct-strategy Finding 1, High severity):

- Tool restriction: 3-section edit to 1 file (orchestrator.agent.md) + 1 supporting file
- Memory elimination: 23-file blast radius with significant per-agent variation

These are logically independent. The tool restriction doesn't change the memory.md write equation because the orchestrator never had file-write tools — it already delegated all writes to subagents or used the `memory` tool (which, as ct-strategy Finding 3 notes, couldn't actually write files anyway).

### 13.2 Lessons Learned Keyword Heuristic — Removed

The original design proposed a keyword extraction algorithm (§13.2) that searched implementer `.mem.md` files for "Issue:", "→ Resolution:", "Problem:", "Workaround:", "Lesson:" patterns. CT scalability (Finding 5) measured this against real data: **zero matches** across 12 implementer `.mem.md` files. The mechanism was dead on arrival.

Since Phase 1 preserves memory.md, the existing Lessons Learned accumulation in memory.md continues to work. The broken heuristic is simply not needed.

If Phase 2 proceeds with memory.md elimination, a viable Lessons Learned mechanism would need to either:
- (a) Add a required `## Lessons Learned` section to the `.mem.md` format schema, or
- (b) Pass full `.mem.md` contents in dispatch prompts (costly but reliable), or
- (c) Accept that Lessons Learned propagation is optional context and may be lost

### 13.3 Advisory vs. Runtime Enforcement

CT security (Finding 2) and ct-strategy (Finding 7) both note that the YAML `tools:` field is advisory-only with no runtime enforcement. This is accepted as a known limitation:

- The YAML field serves as documentation and convention
- The prose restriction in Operating Rule 5 is the primary enforcement
- The Anti-Drift Anchor reinforces the restriction
- The removed tools are read-only — unauthorized use causes exploration overhead, not data corruption
- This is consistent with how all 20 other agents enforce their tool boundaries (prose only)

---

## 14. CT Findings Disposition

### 14.1 Findings Addressed by Phase 1

| CT Finding | Source | Severity | How Addressed |
|------------|--------|----------|---------------|
| Scope coupling — independent changes bundled | ct-strategy §1 | High | **Resolved.** Phased into Phase 1 (tool restriction) and Phase 2 (memory redesign). |
| Context window fragility | ct-strategy §2, ct-security §3 | High | **Resolved.** Memory.md preserved — self-verification, warnings, lessons all remain in persistent storage. |
| "3 universal changes" masks per-agent variation | ct-maintainability §1 | High | **Resolved.** Phase 1 does not touch any subagent files. |
| PostMortem degradation (5.2× token increase) | ct-scalability §2 | High | **Resolved.** Memory.md preserved — PostMortem still reads one aggregated file. |
| Lessons Learned extraction non-functional | ct-scalability §5 | Medium | **Resolved.** Keyword heuristic dropped. Existing memory.md accumulation preserved. |
| Post-mortem undocumented edit points | ct-maintainability §2 | High | **Resolved.** Post-mortem agent is not modified in Phase 1. |
| Orchestrator context window pressure | ct-scalability §3 | High | **Resolved.** No additional context window load — memory.md handles persistence. |
| Memory file count O(tasks) scaling | ct-scalability §4 | High | **Resolved.** Memory.md pruning is preserved. |
| Prompt injection via Lessons Learned | ct-security §1 | High | **Resolved.** The broken keyword extraction mechanism is not implemented. |
| Invalidation weakened by dispatch-only text | ct-strategy §6, ct-security §5, ct-maintainability §5 | Medium | **Resolved.** File-based `[INVALIDATED]` markers preserved. |
| Token cost underestimate | ct-scalability §1 | High | **Documented.** Correct numbers recorded in §10.2 for Phase 2 planning. |

### 14.2 Findings Addressed by Existing Architecture (No Action Needed)

| CT Finding | Source | Severity | Status |
|------------|--------|----------|--------|
| Advisory-only tool enforcement | ct-security §2, ct-strategy §7 | High/Low | **Accepted.** Consistent with all other agents. YAML + prose + anchor is the strongest available enforcement. |
| Memory tool conflation risk | ct-security §4, ct-maintainability iter 2 (High), ct-security iter 2 (Medium) | Medium→High | **Resolved.** Phase 1 now adds disambiguation in 3 locations AND corrects all 5 contradicting references (Global Rule 12, Step 0.1, Step 8.2, Memory Lifecycle Table, merge step 1.1m). See §3.5–§3.9. |
| Stale path in orchestrator (L96) | ct-security §8 | Low | **Out of scope** for this feature — pre-existing issue unrelated to tool restriction. |

### 14.3 Findings Deferred to Phase 2

| CT Finding | Source | Severity | Deferral Rationale |
|------------|--------|----------|--------------------|
| Subagent memory.md references (20 files) | ct-maintainability §1 | High | Requires 20-file mass edit — deferred to Phase 2 (memory architecture redesign). |
| Memory Lifecycle table removal | original design §3.8 | N/A | Lifecycle is preserved in Phase 1. Phase 2 would remove it. |
| Context window as sole persistent storage | ct-strategy §2 | High | Not applicable — memory.md provides persistent storage in Phase 1. |
| PostMortem reads 20+ files | ct-scalability §2 | High | Not applicable — PostMortem still reads memory.md in Phase 1. |
| Planner undocumented memory.md reference | ct-maintainability §3 | Medium | Planner is not modified in Phase 1. |
| N+1 read pattern | ct-scalability §6 | Medium | Not applicable — merge pattern preserved. |

### 14.4 CT Iteration 2 Findings — Addressed in This Revision

| CT Finding | Source | Severity | How Addressed |
|------------|--------|----------|---------------|
| Memory tool disambiguation contradicts preserved merge references | ct-maintainability iter 2, ct-security iter 2 | High / Medium | **Resolved.** All 5 contradicting references corrected to subagent delegation (§3.5–§3.9). Disambiguation + correction = no contradictions remain. |
| Anti-Drift prohibition rationale gap (read tools not covered by "all file writes are delegated") | ct-maintainability iter 2 | Medium | **Resolved.** Justification updated: "all file writes are delegated to subagents; search/discovery tools are unnecessary because the orchestrator reads known deterministic paths only" (§3.4 Change #2). |
| feature.md describes full 23-file scope with all 15 ACs but Phase 1 implements only 5 ACs | ct-strategy iter 2 | Medium | **Noted.** feature.md should be updated to reflect Phase 1 scope (see §16.5). This is a spec-maintenance task, not a design issue. The v-feature agent should verify against the Phase 1 AC subset listed in §16.3, not the full feature.md AC list. |
| Optional workflow doc drift risk | ct-maintainability iter 2 | Low | **Accepted.** The `feature-workflow.prompt.md` change remains optional. Drift risk is minimal since memory.md is preserved. |
| Phase 2 process gating | ct-scalability iter 2 | Low | **Accepted.** Phase 2 CT-measured numbers documented in §10.2 but no enforcement mechanism. Phase 2 is a separate feature that would have its own planning. |

---

## 15. Phase 2 — Future Work (Memory Architecture Redesign)

Phase 2 is a **separate future feature** that would address eliminating `memory.md` (Pattern B). It is NOT part of this implementation. The following captures context for future planning:

### 15.1 Scope

- Eliminate `memory.md` — remove initialization, merge, prune, invalidation
- Update all 20 subagent definitions (remove `memory.md` from Inputs, update Operating Rule 6, update Anti-Drift Anchors)
- Update `dispatch-patterns.md`
- Design a viable Lessons Learned propagation mechanism (not the broken keyword heuristic)
- Address PostMortem token scaling (5.2× increase documented in §10.2)
- Address context window pressure for self-verification logging

### 15.2 Prerequisites

- Phase 1 must be implemented and validated first
- A proper `.mem.md` schema revision may be needed (e.g., add `## Lessons Learned` section)
- Context window sizing analysis for orchestrator operational state
- Per-agent variation analysis for the 20-file mass edit (ct-maintainability evidence shows 4+ input format variants, 8+ Operating Rule 6 variants, 3+ Anti-Drift variants)

### 15.3 CT Findings to Incorporate

All deferred findings from §14.3, plus:
- ct-maintainability Finding 1: Document exact per-agent wording variants before designing universal changes
- ct-scalability Finding 1: Use real token costs (avg 488 per `.mem.md`) in all analysis
- ct-strategy Finding 3: Acknowledge memory.md was already broken in practice (orchestrator couldn't actually write to it via `memory` tool) — elimination formalizes existing reality

---

## 16. Implementation Checklist & Deliverables

### 16.1 Files to Modify

| # | File | Changes | Priority |
|---|------|---------|----------|
| 1 | `.github/agents/orchestrator.agent.md` | YAML `tools:` field, Global Rule 1, Operating Rule 1, Operating Rule 5, Anti-Drift Anchor, Global Rule 12 (memory tool fix), Step 0.1 (memory tool fix), Step 8.2 (memory tool fix), Memory Lifecycle Table (memory tool fix), all merge steps 1.1m/2m/3m/3b.2/4m/between-waves/6.3/7.3 (merge wording fix) | Critical |
| 2 | `.github/prompts/feature-workflow.prompt.md` | Add orchestrator tool restriction note to Rules section | Low |

### 16.2 Files NOT Modified

- All 20 subagent `.agent.md` files — unchanged (deferred to Phase 2)
- `.github/agents/dispatch-patterns.md` — unchanged (no tool references to update)
- `.github/agents/evaluation-schema.md` — unchanged
- No new files created

### 16.3 Acceptance Criteria Mapping (Phase 1 Subset)

| AC | Implementation Section | Status |
|----|----------------------|--------|
| AC-1 (YAML tools field) | §2 (YAML Frontmatter) | In scope |
| AC-2 (Prose matches YAML) | §3.3 (Operating Rule 5) | In scope |
| AC-3 (Removed tool refs eliminated) | §3.1, §3.2, §3.3, §3.4 | In scope |
| AC-11 (Memory tool disambiguated) | §3.1, §3.3, §3.4, §3.5–§3.9 | In scope |
| AC-12 (Anti-Drift updated) | §3.4 | In scope |
| AC-4 (No memory.md refs in orchestrator) | N/A | **Deferred** — memory.md is preserved |
| AC-5 (No memory.md in subagents) | N/A | **Deferred** to Phase 2 |
| AC-6 (No memory.md in supporting docs) | N/A | **Deferred** to Phase 2 |
| AC-7 (Lifecycle table removed) | N/A | **Deferred** — table preserved |
| AC-8 (Merge steps removed) | N/A | **Deferred** — merges preserved |
| AC-9 (Upstream memory paths) | N/A | **Deferred** to Phase 2 |
| AC-10 (Lessons Learned propagation) | N/A | **Deferred** — existing mechanism preserved |
| AC-13 (Cluster decisions unaffected) | §7.1 | In scope (no changes needed) |
| AC-14 (Self-verification preserved) | §7.1 | In scope (no changes needed) |
| AC-15 (Subagent fallback preserved) | N/A | **Deferred** — subagents untouched |

### 16.4 Implementation Order

1. **Add YAML `tools:` field** to orchestrator frontmatter (AC-1)
2. **Update Global Rule 1** — remove discovery tools, add `memory` disambiguation (AC-3, AC-11)
3. **Update Operating Rule 1** — remove `semantic_search`/`grep_search` preference (AC-3)
4. **Update Operating Rule 5** — restrict tool list, add `memory` disambiguation (AC-2, AC-3, AC-11)
5. **Update Anti-Drift Anchor** — specify `read_file`/`list_dir`, expand prohibited list, add `memory` disambiguation, fix prohibition rationale (AC-12, AC-11)
6. **Fix Global Rule 12** — replace "via `memory` tool" with subagent delegation (AC-11)
7. **Fix Step 0.1** — replace "Use the `memory` tool" with subagent delegation (AC-11)
8. **Fix Step 8.2** — replace "using `memory` tool" with subagent delegation (AC-11)
9. **Fix Memory Lifecycle Table** — replace "Use `memory` tool" with subagent delegation (AC-11)
10. **Fix all merge steps** — replace "orchestrator merges" with "orchestrator dispatches a subagent to merge" (wording correction)
11. **Update feature-workflow.prompt.md** — add tool restriction note (optional)
12. **Verify** — run static tests T-1 through T-13 and T-N1 through T-N4

### 16.5 Feature Spec Scope Note

**Important:** The `feature.md` currently describes the full scope (23 files, 15 ACs) including both tool restriction and memory.md elimination. Phase 1 implements only a subset (2 files, 7 ACs: AC-1, AC-2, AC-3, AC-11, AC-12, AC-13, AC-14). The v-feature verification agent should verify against the Phase 1 AC subset listed in §16.3, not the full feature.md AC list. If feature.md is not updated before implementation, the implementer and verifier should be explicitly instructed that AC-4 through AC-10 and AC-15 are deferred to Phase 2.

---

## 17. Edge Cases Summary

| Edge Case | Design Response |
|-----------|----------------|
| LLM uses prohibited tool (`grep_search`, etc.) | Advisory enforcement only. Removed tools are read-only — no data corruption possible. Anti-Drift Anchor reinforces restriction. |
| `memory` tool used for file operations | Triple disambiguation (Global Rule 1, Operating Rule 5, Anti-Drift) + all 5 contradicting references corrected to subagent delegation. No conflating language remains in the orchestrator prompt. |
| Subagent still references `memory.md` | Expected — subagents are not modified in Phase 1. Memory.md is preserved, so references remain valid. |
| Pipeline run with new orchestrator, old feature-workflow | No conflict — memory.md is preserved, tool restriction is orchestrator-internal. |

---

## Assumptions

1. The VS Code `memory` tool is strictly for cross-session codebase knowledge and CANNOT write to arbitrary file paths.
2. The orchestrator's 4 removed tools (`grep_search`, `semantic_search`, `file_search`, `get_errors`) are never used in deterministic pipeline logic — all reads target known paths.
3. Advisory enforcement (YAML + prose + Anti-Drift) is sufficient for read-only tool restrictions where unauthorized use is non-destructive.
4. Phase 2 (memory architecture redesign) may or may not proceed — Phase 1 delivers independent value regardless.
