# Research: Architecture

## Focus Area

architecture

## Summary

The project is a multi-agent orchestration system defined entirely via markdown agent prompt files (`.agent.md`) and workflow configuration files, using a shared `memory.md` with strict read-only/read-write access rules, three cluster dispatch patterns (A/B/C), and three aggregator agents (CT/V/R) that merge parallel sub-agent outputs into single combined artifacts.

## Findings

### 1. Repository Structure and Layout

The repository has two main directories containing agent definitions:

- **`NewAgentsAndPrompts/`** — The current/active agent prompt definitions (25 files). This is the target directory for the feature request.
- **`.github/agents/`** — An older generation of agent definitions (11 files: orchestrator, researcher, spec, designer, planner, implementer, documentation-writer, critical-thinker, reviewer, verifier). These appear to be the predecessor single-agent versions before the cluster decomposition (e.g., monolithic `critical-thinker` before the CT cluster, monolithic `reviewer` before the R cluster, monolithic `verifier` before the V cluster).
- **`.github/prompts/`** — Contains a `feature-workflow.prompt.md` (older version).
- **`docs/feature/`** — Feature documentation output directory. Each feature gets a subdirectory with a slug-based name containing all pipeline artifacts.

The root also contains `LICENSE`, `README.md`, and a `.git/` directory.

### 2. File Naming Conventions

All agent files follow a strict naming convention:

- **Agent files:** `<agent-name>.agent.md` — e.g., `orchestrator.agent.md`, `ct-security.agent.md`
- **Workflow prompts:** `<name>.prompt.md` — e.g., `feature-workflow.prompt.md`
- **Reference docs:** `<name>.md` — e.g., `dispatch-patterns.md`
- **Cluster sub-agents:** Prefixed with their cluster abbreviation: `ct-` (Critical Thinking), `v-` (Verification), `r-` (Review)
- **Aggregator agents:** `<cluster>-aggregator.agent.md` — e.g., `ct-aggregator.agent.md`

### 3. Agent File Format Conventions

Every `.agent.md` file follows a consistent structure:

1. **YAML front matter:** Contains `name` and `description` fields, wrapped in triple-dash delimiters inside a chatagent code fence.
2. **Title:** A markdown H1 with the agent's name and "Agent Workflow" or "Agent" suffix.
3. **Identity paragraph:** "You are the **\<Name\> Agent**." followed by a concise role description.
4. **Anti-pattern statements:** "You NEVER..." statements establishing guardrails.
5. **Experimental directive:** `Use detailed thinking to reason through complex decisions before acting. <!-- experimental: model-dependent -->`
6. **Inputs section:** Explicit list of files the agent reads, always starting with `memory.md (read first — operational memory)`.
7. **Outputs section:** Explicit list of files the agent writes.
8. **Operating Rules section:** Standardized 6-rule block present in every agent:
   - Rule 1: Context-efficient reading
   - Rule 2: Error handling (transient, persistent, security, missing context, retry budget)
   - Rule 3: Output discipline
   - Rule 4: File boundaries
   - Rule 5: Tool preferences (varies per agent)
   - Rule 6: Memory-first reading
9. **Workflow section:** Step-by-step numbered instructions.
10. **Completion Contract:** Exactly one line return — `DONE:`, `NEEDS_REVISION:`, or `ERROR:` (not all agents support all three).
11. **Anti-Drift Anchor:** A `**REMEMBER:**` block reinforcing the agent's role constraints.

### 4. Current Memory System Architecture

The memory system is centralized around a single shared `memory.md` file:

- **Location:** `docs/feature/<feature-slug>/memory.md`
- **Structure:** Four root sections — Artifact Index (table), Recent Decisions, Lessons Learned (never pruned), Recent Updates.
- **Lifecycle:** Initialized at Step 0; pruned at checkpoints (after Steps 1.2, 2, 4); invalidated on revision; validated after aggregator returns.
- **Memory failure is non-blocking** — if `memory.md` cannot be created or is corrupted, agents fall back to direct artifact reads.

