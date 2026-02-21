# Research: Architecture

## Focus Area

**architecture** — Repository structure, project layout, architecture patterns, layers, .github instructions, coding/folder/contributor conventions

## Summary

Forge is a pure agent-definition repository with 21 active agents (plus 1 deprecated) coordinated by an 8-step deterministic pipeline orchestrator, using `.agent.md` files with YAML frontmatter, a dual-layer memory system, three cluster dispatch patterns, and strict file-boundary/output-discipline conventions throughout.

---

## Findings

### 1. Repository Structure

The repository is a **pure agent-definition project** — it contains no application source code, only agent definitions, prompts, documentation, and metadata.

```
OrchestratorAgents/
├── .github/
│   ├── agents/              # 21 .agent.md files + 1 reference doc
│   │   ├── orchestrator.agent.md
│   │   ├── researcher.agent.md
│   │   ├── spec.agent.md
│   │   ├── designer.agent.md
│   │   ├── planner.agent.md
│   │   ├── implementer.agent.md
│   │   ├── documentation-writer.agent.md
│   │   ├── critical-thinker.agent.md      # DEPRECATED
│   │   ├── ct-security.agent.md
│   │   ├── ct-scalability.agent.md
│   │   ├── ct-maintainability.agent.md
│   │   ├── ct-strategy.agent.md
│   │   ├── v-build.agent.md
│   │   ├── v-tests.agent.md
│   │   ├── v-tasks.agent.md
│   │   ├── v-feature.agent.md
│   │   ├── r-quality.agent.md
│   │   ├── r-security.agent.md
│   │   ├── r-testing.agent.md
│   │   ├── r-knowledge.agent.md
│   │   └── dispatch-patterns.md           # Reference doc (not an agent)
│   └── prompts/
│       └── feature-workflow.prompt.md     # Single entry-point prompt
├── docs/
│   └── feature/
│       └── <feature-slug>/               # Per-feature artifact tree
│           ├── initial-request.md
│           ├── memory.md
│           ├── memory/                    # Agent-isolated memory files
│           ├── research/
│           ├── ct-review/
│           ├── verification/
│           ├── review/
│           ├── tasks/
│           ├── feature.md
│           ├── design.md
│           ├── plan.md
│           └── decisions.md
├── README.md                              # 422 lines, comprehensive docs
└── LICENSE                                # MIT
```

**Key observation:** No `.github/instructions/` directory exists. The `r-knowledge` agent references it as optional input, but it has not been created. No `decisions.md` exists yet — it is created during pipeline execution by the `r-knowledge` agent.

### 2. Agent Definition Format (`.agent.md`)

All agents are defined as Markdown files with a consistent structure:

#### YAML Frontmatter

```yaml
---
name: <agent-name> # lowercase-kebab-case identifier
description: "<one-line role description>"
---
```

The file is wrapped in a triple-backtick `chatagent` code fence. Only two frontmatter fields are used: `name` and `description`.

#### Standard Section Structure (observed across all agents)

| Section                                  | Purpose                                                                                                               | Present In |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | ---------- |
| Title + Role Description                 | Identity and behavioral constraints                                                                                   | All agents |
| `<!-- experimental: model-dependent -->` | Hint for detailed thinking                                                                                            | All agents |
| Inputs                                   | Explicit list of input files with access mode (primary, selective, conditional)                                       | All agents |
| Outputs                                  | Explicit list of output files (strict boundary)                                                                       | All agents |
| Operating Rules (6 standardized)         | Context-efficient reading, error handling, output discipline, file boundaries, tool preferences, memory-first reading | All agents |
| Workflow / Review Process                | Agent-specific step-by-step procedure                                                                                 | All agents |
| Write Isolated Memory                    | Instructions to write `memory/<agent>.mem.md`                                                                         | All agents |
| Completion Contract                      | `DONE:` / `NEEDS_REVISION:` / `ERROR:`                                                                                | All agents |
| Anti-Drift Anchor                        | Final role-reinforcement block (exploits LLM recency bias)                                                            | All agents |

#### Standardized Operating Rules (shared by all agents)

All 21 agents include the same 6 operating rules, with agent-specific variations only in rule 5 (tool preferences):

