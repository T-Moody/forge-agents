# Research: Patterns

## Focus Area

patterns

## Summary

All 24 agent files follow a rigorous, layered convention system with standardized sections (Operating Rules, Completion Contract, Anti-Drift Anchor), consistent memory-first reading, uniform error handling with retry budgets, and a clear read-only vs read-write memory separation enforced by the orchestrator.

## Findings

### Common Agent File Conventions

Every agent file follows a consistent structure, observed across all 24 files in `NewAgentsAndPrompts/`:

#### 1. YAML Frontmatter

All files open with a `chatagent` fenced code block containing YAML metadata:

```yaml
---
name: <agent-name>
description: "<short description>"
---
```

#### 2. Standard Sections (in order)

Every agent file includes these sections in consistent order:

1. **Title & Role Statement** — `# <Agent Name> Workflow` followed by a bold "You are the **<Agent Name>**" identity statement.
2. **Role Prohibition** — "You NEVER write/modify X" constraints specific to the agent.
3. **Detailed Thinking Directive** — `Use detailed thinking to reason through complex decisions before acting.` with `<!-- experimental: model-dependent -->` comment (present in all agents).
4. **Inputs** — Explicit list of input files with `memory.md` always listed first (annotated "read first — operational memory" or "read first — do NOT write").
5. **Outputs** — Explicit list of output files. Agents are strictly bounded to these files.
6. **Operating Rules** — Numbered list (5–6 rules), nearly identical across all agents (see Error Handling Patterns below).
7. **Workflow** — Numbered steps, always starting with "Read Memory" (step 0 or 1).
8. **Completion Contract** — Standardized return format (see below).
9. **Anti-Drift Anchor** — Final section, bold "REMEMBER:" reminder of agent identity and constraints.

#### 3. Read-Only Enforcement Section

Non-code-modifying agents include an explicit "Read-Only Enforcement" section stating they "MUST NOT modify source code, test files, or configuration files" and listing the sole file they write (their output artifact). Found in: v-build, v-tests, v-tasks, v-feature, r-quality, r-security, r-testing, r-knowledge, documentation-writer.

#### 4. Self-Verification Step

Several agents include a self-verification step before returning:

- **spec:** Verifies all acceptance criteria are testable, all requirements have criteria, edge cases cover failure modes.
- **designer:** Verifies design addresses all requirements, every acceptance criterion has implementation path, security considerations addressed.
- **implementer:** Self-Reflection step for Medium/High effort tasks ("Would a senior engineer approve this code?").
- **r-quality:** Self-Reflection verifying all changed files reviewed, comments are actionable, every finding includes file path and rationale.

### Memory Access Patterns

#### Current Architecture: Shared `memory.md`

All agents read a single shared `docs/feature/<feature-slug>/memory.md`. Memory access is divided into two tiers:

##### Read-Only (Parallel) Agents

These agents read `memory.md` but do NOT write to it. They are safe for parallel dispatch:

- **Research cluster:** researcher (focused mode) ×4
- **CT cluster:** ct-security, ct-scalability, ct-maintainability, ct-strategy
- **V cluster:** v-build, v-tests, v-tasks, v-feature
- **R cluster:** r-quality, r-security, r-testing, r-knowledge
- **Implementation:** implementer ×N, documentation-writer

Each read-only agent has an explicit "No Memory Write" workflow step (e.g., `### 5. No Memory Write`) with a boilerplate comment:

```
(No memory write step — findings are communicated through `<artifact-path>`. The <Aggregator> will consolidate relevant findings into memory after all sub-agents complete.)
```

Alternatively, some use input annotation: `memory.md (read first — do NOT write)`.

##### Read-Write (Sequential) Agents

These agents both read and write to `memory.md`. They run sequentially (never in parallel):

- **Sequential pipeline agents:** spec, designer, planner
- **Aggregator agents:** ct-aggregator, v-aggregator, r-aggregator
- **Synthesis agent:** researcher (synthesis mode)

Write pattern for sequential agents (spec, designer, planner):

```markdown
Update `memory.md`: append to the appropriate sections:

- **Artifact Index:** Add path and key sections of the output artifact.
- **Recent Decisions:** If decisions were made (≤2 sentences each).
- **Recent Updates:** Summary of output produced (≤2 sentences).
```

Write pattern for aggregator agents additionally includes:

- **Lessons Learned** (v-aggregator, r-aggregator)

##### Memory-First Reading Protocol

All agents follow this protocol (Operating Rule 6 in every agent):

