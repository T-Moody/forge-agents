# Technical Design: Forge Architecture Upgrade

## Revision History

| Date       | Revision                                 | Trigger                                                                                                |
| ---------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| 2026-02-17 | **R1: Address critical review findings** | `design_critical_review.md` returned NEEDS_REVISION with 3 critical issues and 6 significant concerns. |

**Changes in R1:**

1. **[Critical Fix] Concurrent memory writes redesigned (§1).** Parallel sub-agents (CT ×4, V ×3, R ×4, researchers ×4, implementers ×N) **no longer write to `memory.md`**. Only aggregators and sequential agents write to memory. Sub-agents communicate findings through their intermediate output files (which they already produce). The "Cluster Workspaces" section is removed from `memory.md`. This resolves the fundamental concurrency flaw: VS Code's `replace_string_in_file` tool cannot support concurrent modifications to the same file by multiple agents.
2. **[Critical Fix] Canonical agent location clarified (§1.1, §11, Appendices A–B).** `.github/agents/` is the VS Code Copilot runtime's canonical discovery location. `NewAgentsAndPrompts/` is the development/authoring directory. Both exist with matching content. The design now targets `.github/agents/` as the primary deployment location, with `NewAgentsAndPrompts/` kept in sync. All file inventories updated.
3. **[Critical Fix] Baseline line counts corrected (§3.1, §4.1, §5.1, §7, Appendix D).** Actual measured line counts: critical-thinker=86 (was 139), verifier=109 (was 179), reviewer=130 (was 220), orchestrator=234 (was 323). Orchestrator post-upgrade size re-projected to ~384–434 lines. Line cap revised from 550 to 450.
4. **[Significant] Aggregator overhead budget revised (§2.5).** Changed from ≤25% to 40–60% of longest sub-agent time, acknowledging semantic deduplication and cross-cutting synthesis require LLM reasoning. Speedup projections revised accordingly.
5. **[Significant] CT fragmentation tradeoff acknowledged (§3.1).** Added explicit acknowledgment that holistic cross-domain analysis may suffer. Design notes that 2–3 larger sub-agents is a viable alternative to evaluate during implementation.
6. **[Significant] End-to-end speedup revised (Design Overview, §7).** Changed from 15–20% to 10–18% to account for ~20 additional agent invocation overhead.
7. **[Significant] V cluster replan speedup clarified (§4).** Explicit acknowledgment that V cluster provides minimal speedup (~1.0–1.25×) during replan iterations where build failures dominate.
8. **[Minor] Emergency pruning preserves Artifact Index (§1.6).** Emergency pruning now retains Artifact Index alongside Lessons Learned.
9. **[Minor] CT intermediate files moved to `ct-review/` (§1.1, §3).** CT intermediate outputs stored in `ct-review/` rather than `research/` to avoid mixing Step 1 and Step 3b outputs.
10. **[Minor] Unresolved tensions forwarding specified (§3.3).** Unresolved tensions surviving the max-1 CT loop are forwarded to the planner as explicit planning constraints.
11. **[Minor] Memory-first reading is optional at Step 1 (§8.1).** Added note that memory-first reading provides no value for researchers at Step 1.1 (memory is empty), but the rule is kept for consistency.
12. **[Minor] R-Knowledge high-value suggestions surfaced in `review.md` (§5.5).** Aggregator includes a summary of high-value suggestions.

---

## Design Overview