#### Memory Access Control — Read-Only vs Read-Write

The orchestrator enforces strict memory write safety rules:

**Read-only (parallel-safe) agents:** researcher (focused mode), implementer, documentation-writer, all CT sub-agents (ct-security, ct-scalability, ct-maintainability, ct-strategy), all V sub-agents (v-build, v-tests, v-tasks, v-feature), all R sub-agents (r-quality, r-security, r-testing, r-knowledge).

**Read-write (sequential only) agents:** researcher (synthesis mode), spec, designer, planner, ct-aggregator, v-aggregator, r-aggregator.

This is documented in orchestrator Global Rule 12. The pattern is that **parallel sub-agents are read-only** and **sequential/aggregator agents are read-write**. This prevents concurrent write conflicts to the shared `memory.md`.

#### Memory-First Reading Protocol

Every agent has Operating Rule 6: "Read `memory.md` FIRST before accessing any artifact. Use the Artifact Index to navigate directly to relevant sections rather than reading full artifacts." This means `memory.md` serves as a navigation index to avoid reading large artifact files in full.

### 5. Cluster Dispatch Patterns

Three reusable dispatch patterns are defined:

#### Pattern A — Fully Parallel

Used by: CT cluster (Step 3b), R cluster (Step 7), Research focused (Step 1.1).

1. Dispatch N sub-agents in parallel (≤4)
2. Wait for all N to return
3. Retry errors once per Global Rule 4
4. If ≥2 sub-agent outputs available → invoke aggregator
5. If <2 outputs after retries → cluster ERROR
6. Check aggregator completion contract

#### Pattern B — Sequential Gate + Parallel

Used by: V cluster (Step 6).

1. Dispatch gate agent (V-Build) sequentially
2. If gate ERROR → retry once; still ERROR → skip parallel, forward to aggregator
3. If gate DONE → dispatch N-1 sub-agents in parallel
4. Wait for all parallel agents; retry errors once each
5. Invoke aggregator with all available outputs
6. Check aggregator completion contract

#### Pattern C — Replan Loop (wraps Pattern B)

Used by: V cluster (Step 6), max 3 iterations.

1. Run Pattern B (full V cluster)
2. If aggregator DONE → break
3. If NEEDS_REVISION or ERROR → increment iteration, invalidate V-related memory, invoke planner in replan mode, execute fix tasks, re-run full V cluster
4. After 3 iterations without DONE → proceed with findings in verifier.md

### 6. Aggregator Agent Architecture

Three aggregator agents exist, all following the same structural pattern:

#### CT-Aggregator (`ct-aggregator.agent.md`)

- **Reads:** 4 CT sub-agent outputs (`ct-review/ct-*.md`), `design.md`
- **Writes:** `design_critical_review.md`, `memory.md`
- **Role:** Merge-only — combines, deduplicates, categorizes, attributes findings; surfaces Unresolved Tensions
- **Completion contract:** DONE (≤Medium severity), NEEDS_REVISION (Critical/High findings), ERROR (<2 outputs)

#### V-Aggregator (`v-aggregator.agent.md`)

- **Reads:** 4 V sub-agent outputs (`verification/v-*.md`)
- **Writes:** `verifier.md`, `memory.md`
- **Role:** Merge-only — compiles unified verification report; critical output is task ID failure mapping for replanning
- **Completion contract:** DONE, NEEDS_REVISION, ERROR (per decision table based on sub-agent statuses)

#### R-Aggregator (`r-aggregator.agent.md`)

- **Reads:** 4 R sub-agent outputs (`review/r-*.md`), `knowledge-suggestions.md`
- **Writes:** `review.md`, `memory.md`
- **Role:** Merge-only — merges findings, enforces R-Security pipeline override, surfaces knowledge suggestions
- **Completion contract:** DONE, NEEDS_REVISION, ERROR (R-Security override: security blocks pipeline)
- **Special rules:** R-Security is mandatory/blocking; R-Knowledge is non-blocking