1. **Context-efficient reading** — prefer semantic_search/grep_search, limit read_file to ~200 lines
2. **Error handling** — transient retries (2x), no retry on deterministic failures, retry budget (3 internal × 2 orchestrator = 6 max)
3. **Output discipline** — produce only specified deliverables
4. **File boundaries** — never write outside output scope
5. **Tool preferences** — varies by agent role (e.g., implementer uses `multi_replace_string_in_file`; reviewers use read-only tools only)
6. **Memory-first reading** — read `memory.md` FIRST, then upstream memories, use Artifact Index for navigation

### 3. Pipeline Architecture (8-Step Deterministic Pipeline)

The orchestrator drives a fixed-order pipeline:

```
Step 0: Setup → initial-request.md + memory.md + memory/
Step 1: Researcher ×4 (parallel, Pattern A) → merge memories
Step 2: Spec (sequential) → merge memory
Step 3: Designer (sequential) → merge memory
Step 3b: CT cluster ×4 (parallel, Pattern A) → orchestrator CT evaluation → merge
Step 4: Planner (sequential) → merge memory
Step 5: Implementation waves (≤4 per sub-wave, parallel) → merge between waves
Step 6: V cluster (Pattern B + C) → orchestrator V evaluation → merge
Step 7: R cluster ×4 (parallel, Pattern A) → orchestrator R evaluation → merge
```

#### Agent Roster (21 Active + 1 Deprecated)

| Category         | Agents                                                                               | Count              |
| ---------------- | ------------------------------------------------------------------------------------ | ------------------ |
| Core Pipeline    | orchestrator, researcher, spec, designer, planner, implementer, documentation-writer | 7                  |
| CT Cluster       | ct-security, ct-scalability, ct-maintainability, ct-strategy                         | 4                  |
| V Cluster        | v-build, v-tests, v-tasks, v-feature                                                 | 4                  |
| R Cluster        | r-quality, r-security, r-testing, r-knowledge                                        | 4                  |
| Deprecated       | critical-thinker                                                                     | 1                  |
| **Total active** |                                                                                      | **19 agent files** |

**Note:** The README references aggregator agents (`ct-aggregator`, `v-aggregator`, `r-aggregator`) that **do not exist** in the `.github/agents/` directory. The orchestrator agent definition explicitly states: "No aggregator agents exist in the pipeline" — the orchestrator evaluates cluster results directly via isolated memory files. This is a divergence between README (which says 22 agents) and the actual codebase (19 active agent files + 1 deprecated + 1 reference doc).

### 4. Cluster Dispatch Patterns

Three reusable dispatch patterns defined in `dispatch-patterns.md`:

| Pattern                            | Used By                                     | Behavior                                                                                                  |
| ---------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| **A — Fully Parallel**             | Research (Step 1), CT (Step 3b), R (Step 7) | Dispatch ≤4 concurrent → wait all → retry errors once → orchestrator reads memories → ≥2 outputs required |
| **B — Sequential Gate + Parallel** | V cluster (Step 6)                          | Dispatch gate (v-build) → if DONE, dispatch ≤3 parallel → orchestrator reads memories                     |
| **C — Replan Loop**                | V cycle (Steps 5-6)                         | Wraps Pattern B: run V → if NEEDS_REVISION → replan → re-implement → re-verify → max 3 iterations         |

All patterns follow the **memory-first architecture**: sub-agents write isolated memories, orchestrator reads them for routing decisions (never reads full artifacts for dispatch logic), then merges into shared `memory.md`.

### 5. Memory System (Dual-Layer)

#### Layer 1: Isolated Agent Memory

- Path: `docs/feature/<feature-slug>/memory/<agent-name>.mem.md`
- Written by: each individual agent
- Format: Status, Key Findings (≤5 bullets), Highest Severity, Decisions Made, Artifact Index
- Purpose: compact summary for orchestrator routing decisions

#### Layer 2: Shared Pipeline Memory

- Path: `docs/feature/<feature-slug>/memory.md`
- Written by: orchestrator only (sole writer)
- Format: Artifact Index table, Recent Decisions, Lessons Learned, Recent Updates
- Purpose: cross-agent coordination and context passing

#### Memory Lifecycle