This document provides the implementation-ready technical design for the Forge Architecture Upgrade. The upgrade introduces an operational memory system, parallelizes three sequential bottlenecks into agent clusters (Critical Thinking ×4, Verification ×4, Review ×4 with Knowledge Evolution), expands research from 3→4 agents, and evolves the orchestrator to coordinate all new capabilities. All changes are markdown-only edits to agent definitions. Agent files are authored in `NewAgentsAndPrompts/` and deployed to `.github/agents/` (the VS Code Copilot runtime's canonical discovery location). The agent count grows from 10+1 to 25+1.

**Design Goals:**

1. Reduce redundant artifact reads via operational memory.
2. Achieve ~10–18% end-to-end pipeline speedup through parallelization (accounting for ~20 additional agent invocations).
3. Add Knowledge Evolution as a non-blocking, suggestion-only capability.
4. Preserve deterministic, artifact-grounded behavior and the two-layer verification model.
5. Keep all changes within the existing markdown-only architecture (no code, databases, or YAML state files).

**Context & Inputs:**

- [initial-request.md](initial-request.md) — original requirements and constraints.
- [analysis.md](analysis.md) — synthesized research on architecture, impact, and dependencies.
- [feature.md](feature.md) — formal feature specification with acceptance criteria.

---

## 1. Memory System Architecture

### 1.1 File Structure

Memory is a **single markdown file** at `docs/feature/<feature-slug>/memory.md`. This aligns with Forge's file-based communication model and avoids introducing a new directory or multiple state files.

**Rationale for single-file:** A single file is simpler to initialize, validate, and prune. The 200-line cap (MEM-AC-8) keeps it manageable.

> **[R1 — Critical Fix: Concurrent Memory Writes]** The original design assumed parallel sub-agents could safely write to per-group "Cluster Workspace" subsections within `memory.md`. This is technically impossible with VS Code's file editing tools: `replace_string_in_file` requires exact match of `oldString`, so concurrent read-modify-write cycles by parallel agents will fail when one agent's write changes the file before another's write lands. **Resolution:** Parallel sub-agents (CT ×4, V ×3, R ×4, researchers ×4, implementers ×N) **do not write to `memory.md`**. They communicate findings exclusively through their intermediate output files (e.g., `ct-security.md`, `v-tests.md`, `r-quality.md`). Only aggregators and sequential agents (spec, designer, planner, document writer) write to `memory.md` — these always run sequentially, making concurrent writes impossible. The "Cluster Workspaces" section has been removed from `memory.md`.

> **[R1 — Critical Fix: Canonical Agent Location]** Agent definitions exist in two locations: `NewAgentsAndPrompts/` (development/authoring) and `.github/agents/` (VS Code Copilot runtime discovery — the standard convention). Both directories contain matching agent files. The design targets `NewAgentsAndPrompts/` for authoring per the initial request, with a mandatory sync step to `.github/agents/` for deployment. New sub-agent and aggregator files (15 total) must be created in both locations. `.github/prompts/` contains the feature workflow prompt. Only `.github/instructions/` does not yet exist.

```
docs/feature/<feature-slug>/
├── memory.md              ← NEW: operational memory (single file)
├── initial-request.md
├── research/
│   ├── architecture.md
│   ├── impact.md
│   ├── dependencies.md
│   └── patterns.md        ← NEW: 4th research focus
├── analysis.md
├── feature.md
├── design.md
├── design_critical_review.md
├── ct-review/              ← NEW: CT cluster intermediate outputs (Step 3b)
│   ├── ct-security.md
│   ├── ct-scalability.md
│   ├── ct-maintainability.md
│   └── ct-strategy.md
├── plan.md
├── tasks/*.md
├── verification/           ← NEW: intermediate V cluster outputs
│   ├── v-build.md
│   ├── v-tests.md
│   ├── v-tasks.md
│   └── v-feature.md
├── review/                 ← NEW: intermediate R cluster outputs
│   ├── r-quality.md
│   ├── r-security.md
│   ├── r-testing.md
│   ├── r-knowledge.md
│   └── knowledge-suggestions.md
├── verifier.md
├── review.md
└── decisions.md
```

**Naming rationale:** CT intermediate files go under `ct-review/` (not `research/`) because they are Step 3b outputs, not Step 1 research outputs — mixing them in `research/` would confuse agents that read `research/` expecting only Step 1 outputs. V and R clusters get dedicated subdirectories (`verification/`, `review/`) because they produce operationally distinct outputs referenced by different downstream agents.

### 1.2 Memory Index Schema

The full `memory.md` template initialized at Step 0:

```markdown
# Operational Memory

## Artifact Index

<!-- Updated by each agent after producing output. Max 2 sentences per entry. -->

| Artifact | Key Sections | Last Updated By |
| -------- | ------------ | --------------- |

## Recent Decisions

<!-- Append-only within a phase; pruned at phase boundaries. -->
<!-- Format: - [agent-name, step-N] Decision summary. Rationale: ... -->

## Lessons Learned

<!-- Append-only; persists for full pipeline run. Never pruned. -->
<!-- Format: - [agent-name, step-N] Issue encountered → Resolution applied. -->

## Recent Updates

<!-- Append-only within a phase; pruned at phase boundaries. -->
<!-- Format: - [agent-name, step-N] Updated `artifact-path` — summary of change. -->
```

> **[R1 — Cluster Workspaces Removed]** The original template included a "Cluster Workspaces" section with per-group subsections for parallel agents to write to. This has been removed because parallel sub-agents no longer write to `memory.md` (see §1.1 concurrency fix). Sub-agents communicate findings through their intermediate output files. Aggregators consolidate relevant findings into root-level memory sections after all parallel work completes.

**Field definitions:**

| Field            | Type                 | Max Length   | Description                                                 |
| ---------------- | -------------------- | ------------ | ----------------------------------------------------------- |
| Artifact         | File path            | —            | Relative path to the artifact file                          |
| Key Sections     | Comma-separated list | ≤100 chars   | Top-level section headings in the artifact (for navigation) |
| Last Updated By  | Agent name + step    | —            | Which agent last wrote this entry                           |
| Decision summary | Free text            | ≤2 sentences | What was decided and the core rationale                     |
| Lesson entry     | Free text            | ≤2 sentences | Issue encountered and how it was resolved                   |
| Update entry     | Free text            | ≤2 sentences | Which artifact changed and what changed                     |

### 1.3 Decision Log Schema

Memory's Recent Decisions section captures **operational decisions** made during the current pipeline run. This is distinct from `decisions.md` which captures **architectural decisions** that persist across features.

**Entry format:**

```markdown
- [designer, step-3] Chose event-driven architecture over polling. Rationale: lower latency and better scalability for the notification system.
```

**Rules:**

- Each entry is ≤2 sentences (MEM-AC-7).
- Entries include the agent name and pipeline step for traceability.
- Entries are append-only within a phase.
- Pruned at phase boundaries (entries older than 2 completed phases removed — see §1.6).
- Lessons Learned entries are never pruned; they persist for the full pipeline run.

### 1.4 Lessons Learned Schema

**Entry format:**

```markdown
- [implementer, step-5] `UserService.validate()` threw on null input despite type annotations → Added explicit null guard. Typescript strict mode doesn't catch runtime nulls from API responses.
```

**Rules:**

- Each entry is ≤2 sentences.
- Never pruned during the pipeline run.
- Written primarily by implementers (encountering issues during TDD) and verifier/verification aggregator (encountering integration issues).
- All agents read these to avoid repeating past mistakes within the same pipeline run.

### 1.5 Agent Read/Write Matrix

> **[R1 — Critical Fix]** Parallel agents no longer write to `memory.md`. Only aggregators and sequential agents write. This eliminates concurrent write conflicts.

| Agent                                  | Reads Memory                                     | Writes Memory                     | Sections Written                                                                       |
| -------------------------------------- | ------------------------------------------------ | --------------------------------- | -------------------------------------------------------------------------------------- |
| **Orchestrator**                       | Yes (lifecycle)                                  | Yes (init, invalidation, pruning) | All sections (management only)                                                         |
| **Researcher (×4 focused)**            | Yes (low value at Step 1 — memory is near-empty) | **No**                            | — (findings go to `research/<focus>.md`)                                               |
| **Researcher (synthesis)**             | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Updates                                                         |
| **Spec Agent**                         | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Decisions, Recent Updates                                       |
| **Designer**                           | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Decisions, Recent Updates                                       |
| **CT Sub-Agents (×4)**                 | Yes                                              | **No**                            | — (findings go to `ct-review/ct-<focus>.md`)                                           |
| **CT Aggregator**                      | Yes                                              | Yes (sequential)                  | Recent Decisions, Recent Updates                                                       |
| **Planner**                            | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Decisions, Recent Updates                                       |
| **Implementer (×N)**                   | Yes                                              | **No**                            | — (findings go to task output; Lessons Learned captured by orchestrator between waves) |
| **V-Build**                            | Yes                                              | **No**                            | — (findings go to `verification/v-build.md`)                                           |
| **V-Tests / V-Tasks / V-Feature**      | Yes                                              | **No**                            | — (findings go to `verification/v-*.md`)                                               |
| **V Aggregator**                       | Yes                                              | Yes (sequential)                  | Recent Updates, Lessons Learned                                                        |
| **R-Quality / R-Security / R-Testing** | Yes                                              | **No**                            | — (findings go to `review/r-*.md`)                                                     |
| **R-Knowledge**                        | Yes                                              | **No**                            | — (findings go to `review/r-knowledge.md`)                                             |
| **R Aggregator**                       | Yes                                              | Yes (sequential)                  | Recent Decisions, Recent Updates, Lessons Learned                                      |
| **Documentation Writer**               | Yes                                              | Yes (sequential)                  | Artifact Index, Recent Updates                                                         |

**Key constraint:** Only aggregators and sequential agents write to `memory.md`. Parallel agents (researchers, implementers, cluster sub-agents) communicate through their intermediate output files only. Since aggregators run after all sub-agents complete, and sequential agents run one at a time, concurrent writes to `memory.md` are impossible. This is the fundamental concurrency invariant.

**Implementer Lessons Learned:** Implementers capture issues in their task output. The orchestrator extracts Lessons Learned entries from completed task outputs and appends them to `memory.md` between implementation waves (a sequential operation).

### 1.6 Memory Lifecycle

```
Step 0: Orchestrator creates memory.md with empty templates
  │
  ├─ Step 1.1: Researchers ×4 run in parallel (read memory, do NOT write to it)
  │            Findings go to research/<focus>.md files
  ├─ Step 1.2: Synthesis agent reads research partials → writes analysis.md
  │            Synthesis agent writes to memory (Artifact Index, Recent Updates)
  │            Orchestrator prunes entries older than 2 phases (none yet)
  │
  ├─ Step 2: Spec writes to memory (sequential — safe)
  │            Orchestrator prunes (removes pre-Step-1 entries from Decisions/Updates)
  │
  ├─ Step 3: Designer writes to memory (sequential — safe)
  │
  ├─ Step 3b: CT sub-agents ×4 run in parallel (read memory, do NOT write to it)
  │            Findings go to ct-review/ct-<focus>.md files
  │            CT Aggregator reads sub-agent outputs → writes memory (sequential — safe)
  │            IF NEEDS_REVISION:
  │              Orchestrator marks affected entries [INVALIDATED — revision in progress]
  │              Designer revises → writes corrected entries (sequential — safe)
  │              Full CT cluster re-runs
  │
  ├─ Step 4: Planner writes to memory (sequential — safe)
  │            Orchestrator prunes (removes entries >2 phases old)
  │
  ├─ Step 5: Implementers ×N run in parallel (read memory, do NOT write to it)
  │            Findings go to task outputs; issues logged in task output files
  │            Between waves: orchestrator extracts Lessons Learned from task outputs
  │            and appends to memory (sequential — safe)
  │
  ├─ Step 6: V-Build runs (reads memory, does NOT write to it)
  │            V-Tests/V-Tasks/V-Feature run in parallel (read memory, do NOT write)
  │            V Aggregator reads sub-agent outputs → writes memory (sequential — safe)
  │            IF NEEDS_REVISION → replan loop:
  │              Orchestrator marks affected entries [INVALIDATED]
  │              Planner → Implementers → Full V cluster re-run (max 3 iterations)
  │
  ├─ Step 7: R sub-agents ×4 run in parallel (read memory, do NOT write to it)
  │            R Aggregator reads sub-agent outputs → writes memory (sequential — safe)
  │
  └─ Pipeline complete: memory.md persists alongside final artifacts
```

> **[R1 — Concurrency Invariant]** At every point in the lifecycle above, exactly zero or one agent writes to `memory.md` at any given time. Parallel agents only read. This is enforced by design: sub-agents never write to memory, and orchestrator/aggregator/sequential-agent writes occur at sequential pipeline points.

**Pruning rules:**

- **Phase boundary pruning:** After each completed step, the orchestrator removes entries from Recent Decisions, Recent Updates, and Artifact Index that are more than 2 completed phases old. This keeps the file bounded.
- **Lessons Learned:** Never pruned. Persists for the full pipeline run.
- **Emergency pruning:** If `memory.md` exceeds 200 lines (MEM-AC-8), the orchestrator removes all entries except **Lessons Learned, Artifact Index, and current-phase entries**. (R1: Artifact Index is now preserved during emergency pruning to maintain navigational value — removing it would cause all downstream agents to fall back to full artifact reads, defeating memory's purpose.)

**Invalidation protocol:**
When a revision loop is triggered (NEEDS_REVISION from CT, V, or R aggregator):

1. Orchestrator identifies memory entries written by agents whose output is now stale.
2. Orchestrator prepends `[INVALIDATED — <reason>]` to each affected entry.
3. The revising agent (designer for CT, implementer for V/R) writes corrected entries that replace the invalidated ones.
4. Invalidated entries that are not replaced after the revising agent completes are removed by the orchestrator.

### 1.7 Failure Modes

| Failure                                                | Detection                                                                | Recovery                                                                                                                        | Severity |
| ------------------------------------------------------ | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- | -------- |
| `memory.md` missing when agent reads                   | Agent checks for file existence before reading                           | Agent logs warning, proceeds without memory (falls back to direct artifact reads). Pipeline continues.                          | High     |
| `memory.md` corrupted or unparseable                   | Orchestrator validates section headers after each step                   | Orchestrator re-initializes memory with empty templates. Downstream agents start fresh. Event logged.                           | High     |
| Memory exceeds 200-line cap                            | Orchestrator counts lines after each step                                | Emergency pruning: remove all except Lessons Learned + Artifact Index + current-phase entries                                   | Medium   |
| Stale memory after partial revision                    | Orchestrator checks invalidation markers remain after revision completes | Remove orphaned [INVALIDATED] entries                                                                                           | Medium   |
| Memory-artifact drift (memory says X, artifact says Y) | Cannot be automatically detected                                         | Agents are instructed: "For critical decisions, verify against the source artifact." Memory is navigational, not authoritative. | High     |

> **[R1]** The "Parallel agent write conflict" failure mode has been removed. Since parallel agents no longer write to `memory.md` (§1.5), concurrent write conflicts are impossible by design.

---

## 2. Aggregator Pattern (Shared)

All three clusters (CT, V, R) use aggregators that follow the same structural pattern. The researcher synthesis mode is the closest existing precedent. This section defines the reusable aggregator template.

### 2.1 Aggregator Role Definition

An aggregator is a specialized agent that:

1. Consumes N sub-agent output files.
2. Merges, deduplicates, and organizes findings.
3. Surfaces conflicts as "Unresolved Tensions" — never resolves them.
4. Produces a single output artifact with a single completion contract.
5. Writes consolidated findings to `memory.md` (the aggregator is the only cluster participant that writes to memory — all sub-agents are read-only with respect to memory).

**Structural pattern (v2 template):**

```
YAML frontmatter → Role statement → Inputs → Outputs → Operating Rules (5) →
Workflow → Output spec → Completion Contract → Anti-Drift Anchor
```

Each aggregator file follows this template exactly, substituting cluster-specific details.

### 2.2 Input Contract

Every aggregator receives:

| Input                       | Source             | Description                                            |
| --------------------------- | ------------------ | ------------------------------------------------------ |
| Sub-agent output files (×N) | Cluster sub-agents | Intermediate findings files                            |
| `memory.md`                 | Operational memory | Current memory state                                   |
| Context artifact(s)         | Varies by cluster  | The artifact being reviewed (e.g., `design.md` for CT) |

**Input validation rules:**

- If a sub-agent output file is missing (sub-agent ERROR'd), the aggregator notes the gap and proceeds with available outputs.
- If 2+ sub-agent outputs are missing, the aggregator returns ERROR (insufficient data for meaningful aggregation).
- If `memory.md` is missing, the aggregator proceeds without memory context (non-blocking).

### 2.3 Output Contract

Every aggregator produces:

| Output          | Description                                                |
| --------------- | ---------------------------------------------------------- |
| Merged artifact | Single markdown file combining all sub-agent findings      |
| Memory updates  | Consolidated entries written to root-level memory sections |

**Merged artifact structure (common sections):**

```markdown
# <Cluster Name> — Aggregated Report

## Summary

<!-- One-paragraph synthesis of all sub-agent findings -->

## Findings by Severity

### Critical

### High

### Medium

### Low

## Cross-Cutting Concerns

<!-- Synthesized from all sub-agents' Cross-Cutting Observations sections -->

## Unresolved Tensions

<!-- Conflicts between sub-agents, presented with both sides and attribution -->

## Coverage

<!-- Which sub-agents contributed, any gaps from missing sub-agents -->

## Sub-Agent Attribution

<!-- Per-finding attribution to originating sub-agent for traceability -->
```

**Completion contract aggregation logic (universal):**

| Scenario                     | Result                                                                                        |
| ---------------------------- | --------------------------------------------------------------------------------------------- |
| All sub-agents DONE          | Aggregator applies its own severity threshold to determine DONE vs. NEEDS_REVISION            |
| 1 sub-agent ERROR, rest DONE | Aggregator proceeds with available outputs; notes coverage gap; result per severity threshold |
| 2+ sub-agents ERROR          | ERROR (insufficient coverage)                                                                 |
| Any sub-agent output empty   | Treat as DONE with no findings for that area; log warning                                     |

### 2.4 Conflict Resolution

**Design principle:** Aggregators surface conflicts, they do not resolve them.

When two sub-agents produce contradictory findings:

1. Both findings are included verbatim in the "Unresolved Tensions" section.
2. Each finding is attributed to its originating sub-agent.
3. The tension is categorized (e.g., "security vs. performance tradeoff").
4. The downstream consumer (designer for CT, implementer for V/R) resolves the tension.

**Example:**

```markdown
## Unresolved Tensions

### Tension: Input Validation Thoroughness

- **CT-Security** (critical): "Every API endpoint must validate all input fields against schema before processing."
- **CT-Scalability** (medium): "Per-field schema validation on every request adds ~5ms latency at scale; consider lazy validation for read-only endpoints."
- **Category:** Security vs. Performance tradeoff
- **Resolution required by:** Designer (during revision)
```

### 2.5 Bottleneck Mitigation

The aggregator must not become a new sequential bottleneck that offsets parallelization gains.

**Mitigation strategies:**

1. **Merge-only, no re-analysis:** Aggregators merge sub-agent outputs structurally. They do NOT re-read source artifacts, re-analyze code, or perform independent verification. Their sole job is combining, deduplicating, and categorizing existing findings.

2. **Structured input format:** Sub-agents produce findings in a consistent, parseable markdown format (see §3.1, §4.1, §5.1) so the aggregator can process them mechanically.

3. **Severity-based prioritization:** Findings are ordered by severity (critical → high → medium → low). The aggregator stops detailing low-severity findings if the output exceeds a reasonable length (~100 lines for the aggregated section).

4. **Deduplication by content hash (conceptual):** When multiple sub-agents flag the same issue (identified by matching file references and concern descriptions), the aggregator merges them into a single entry with multi-source attribution.

5. **Time budget:** Aggregation should consume **≤40–60%** of the longest sub-agent execution time. The aggregator performs semantic deduplication, cross-cutting synthesis, and severity threshold evaluation — these are lightweight reasoning tasks, not mechanical operations, but they are scoped to combining existing findings rather than generating new analysis.

> **[R1 — Significant Revision]** The original 25% overhead budget was unrealistic. Semantic deduplication (recognizing when differently-worded findings describe the same issue), cross-cutting synthesis (combining observations into a coherent narrative), and severity threshold determination all require LLM reasoning. The revised 40–60% budget is more realistic. **Impact on speedup projections:** CT cluster effective speedup drops from ~2–2.5× to ~1.5–2×. V cluster effective speedup remains ~1.3–1.7× (first pass). R cluster effective speedup drops from ~2.5–3× to ~1.8–2.5×.

---

## 3. Critical Thinking Cluster

### 3.1 Sub-Agent Definitions

The current `critical-thinker.agent.md` (86 lines) is split into 4 focused sub-agents. Each inherits the critical thinker's adversarial mindset and verification-first methodology, but scopes its probing to specific risk categories.

> **[R1 — Critical Fix]** Baseline corrected from 139 to 86 lines (actual measured value).

> **[R1 — Significant: Fragmentation Tradeoff]** Splitting the critical thinker into 4 scoped sub-agents improves depth of analysis per category but risks losing holistic cross-domain insights (e.g., "this security decision creates a scalability bottleneck"). The Cross-Cutting Observations mechanism partially mitigates this but cannot fully replicate the single agent's ability to reason across all domains simultaneously. **Alternative to evaluate during implementation:** 2–3 larger sub-agents (e.g., "Technical" covering security+scalability+performance, "Strategic" covering maintainability+scope+approach) would preserve more cross-domain insight while still providing parallelism. The 4-agent split is the current design but the planner should assess whether fewer, broader sub-agents yield better review quality.

| Sub-Agent              | File                          | Risk Categories                                                | Key Questions                                                                |
| ---------------------- | ----------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **CT-Security**        | `ct-security.agent.md`        | Security vulnerabilities, backwards compatibility              | "What can be exploited?" "What breaks for existing users?"                   |
| **CT-Scalability**     | `ct-scalability.agent.md`     | Scalability bottlenecks, performance implications              | "What happens at 10× load?" "Where are the N+1 patterns?"                    |
| **CT-Maintainability** | `ct-maintainability.agent.md` | Maintainability, complexity, integration risks                 | "Will a new dev understand this in 6 months?" "What's the coupling surface?" |
| **CT-Strategy**        | `ct-strategy.agent.md`        | Strategic risks, scope risks, edge cases, fundamental approach | "Is this the right approach at all?" "What did the design quietly drop?"     |

**Prompt splitting strategy:**

From the current critical-thinker, each sub-agent retains:

- The adversarial mindset section (identical across all 4).
- The workflow steps for reading `initial-request.md`, `design.md`, `feature.md`, and codebase verification.
- The self-verification step.
- The v2 structural template.

Each sub-agent receives:

- A focused subset of the "Risk Categories" section.
- A scoped "Challenge the Fundamentals" prompt tailored to their area.
- A **mandatory "Cross-Cutting Observations"** section in their output — for issues that span beyond their assigned categories.

**Sub-agent output format (standardized):**

```markdown
# Critical Review: <Focus Area>

## Findings

### [Severity: Critical/High/Medium/Low] Finding Title

- **What:** Specific risk description
- **Where:** File, component, or design section
- **Likelihood:** High / Medium / Low
- **Impact:** High / Medium / Low
- **Assumption at risk:** What underlying assumption this challenges

## Cross-Cutting Observations

<!-- Issues that span beyond this sub-agent's primary scope -->

- Observation with reference to which other sub-agent's scope it belongs in

## Requirement Coverage

<!-- Requirements from feature.md relevant to this focus area -->

| Requirement | Coverage Status | Notes |
| ----------- | --------------- | ----- |
```

**Sub-agent completion contract:** `DONE:` / `ERROR:` only. Sub-agents never return `NEEDS_REVISION` — only the aggregator makes that determination.

**Sub-agent inputs:**

- `initial-request.md`
- `design.md`
- `feature.md`
- `memory.md`

**Sub-agent output:**

- `docs/feature/<feature-slug>/ct-review/ct-<focus>.md`

### 3.2 Dispatch Protocol

```
Orchestrator at Step 3b:
  1. Verify design.md exists (prerequisite).
  2. Dispatch ct-security, ct-scalability, ct-maintainability, ct-strategy
     in parallel (4 concurrent — within Global Rule 7 cap).
  3. Each sub-agent receives: initial-request.md, design.md, feature.md, memory.md
     via explicit file path arguments.
  4. Wait for ALL 4 to return DONE or ERROR.
  5. Handle errors:
     - If 1 sub-agent ERROR'd: retry it once (Global Rule 4).
     - If 2+ sub-agents ERROR'd: retry entire cluster once.
     - If errors persist after retry: CT Aggregator proceeds with available outputs
       (if ≥2 outputs available) or returns ERROR (if <2 outputs).
  6. Invoke ct-aggregator with all available sub-agent output files.
  7. CT Aggregator returns DONE / NEEDS_REVISION / ERROR.
  8. If NEEDS_REVISION:
     a. Route design_critical_review.md to designer for revision.
     b. Designer updates design.md.
     c. Orchestrator invalidates stale CT memory entries.
     d. Re-run full CT cluster (all 4 sub-agents + aggregator). Not a subset.
     e. Max 1 revision loop total. If still NEEDS_REVISION, proceed with warning.
  9. If ERROR: retry aggregator once. If persistent, halt pipeline.
```

### 3.3 Aggregation Details

**Aggregator file:** `ct-aggregator.agent.md`

**Inputs:**

- `ct-review/ct-security.md`, `ct-review/ct-scalability.md`, `ct-review/ct-maintainability.md`, `ct-review/ct-strategy.md`
- `design.md` (for context reference only — not for re-analysis)
- `memory.md`

**Output:** `design_critical_review.md`

**Aggregation workflow:**

1. Read all available sub-agent output files.
2. Collect all findings into a unified list.
3. Deduplicate: merge findings with matching `Where` and `What` fields, attributing to all originating sub-agents.
4. Sort by severity (Critical → High → Medium → Low).
5. Synthesize Cross-Cutting Observations from all 4 sub-agents into a dedicated "Cross-Cutting Concerns" section.
6. Surface any contradictions as "Unresolved Tensions" (see §2.4).
7. Merge requirement coverage tables.
8. Determine completion contract:
   - `NEEDS_REVISION` if any finding rated Critical or High severity.
   - `DONE` if all findings are Medium or Low severity.
   - `ERROR` if aggregation cannot proceed (< 2 sub-agent outputs available after retries).
9. Write consolidated findings to `memory.md` (Recent Decisions, Recent Updates).

> **[R1 — Minor: Unresolved Tensions Forwarding]** If the max-1 CT revision loop completes and `design_critical_review.md` still contains Unresolved Tensions, the aggregator must format them as **explicit planning constraints** in a dedicated "Planning Constraints" section at the end of `design_critical_review.md`. The planner reads this section and incorporates unresolved tensions as constraints on task design (e.g., "Task X must support both encryption-at-rest AND the low-latency path — implementer must resolve this tradeoff").

**Output format of `design_critical_review.md`:**

```markdown
# Design Critical Review

## Summary

<!-- One-line verdict -->

## Overall Risk Level

<!-- Low / Medium / High / Critical — with justification -->

## Findings by Severity

### Critical

<!-- Merged, deduplicated, attributed findings -->

### High

### Medium

### Low

## Cross-Cutting Concerns

<!-- Synthesized from all sub-agents' Cross-Cutting Observations -->

## Unresolved Tensions

<!-- Contradictions between sub-agents -->

## Requirement Coverage Gaps

<!-- Aggregated from all sub-agents' coverage tables -->

## Recommendations

<!-- Prioritized list for designer, if NEEDS_REVISION -->
```

### 3.4 Orchestrator Integration

**Changed orchestrator section:** Step 3b (currently lines ~145–156).

**Before (current):**

```
Step 3b: Invoke critical-thinker → design_critical_review.md → check DONE/NEEDS_REVISION/ERROR
```

**After (proposed):**

```
Step 3b:
  3b.1: Dispatch [ct-security, ct-scalability, ct-maintainability, ct-strategy] in parallel (×4)
  3b.2: Wait for all 4 → handle errors per §3.2
  3b.3: Invoke ct-aggregator
  3b.4: Check aggregator contract → route NEEDS_REVISION to designer (max 1 loop)
```

**Orchestrator additions (~35–40 lines):** Cluster dispatch logic, await-all pattern, aggregator invocation, error handling for partial failures, NEEDS_REVISION routing with full re-run, memory invalidation on revision.

---

## 4. Verification Cluster

### 4.1 Sub-Agent Definitions

The current `verifier.agent.md` (109 lines) is split into 4 sub-agents with a mandatory sequential gate (V-Build must pass before others run).

> **[R1 — Critical Fix]** Baseline corrected from 179 to 109 lines (actual measured value).

> **[R1 — Significant: Replan Speedup Clarification]** The V cluster provides meaningful speedup (~1.3–1.7×) on the **first verification pass** where V-Build succeeds and all 3 parallel sub-agents run. During **replan iterations** (the most time-sensitive scenario), build failures are common and dominate wall-clock time; in these cases the cluster provides minimal speedup (~1.0–1.25×). The V cluster's primary value in replan scenarios is **structured, granular reporting** (per-task verification, separate test/feature/task assessment) that enables more targeted replanning, not wall-clock speedup.

| Sub-Agent     | File                 | Responsibility                                                | Execution           | Completion Contract                    |
| ------------- | -------------------- | ------------------------------------------------------------- | ------------------- | -------------------------------------- |
| **V-Build**   | `v-build.agent.md`   | Build system detection, compilation, build artifact reporting | **Sequential gate** | `DONE:` / `ERROR:`                     |
| **V-Tests**   | `v-tests.agent.md`   | Full test suite execution, result analysis                    | After V-Build DONE  | `DONE:` / `NEEDS_REVISION:` / `ERROR:` |
| **V-Tasks**   | `v-tasks.agent.md`   | Per-task acceptance criteria verification                     | After V-Build DONE  | `DONE:` / `NEEDS_REVISION:` / `ERROR:` |
| **V-Feature** | `v-feature.agent.md` | Feature-level acceptance criteria verification                | After V-Build DONE  | `DONE:` / `NEEDS_REVISION:` / `ERROR:` |

**Prompt splitting strategy:**

From the current verifier, each sub-agent receives:

- The read-only enforcement section (identical across all 4).
- The v2 structural template.
- Its specific workflow steps from the current 6-step workflow.

**V-Build specifics:**

- Inherits Step 1 (Detect Build System) and Step 2 (Build) from current verifier.
- Outputs `verification/v-build.md` containing:
  - Build system detected (name, version, configuration file path).
  - Build command executed.
  - Build status (pass/fail).
  - Build errors and warnings (with file paths and line numbers).
  - Build artifact paths (for downstream sub-agents).
  - Environment details (language version, OS, key tool versions).
- **Critical constraint:** All build state is captured in `v-build.md` on disk. No reliance on terminal state, environment variables, or shared process context.

**V-Tests specifics:**

- Inherits Step 3 (Test Execution) from current verifier.
- Reads `v-build.md` for build context (build artifact locations, environment).
- Outputs `verification/v-tests.md` containing:
  - Test command executed.
  - Pass/fail/skip counts.
  - Failing test details with stack traces.
  - Flag if previously-passing unit tests now fail (indicates cross-task integration issue — high severity).

**V-Tasks specifics:**

- Inherits Step 4 (Task-Level Verification) from current verifier.
- Reads `v-build.md`, `plan.md`, `tasks/*.md`.
- Outputs `verification/v-tasks.md` containing:
  - Per-task verification status (verified / partially-verified / failed).
  - Acceptance criteria checklist per task.
  - Specific task IDs that failed (critical for targeted replan).

**V-Feature specifics:**

- Inherits Step 5 (Feature-Level Verification) from current verifier.
- Reads `v-build.md`, `feature.md`.
- Outputs `verification/v-feature.md` containing:
  - Feature-level acceptance criteria status.
  - Regression check results.
  - Overall feature readiness assessment.

**Sub-agent output format (standardized):**

```markdown
# Verification: <Focus Area>

## Status

<!-- PASS / FAIL / PARTIAL -->

## Results

<!-- Focus-area-specific results -->

## Issues Found

### [Severity: Critical/High/Medium/Low] Issue Title

- **What:** Specific issue
- **Where:** File path, test name, or task ID
- **Impact:** What breaks or doesn't work
- **Suggested Action:** What needs to be fixed (for replan)

## Cross-Cutting Observations

<!-- Issues spanning beyond this sub-agent's scope -->
```

### 4.2 Dispatch Protocol

```
Orchestrator at Step 6:
  1. Dispatch v-build (sequential — single agent).
  2. Wait for V-Build to return.
  3. If V-Build ERROR:
     a. Retry once (Global Rule 4).
     b. If still ERROR: enter replan loop directly (skip V-Tests/V-Tasks/V-Feature).
        Invoke planner in replan mode with V-Build error details.
  4. If V-Build DONE:
     a. Dispatch v-tests, v-tasks, v-feature in parallel (×3).
        Each receives: v-build.md, memory.md, plus their specific inputs.
     b. Wait for all 3 to return.
     c. Handle errors: retry individual ERROR'd sub-agents once.
  5. Invoke v-aggregator with all available sub-agent outputs.
  6. V-Aggregator returns DONE / NEEDS_REVISION / ERROR.
  7. If NEEDS_REVISION or ERROR:
     a. Enter replan loop (see §4.4).
```

### 4.3 Aggregation Details

**Aggregator file:** `v-aggregator.agent.md`

**Inputs:**

- `verification/v-build.md`, `verification/v-tests.md`, `verification/v-tasks.md`, `verification/v-feature.md`
- `memory.md`

**Output:** `verifier.md`

**Aggregation workflow:**

1. Read all available sub-agent verification outputs.
2. Compile a unified verification report:
   - Build Results section (from V-Build).
   - Test Results section (from V-Tests).
   - Per-Task Verification section (from V-Tasks, with task IDs for replan targeting).
   - Feature-Level Verification section (from V-Feature).
3. Merge Issues Found from all sub-agents, sorted by severity.
4. Map failing items to specific task IDs (critical for targeted replan — the planner needs to know which tasks to re-plan).
5. Determine completion contract:
   - `DONE` if V-Build passed AND all 3 parallel sub-agents DONE.
   - `NEEDS_REVISION` if V-Build passed AND any parallel sub-agent NEEDS_REVISION.
   - `ERROR` if V-Build ERROR'd OR any parallel sub-agent ERROR'd persistently OR 2+ sub-agents missing.
   - Mixed NEEDS_REVISION + ERROR: ERROR takes priority.

**Output format of `verifier.md`:**

```markdown
# Verification Report

## Summary

<!-- Brief outcome and verdict -->

## Build Results

<!-- From V-Build -->

## Test Results

<!-- From V-Tests -->

## Per-Task Verification

<!-- From V-Tasks — includes task IDs -->

| Task ID | Status | Issues |
| ------- | ------ | ------ |

## Feature-Level Verification

<!-- From V-Feature -->

## Issues Summary

### Critical

### High

### Medium

### Low

## Actionable Items

<!-- Prioritized fixes mapped to specific task IDs for replan -->

## Verification Scope

<!-- What was verified, any gaps from missing sub-agents -->
```

### 4.4 NEEDS_REVISION Routing

When V-Aggregator returns NEEDS_REVISION or ERROR:

```
V-Aggregator → NEEDS_REVISION
  │
  ├─ Orchestrator reads verifier.md to identify failing task IDs
  ├─ Orchestrator invalidates V-related memory entries
  ├─ Invoke planner in replan mode with verifier.md
  │    Planner creates/updates ONLY tasks addressing failures
  ├─ Re-run implementation loop for fix tasks only
  ├─ Re-run FULL verification cluster (V-Build → parallel ×3 → V-Aggregator)
  │    Not a subset — full cluster because fixes may introduce new issues
  ├─ Check V-Aggregator result again
  └─ Max 3 total iterations of this loop
      If issues persist after 3: proceed with findings documented in verifier.md
```

### 4.5 Replan Loop Integration

The existing replan loop (max 3 iterations) is preserved but now operates on the V cluster instead of a single verifier:

**Per iteration:**

1. Planner reads `verifier.md` → creates targeted fix tasks.
2. Implementers execute fix tasks (same wave/sub-wave logic as Step 5).
3. Full V cluster re-runs: V-Build → (if DONE) → V-Tests/V-Tasks/V-Feature → V-Aggregator.
4. Check aggregator result.

**Iteration counting:** The counter tracks full V cluster runs, not individual sub-agent invocations. Each complete V-Build → parallel ×3 → aggregate cycle counts as 1 iteration.

**Build failure during replan:** If V-Build fails during a replan iteration, the iteration still counts. The orchestrator enters the next replan iteration (if available) targeting build fix.

---

## 5. Review Cluster + Knowledge Evolution

### 5.1 Sub-Agent Definitions

The current `reviewer.agent.md` (130 lines) is split into 4 focused sub-agents. All run fully in parallel.

> **[R1 — Critical Fix]** Baseline corrected from 220 to 130 lines (actual measured value).

| Sub-Agent       | File                   | Responsibility                                                                                 | Completion Contract                    |
| --------------- | ---------------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------- |
| **R-Quality**   | `r-quality.agent.md`   | Code quality, readability, maintainability, naming, conventions, architectural alignment       | `DONE:` / `NEEDS_REVISION:` / `ERROR:` |
| **R-Security**  | `r-security.agent.md`  | Security scanning, OWASP, secrets/PII detection, dependency vulnerabilities                    | `DONE:` / `NEEDS_REVISION:` / `ERROR:` |
| **R-Testing**   | `r-testing.agent.md`   | Test quality, coverage, test patterns, missing test scenarios                                  | `DONE:` / `NEEDS_REVISION:` / `ERROR:` |
| **R-Knowledge** | `r-knowledge.agent.md` | Knowledge evolution: instruction suggestions, skill suggestions, pattern capture, decision log | `DONE:` / `ERROR:`                     |

**Prompt splitting strategy:**

From the current reviewer, each sub-agent receives:

- The read-only enforcement section (identical across all 4).
- The Quality Standard section ("Would a staff engineer approve this code?").
- The v2 structural template.

**R-Quality specifics:**

- Inherits review workflow Step 4 (code review) from current reviewer.
- Inherits Review Depth Tiers model — receives tier designation from orchestrator at dispatch.
- Reviews: maintainability, readability, naming conventions, architectural alignment with `design.md`, DRY/KISS/YAGNI compliance.
- Output: `review/r-quality.md`.

**R-Security specifics:**

- Inherits Security Review section from current reviewer (both standard scan and Full-tier OWASP).
- Receives tier designation from orchestrator.
- Output: `review/r-security.md`.
- **Override rule:** R-Security ERROR or NEEDS_REVISION with Critical severity findings overrides the aggregated result to ERROR — security is a pipeline blocker.

**R-Testing specifics:**

- Inherits test quality review from current reviewer Step 4.
- Reviews: test coverage adequacy, test quality (behavioral vs. implementation testing), missing test scenarios, test data quality.
- Output: `review/r-testing.md`.

**R-Knowledge specifics (new capability):**

- **Does NOT inherit** from current reviewer — this is entirely new.
- Analyzes the pipeline's implementation to identify:
  1. **Instruction updates:** Improvements/additions/removals for `.github/instructions/*.instructions.md`.
  2. **Skill suggestions:** Improvements for agent skills and capabilities.
  3. **Pattern capture:** Reusable workflow, code, or architectural patterns observed during implementation.
  4. **Decision log updates:** Significant architectural decisions to append to `decisions.md`.
- Output:
  - `review/r-knowledge.md` (analysis and rationale).
  - `review/knowledge-suggestions.md` (actionable suggestions buffer).
  - `decisions.md` (append-only updates — creates if not exists).
- **Critical constraint:** R-Knowledge NEVER modifies agent definitions in `NewAgentsAndPrompts/`. All suggestions go to `knowledge-suggestions.md`.
- **Non-blocking:** R-Knowledge ERROR does not block the pipeline. The aggregator proceeds without knowledge analysis.

**Sub-agent output format (for R-Quality, R-Security, R-Testing):**

```markdown
# Review: <Focus Area>

## Review Tier

<!-- Full / Standard / Lightweight — with rationale -->

## Findings

### [Severity: Blocker/Major/Minor] Finding Title

- **What:** Specific issue
- **Where:** File path and line reference
- **Why:** Rationale for the concern
- **Suggested Fix:** Actionable recommendation
- **Affects Tasks:** Task IDs if identifiable

## Cross-Cutting Observations

<!-- Issues spanning beyond this sub-agent's scope -->

## Summary

<!-- Issue counts by severity -->
```

**R-Knowledge output format:**

```markdown
# Knowledge Evolution Analysis

## Instruction Suggestions

### [Category: instruction-update] Title

- **File:** `.github/instructions/filename.instructions.md`
- **Change:** Add / Update / Remove
- **Rationale:** Why this change improves future pipeline runs
- **Before:** (existing content, if update/remove)
- **After:** (proposed content)

## Pattern Captures

### [Category: pattern-capture] Title

- **Pattern:** Description of the reusable pattern
- **Evidence:** Where it was observed (file paths, task IDs)
- **Applicability:** When this pattern should be reused

## Decision Log Entries

### [Decision Title]

- **Context / Decision / Rationale / Scope / Affected Components**
```

**`knowledge-suggestions.md` format:**

````markdown
# Knowledge Evolution Suggestions

> **WARNING:** These are proposals only. Do NOT apply without human review.
> Generated by R-Knowledge agent during pipeline execution.

## Suggestions

### 1. [Category] Title

- **Target file:** Path to file that would be modified
- **Change type:** Add / Update / Remove
- **Rationale:** Why
- **Diff:**
  ```diff
  - old content
  + new content
  ```
````

- **Risk assessment:** Low / Medium / High

## Rejected Suggestions

<!-- Suggestions that were filtered by KE-SAFE-6 (attempted to weaken safety constraints) -->

```

### 5.2 Knowledge Evolution Agent Details

**R-Knowledge workflow:**
1. Read `memory.md` for pipeline context.
2. Read `initial-request.md`, `feature.md`, `design.md`, `plan.md`, `verifier.md` for full pipeline history.
3. Examine git diff and codebase for implementation patterns.
4. Read existing `decisions.md` (if present) to avoid duplicating prior decisions.
5. Read existing `.github/instructions/` (if present) to understand current instruction state.
6. Identify instruction updates, skill updates, patterns, and decisions.
7. Apply safety filters (KE-SAFE-6): reject any suggestion that removes or weakens safety constraints, error handling, or verification steps.
8. Write `review/r-knowledge.md` (analysis and rationale).
9. Write `review/knowledge-suggestions.md` (actionable suggestions with diff-style before/after).
10. Append to `decisions.md` (if significant decisions identified; create if not exists).
11. (R-Knowledge does NOT write to `memory.md` — findings are in `review/r-knowledge.md`. The R Aggregator consolidates relevant findings into memory.)

### 5.3 Safeguards

| ID | Safeguard | Implementation |
|----|-----------|---------------|
| KE-SAFE-1 | Writes to `knowledge-suggestions.md` ONLY | R-Knowledge's File Boundaries operating rule explicitly lists only its output files. Anti-drift anchor reinforces this. |
| KE-SAFE-2 | Each suggestion includes what/why/file/diff | Enforced in the output format specification of r-knowledge.agent.md. |
| KE-SAFE-3 | Categorized suggestions | Four categories: `instruction-update`, `skill-update`, `pattern-capture`, `workflow-improvement`. |
| KE-SAFE-4 | No auto-apply | Orchestrator NEVER reads `knowledge-suggestions.md` for auto-application. The file persists as a human-review artifact. |
| KE-SAFE-5 | Warning header | `knowledge-suggestions.md` template includes "WARNING: These are proposals only." header. |
| KE-SAFE-6 | Safety constraint filter | R-Knowledge prompt includes explicit instruction: "You MUST NOT suggest removing or weakening any safety constraint, error handling, or verification step. If you identify such a suggestion during your analysis, log it in the Rejected Suggestions section with the reason 'safety constraint — cannot weaken.'" |
| KE-SAFE-7 | `decisions.md` append-only | R-Knowledge's workflow step for `decisions.md` says "append only — never modify or delete existing entries." Read-only verification step checks existing entries are unchanged. |

### 5.4 Dispatch Protocol

```

Orchestrator at Step 7:

1. Determine review tier (Full/Standard/Lightweight) based on changed files.
2. Dispatch r-quality, r-security, r-testing, r-knowledge in parallel (×4).
   Each receives: tier designation, initial-request.md, memory.md, git diff context.
   R-Knowledge additionally receives: feature.md, design.md, plan.md, verifier.md.
3. Wait for all 4 to return.
4. Handle errors:
   - R-Knowledge ERROR: non-blocking. Aggregator proceeds with 3 outputs.
   - R-Security ERROR: critical. Retry once. If persistent, aggregator returns ERROR.
   - R-Quality or R-Testing ERROR: retry once. If persistent, aggregator proceeds with available.
5. Invoke r-aggregator with all available sub-agent outputs.
6. R-Aggregator returns DONE / NEEDS_REVISION / ERROR.
7. Handle result per §5.6.

````

### 5.5 Aggregation Details

**Aggregator file:** `r-aggregator.agent.md`

**Inputs:**
- `review/r-quality.md`, `review/r-security.md`, `review/r-testing.md`, `review/r-knowledge.md` (if available)
- `memory.md`

**Output:** `review.md`

**Aggregation workflow:**
1. Read all available sub-agent review outputs.
2. Compile unified review report:
   - Review Tier (from dispatched designation).
   - Quality Findings (from R-Quality).
   - Security Findings (from R-Security).
   - Testing Findings (from R-Testing).
   - Knowledge Evolution Summary (from R-Knowledge — if available; otherwise "Knowledge analysis unavailable").
3. Merge, deduplicate, and sort findings by severity.
4. Surface Unresolved Tensions (e.g., R-Quality flags complexity vs. R-Security requires additional checks).
5. Determine completion contract:
   - `DONE` if all available sub-agents DONE and no blocker/major findings requiring code changes.
   - `NEEDS_REVISION` if R-Quality or R-Testing NEEDS_REVISION (merge failing items for implementers).
   - `ERROR` if R-Security NEEDS_REVISION with critical findings, or R-Security ERROR.
   - R-Knowledge DONE with suggestions: `DONE` (suggestions are non-blocking).
   - R-Knowledge ERROR: `DONE` with warning (knowledge failure is non-blocking).
6. Write consolidated findings to `memory.md` (Recent Decisions, Recent Updates, Lessons Learned).
7. **[R1]** If R-Knowledge produced suggestions, include a "High-Value Suggestions Summary" in `review.md` — a 3–5 line summary of the most impactful suggestions with a reference to `knowledge-suggestions.md` for full details. This increases visibility and adoption likelihood.

**`decisions.md` ownership:** R-Knowledge agent owns `decisions.md` writes. The aggregator does NOT touch `decisions.md`. If R-Knowledge didn't run (ERROR), `decisions.md` is not updated for this pipeline run.

**Output format of `review.md`:**

```markdown
# Code Review Report

## Summary
<!-- Overall verdict: OK / Minor / Major / Blocking -->

## Review Tier
<!-- Full / Standard / Lightweight — with rationale -->

## Quality Review
<!-- From R-Quality -->

## Security Review
<!-- From R-Security -->

## Testing Review
<!-- From R-Testing -->

## Knowledge Evolution
<!-- Summary from R-Knowledge or "unavailable" -->
<!-- Reference to knowledge-suggestions.md for full proposals -->

## Issues by Severity
### Blocker
### Major
### Minor

## Unresolved Tensions
<!-- Cross-sub-agent conflicts -->

## Suggested Fixes
<!-- Actionable recommendations mapped to files/tasks -->

## Coverage
<!-- Which sub-agents contributed, any gaps -->
````

### 5.6 NEEDS_REVISION Routing

```
R-Aggregator → NEEDS_REVISION
  │
  ├─ Orchestrator reads review.md to identify affected task IDs
  ├─ Route findings to affected implementer(s) for lightweight fix pass
  ├─ Each implementer receives: its task file + relevant review findings
  ├─ Max 1 fix loop
  └─ If still NEEDS_REVISION after fix: escalate to planner for full replan
```

```
R-Aggregator → ERROR (from R-Security)
  │
  ├─ Security is a critical-path blocker
  ├─ Orchestrator retries R-Security once
  ├─ If persistent: escalate to planner for full replan
  └─ Pipeline does not proceed past security ERROR
```

---

## 6. Research Phase Expansion

### 6.1 Fourth Focus Area

The 4th research focus area is **Patterns & Conventions**.

| Focus Area           | Scope                                                                                                                                        | Output                     |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| **architecture**     | Repository structure, project layout, architecture patterns, layers, .github instructions                                                    | `research/architecture.md` |
| **impact**           | Affected files, modules, components; what changes and where                                                                                  | `research/impact.md`       |
| **dependencies**     | Module interactions, data flow, API/interface contracts, external dependencies                                                               | `research/dependencies.md` |
| **patterns** _(NEW)_ | Testing strategy, code conventions, developer experience patterns, error handling patterns, operational concerns, existing reusable patterns | `research/patterns.md`     |

**Rationale:** The existing 3 focus areas cover structure (architecture), change surface (impact), and data flow (dependencies). None systematically captures the project's testing conventions, code style patterns, error handling approaches, or operational practices. This information is critical for the planner (to generate convention-compliant tasks) and implementer (to follow established patterns).

**Patterns focus area scope:**

- Testing strategy: framework, patterns, naming conventions, fixture approaches.
- Code conventions: naming, file organization, import ordering, comment style.
- Error handling patterns: try/catch conventions, error types, logging patterns.
- Developer experience: build scripts, development workflows, tooling configuration.
- Operational concerns: logging, monitoring, configuration management patterns.

### 6.2 Updated Dispatch

**Changes to `researcher.agent.md`:**

- Add `patterns` to the Focus Area table:
  ```
  | **patterns** | Testing strategy, code conventions, developer experience patterns, error handling patterns, operational concerns |
  ```
- No other changes to the researcher agent. The existing focused research workflow, synthesis mode, retrieval strategy, and completion contract all accommodate a 4th focus area without modification.

**Changes to `orchestrator.agent.md` Step 1.1:**

- Update the dispatch table from 3 to 4 agents:
  ```
  | researcher (4) | `patterns` | `docs/feature/<feature-slug>/research/patterns.md` |
  ```
- Update the narrative from "three" to "four" researcher instances.
- The concurrency cap (max 4) accommodates this directly — 4 research agents is at the cap limit.

**Changes to `researcher.agent.md` synthesis mode:**

- The synthesis instructions already say "Read ALL partial research files from the `research/` directory." Since `patterns.md` will be in `research/`, it is automatically included with no change needed.

**Impact:** ~5–10 lines changed across 2 files. Zero risk — purely additive, follows existing dispatch pattern.

---

## 7. Orchestrator Evolution

### 7.1 Updated Pipeline

```
Step 0: Setup
  → Create initial-request.md
  → Initialize memory.md with empty templates                              [NEW]

Step 1: Research (Parallel)
  1.1: Dispatch 4 researcher agents in parallel                            [CHANGED: 3→4]
       [architecture, impact, dependencies, patterns]
       (Sub-agents read memory but do NOT write to it)
  1.2: Invoke researcher synthesis → analysis.md
       (Synthesis writes to memory — sequential, safe)
  1.2a: (Conditional) Approval gate
  → Orchestrator: prune memory                                             [NEW]

Step 2: Specification
  → Invoke spec → feature.md
  → Orchestrator: prune memory                                             [NEW]

Step 3: Design
  → Invoke designer → design.md

Step 3b: Critical Review (Cluster)                                         [CHANGED]
  3b.1: Dispatch [ct-security, ct-scalability, ct-maintainability, ct-strategy] ×4 parallel
         (Sub-agents read memory but do NOT write to it)
  3b.2: Wait all → handle errors
  3b.3: Invoke ct-aggregator → design_critical_review.md
         (Aggregator writes to memory — sequential, safe)
  3b.4: If NEEDS_REVISION: invalidate memory → designer revises → full re-run (max 1 loop)

Step 4: Planning
  → Invoke planner → plan.md + tasks/*.md
  → Orchestrator: prune memory                                             [NEW]
  4a: (Conditional) Approval gate

Step 5: Implementation (Parallel Waves)
  → Execute waves per existing logic
  → Between waves: orchestrator extracts Lessons Learned from task outputs  [NEW]
    and appends to memory (sequential, safe)

Step 6: Verification (Cluster)                                             [CHANGED]
  6.1: Dispatch v-build (sequential gate — reads memory, does NOT write)
  6.2: If DONE: dispatch [v-tests, v-tasks, v-feature] ×3 parallel
       (Sub-agents read memory but do NOT write to it)
  6.3: Invoke v-aggregator → verifier.md
       (Aggregator writes to memory — sequential, safe)
  6.4: If NEEDS_REVISION: replan loop (max 3, full cluster re-run each time)

Step 7: Final Review (Cluster)                                             [CHANGED]
  7.1: Determine review tier
  7.2: Dispatch [r-quality, r-security, r-testing, r-knowledge] ×4 parallel
       (Sub-agents read memory but do NOT write to it)
  7.3: Invoke r-aggregator → review.md
       (Aggregator writes to memory — sequential, safe)
  7.4: If NEEDS_REVISION: route to implementers (max 1 fix loop)
  7.5: Preserve knowledge-suggestions.md (no auto-apply)

Pipeline Complete
```

### 7.2 Memory Coordination

The orchestrator is responsible for memory lifecycle management. These are the specific memory actions inserted into the orchestrator's workflow:

| Orchestrator Action               | When                                           | What                                                                                                                          |
| --------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Initialize memory                 | Step 0, after creating initial-request.md      | Create `memory.md` with empty template (see §1.2)                                                                             |
| Verify memory-writing agent wrote | After aggregators and sequential agents return | Check that the agent wrote to memory. Log warning if not (non-blocking).                                                      |
| Extract Lessons Learned           | Between implementation waves                   | Read completed task outputs for issue/resolution entries; append to memory Lessons Learned section                            |
| Prune memory                      | After Steps 1.2, 2, 4 (phase boundaries)       | Remove entries from Recent Decisions, Recent Updates, Artifact Index older than 2 completed phases. Preserve Lessons Learned. |
| Invalidate on revision            | Before dispatching revision agent              | Mark affected entries with `[INVALIDATED — <reason>]`                                                                         |
| Clean invalidated entries         | After revision agent completes                 | Remove any remaining `[INVALIDATED]` entries that weren't replaced                                                            |
| Emergency prune                   | When memory exceeds 200 lines                  | Remove all except Lessons Learned + Artifact Index + current-phase entries                                                    |

> **[R1]** Removed "Clear cluster workspace" and "Verify agent memory write (after every agent)" actions. Cluster Workspaces no longer exist in `memory.md`. Only aggregators and sequential agents write to memory, so verification is scoped to those agents only. Emergency pruning now preserves Artifact Index.

**Memory failure handling:** If `memory.md` cannot be initialized (e.g., filesystem error), the orchestrator logs a warning and proceeds. All agents fall back to direct artifact reads. Memory is beneficial but not required for pipeline correctness.

### 7.3 Cluster Management

The orchestrator must manage 3 distinct cluster execution patterns:

**Pattern A — Fully Parallel (CT cluster at Step 3b, R cluster at Step 7):**

```
1. Dispatch N sub-agents in parallel (≤4, within Global Rule 7).
2. Wait for all N to return.
3. Handle individual errors (retry once per Global Rule 4).
4. If ≥2 sub-agents available: invoke aggregator.
5. If <2 sub-agents available (after retries): cluster ERROR.
6. Check aggregator completion contract.
```

**Pattern B — Sequential Gate + Parallel (V cluster at Step 6):**

```
1. Dispatch gate agent (V-Build).
2. Wait for gate agent.
3. If gate ERROR: retry once. If still ERROR → handle (replan or halt).
4. If gate DONE: dispatch N-1 sub-agents in parallel.
5. Wait for all N-1 to return.
6. Handle individual errors (retry once).
7. Invoke aggregator with all available outputs.
8. Check aggregator completion contract.
```

**Pattern C — Replan Loop (wraps Pattern B for V cluster):**

```
iteration_count = 0
while iteration_count < 3:
  Run Pattern B (full V cluster)
  If aggregator DONE: break
  If aggregator NEEDS_REVISION or ERROR:
    iteration_count += 1
    Invoke planner (replan mode) with verifier.md
    Execute fix tasks (Step 5 logic)
    Invalidate V-related memory entries
If iteration_count == 3 and not DONE: proceed with findings documented
```

### 7.4 Updated Completion Contracts

All new agents follow the established three-state or two-state completion contract pattern:

| Agent              | DONE                             | NEEDS_REVISION                     | ERROR                                       |
| ------------------ | -------------------------------- | ---------------------------------- | ------------------------------------------- |
| CT sub-agents (×4) | Findings produced                | N/A (only aggregator decides)      | Cannot complete review                      |
| CT Aggregator      | All findings ≤Medium severity    | ≥1 Critical/High finding           | <2 sub-agent outputs or aggregation failure |
| V-Build            | Build passes                     | N/A                                | Build fails                                 |
| V-Tests            | All tests pass                   | Test failures found                | Cannot run tests                            |
| V-Tasks            | All tasks verified               | Tasks failed verification          | Cannot verify                               |
| V-Feature          | Feature criteria met             | Feature criteria not met           | Cannot verify                               |
| V Aggregator       | All pass                         | Any NEEDS_REVISION                 | V-Build ERROR or 2+ sub-agent ERROR         |
| R-Quality          | No blocker/major issues          | Blocker/major issues found         | Cannot complete review                      |
| R-Security         | No security issues               | Security issues found              | Cannot complete scan                        |
| R-Testing          | Test coverage adequate           | Test gaps found                    | Cannot complete review                      |
| R-Knowledge        | Analysis complete (±suggestions) | N/A                                | Cannot complete analysis                    |
| R Aggregator       | All available DONE               | R-Quality/R-Testing NEEDS_REVISION | R-Security critical issue                   |

### 7.5 Updated NEEDS_REVISION Routing Table

| Returning Agent                                  | Routes To                                                                       | Max Loops | Escalation                          |
| ------------------------------------------------ | ------------------------------------------------------------------------------- | --------- | ----------------------------------- |
| CT Aggregator                                    | Designer (with `design_critical_review.md`)                                     | 1         | Proceed to planning with warning    |
| V Aggregator                                     | Planner (replan mode with `verifier.md`) → Implementers → Full V cluster re-run | 3         | Proceed with findings documented    |
| R Aggregator                                     | Implementer(s) for affected tasks                                               | 1         | Escalate to planner for full replan |
| R-Security (via R Aggregator ERROR)              | Retry R-Security once → Planner if persistent                                   | 1         | Halt pipeline                       |
| All sub-agents                                   | N/A — sub-agents route through their aggregator                                 | —         | —                                   |
| All other agents (spec, designer, planner, etc.) | N/A — these return DONE or ERROR only                                           | —         | —                                   |

### 7.6 Updated Parallel Execution Summary

```
Step 0: Setup → initial-request.md + memory.md

Step 1:
Research:   [Architecture] ──┐
            [Impact]       ──┤── parallel ×4 (max 4) → wait
            [Dependencies] ──┤
            [Patterns]     ──┘
            → Synthesize → analysis.md
            ↓ (approval gate if APPROVAL_MODE)

Step 2-3:   Spec → Design (sequential)

Step 3b:
CT Cluster: [CT-Security]        ──┐
            [CT-Scalability]      ──┤── parallel ×4 → wait
            [CT-Maintainability]  ──┤
            [CT-Strategy]         ──┘
            → CT-Aggregator → design_critical_review.md
            ↓ (NEEDS_REVISION? → Designer → full re-run, max 1 loop)

Step 4:     Planning (sequential)
            ↓ (approval gate if APPROVAL_MODE)

Step 5:
Impl Waves: [Task A] ──┐
            [Task B] ──┤── sub-wave ≤4, parallel → wait
            [Task C] ──┘
            [Task D] ──┐
            [Task E] ──┤── sub-wave ≤4, parallel → wait
            ...

Step 6:
V Cluster:  V-Build (sequential gate)
            ↓ (DONE?)
            [V-Tests]   ──┐
            [V-Tasks]   ──┤── parallel ×3 → wait
            [V-Feature] ──┘
            → V-Aggregator → verifier.md
            ↓ (NEEDS_REVISION? → Replan → Re-implement → full V re-run, max 3 loops)

Step 7:
R Cluster:  [R-Quality]  ──┐
            [R-Security]  ──┤── parallel ×4 → wait
            [R-Testing]   ──┤
            [R-Knowledge] ──┘
            → R-Aggregator → review.md + knowledge-suggestions.md
            ↓ (NEEDS_REVISION? → Implementers fix → max 1 loop)

Pipeline Complete
```

**Updated Orchestrator Expectations Table:**

| Agent                   | On DONE                             | On NEEDS_REVISION                           | On ERROR                                                               |
| ----------------------- | ----------------------------------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| Researcher (focused ×4) | Collect result; wait for all 4      | N/A                                         | Retry once; if still ERROR, synthesis proceeds with available partials |
| Researcher (synthesis)  | Proceed to spec (or approval gate)  | N/A                                         | Retry once; halt if persistent                                         |
| Spec                    | Proceed to design                   | N/A                                         | Retry once; halt if persistent                                         |
| Designer                | Proceed to CT cluster               | N/A                                         | Retry once; halt if persistent                                         |
| CT sub-agents (×4)      | Collect result; wait for all 4      | N/A                                         | Retry once; aggregator proceeds with ≥2 outputs                        |
| CT Aggregator           | Proceed to planning                 | Route to designer (max 1 loop)              | Retry once; halt if persistent                                         |
| Planner                 | Proceed to implementation (or gate) | N/A                                         | Retry once; halt if persistent                                         |
| Implementer (×N)        | Collect; wait for sub-wave          | N/A (DONE/ERROR only)                       | Record failure; wait; proceed to verification                          |
| Documentation Writer    | Collect (same as implementer)       | N/A                                         | Record failure; continue                                               |
| V-Build                 | Dispatch V-Tests/V-Tasks/V-Feature  | N/A (DONE/ERROR only)                       | Retry once; if persistent, enter replan                                |
| V-Tests                 | Collect; wait for parallel group    | Route through V-Aggregator                  | Retry once; aggregator proceeds with available                         |
| V-Tasks                 | Collect; wait for parallel group    | Route through V-Aggregator                  | Retry once; aggregator proceeds with available                         |
| V-Feature               | Collect; wait for parallel group    | Route through V-Aggregator                  | Retry once; aggregator proceeds with available                         |
| V Aggregator            | Proceed to review                   | Trigger replan loop (max 3)                 | Trigger replan loop                                                    |
| R-Quality               | Collect; wait for all 4             | Route through R-Aggregator                  | Retry once; aggregator proceeds with available                         |
| R-Security              | Collect; wait for all 4             | Route through R-Aggregator (ERROR override) | Retry once; if persistent, aggregator ERROR                            |
| R-Testing               | Collect; wait for all 4             | Route through R-Aggregator                  | Retry once; aggregator proceeds with available                         |
| R-Knowledge             | Collect; wait for all 4             | N/A (DONE/ERROR only)                       | Non-blocking; aggregator proceeds without                              |
| R Aggregator            | Workflow complete                   | Route findings to implementers (max 1 loop) | Trigger replan via planner                                             |

---

## 8. Per-Agent Memory Integration

### 8.1 Memory Protocol (All Agents)

Every agent definition receives the following additions:

**1. Input addition (Inputs section):**

```markdown
- docs/feature/<feature-slug>/memory.md (read first — operational memory)
```

This is added as the FIRST item in every agent's Inputs list.

> **[R1 — Minor]** For researchers at Step 1.1, `memory.md` is nearly empty (just initialized with blank templates). The memory-first read adds a workflow step with minimal informational value at this stage. The rule is kept for consistency across all agents, but implementers should note that memory provides no navigational value until after Step 1.2 (synthesis).

**2. New Operating Rule (added as Rule 6 in the Operating Rules section):**

```markdown
6. **Memory-first reading:** Read `memory.md` FIRST before accessing any artifact.
   Use the Artifact Index to navigate directly to relevant sections rather than
   reading full artifacts. If `memory.md` is missing, log a warning and proceed
   with direct artifact reads.
```

**3. Workflow Step 1 (prepended to existing workflow):**

```markdown
1. Read `memory.md` to load artifact index, recent decisions, lessons learned,
   and recent updates. Use this to orient before reading source artifacts.
```

All existing workflow steps are renumbered (Step 1 → Step 2, etc.).

**4. Workflow penultimate step — SEQUENTIAL AGENTS ONLY (inserted before completion contract return):**

> **[R1 — Critical Fix]** This memory write step applies ONLY to sequential agents (spec, designer, planner, documentation writer) and aggregators. Parallel agents (researchers in focused mode, CT sub-agents, V sub-agents, R sub-agents, implementers) do NOT write to `memory.md`. Their findings are communicated through intermediate output files.

For **sequential agents and aggregators:**

```markdown
N. Update `memory.md`: append to the appropriate sections:

- **Artifact Index:** If a new artifact was produced, add its path and key sections.
- **Recent Decisions:** If decisions were made during this step (≤2 sentences each).
- **Lessons Learned:** If issues were encountered and resolved (≤2 sentences each).
- **Recent Updates:** Summary of output produced (≤2 sentences).
```

For **parallel agents (sub-agents and implementers):**

```markdown
N. (No memory write step — findings are communicated through intermediate output files.
The aggregator or orchestrator will consolidate relevant findings into memory
after all parallel work completes.)
```

### 8.2 Agent-Specific Changes

Below is the specific memory integration for each existing agent. New sub-agents and aggregators will have memory built into their initial definitions.

**Researcher (`researcher.agent.md`):**

- Input: Add `memory.md` as first input (both focused and synthesis modes).
- Rule 6: Memory-first reading.
- Focused mode workflow: Step 1 reads memory → existing steps renumbered. **No memory write step** (parallel agent — findings go to `research/<focus>.md`).
- Synthesis mode workflow: Step 1 reads memory → existing steps renumbered. Penultimate step writes to Artifact Index and Recent Updates (root-level — synthesis runs sequentially, safe to write).
- Focus area table: Add `patterns` row (see §6.1).

**Spec (`spec.agent.md`):**

- Input: Add `memory.md` as first input.
- Rule 6: Memory-first reading.
- Workflow: Prepend memory read, append memory write (Artifact Index + Recent Decisions + Recent Updates). Sequential agent — safe.

**Designer (`designer.agent.md`):**

- Input: Add `memory.md` as first input.
- Rule 6: Memory-first reading.
- Workflow: Prepend memory read, append memory write (Artifact Index + Recent Decisions + Recent Updates). Sequential agent — safe.

**Critical Thinker (`critical-thinker.agent.md`):**

- This file is superseded by 4 CT sub-agents + CT aggregator. The original file is retained for backward compatibility reference but is no longer dispatched by the orchestrator.
- Alternative: Remove the file entirely and rely on `ct-*.agent.md` files. **Recommendation:** Remove to avoid confusion. Add a note to `docs/feature/<feature-slug>/decisions.md` recording this change.
- Must be removed/superseded in **both** `.github/agents/` and `NewAgentsAndPrompts/`.

**Planner (`planner.agent.md`):**

- Input: Add `memory.md` as first input.
- Rule 6: Memory-first reading.
- Workflow: Prepend memory read, append memory write (Artifact Index + Recent Decisions + Recent Updates). Sequential agent — safe.

**Implementer (`implementer.agent.md`):**

- Input: Add `memory.md` as first input.
- Rule 6: Memory-first reading.
- Workflow: Prepend memory read. **No memory write step** (parallel agent — issues and lessons captured in task output files).
- **Note:** The orchestrator extracts Lessons Learned from completed task outputs between implementation waves and appends them to `memory.md` (sequential operation — safe).

**Verifier (`verifier.agent.md`):**

- This file is superseded by 4 V sub-agents + V aggregator. Same recommendation as critical-thinker: remove and replace.
- Must be removed/superseded in **both** `.github/agents/` and `NewAgentsAndPrompts/`.

**Reviewer (`reviewer.agent.md`):**

- This file is superseded by 4 R sub-agents + R aggregator. Same recommendation as critical-thinker: remove and replace.
- Must be removed/superseded in **both** `.github/agents/` and `NewAgentsAndPrompts/`.

**Documentation Writer (`documentation-writer.agent.md`):**

- Input: Add `memory.md` as first input.
- Rule 6: Memory-first reading.
- Workflow: Prepend memory read, append memory write (Artifact Index + Recent Updates). Sequential agent (runs within implementation waves, but only one doc writer per task) — safe.

**Feature Workflow Prompt (`feature-workflow.prompt.md`):**

- Add references to the memory system and cluster model in the Rules section:
  ```markdown
  - The orchestrator initializes `memory.md` during setup. All agents read memory
    before accessing artifacts. Only aggregators and sequential agents write to memory.
  - Critical Thinking, Verification, and Review are now handled by parallel agent
    clusters (4 sub-agents each) with aggregators. See design.md §3, §4, §5 for details.
  - Knowledge Evolution produces `knowledge-suggestions.md` — review this file
    post-pipeline for actionable improvements to instructions and patterns.
  ```
- No memory read/write changes (this is a prompt file, not an agent — it doesn't directly interact with memory).

**Orchestrator (`orchestrator.agent.md`):**

- Memory management logic is part of orchestrator evolution (§7.2).
- The orchestrator reads memory for lifecycle management, not for operational context.
- Documentation Structure section: add `memory.md` to the file list.
- Add memory lifecycle actions to each workflow step.

---

## 9. Security Considerations

### 9.1 Authentication/Authorization

**N/A with justification:** Forge is a local-execution, single-user pipeline running within a VS Code Copilot agent runtime. There are no network endpoints, no multi-user access, no authentication tokens, and no API keys exchanged between agents. All agent communication is mediated through files on the local filesystem. There is no authentication or authorization surface to protect.

### 9.2 Data Protection

| Concern                                  | Assessment                                                   | Mitigation                                                                                                                                                                                                                                                                                               |
| ---------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Secrets in memory/artifacts              | Agents may encounter secrets during codebase analysis        | Existing Secret/PII scan rules (in implementer, reviewer, verifier) are preserved and extended to all new sub-agents. R-Security inherits the full security scan. Memory entries must never contain secrets — the 2-sentence limit and "no code snippets" rule (MEM-AC-7) provide structural protection. |
| Knowledge suggestions containing secrets | R-Knowledge may capture patterns that include sensitive data | KE-SAFE-6 filters suggestions that weaken safety. Additionally, R-Knowledge's output format requires `diff` blocks — reviewers can inspect these for secrets before applying.                                                                                                                            |
| Memory persistence                       | `memory.md` persists after pipeline completion               | Memory contains only navigational metadata, decisions, and lessons — no source code, no secrets, no PII by design (MEM-AC-7). If the project requires cleanup, memory can be deleted with no impact on artifact integrity.                                                                               |

### 9.3 Threat Model

| Threat                                                               | Likelihood | Impact   | Mitigation                                                                                                                                                                                                                                                               |
| -------------------------------------------------------------------- | ---------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Self-modification via Knowledge Evolution**                        | Medium     | Critical | R-Knowledge writes to `knowledge-suggestions.md` only (KE-SAFE-1). Pipeline never auto-applies suggestions (KE-SAFE-4). Safety constraint filter (KE-SAFE-6) prevents weakening protections. Human approval required.                                                    |
| **Memory poisoning (stale/corrupted entries mislead agents)**        | Medium     | High     | Memory is navigational, not authoritative. Agents verify critical decisions against source artifacts. Invalidation protocol on revision loops. Emergency pruning if corrupted.                                                                                           |
| **Agent scope escalation (sub-agent writes outside its boundaries)** | Low        | High     | File Boundaries operating rule (Rule 4) in every agent. Anti-drift anchor reinforces role constraints. Aggregators never write to source code or agent definitions.                                                                                                      |
| **Cross-cluster information leakage**                                | Low        | Low      | Clusters run at different pipeline stages (never concurrent). Sub-agent intermediate files are scoped to per-cluster directories (`ct-review/`, `verification/`, `review/`). No shared state beyond memory and artifacts, which are designed for cross-agent visibility. |

### 9.4 Input Validation

| Input                                      | Validation                                                                                                                                                              | Owner        |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| Sub-agent output files (aggregator inputs) | Aggregator validates: file exists, has expected section headers, is non-empty. If invalid, treats as missing and proceeds with available outputs.                       | Aggregator   |
| `memory.md`                                | Orchestrator validates section headers after each step. If malformed, re-initializes with empty template.                                                               | Orchestrator |
| `v-build.md` (V cluster)                   | V-Tests/V-Tasks/V-Feature validate: file exists, contains build status and artifact paths. If missing or invalid, return ERROR (cannot verify without build context).   | V sub-agents |
| `knowledge-suggestions.md`                 | Orchestrator verifies: file exists after R-Knowledge, contains WARNING header. If header missing, orchestrator adds it. Never auto-applied regardless.                  | Orchestrator |
| `decisions.md` (R-Knowledge writes)        | R-Knowledge verifies existing entries are unchanged after its append. If existing entries were modified (shouldn't happen given the agent's constraints), logs warning. | R-Knowledge  |

---

## 10. Failure Analysis

### 10.1 Single Points of Failure

| Component          | SPOF Risk               | Impact                                               | Mitigation                                                                                                                                       |
| ------------------ | ----------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `memory.md`        | Medium                  | All agents read memory first; corruption affects all | Graceful degradation: agents fall back to direct artifact reads. Orchestrator re-initializes on corruption.                                      |
| Orchestrator       | High (inherent)         | Controls entire pipeline; failure halts everything   | Unchanged from current architecture. Retry + error handling per Global Rules.                                                                    |
| V-Build            | High (within V cluster) | Build failure blocks all verification                | Design choice: V-Build must pass before others run. Replan loop provides recovery. Build failures are explicitly logged with actionable details. |
| R-Security         | Medium                  | Security ERROR blocks pipeline (design choice)       | Retry once. If persistent, pipeline halts — this is intentional (security is non-negotiable).                                                    |
| CT/V/R Aggregators | Medium                  | Aggregator failure loses all sub-agent work          | Retry once with same inputs. Sub-agent intermediate files persist on disk for manual inspection even if aggregator fails.                        |

### 10.2 Cascading Failure Scenarios

**Scenario 1: Memory corruption during orchestrator write**

```
Trigger: Orchestrator or aggregator writes to memory.md with malformed content (e.g., truncated markdown, broken table).
Cascade: Subsequent agents read corrupted memory → incorrect navigation → suboptimal artifact reads.
Impact: Performance degradation (more artifact reads), NOT correctness issues (artifacts are source of truth).
Recovery: Orchestrator detects via section-header validation → re-initializes memory with empty templates → agents fall back to direct reads.
```

> **[R1]** Original scenario described concurrent implementer writes corrupting memory. Since parallel agents no longer write to `memory.md`, concurrent write corruption is no longer possible. The revised scenario covers the remaining (lower-probability) corruption vector.

**Scenario 2: V-Build fails 3 times in replan loop**

```
Trigger: Build error not addressable by planner/implementer (e.g., fundamental architecture issue, external dependency broken).
Cascade: 3 replan iterations exhausted → V cluster never reaches V-Tests/V-Tasks/V-Feature → pipeline proceeds with incomplete verification.
Impact: Feature ships with unknown integration status.
Recovery: Pipeline documents failure in verifier.md. Human intervention required. This is the existing behavior — the cluster doesn't make it worse.
```

**Scenario 3: CT sub-agents disagree fundamentally**

```
Trigger: CT-Security says "must encrypt all data at rest" (critical). CT-Scalability says "encryption adds 40ms per operation, unacceptable for real-time path" (high).
Cascade: Both findings appear in Unresolved Tensions. Designer must resolve during revision.
Impact: If designer doesn't resolve: tension persists into planning. Planner receives unresolved tensions as explicit planning constraints (see §3.3 R1 note).
Recovery: Aggregator surfaces tension explicitly. Max 1 revision loop allows designer to address. If unresolved after revision, tensions are formatted as planning constraints in `design_critical_review.md` so the planner can incorporate them into task design.
```

**Scenario 4: R-Knowledge produces harmful suggestions**

```
Trigger: R-Knowledge suggests removing an error handling pattern or weakening a validation rule.
Cascade: If auto-applied, weakens system safety.
Impact: Critical — if it bypassed safeguards.
Recovery: Multiple defense layers:
  1. KE-SAFE-6 filters the suggestion during R-Knowledge execution (logged in Rejected Suggestions).
  2. KE-SAFE-4 ensures no auto-apply (suggestions are buffered only).
  3. KE-SAFE-5 adds WARNING header to suggestions file.
  4. Human reviews before any application.
  5. Even if all above fail: the suggestion is in a separate file, not in agent definitions.
```

### 10.3 Error Recovery Layers

Error recovery operates at 4 layers (3 existing + 1 new):

| Layer                     | Scope                                | Mechanism                                                                              | Max Retries            |
| ------------------------- | ------------------------------------ | -------------------------------------------------------------------------------------- | ---------------------- |
| **Tool-level**            | Individual tool call within an agent | Agent retries transient errors                                                         | 2                      |
| **Agent-level**           | Single agent invocation              | Orchestrator retries ERROR'd agents                                                    | 1                      |
| **Cluster-level** _(NEW)_ | Partial cluster failure              | Aggregator proceeds with available outputs; orchestrator retries individual sub-agents | 1 per sub-agent        |
| **Pipeline-level**        | Pipeline stage failure               | Replan loop (V cluster), revision loop (CT cluster), fix loop (R cluster)              | 1–3 depending on stage |

**Worst-case retry analysis:**

- Tool: 1 + 2 retries = 3 tool calls
- Agent: × 2 orchestrator attempts = 6 total tool calls per sub-agent
- Cluster: × 4 sub-agents = 24 tool calls for full cluster failure (unlikely — usually 1 sub-agent fails)
- Pipeline: × 3 replan iterations = 72 tool calls (absolute worst case for V cluster)

**Practical bound:** Most failures are isolated to 1 sub-agent. The realistic worst case is ~12 tool calls (1 sub-agent × 6 tool calls × 2 for retry at cluster level).

---

## 11. Migration Strategy

### 11.1 Phased Implementation Order

The implementation follows the dependency-optimized plan from the analysis, with orchestrator changes applied atomically after all component designs are finalized.

**Phase 1: Foundation (can be parallelized)**
| Work Item | Files Created/Modified | Dependencies |
|-----------|----------------------|-------------|
| Memory schema design | (design artifact — no files yet) | None |
| Aggregator pattern template | (design artifact — no files yet) | None |
| Research expansion (C5) | `researcher.agent.md`, `orchestrator.agent.md` | None — ship independently |

**Phase 2: Sub-Agent Creation (parallelizable)**
| Work Item | Files Created | Dependencies |
|-----------|--------------|-------------|
| CT cluster sub-agents | `ct-security.agent.md`, `ct-scalability.agent.md`, `ct-maintainability.agent.md`, `ct-strategy.agent.md`, `ct-aggregator.agent.md` | Aggregator pattern template |
| V cluster sub-agents | `v-build.agent.md`, `v-tests.agent.md`, `v-tasks.agent.md`, `v-feature.agent.md`, `v-aggregator.agent.md` | Aggregator pattern template |
| R cluster sub-agents | `r-quality.agent.md`, `r-security.agent.md`, `r-testing.agent.md`, `r-knowledge.agent.md`, `r-aggregator.agent.md` | Aggregator pattern template |
| Memory integration (non-split agents) | `spec.agent.md`, `designer.agent.md`, `planner.agent.md`, `implementer.agent.md`, `documentation-writer.agent.md`, `feature-workflow.prompt.md` | Memory schema design |

**Phase 3: Orchestrator Update (sequential — bottleneck)**
| Work Item | Files Modified | Dependencies |
|-----------|---------------|-------------|
| Orchestrator memory init + global rules | `orchestrator.agent.md` | Phase 1 |
| Orchestrator Step 1.1 (research ×4) | `orchestrator.agent.md` | Phase 1 |
| Orchestrator Step 3b (CT cluster) | `orchestrator.agent.md` | Phase 2 (CT sub-agents) |
| Orchestrator Step 6 (V cluster) | `orchestrator.agent.md` | Phase 2 (V sub-agents) |
| Orchestrator Step 7 (R cluster) | `orchestrator.agent.md` | Phase 2 (R sub-agents) |
| Orchestrator routing/expectations tables | `orchestrator.agent.md` | All above |

**Recommendation:** Apply all orchestrator changes in a single atomic update to avoid intermediate broken states. The orchestrator is the serialization bottleneck — partial updates create an inconsistent pipeline.

**Phase 4: Cleanup and Memory Integration for New Agents**
| Work Item | Files Modified | Dependencies |
|-----------|---------------|-------------|
| Memory integration for all 15 sub-agents + 3 aggregators | All new `.agent.md` files | Phase 2, Phase 3 |
| Remove superseded files | `critical-thinker.agent.md`, `verifier.agent.md`, `reviewer.agent.md` | Phase 3 |

**Phase 5: Validation**
| Work Item | Method | Dependencies |
|-----------|--------|-------------|
| Full pipeline run on representative feature | Execute feature-workflow prompt end-to-end | All phases |
| Verify all acceptance criteria | Manual inspection of outputs | Phase 5 pipeline run |

### 11.2 Backward Compatibility

| Concern                   | Strategy                                                                                            |
| ------------------------- | --------------------------------------------------------------------------------------------------- |
| Existing artifact formats | Unchanged. `feature.md`, `design.md`, `plan.md`, `verifier.md`, `review.md` retain their structure. |
| Pipeline step order       | 8-step pipeline preserved. Clusters are internal to existing steps.                                 |
| Revision loop limits      | Max 1 CT loop, max 3 V loops, max 1 R loop — all preserved.                                         |
| Two-layer verification    | Implementer unit-level TDD + V cluster integration-level verification — preserved.                  |
| Concurrency cap           | Max 4 concurrent subagents — preserved and sufficient for all clusters.                             |

### 11.3 Rollback Strategy

If the upgrade causes pipeline failures:

1. **Per-cluster rollback:** Replace cluster dispatch in orchestrator with single-agent dispatch (revert to the superseded agent file).
2. **Memory rollback:** Remove memory steps from orchestrator and agents. Pipeline functions without memory (current behavior).
3. **Full rollback:** Revert all `.agent.md` files to pre-upgrade versions. Git-based rollback is the simplest approach since all changes are markdown files.

Superseded agent files (`critical-thinker.agent.md`, `verifier.agent.md`, `reviewer.agent.md`) should be retained in a `_deprecated/` directory or via git history until the upgrade is validated through at least one full pipeline run. Rollback must be applied to **both** `.github/agents/` and `NewAgentsAndPrompts/`.

---

## Appendix A: New File Inventory

| #   | File                          | Type       | Cluster |
| --- | ----------------------------- | ---------- | ------- |
| 1   | `ct-security.agent.md`        | Sub-agent  | CT      |
| 2   | `ct-scalability.agent.md`     | Sub-agent  | CT      |
| 3   | `ct-maintainability.agent.md` | Sub-agent  | CT      |
| 4   | `ct-strategy.agent.md`        | Sub-agent  | CT      |
| 5   | `ct-aggregator.agent.md`      | Aggregator | CT      |
| 6   | `v-build.agent.md`            | Sub-agent  | V       |
| 7   | `v-tests.agent.md`            | Sub-agent  | V       |
| 8   | `v-tasks.agent.md`            | Sub-agent  | V       |
| 9   | `v-feature.agent.md`          | Sub-agent  | V       |
| 10  | `v-aggregator.agent.md`       | Aggregator | V       |
| 11  | `r-quality.agent.md`          | Sub-agent  | R       |
| 12  | `r-security.agent.md`         | Sub-agent  | R       |
| 13  | `r-testing.agent.md`          | Sub-agent  | R       |
| 14  | `r-knowledge.agent.md`        | Sub-agent  | R       |
| 15  | `r-aggregator.agent.md`       | Aggregator | R       |

> **[R1 — Critical Fix: Canonical Agent Location]** All 15 new files are authored in `NewAgentsAndPrompts/` and **must also be deployed** to `.github/agents/` (the VS Code Copilot runtime's canonical discovery location). Both locations must contain identical copies. Total: 15 new files × 2 locations = 30 file operations.

## Appendix B: Modified File Inventory

| #   | File                            | Change Type   | Scope                                                                                          |
| --- | ------------------------------- | ------------- | ---------------------------------------------------------------------------------------------- |
| 1   | `orchestrator.agent.md`         | Major rewrite | ~150–200 lines modified/added (R1: revised from ~200–250 based on corrected 234-line baseline) |
| 2   | `researcher.agent.md`           | Minor update  | Add patterns focus area (~5 lines)                                                             |
| 3   | `spec.agent.md`                 | Minor update  | Memory integration (~10 lines)                                                                 |
| 4   | `designer.agent.md`             | Minor update  | Memory integration (~10 lines)                                                                 |
| 5   | `planner.agent.md`              | Minor update  | Memory integration (~10 lines)                                                                 |
| 6   | `implementer.agent.md`          | Minor update  | Memory integration (~8 lines — read-only, no write step)                                       |
| 7   | `documentation-writer.agent.md` | Minor update  | Memory integration (~10 lines)                                                                 |
| 8   | `feature-workflow.prompt.md`    | Minor update  | Memory + cluster model + knowledge evolution references (~8 lines)                             |

> **[R1 — Critical Fix]** All 8 modified files exist in both `NewAgentsAndPrompts/` and `.github/agents/` (or `.github/prompts/` for the prompt file). Changes must be applied to both locations. Orchestrator addition estimate revised from ~200–250 to ~150–200 lines based on corrected 234-line baseline (post-upgrade target: ~384–434 lines, well within the revised 450-line cap).

## Appendix C: Acceptance Criteria Traceability

| Acceptance Criterion              | Design Section              | Implementation Path                                                |
| --------------------------------- | --------------------------- | ------------------------------------------------------------------ |
| MEM-AC-1 through MEM-AC-8         | §1 (Memory System)          | Orchestrator Step 0 init; per-agent memory protocol; pruning logic |
| CT-AC-1 through CT-AC-9           | §3 (CT Cluster)             | 4 sub-agent files + aggregator; orchestrator Step 3b rewrite       |
| V-AC-1 through V-AC-10            | §4 (V Cluster)              | 4 sub-agent files + aggregator; orchestrator Step 6 rewrite        |
| R-AC-1 through R-AC-11            | §5 (R Cluster)              | 4 sub-agent files + aggregator; orchestrator Step 7 rewrite        |
| KE-SAFE-1 through KE-SAFE-7       | §5.3 (Safeguards)           | R-Knowledge agent definition constraints                           |
| RES-AC-1 through RES-AC-6         | §6 (Research Expansion)     | Researcher + orchestrator minor updates                            |
| ORCH-AC-1 through ORCH-AC-14      | §7 (Orchestrator Evolution) | Orchestrator rewrite                                               |
| MEM-INT-AC-1 through MEM-INT-AC-7 | §8 (Per-Agent Memory)       | Per-agent additions across all files                               |

## Appendix D: Open Design Decisions Resolved

| Question (from analysis.md)             | Resolution                                                                                                                                     | Rationale                                                                                                                                                                                                                                                                            |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Memory concurrency model                | **[R1 — REVISED]** Aggregator-only writes                                                                                                      | Parallel sub-agents do NOT write to `memory.md`. Only aggregators and sequential agents write. Sub-agents communicate through intermediate output files. This avoids concurrent write conflicts with VS Code's `replace_string_in_file` tool.                                        |
| 4th research focus area                 | Patterns & Conventions                                                                                                                         | Covers testing strategy, code conventions, error handling patterns — gaps not addressed by architecture/impact/dependencies.                                                                                                                                                         |
| Orchestrator decomposition              | Single agent (no sub-orchestrators)                                                                                                            | Per ORCH-AC-13. Cluster patterns are embedded in orchestrator steps as named protocol references. Decomposition increases file count without clear benefit at current complexity. **[R1]** Revisit if orchestrator exceeds 450 lines (revised from 550 based on corrected baseline). |
| Aggregator placement                    | Standalone agents dispatched by orchestrator                                                                                                   | Consistent with agent isolation principle. Aggregators are full v2-template agents.                                                                                                                                                                                                  |
| Knowledge Evolution scope               | Suggestion-only to `knowledge-suggestions.md`                                                                                                  | Per KE-SAFE-1 through KE-SAFE-4. No auto-modification. Human approval required.                                                                                                                                                                                                      |
| Sub-agent naming                        | Prefix-based (`ct-security.agent.md`)                                                                                                          | **[R1]** Primary location: `.github/agents/`. Synced to `NewAgentsAndPrompts/`. Prefix groups them logically. Directory-based would add nesting without clear benefit in the Copilot agent runtime.                                                                                  |
| Backward compatibility during migration | Atomic orchestrator update; superseded files retained                                                                                          | Partial updates create broken states. Retain old files until validated.                                                                                                                                                                                                              |
| Build state sharing (V cluster)         | V-Build writes all state to `v-build.md` on disk                                                                                               | No reliance on terminal state or environment variables. File-based communication is consistent with Forge's architecture. `v-build.md` provides informational context; actual build artifacts persist on the shared filesystem (which is what V-Tests functionally depends on).      |
| `decisions.md` ownership                | R-Knowledge owns all writes                                                                                                                    | R-Knowledge has the broadest context (reads all pipeline artifacts). Aggregator doesn't touch it.                                                                                                                                                                                    |
| `.github` directory                     | **[R1 — REVISED]** `.github/agents/` and `.github/prompts/` already exist with production agent files. Only `.github/instructions/` is absent. | The analysis.md claim that "No `.github` directory currently exists" was factually incorrect. `.github/agents/` is the VS Code Copilot runtime's canonical discovery location. R-Knowledge may suggest creating `.github/instructions/` via `knowledge-suggestions.md`.              |
| **[R1 NEW]** Agent canonical location   | `.github/agents/` is primary (runtime); `NewAgentsAndPrompts/` is secondary (authoring)                                                        | Both directories exist with matching content. The VS Code Copilot runtime reads from `.github/agents/`. All new and modified files must be deployed to `.github/agents/` with copies in `NewAgentsAndPrompts/`.                                                                      |