All three aggregators share key traits:

- They are the **only** agents in their cluster that write to `memory.md`
- They are **merge-only** — they never re-analyze source artifacts or generate new findings
- They produce a single combined output artifact from multiple sub-agent inputs
- They deduplicate findings, sort by severity, and surface contradictions as Unresolved Tensions

### 7. Pipeline Flow and Agent Composition (8-Step Pipeline)

The orchestrator runs a deterministic 8-step pipeline:

```
Step 0: Setup → initial-request.md + memory.md
Step 1.1: Researcher ×4 (parallel, Pattern A) → research/*.md
Step 1.2: Researcher (synthesis mode, sequential) → analysis.md
Step 2: Spec (sequential) → feature.md
Step 3: Designer (sequential) → design.md
Step 3b: CT ×4 (parallel, Pattern A) → ct-review/*.md → CT-Aggregator → design_critical_review.md
Step 4: Planner (sequential) → plan.md + tasks/*.md
Step 5: Implementation waves (≤4 parallel per sub-wave) → code + tests
Step 6: V-Build (gate) → V ×3 (parallel) → V-Aggregator → verifier.md (Pattern C: max 3 loops)
Step 7: R ×4 (parallel, Pattern A) → review/*.md → R-Aggregator → review.md
```

### 8. Documentation Structure (Output Artifacts)

All pipeline artifacts live under `docs/feature/<feature-slug>/`:

| Path                              | Created By                                                  | Purpose                                  |
| --------------------------------- | ----------------------------------------------------------- | ---------------------------------------- |
| `initial-request.md`              | Orchestrator                                                | User's original request                  |
| `memory.md`                       | Orchestrator (init), aggregators/sequential agents (append) | Operational memory                       |
| `research/*.md`                   | Researcher (focused ×4)                                     | Partial research outputs                 |
| `analysis.md`                     | Researcher (synthesis)                                      | Merged research analysis                 |
| `feature.md`                      | Spec                                                        | Feature specification                    |
| `design.md`                       | Designer                                                    | Technical design                         |
| `ct-review/ct-*.md`               | CT sub-agents ×4                                            | Individual CT reviews                    |
| `design_critical_review.md`       | CT-Aggregator                                               | Combined CT review                       |
| `plan.md`                         | Planner                                                     | Implementation plan                      |
| `tasks/*.md`                      | Planner                                                     | Individual task files                    |
| `verification/v-*.md`             | V sub-agents ×4                                             | Individual verification reports          |
| `verifier.md`                     | V-Aggregator                                                | Combined verification report             |
| `review/r-*.md`                   | R sub-agents ×4                                             | Individual review reports                |
| `review/knowledge-suggestions.md` | R-Knowledge                                                 | Knowledge improvement proposals          |
| `review.md`                       | R-Aggregator                                                | Combined review report                   |
| `decisions.md`                    | R-Knowledge                                                 | Cross-feature architectural decision log |

### 9. Three-State Completion Contract

All agents return one of three states:

- **`DONE:`** — Agent completed successfully. Orchestrator proceeds.
- **`NEEDS_REVISION:`** — Issues found requiring upstream correction. Only supported by aggregator agents.
- **`ERROR:`** — Agent failed. Orchestrator retries once (Global Rule 4).

Sub-agents (parallel workers) can only return DONE or ERROR. NEEDS_REVISION is routed through their aggregator. The orchestrator maintains a NEEDS_REVISION routing table mapping each returning agent to its upstream correction target (e.g., CT-Aggregator → Designer, V-Aggregator → Planner → Implementers, R-Aggregator → Implementers).

### 10. Concurrency and Parallelism Model

- **Maximum 4 concurrent sub-agents** per wave (Global Rule 8)
- Waves with >4 tasks are partitioned into sub-waves of ≤4
- Parallel agents are **read-only** with respect to `memory.md`
- Between implementation waves, the orchestrator extracts Lessons Learned and updates memory (sequential, safe)