```
Read `memory.md` FIRST before accessing any artifact.
Use the Artifact Index to navigate directly to relevant sections rather than reading full artifacts.
If `memory.md` is missing, log a warning and proceed with direct artifact reads.
```

This is also the first step of every agent's Workflow section.

##### Memory Template

The orchestrator initializes `memory.md` with this structure:

```markdown
# Operational Memory

## Artifact Index

| Artifact | Key Sections | Last Updated By |

## Recent Decisions

## Lessons Learned

## Recent Updates
```

##### Memory Lifecycle (Orchestrator-Managed)

| Action                 | When                                       | What                                                                                                                            |
| ---------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| Initialize             | Step 0                                     | Create with empty template                                                                                                      |
| Prune                  | After Steps 1.2, 2, 4                      | Remove entries older than 2 phases from Recent Decisions and Recent Updates; preserve Artifact Index and Lessons Learned always |
| Extract Lessons        | Between implementation waves               | Append issue/resolution entries to Lessons Learned                                                                              |
| Invalidate on revision | Before dispatching revision agent          | Mark affected entries with `[INVALIDATED — <reason>]`                                                                           |
| Clean invalidated      | After revision agent completes             | Remove remaining `[INVALIDATED]` entries                                                                                        |
| Validate               | After aggregators/sequential agents return | Check agent wrote to memory; log warning if not (non-blocking)                                                                  |

##### Key Design Property

Memory is **non-blocking**: if `memory.md` cannot be created, becomes corrupted, or is missing, all agents fall back to direct artifact reads. This makes memory beneficial but not required (Global Rule 6 in orchestrator).

### Completion Contract Patterns

Three distinct completion contract patterns are used depending on agent role:

#### Pattern 1: DONE/ERROR only (most agents)

Used by: researcher (focused), spec, designer, planner, implementer, documentation-writer, v-build, all CT sub-agents (ct-security, ct-scalability, ct-maintainability, ct-strategy).

```
Return exactly one line:
- DONE: <context-specific summary>
- ERROR: <reason>
```

Sub-agents explicitly state: "Sub-agents never return `NEEDS_REVISION` — only the aggregator makes that determination."

Variations in DONE format:

- **researcher (focused):** `DONE: <focus-area> — <summary>`
- **researcher (synthesis):** `DONE: synthesis — <summary>`
- **spec:** `DONE: <one-line summary>`
- **planner:** `DONE: <N> tasks created, <M> waves`
- **implementer/doc-writer:** `DONE: <task-id>`
- **v-build:** `DONE: build passed — <build system> (<language version>)`
- **ct-security:** `DONE: security — <summary>`

#### Pattern 2: DONE/NEEDS_REVISION/ERROR (aggregators + some V/R sub-agents)

Used by: ct-aggregator, v-aggregator, r-aggregator, v-tests, v-tasks, v-feature, r-quality, r-security, r-testing.

```
Return exactly one line:
- DONE: <summary>
- NEEDS_REVISION: <summary> — <details>
- ERROR: <reason>
```

Key distinctions:

- **ct-aggregator:** NEEDS_REVISION when any Critical/High severity finding exists; DONE when all ≤Medium.
- **v-aggregator:** Uses a decision table (build status × test status × tasks status × feature status) to determine contract.
- **r-aggregator:** ERROR overrides for R-Security (Critical/Blocker findings block pipeline); R-Knowledge ERROR is non-blocking.

#### Pattern 3: Orchestrator (no NEEDS_REVISION self-return)

The orchestrator does not return NEEDS_REVISION itself. It handles NEEDS_REVISION from aggregators by routing to appropriate agents per the NEEDS_REVISION Routing Table.

Final orchestrator contract:

```
Workflow completes only when R Aggregator returns DONE.
ERROR: <summary of unresolved issues> (if workflow cannot complete)
```

### Cluster Dispatch Patterns

Three reusable dispatch patterns defined in `dispatch-patterns.md` and `orchestrator.agent.md`:

#### Pattern A — Fully Parallel

**Used by:** CT cluster (Step 3b), R cluster (Step 7), Research focused (Step 1.1).

Steps:

1. Dispatch N sub-agents in parallel (≤4 concurrency cap).
2. Wait for all N to return.
3. Handle individual errors: retry once per Global Rule 4.
4. If ≥2 sub-agent outputs available: invoke aggregator.
5. If <2 outputs available after retries: cluster ERROR.
6. Check aggregator completion contract.

Key convention: **≥2 outputs required** for aggregation to proceed. <2 = cluster ERROR.

#### Pattern B — Sequential Gate + Parallel

**Used by:** V cluster (Step 6).

Steps:

1. Dispatch gate agent (V-Build) — sequential.
2. Wait for gate agent.
3. If gate ERROR: retry once. If still ERROR → skip parallel, forward ERROR to aggregator.
4. If gate DONE: dispatch N-1 sub-agents in parallel.
5. Wait for all N-1 to return. Handle errors: retry once each.
6. Invoke aggregator with all available outputs.
7. Check aggregator completion contract.

#### Pattern C — Replan Loop (wraps Pattern B)

**Used by:** V cluster (Step 6), exclusive to verification.

```
iteration = 0
while iteration < 3:
    Run Pattern B (full V cluster)
    If aggregator DONE: break
    If NEEDS_REVISION or ERROR:
        iteration += 1
        Invalidate V-related memory entries
        Invoke planner (replan mode) with verifier.md
        Execute fix tasks (Step 5 logic)
If iteration == 3 and not DONE: proceed with findings documented in verifier.md
```

Key convention: max 3 iterations; after exhaustion, proceed with findings (does not halt pipeline).

#### Concurrency Cap

Global Rule 8: Maximum 4 concurrent subagent invocations. Waves with >4 tasks are partitioned into sub-waves of ≤4, dispatched sequentially.

### Error Handling Patterns

#### Standardized Operating Rules (verbatim across all agents)

Every agent includes identical error handling rules as Operating Rules 1-4:

```
1. Context-efficient reading: Prefer semantic_search and grep_search. Limit read_file to ~200 lines/call.
2. Error handling:
   - Transient errors: Retry up to 2 times with brief delay. Do NOT retry deterministic failures.
   - Persistent errors: Include in output and continue. Do not retry.
   - Security issues: Flag immediately with severity: critical.
   - Missing context: Note the gap and proceed with available information.
   - Retry budget: 3 internal attempts × 2 orchestrator attempts = 6 max tool calls.
     Agents MUST NOT retry deterministic failures.
3. Output discipline: Produce only the deliverables specified in Outputs.
4. File boundaries: Only write to files listed in Outputs.
```

#### Orchestrator-Level Error Handling

- **Global Rule 4:** Retry failed subagent invocations (ERROR) once. Do NOT retry NEEDS_REVISION.
- **NEEDS_REVISION Routing Table:** Defines routing for each agent's NEEDS_REVISION (see orchestrator.agent.md lines 314-330).
- **Max Loops:** CT revision = 1 loop; V replan = 3 loops; R implementer fix = 1 loop.
- **Escalation:** If loops exhausted → proceed with findings/warnings or escalate to planner for full replan.

#### Special Error Overrides

- **R-Security:** ERROR is critical — blocks pipeline. Retry once, then aggregator ERROR.
- **R-Knowledge:** ERROR is non-blocking — aggregator proceeds without knowledge section.
- **V-Build:** Binary gate — DONE or ERROR only. No NEEDS_REVISION.

#### Non-Blocking Robustness Pattern

Several components are explicitly non-blocking:

- Memory system: if `memory.md` missing/corrupt → fall back to direct artifact reads.
- R-Knowledge: ERROR doesn't affect aggregated result.
- Approval gates: if runtime doesn't support interactive pause → proceed autonomously.

### Reusable Patterns for New Memory System

Based on existing conventions, the following patterns can be reused or adapted for agent-isolated memory:

#### 1. Memory-First Reading → Memory-First Writing

The existing "read `memory.md` first" pattern can be extended to "create isolated memory file first" for each agent. The protocol structure is already established.

#### 2. Memory Template Structure

The current 4-section template (Artifact Index, Recent Decisions, Lessons Learned, Recent Updates) can be adapted for isolated memory. Each agent's isolated memory would use a subset relevant to its role.

#### 3. No Memory Write → Own-Memory Write

The explicit "No Memory Write" workflow step in read-only agents can be replaced with an "Write Own Memory" step. The boilerplate pattern is:

```
(No memory write step — findings are communicated through `<artifact-path>`. The <Aggregator> will consolidate...)
```

This would become:

```
Write findings to `memory/<agent-name>.memory.md`. Include: key findings, decisions made, artifacts produced.
```

#### 4. Operating Rules Consistency

All 6 Operating Rules are verbatim across agents. This pattern should be preserved — new memory rules should be added as a consistent modification to existing Operating Rules (likely Rule 6 modification).

#### 5. Completion Contract Extension

The existing DONE/ERROR format can be extended to include memory file path:

```
DONE: <summary> — memory: <path-to-isolated-memory>
```

Or the orchestrator can infer the path by convention (e.g., `memory/<agent-name>.memory.md`).