| Action          | When                              | Description                                                                                         |
| --------------- | --------------------------------- | --------------------------------------------------------------------------------------------------- |
| Initialize      | Step 0                            | Create `memory.md` with empty template + `memory/` directory                                        |
| Merge           | After each agent/cluster          | Read isolated `memory/<agent>.mem.md`, merge Key Findings/Decisions/Artifact Index into `memory.md` |
| Prune           | After Steps 1.1m, 2m, 4m          | Remove Recent Decisions/Updates older than 2 phases; preserve Artifact Index + Lessons Learned      |
| Extract Lessons | Between implementation waves      | Read implementer memory files for issue/resolution entries                                          |
| Invalidate      | Before dispatching revision agent | Mark affected entries with `[INVALIDATED — <reason>]`                                               |
| Clean           | After revision completes          | Remove `[INVALIDATED]` entries not replaced                                                         |
| Validate        | After each agent completes        | Check isolated memory file exists (non-blocking)                                                    |

Memory failure is **non-blocking** — if `memory.md` cannot be created or becomes corrupted, agents fall back to direct artifact reads.

### 6. Completion Contract Pattern

All agents return exactly one of three completion states:

| State                                    | Meaning                                | Orchestrator Response                           |
| ---------------------------------------- | -------------------------------------- | ----------------------------------------------- |
| `DONE: <summary>`                        | Task completed successfully            | Proceed to next step                            |
| `NEEDS_REVISION: <what needs to change>` | Output needs improvement (not failure) | Route to appropriate agent per routing table    |
| `ERROR: <reason>`                        | Task failed                            | Retry once (Global Rule 4), then report failure |

**Key nuances:**

- The orchestrator itself never returns `NEEDS_REVISION:` — it handles revisions by routing to the appropriate agent.
- Not all agents use all three states: some agents (spec, designer, planner, CT sub-agents, implementer) return only `DONE:` or `ERROR:`. Only V sub-agents and R sub-agents use `NEEDS_REVISION:`.
- The researcher uses `DONE: <focus-area> — <one-line summary>` or `ERROR: <reason>`.

### 7. Orchestrator Coordination Model

The orchestrator (486 lines) is the largest and most complex agent. Key architectural properties:

- **Never writes code/docs/tests directly** — all work delegated via `runSubagent`
- **Sole writer to shared `memory.md`** — enforces no concurrent writes
- **Concurrency cap: max 4 concurrent subagent invocations** — waves >4 split into sub-waves
- **Direct cluster evaluation** — reads sub-agent isolated memory files and applies decision logic (CT/V/R Decision Flows) without aggregator agents
- **NEEDS_REVISION routing table** — defines which agent handles revisions from each source
- **Global Rules (12 total)** govern all orchestrator behavior: no direct work, explicit file paths, three-state contracts, auto-retry on ERROR, custom agents only, memory-first protocol, implementer TDD + V integration, concurrency cap, task agent routing, optional approval gates, display step info, memory write safety

#### Cluster Decision Logic (Orchestrator-Internal)

| Cluster | Decision Method                          | Key Rules                                                                                         |
| ------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------- |
| CT      | Read severity from 4 CT memories         | Any Critical/High → NEEDS_REVISION; All Medium/Low → DONE; <2 available → ERROR                   |
| V       | Read status from 4 V memories            | Decision table with V-Build as gate; Pattern C replan on failure                                  |
| R       | Read severity + status from 4 R memories | R-Security Blocker → pipeline ERROR; R-Knowledge non-blocking; <2 non-knowledge available → ERROR |

### 8. Prompt System

Single entry-point prompt: `.github/prompts/feature-workflow.prompt.md`

#### Prompt Format

```yaml
---
name: Feature Workflow
agent: orchestrator # Routes to the orchestrator agent
---
```

Body contains rules, key artifact descriptions, and template variables:

| Variable            | Required | Default | Purpose                                                |
| ------------------- | -------- | ------- | ------------------------------------------------------ |
| `{{USER_FEATURE}}`  | Yes      | —       | The user's feature request                             |
| `{{APPROVAL_MODE}}` | No       | `false` | Enables human approval gates after research + planning |

### 9. Documentation Structure Convention

All per-feature artifacts live under `docs/feature/<feature-slug>/`:

| Subdirectory  | Contents                                                                            | Populated By                                       |
| ------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------- |
| (root)        | initial-request.md, memory.md, feature.md, design.md, plan.md, decisions.md         | orchestrator, spec, designer, planner, r-knowledge |
| research/     | architecture.md, impact.md, dependencies.md, patterns.md                            | researcher ×4                                      |
| memory/       | `<agent-name>.mem.md` files                                                         | each individual agent                              |
| ct-review/    | ct-security.md, ct-scalability.md, ct-maintainability.md, ct-strategy.md            | CT sub-agents                                      |
| tasks/        | `*.md` task files                                                                   | planner                                            |
| verification/ | v-build.md, v-tests.md, v-tasks.md, v-feature.md                                    | V sub-agents                                       |
| review/       | r-quality.md, r-security.md, r-testing.md, r-knowledge.md, knowledge-suggestions.md | R sub-agents                                       |

### 10. Agent Naming Conventions

| Pattern                   | Examples                                                  | Usage                                    |
| ------------------------- | --------------------------------------------------------- | ---------------------------------------- |
| Lowercase kebab-case      | `researcher`, `spec`, `designer`                          | Core pipeline agents                     |
| Cluster prefix + focus    | `ct-security`, `v-build`, `r-quality`                     | Cluster sub-agents                       |
| Role-based task ID suffix | `implementer-<task-id>`, `documentation-writer-<task-id>` | Memory file naming for wave execution    |
| Focus area suffix         | `researcher-architecture`, `researcher-impact`            | Memory file naming for parallel research |

### 11. Tool Usage Patterns Across Agents

| Agent Type                     | Key Tools                                                                                          | Read-Only?                              |
| ------------------------------ | -------------------------------------------------------------------------------------------------- | --------------------------------------- |
| Orchestrator                   | `runSubagent` (sole tool for delegation)                                                           | Yes (except memory.md)                  |
| Researcher                     | `semantic_search`, `grep_search`, `file_search`, `read_file`                                       | Yes                                     |
| Spec, Designer                 | `semantic_search`, `grep_search`, `read_file`                                                      | Yes (write only outputs)                |
| Planner                        | `semantic_search`, `grep_search`, `read_file`                                                      | Yes (write only outputs)                |
| Implementer                    | `multi_replace_string_in_file`, `get_errors`, `list_code_usages`, `grep_search`, `run_in_terminal` | No (writes code + tests)                |
| Documentation Writer           | `semantic_search`, `grep_search`, `read_file`                                                      | No (writes docs only)                   |
| V-Build                        | `run_in_terminal`, `grep_search`                                                                   | Yes (write only outputs)                |
| V-Tests/V-Tasks/V-Feature      | `run_in_terminal`, `grep_search`, `read_file`                                                      | Yes (write only outputs)                |
| R-Quality/R-Security/R-Testing | `grep_search`, `read_file`                                                                         | Yes (write only outputs)                |
| R-Knowledge                    | `grep_search`, `read_file`                                                                         | Yes (write only outputs + decisions.md) |

**Critical observation for the self-improvement feature:** The orchestrator currently uses `runSubagent` for delegation but its tool access is not restricted by the agent definition itself — restriction is behavioral (instructions say "never write code"). The initial request asks to restrict it to only `[agent, agent/runSubagent, memory]` tools.

### 12. README vs. Codebase Divergences

| README Claims                                                              | Actual Codebase                                                         | Impact                               |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------ |
| 22 specialized agents                                                      | 19 active agent files (no aggregators exist)                            | README overstates count              |
| `ct-aggregator.agent.md`, `v-aggregator.agent.md`, `r-aggregator.agent.md` | Files do not exist                                                      | README references dead links         |
| `analysis.md` (synthesized research)                                       | Not referenced in orchestrator — 4 partial research files used directly | README shows stale v1 artifact       |
| `design_critical_review.md`                                                | Orchestrator uses individual `ct-review/ct-*.md` files                  | README shows stale artifact name     |
| `verifier.md`                                                              | Orchestrator uses individual `verification/v-*.md` files                | README shows stale artifact name     |
| `review.md`                                                                | Orchestrator uses individual `review/r-*.md` files                      | README shows stale artifact name     |
| `research/` has 3 files (architecture, impact, dependencies)               | Actually 4 files (+ patterns.md)                                        | README artifact structure incomplete |