### 11. Relationship Between `.github/agents/` and `NewAgentsAndPrompts/`

The `.github/agents/` directory contains the older monolithic agent definitions (11 files). The `NewAgentsAndPrompts/` directory is the new cluster-decomposed architecture (25 files). Key decompositions:

- `critical-thinker.agent.md` (old, deprecated with deprecation notice) → `ct-security`, `ct-scalability`, `ct-maintainability`, `ct-strategy`, `ct-aggregator` (new)
- `verifier.agent.md` (old) → `v-build`, `v-tests`, `v-tasks`, `v-feature`, `v-aggregator` (new)
- `reviewer.agent.md` (old) → `r-quality`, `r-security`, `r-testing`, `r-knowledge`, `r-aggregator` (new)

### 12. Input/Output Chain Across the Pipeline

Key artifact dependencies that define the data flow:

- **analysis.md** is consumed by: spec, designer, planner (3 downstream agents)
- **design_critical_review.md** is consumed by: planner (as planning constraints), designer (during revision cycle)
- **verifier.md** is consumed by: planner (replan mode, to identify failing tasks)
- **review.md** is the terminal artifact; consumed only by the orchestrator for pass/fail determination
- **memory.md** is consumed by ALL agents as their first read

### 13. Orchestrator File Scope

The orchestrator only creates/writes two files directly:

1. `initial-request.md` (Step 0)
2. `memory.md` (lifecycle management — init template, prune, validate, extract lessons)

All other file creation is delegated to sub-agents.

## File References

| File                                                                                                   | Rationale                                                                                                                     |
| ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| [NewAgentsAndPrompts/orchestrator.agent.md](NewAgentsAndPrompts/orchestrator.agent.md)                 | Central orchestrator — defines pipeline steps, memory lifecycle, dispatch patterns, concurrency rules, NEEDS_REVISION routing |
| [NewAgentsAndPrompts/dispatch-patterns.md](NewAgentsAndPrompts/dispatch-patterns.md)                   | Reference definitions for Pattern A and Pattern B                                                                             |
| [NewAgentsAndPrompts/feature-workflow.prompt.md](NewAgentsAndPrompts/feature-workflow.prompt.md)       | Entry point prompt that invokes the orchestrator                                                                              |
| [NewAgentsAndPrompts/researcher.agent.md](NewAgentsAndPrompts/researcher.agent.md)                     | Dual-mode agent — focused (parallel) and synthesis (sequential); synthesis mode produces analysis.md                          |
| [NewAgentsAndPrompts/ct-aggregator.agent.md](NewAgentsAndPrompts/ct-aggregator.agent.md)               | CT cluster merge agent — to be removed                                                                                        |
| [NewAgentsAndPrompts/v-aggregator.agent.md](NewAgentsAndPrompts/v-aggregator.agent.md)                 | V cluster merge agent — to be removed                                                                                         |
| [NewAgentsAndPrompts/r-aggregator.agent.md](NewAgentsAndPrompts/r-aggregator.agent.md)                 | R cluster merge agent — to be removed                                                                                         |
| [NewAgentsAndPrompts/spec.agent.md](NewAgentsAndPrompts/spec.agent.md)                                 | Reads analysis.md — input will change                                                                                         |
| [NewAgentsAndPrompts/designer.agent.md](NewAgentsAndPrompts/designer.agent.md)                         | Reads analysis.md and design_critical_review.md — inputs will change                                                          |
| [NewAgentsAndPrompts/planner.agent.md](NewAgentsAndPrompts/planner.agent.md)                           | Reads analysis.md, design_critical_review.md, verifier.md — inputs will change                                                |
| [NewAgentsAndPrompts/implementer.agent.md](NewAgentsAndPrompts/implementer.agent.md)                   | Currently no memory output — will need memory write added                                                                     |
| [NewAgentsAndPrompts/documentation-writer.agent.md](NewAgentsAndPrompts/documentation-writer.agent.md) | Currently no memory output — will need memory write added                                                                     |
| [NewAgentsAndPrompts/ct-security.agent.md](NewAgentsAndPrompts/ct-security.agent.md)                   | Representative CT sub-agent — currently no memory output                                                                      |
| [NewAgentsAndPrompts/v-build.agent.md](NewAgentsAndPrompts/v-build.agent.md)                           | Sequential gate for V cluster — currently no memory output                                                                    |
| [NewAgentsAndPrompts/r-quality.agent.md](NewAgentsAndPrompts/r-quality.agent.md)                       | Representative R sub-agent — currently no memory output                                                                       |
| [NewAgentsAndPrompts/r-knowledge.agent.md](NewAgentsAndPrompts/r-knowledge.agent.md)                   | Knowledge evolution agent — unique outputs (knowledge-suggestions.md, decisions.md)                                           |
| [NewAgentsAndPrompts/critical-thinker.agent.md](NewAgentsAndPrompts/critical-thinker.agent.md)         | Deprecated — retained for reference; no changes needed                                                                        |
| [.github/agents/](../../.github/agents/)                                                               | Older-generation agent definitions (pre-cluster); not in scope for changes                                                    |