#### 6. Input/Output Section Pattern

Currently all agents list `memory.md` in Inputs. The new pattern would:

- Remove shared `memory.md` from Inputs for parallel agents.
- Add isolated memory file to Outputs for all agents.
- Aggregator removal means each sub-agent must be self-sufficient in its output.

#### 7. Aggregator Merge Pattern → Orchestrator Merge

The aggregator workflow (read all sub-agent outputs → merge → deduplicate → categorize → determine completion contract) currently lives in ct-aggregator, v-aggregator, and r-aggregator. This logic moves to the orchestrator. Key reusable elements:

- Input validation rules (≥2 outputs required for CT/R, mandatory R-Security)
- Deduplication by matching file references and concern descriptions
- Severity-based sorting (Critical → High → Medium → Low for CT; Blocker → Major → Minor for R)
- Decision tables for determining completion contract
- Unresolved Tensions surfacing pattern

#### 8. Dispatch Pattern Conventions

Patterns A, B, C remain structurally sound without aggregators — the aggregation step simply moves to the orchestrator reading individual memories instead of dispatching a separate aggregator agent.

#### 9. Anti-Drift Anchor Pattern

Every agent's Anti-Drift Anchor includes memory access constraints (e.g., "You do NOT write to `memory.md`" or "You are the only cluster agent that writes to `memory.md`"). These must be updated to reflect the new isolated memory model.

#### 10. Severity Taxonomy

Two severity scales are used:

- **CT cluster:** Critical / High / Medium / Low
- **R cluster:** Blocker / Major / Minor
- **V cluster:** Binary (PASS/FAIL) + task-level status

These should remain consistent in the new system since they're intrinsic to the domain, not the memory architecture.

## File References

| File                                                                               | Relevance                                                                                                                                          |
| ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| [orchestrator.agent.md](NewAgentsAndPrompts/orchestrator.agent.md)                 | Primary coordination logic; Global Rules, Memory Lifecycle, NEEDS_REVISION Routing Table, Parallel Execution Summary, cluster dispatch definitions |
| [dispatch-patterns.md](NewAgentsAndPrompts/dispatch-patterns.md)                   | Pattern A and Pattern B full definitions                                                                                                           |
| [feature-workflow.prompt.md](NewAgentsAndPrompts/feature-workflow.prompt.md)       | User-facing workflow trigger; variables, rules summary                                                                                             |
| [researcher.agent.md](NewAgentsAndPrompts/researcher.agent.md)                     | Dual-mode agent (focused + synthesis); synthesis mode writes to memory                                                                             |
| [spec.agent.md](NewAgentsAndPrompts/spec.agent.md)                                 | Sequential agent; memory read-write; self-verification pattern                                                                                     |
| [designer.agent.md](NewAgentsAndPrompts/designer.agent.md)                         | Sequential agent; memory read-write; self-verification pattern                                                                                     |
| [planner.agent.md](NewAgentsAndPrompts/planner.agent.md)                           | Sequential agent; memory read-write; mode detection pattern (initial/replan/extension)                                                             |
| [ct-aggregator.agent.md](NewAgentsAndPrompts/ct-aggregator.agent.md)               | Aggregator pattern; merge-only; NEEDS_REVISION contract; writes to memory                                                                          |
| [v-aggregator.agent.md](NewAgentsAndPrompts/v-aggregator.agent.md)                 | Aggregator pattern; merge-only; decision table for completion; task-ID failure mapping                                                             |
| [r-aggregator.agent.md](NewAgentsAndPrompts/r-aggregator.agent.md)                 | Aggregator pattern; merge-only; R-Security override; R-Knowledge non-blocking                                                                      |
| [ct-security.agent.md](NewAgentsAndPrompts/ct-security.agent.md)                   | Representative CT sub-agent; read-only; DONE/ERROR only; "No Memory Write" step                                                                    |
| [ct-scalability.agent.md](NewAgentsAndPrompts/ct-scalability.agent.md)             | CT sub-agent; same pattern as ct-security                                                                                                          |
| [ct-maintainability.agent.md](NewAgentsAndPrompts/ct-maintainability.agent.md)     | CT sub-agent; same pattern as ct-security                                                                                                          |
| [ct-strategy.agent.md](NewAgentsAndPrompts/ct-strategy.agent.md)                   | CT sub-agent; same pattern as ct-security                                                                                                          |
| [v-build.agent.md](NewAgentsAndPrompts/v-build.agent.md)                           | V gate agent; binary DONE/ERROR; "No Memory Write" step                                                                                            |
| [v-tests.agent.md](NewAgentsAndPrompts/v-tests.agent.md)                           | V parallel sub-agent; DONE/NEEDS_REVISION/ERROR; "No Memory Write" step                                                                            |
| [v-tasks.agent.md](NewAgentsAndPrompts/v-tasks.agent.md)                           | V parallel sub-agent; DONE/NEEDS_REVISION/ERROR; "No Memory Write" step                                                                            |
| [v-feature.agent.md](NewAgentsAndPrompts/v-feature.agent.md)                       | V parallel sub-agent; DONE/NEEDS_REVISION/ERROR; "No Memory Write" step                                                                            |
| [r-quality.agent.md](NewAgentsAndPrompts/r-quality.agent.md)                       | R parallel sub-agent; DONE/NEEDS_REVISION/ERROR; "No Memory Write" step                                                                            |
| [r-security.agent.md](NewAgentsAndPrompts/r-security.agent.md)                     | R parallel sub-agent; pipeline blocker override; "No Memory Write" step                                                                            |
| [r-testing.agent.md](NewAgentsAndPrompts/r-testing.agent.md)                       | R parallel sub-agent; "No Memory Write" step                                                                                                       |
| [r-knowledge.agent.md](NewAgentsAndPrompts/r-knowledge.agent.md)                   | R parallel sub-agent; non-blocking ERROR; writes to decisions.md; "No Memory Write" step                                                           |
| [implementer.agent.md](NewAgentsAndPrompts/implementer.agent.md)                   | Implementation agent; TDD workflow; read-only memory; DONE/ERROR only                                                                              |
| [documentation-writer.agent.md](NewAgentsAndPrompts/documentation-writer.agent.md) | Documentation agent; read-only memory; DONE/ERROR only                                                                                             |
| [critical-thinker.agent.md](NewAgentsAndPrompts/critical-thinker.agent.md)         | Deprecated — superseded by CT cluster. Retained for migration reference.                                                                           |