---

## File References

| File/Folder                                               | Rationale                                                                |
| --------------------------------------------------------- | ------------------------------------------------------------------------ |
| `.github/agents/orchestrator.agent.md` (486 lines)        | Central pipeline coordination, memory management, cluster decision logic |
| `.github/agents/researcher.agent.md` (111 lines)          | Research agent pattern (parallel focus areas, isolated memory)           |
| `.github/agents/spec.agent.md` (106 lines)                | Spec agent pattern (memory-first reads, self-verification)               |
| `.github/agents/designer.agent.md` (120 lines)            | Design agent pattern (revision mode with CT memories)                    |
| `.github/agents/planner.agent.md` (260 lines)             | Planning agent (mode detection: initial/replan/extension, pre-mortem)    |
| `.github/agents/implementer.agent.md` (206 lines)         | TDD implementation agent (strict input/output boundaries)                |
| `.github/agents/documentation-writer.agent.md` (92 lines) | Documentation agent (read-only enforcement)                              |
| `.github/agents/ct-security.agent.md` (150 lines)         | CT cluster sub-agent pattern                                             |
| `.github/agents/v-build.agent.md` (151 lines)             | V cluster gate agent pattern                                             |
| `.github/agents/r-quality.agent.md` (211 lines)           | R cluster sub-agent pattern                                              |
| `.github/agents/r-knowledge.agent.md` (334 lines)         | Knowledge evolution agent (longest R agent, most complex outputs)        |
| `.github/agents/r-security.agent.md` (270 lines)          | Security review agent (pipeline-blocking severity)                       |
| `.github/agents/dispatch-patterns.md`                     | Reference doc for cluster dispatch patterns A, B, C                      |
| `.github/prompts/feature-workflow.prompt.md`              | Entry-point prompt with variables                                        |
| `docs/feature/self-improvement-system/`                   | Current feature's artifact directory                                     |
| `README.md` (422 lines)                                   | Project documentation (partially stale — see divergences)                |

---

## Assumptions & Limitations

- **Assumption:** The `.github/agents/` directory is the canonical location for all agent definitions. No other agent definition locations exist.
- **Assumption:** The `dispatch-patterns.md` file is the only non-agent reference document in the agents directory.
- **Limitation:** Could not verify runtime tool access restrictions for agents — the agent definitions contain behavioral instructions but actual tool access may be platform-controlled.
- **Limitation:** No sample pipeline execution artifacts were available to examine actual memory file formatting or artifact content structure.
- **Assumption:** The aggregator agents referenced in the README were removed as part of the v2 upgrade (the orchestrator now handles cluster aggregation directly), but the README was not fully updated.

---

## Open Questions

1. **Aggregator agent gap:** Should the README be updated to remove references to `ct-aggregator`, `v-aggregator`, `r-aggregator`? Or are these agents planned but not yet created?
2. **Tool access restriction mechanism:** How does the platform enforce which tools an agent can use? Is this controlled by YAML frontmatter, a separate configuration, or purely behavioral (prompt instructions)?
3. **`.github/instructions/` directory:** The `r-knowledge` agent references this as optional input. Is this a planned addition, or a reference to a GitHub Copilot platform feature that may/may not exist?
4. **`critical-thinker.agent.md` cleanup:** The deprecated agent is retained "for reference during migration validation." Is the migration complete? Can it be removed?
5. **`research/patterns.md`:** The fourth research focus area ("patterns") is used by the orchestrator and researcher but is missing from the README's artifact structure listing.

---

## Research Metadata

- **confidence_level:** high — all 21 agent files, the prompt, the dispatch-patterns reference, and the README were examined in detail.
- **coverage_estimate:** Full coverage of `.github/agents/`, `.github/prompts/`, `docs/feature/`, and `README.md`. Every agent file's frontmatter, inputs, outputs, and structural patterns were verified.
- **gaps:** No runtime execution logs or sample artifacts were available to verify actual memory file content structure or agent tool access enforcement. The README divergences suggest the documentation may not have been updated after the v2 architectural changes (removal of aggregators, addition of patterns.md research focus).