## Assumptions & Limitations

1. **Assumed:** The `.github/agents/` directory contains the older generation of agents and is NOT in scope for the feature request — only `NewAgentsAndPrompts/` is in scope.
2. **Assumed:** The `critical-thinker.agent.md` deprecation notice indicates the cluster decomposition has already been completed and merged; the monolithic agents in `.github/agents/` are the pre-decomposition versions.
3. **Limitation:** This research did not deeply inspect the `.github/agents/` files to compare them with the `NewAgentsAndPrompts/` versions, as only the latter is in scope.
4. **Assumed:** There are no `.github/instructions/` files (none found by file search), which R-Knowledge references but treats as optional.
5. **Assumed:** The `feature-workflow.prompt.md` in `NewAgentsAndPrompts/` is the active/current version; the one in `.github/prompts/` is an older version.

## Open Questions

1. **`.github/agents/` deprecation status:** Are the `.github/agents/` files still used by any workflow, or are they fully superseded by `NewAgentsAndPrompts/`? This matters because the feature changes could introduce incompatibilities if both sets are used.
2. **Memory file naming convention for isolated memory:** The feature request specifies "each subagent creates its own isolated memory file" but does not define the naming convention (e.g., `memory-<agent-name>.md`? `<agent-name>.memory.md`?). This needs to be defined during design.
3. **Planner replan mode dependencies:** The planner in replan mode currently reads `verifier.md` (V-Aggregator output). If V-Aggregator is removed, how does the planner receive the unified failure mapping it needs for targeted replanning? The orchestrator could read individual V memories, but the task-ID-to-failure mapping logic currently lives in V-Aggregator.
4. **R-Security pipeline override enforcement:** Currently, the R-Aggregator enforces the rule that R-Security failures block the pipeline. If R-Aggregator is removed, this enforcement logic must move to the orchestrator.

## Research Metadata

- confidence_level: high
- coverage_estimate: All 25 files in NewAgentsAndPrompts/ were examined. All structural patterns, memory access rules, dispatch patterns, aggregator behavior, and pipeline flow were documented. The .github/agents/ directory was discovered and noted but not deeply inspected (out of scope).
- gaps: The .github/agents/ and .github/prompts/ older files were not deeply compared with NewAgentsAndPrompts/ versions. No runtime/execution environment details were examined since this is a prompt-definition-only project. The ct-scalability.agent.md and ct-maintainability.agent.md files were not individually read (inferred structurally consistent with ct-security.agent.md and ct-strategy.agent.md based on naming convention and the common agent format template). Similarly, v-tests.agent.md, v-tasks.agent.md, v-feature.agent.md, and r-testing.agent.md were not individually read (inferred consistent with their sibling files).