## Assumptions & Limitations

1. **Assumption:** The "No Memory Write" boilerplate in read-only agents is a convention guide, not an enforced runtime constraint. Compliance depends on LLM behavior.
2. **Assumption:** The Operating Rules text is intended to be identical across agents — verified via grep matching. Minor wording variations exist (e.g., v-tasks uses "do NOT write" in Input annotation vs "operational memory").
3. **Assumption:** The `dispatch-patterns.md` file is a reference document read by the orchestrator, not directly executable. Pattern C is intentionally defined inline in the orchestrator.
4. **Limitation:** I did not read the full text of every agent file line-by-line; I used targeted reads and grep searches to verify pattern consistency. Some agent-specific workflow details in mid-file sections may have unique patterns not captured here.
5. **Assumption:** The v-tasks and v-feature agents use a slightly different Input annotation format (`read first — do NOT write`) compared to other agents (`read first — operational memory`), but both achieve the same result.

## Open Questions

1. **Memory file naming convention:** Will isolated memory files follow `memory/<agent-name>.memory.md` or a different path structure?
2. **Orchestrator merge granularity:** When the orchestrator merges isolated memories into shared memory, does it merge all at once (after a cluster completes) or incrementally (after each agent completes)?
3. **Shared memory reading during isolated model:** Will agents in a parallel wave read the shared `memory.md` snapshot OR receive no shared memory at all (only their own isolated memory)?
4. **Sequential agent memory isolation:** Will spec, designer, and planner also use isolated memory (with immediate merge), or continue writing directly to shared memory since they already run sequentially?
5. **Completion contract change:** Will agents include their memory path in the completion contract, or will the orchestrator infer it by naming convention?
6. **Aggregator logic migration:** How much of the aggregator merge logic (deduplication, severity sorting, decision tables, Unresolved Tensions) moves to the orchestrator vs. is simply removed?

## Research Metadata

- confidence_level: high
- coverage_estimate: All 24 agent files in NewAgentsAndPrompts/ were examined via targeted reads and grep searches. Full file reads were performed for orchestrator, dispatch-patterns, feature-workflow, researcher, spec, designer, planner, ct-aggregator, v-aggregator, r-aggregator, ct-security, ct-scalability, implementer, documentation-writer, v-build, v-tests, r-quality, and critical-thinker. Remaining files were verified via grep for key patterns (Completion Contract, Anti-Drift Anchor, memory references, No Memory Write).
- gaps: Did not read full workflow sections of ct-maintainability, ct-strategy, v-tasks, v-feature, r-security, r-testing, r-knowledge mid-file workflow details. These agents follow identical structural patterns to their sibling agents already fully read, so risk of missing unique patterns is low.
