# Architecture Research

## Focus Area

Architecture — Repository structure, project layout, architecture patterns, layers, coding/folder/contributor conventions, orchestration flow, agent structure, and communication patterns.

## Summary

Forge is a deterministic, fixed 8-step multi-agent orchestration pipeline with 10 specialized agents and 1 prompt file, using markdown artifacts as file-based inter-agent communication. Parallelism exists only at two points (research ×3, implementation waves ×4 max), while 6 pipeline stages run sequentially as single-agent bottlenecks.

---

## Findings

### 1. Current Agent Structure

Forge comprises **10 agents** and **1 prompt file**, all defined as `.agent.md` / `.prompt.md` files in `NewAgentsAndPrompts/`:

| Agent                         | File                                       | Role                                                                                                             | Inputs                                                                                      | Primary Output                                                        | Completion Contract                               |
| ----------------------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------- |
| **Orchestrator**              | `orchestrator.agent.md` (323 lines)        | Top-level coordinator; dispatches all agents via `runSubagent`; never writes code/docs directly                  | User feature request                                                                        | `initial-request.md` + coordination                                   | `ERROR:` only (handles NEEDS_REVISION internally) |
| **Researcher** (focused)      | `researcher.agent.md` (137 lines)          | Investigates codebase by focus area (architecture / impact / dependencies); read-only                            | `initial-request.md` + focus area                                                           | `research/<focus-area>.md`                                            | `DONE:` / `ERROR:`                                |
| **Researcher** (synthesis)    | Same file, different mode                  | Merges all research partials into cohesive analysis                                                              | All `research/*.md` files                                                                   | `analysis.md`                                                         | `DONE:` / `ERROR:`                                |
| **Spec**                      | `spec.agent.md` (105 lines)                | Produces formal feature specification with acceptance criteria, edge cases, test scenarios                       | `initial-request.md`, `analysis.md`                                                         | `feature.md`                                                          | `DONE:` / `ERROR:`                                |
| **Designer**                  | `designer.agent.md` (119 lines)            | Creates technical design — architecture, data models, APIs, security, failure analysis                           | `initial-request.md`, `analysis.md`, `feature.md`, (optionally `design_critical_review.md`) | `design.md`                                                           | `DONE:` / `ERROR:`                                |
| **Critical Thinker**          | `critical-thinker.agent.md` (139 lines)    | Devil's advocate adversarial design review; probes risks, assumptions, weaknesses                                | `initial-request.md`, `design.md`, `feature.md`                                             | `design_critical_review.md`                                           | `DONE:` / `NEEDS_REVISION:` / `ERROR:`            |
| **Planner**                   | `planner.agent.md` (244 lines)             | Decomposes work into dependency-aware tasks in execution waves; pre-mortem analysis; task size limits            | `initial-request.md`, `analysis.md`, `feature.md`, `design.md`, (replan: `verifier.md`)     | `plan.md` + `tasks/*.md`                                              | `DONE:` / `ERROR:`                                |
| **Implementer**               | `implementer.agent.md` (198 lines)         | Implements exactly one task using TDD (red-green-refactor); runs `get_errors` after every edit                   | Task file, `feature.md`, `design.md` (explicitly NOT `plan.md`)                             | Code files, test files, updated task file                             | `DONE:` / `ERROR:`                                |
| **Verifier**                  | `verifier.agent.md` (179 lines)            | Integration-level build + full test suite + per-task acceptance criteria verification; read-only                 | `initial-request.md`, `feature.md`, `design.md`, `plan.md`, `tasks/*.md`, codebase          | `verifier.md`                                                         | `DONE:` / `NEEDS_REVISION:` / `ERROR:`            |
| **Reviewer**                  | `reviewer.agent.md` (220 lines)            | Security-aware peer review with tiered depth (Full/Standard/Lightweight); OWASP scanning; maintains decision log | `initial-request.md`, git diff, codebase                                                    | `review.md`, `.github/instructions/*.instructions.md`, `decisions.md` | `DONE:` / `NEEDS_REVISION:` / `ERROR:`            |
| **Documentation Writer**      | `documentation-writer.agent.md` (95 lines) | Optional agent for API docs, Mermaid diagrams, README updates; invoked via task routing                          | Task file, `feature.md`, `design.md`, codebase                                              | Documentation files, updated task file                                | `DONE:` / `ERROR:`                                |
| **Feature Workflow** (prompt) | `feature-workflow.prompt.md` (30 lines)    | Entry-point prompt that invokes the orchestrator with `{{USER_FEATURE}}` and optional `{{APPROVAL_MODE}}`        | User feature text, approval flag                                                            | Delegates to orchestrator                                             | N/A                                               |

#### Agent Definition Pattern

All agents follow a consistent structural pattern (established in the v2 agent-improvements feature):

1. **YAML frontmatter** — `name` and `description` fields inside a `chatagent` code fence
2. **Role statement** — Single paragraph declaring what the agent does and what it NEVER does
3. **Inputs / Outputs** — Explicit file lists (strict boundaries)
4. **Operating Rules** — 5 standardized rules shared across all agents:
   - Context-efficient reading (prefer `semantic_search`/`grep_search`)
   - Error handling with retry budget (max 3 internal × 2 orchestrator = 6)
   - Output discipline
   - File boundaries
   - Tool preferences (agent-specific)
5. **Workflow** — Step-by-step procedure
6. **Output file contents specification** — Defines structure of the output artifact
7. **Completion Contract** — Three-state: `DONE:` / `NEEDS_REVISION:` / `ERROR:` (not all agents use all three)
8. **Anti-Drift Anchor** — Final instruction reinforcing role constraints (exploits LLM recency bias)

#### Agent Role Restrictions

| Agent                | Writes Code | Writes Docs                                         | Reads Codebase         | Modifies Source | Has NEEDS_REVISION |
| -------------------- | ----------- | --------------------------------------------------- | ---------------------- | --------------- | ------------------ |
| Orchestrator         | No          | Only `initial-request.md`                           | No                     | No              | No (handles it)    |
| Researcher           | No          | Research artifacts only                             | Yes (read-only)        | No              | No                 |
| Spec                 | No          | `feature.md` only                                   | Minimal                | No              | No                 |
| Designer             | No          | `design.md` only                                    | Yes (targeted)         | No              | No                 |
| Critical Thinker     | No          | `design_critical_review.md` only                    | Yes (to verify claims) | No              | Yes                |
| Planner              | No          | `plan.md` + `tasks/*.md`                            | Yes (targeted)         | No              | No                 |
| Implementer          | Yes         | Task file updates                                   | Yes                    | Yes             | No                 |
| Verifier             | No          | `verifier.md` only                                  | Yes (read-only)        | No              | Yes                |
| Reviewer             | No          | `review.md`, `decisions.md`, `.github/instructions` | Yes (read-only)        | No              | Yes                |
| Documentation Writer | No          | Doc files per task                                  | Yes (read-only)        | No              | No                 |

---

### 2. Orchestration Pipeline Flow

The orchestrator enforces a **fixed 8-step deterministic pipeline**. Steps are never skipped or reordered.

#### Pipeline Stages

```
Step 0: Setup
  └─ Orchestrator creates initial-request.md

Step 1: Research (PARALLEL)
  ├─ 1.1: Dispatch 3 researcher instances concurrently
  │   ├─ architecture  → research/architecture.md
  │   ├─ impact        → research/impact.md
  │   └─ dependencies  → research/dependencies.md
  ├─ 1.2: Synthesize (sequential) → analysis.md
  └─ 1.2a: (Conditional) Human approval gate if APPROVAL_MODE=true

Step 2: Specification (SEQUENTIAL)
  └─ spec → feature.md

Step 3: Design (SEQUENTIAL)
  └─ designer → design.md

Step 3b: Design Review (SEQUENTIAL, with feedback loop)
  └─ critical-thinker → design_critical_review.md
     └─ If NEEDS_REVISION: route back to designer (max 1 loop)

Step 4: Planning (SEQUENTIAL)
  └─ planner → plan.md + tasks/*.md
  └─ 4a: (Conditional) Human approval gate if APPROVAL_MODE=true

Step 5: Implementation (PARALLEL WAVES)
  ├─ Parse execution waves from plan.md
  ├─ For each wave (sequential between waves):
  │   ├─ Dispatch tasks within wave concurrently (max 4 per sub-wave)
  │   ├─ Agents: implementer or documentation-writer (per task's agent field)
  │   └─ Wait for all in sub-wave to complete before next sub-wave
  └─ On any ERROR: proceed to verification (don't continue to next wave)

Step 6: Verification (SEQUENTIAL, with retry loop)
  └─ verifier → verifier.md
     └─ If NEEDS_REVISION/ERROR: replan → re-implement → re-verify (max 3 loops)

Step 7: Final Review (SEQUENTIAL)
  └─ reviewer → review.md
     └─ If NEEDS_REVISION: route to affected implementer(s) (max 1 loop)
        └─ If still NEEDS_REVISION: escalate to planner for full replan
```

#### NEEDS_REVISION Routing Table (from orchestrator.agent.md, lines 253-261)

| Returning Agent  | Routes To               | Max Loops | Escalation           |
| ---------------- | ----------------------- | --------- | -------------------- |
| Critical Thinker | Designer                | 1         | Proceed with warning |
| Verifier         | Planner (replan loop)   | 3         | Halt and report      |
| Reviewer         | Affected Implementer(s) | 1         | Escalate to planner  |

#### Concurrency Model

- **Max 4 concurrent subagent invocations** (Global Rule 7, orchestrator.agent.md line 34)
- Waves with >4 tasks are partitioned into sequential sub-waves of ≤4
- Research phase: always exactly 3 concurrent agents
- Synthesis, spec, design, critical review, planning, verification, review: all single-agent sequential

---

### 3. Artifact Architecture

All artifacts reside under a deterministic directory structure:

```
docs/feature/<feature-slug>/
├── initial-request.md          # Grounding doc — user's original request
├── research/
│   ├── architecture.md         # Researcher focused output
│   ├── impact.md               # Researcher focused output
│   └── dependencies.md         # Researcher focused output
├── analysis.md                 # Synthesized research (single source of truth)
├── feature.md                  # Formal specification (acceptance criteria)
├── design.md                   # Technical design (architecture, APIs, security)
├── design_critical_review.md   # Adversarial review of design
├── plan.md                     # Dependency graph + execution waves + pre-mortem
├── tasks/
│   ├── 01-task-description.md  # Individual task files with agent routing
│   ├── 02-task-description.md
│   └── ...
├── verifier.md                 # Integration verification report
├── review.md                   # Security-aware peer review
└── decisions.md                # (Optional) Architectural decision log
```

#### Artifact Properties

| Artifact                    | Creator                | Consumers                                                  | Format                                                                    | Lifecycle                                           |
| --------------------------- | ---------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------- |
| `initial-request.md`        | Orchestrator           | ALL agents                                                 | Free-form markdown                                                        | Created once at Step 0; provided to every subagent  |
| `research/*.md`             | Researcher (focused)   | Researcher (synthesis)                                     | Structured markdown with findings, metadata, confidence                   | Created at Step 1.1; consumed at Step 1.2           |
| `analysis.md`               | Researcher (synthesis) | Spec, Designer, Planner                                    | Merged/deduplicated markdown                                              | Created at Step 1.2; read-only after                |
| `feature.md`                | Spec                   | Designer, Critical Thinker, Planner, Implementer, Verifier | Structured: requirements, acceptance criteria, edge cases, test scenarios | Created at Step 2; read-only after                  |
| `design.md`                 | Designer               | Critical Thinker, Planner, Implementer, Verifier           | Structured: architecture, data models, APIs, security, failure modes      | Created at Step 3; may be revised once at Step 3b   |
| `design_critical_review.md` | Critical Thinker       | Designer (if revision), Orchestrator                       | Risk assessment with categories, coverage gaps                            | Created at Step 3b                                  |
| `plan.md`                   | Planner                | Orchestrator, Verifier                                     | Execution waves, dependency graph, pre-mortem, task index                 | Created at Step 4; may be updated on replan         |
| `tasks/*.md`                | Planner                | Implementer/Doc-writer, Verifier                           | Goal, depends_on, agent, scope, acceptance criteria, test reqs, effort    | Created at Step 4                                   |
| `verifier.md`               | Verifier               | Planner (replan), Orchestrator                             | Build results, test results, per-task verification, actionable items      | Created at Step 6; may be updated on re-verify      |
| `review.md`                 | Reviewer               | Orchestrator, Implementer(s) (if fix)                      | Tiered review, issues, security findings, suggestions                     | Created at Step 7                                   |
| `decisions.md`              | Reviewer               | Researcher (if exists)                                     | Append-only decision log with date, context, rationale                    | Created by reviewer if decisions found; accumulates |

#### Key Artifact Design Principles

1. **Markdown as universal format** — All artifacts are human-readable markdown files. No YAML, JSON, or binary formats. (Explicitly rejected YAML state files per `optimization-from-gem-team.md`)
2. **Single source of truth** — Each artifact is the authoritative source for its domain; no duplication between artifacts.
3. **Explicit file paths** — Orchestrator passes exact file paths to each subagent (Global Rule 2, orchestrator.agent.md line 29).
4. **One creator per artifact** — Each artifact has exactly one agent responsible for creating/writing it.
5. **Read-only consumers** — Consumer agents only read artifacts; they never modify artifacts they don't own.
6. **Grounding document** — `initial-request.md` is passed to every subagent for consistent grounding.

---

### 4. Parallelism Model

#### Current Parallel Execution Points

| Stage                   | Parallelism                       | Max Concurrent       | Notes                                                            |
| ----------------------- | --------------------------------- | -------------------- | ---------------------------------------------------------------- |
| Research (Step 1.1)     | 3 focused researchers in parallel | 3 (always exactly 3) | Architecture, impact, dependencies — fixed focus areas           |
| Implementation (Step 5) | Wave-based parallel execution     | 4 per sub-wave       | Tasks within a wave run concurrently; waves execute sequentially |

#### Current Sequential Bottlenecks (Single-Agent Stages)

| Stage                         | Sequential Agent            | Duration Impact                                  | Why Sequential                             |
| ----------------------------- | --------------------------- | ------------------------------------------------ | ------------------------------------------ |
| Research Synthesis (Step 1.2) | Researcher (synthesis mode) | Blocks spec                                      | Must merge all 3 research outputs into one |
| Specification (Step 2)        | Spec                        | Blocks design                                    | Single requirements document               |
| Design (Step 3)               | Designer                    | Blocks critical review                           | Single design document                     |
| Critical Review (Step 3b)     | Critical Thinker            | Blocks planning (+ may loop back to design once) | Single adversarial review                  |
| Planning (Step 4)             | Planner                     | Blocks implementation                            | Must produce coherent dependency graph     |
| Verification (Step 6)         | Verifier                    | Blocks review (+ may loop 3× to planner)         | Must verify full integration               |
| Review (Step 7)               | Reviewer                    | Final gate (+ may loop to implementers once)     | Must review full diff                      |

#### Visual Pipeline Parallelism Map

```
PARALLEL:   [Research×3] ─────────────────────────────────────────────────┐
SEQUENTIAL: ─────────────── Synth → Spec → Design → CritReview → Plan ──┤
PARALLEL:   ──────────────────────────────────────── [Wave1×4] [Wave2×4] ┤
SEQUENTIAL: ──────────────────────────────────────────────── Verify ──────┤
SEQUENTIAL: ──────────────────────────────────────────────── Review ──────┘
```

The pipeline is **heavily sequential** in the middle (Spec → Design → Critical Review → Planning) — a chain of 4 single-agent stages that must execute one after another with no parallelism.

---

### 5. Communication Patterns

#### Primary Communication: File-Based Artifact Passing

All inter-agent communication happens through **markdown files on disk**. There is no in-memory state, no shared database, no message queue, and no cross-agent memory system.

**Data flow pattern:**

1. Orchestrator invokes subagent via `runSubagent` with explicit file paths in the prompt
2. Subagent reads its input artifacts from disk
3. Subagent writes its output artifact to disk
4. Orchestrator reads the completion contract line (`DONE:` / `NEEDS_REVISION:` / `ERROR:`)
5. Orchestrator decides next step based on completion contract

**Key characteristics:**

- **No shared memory** — Agents cannot share runtime context or intermediate state. Each agent starts fresh with only the file paths provided.
- **No artifact caching** — Every agent reads artifacts from disk on every invocation. There is no indexing, summarization, or navigation metadata.
- **Complete file reads** — Agents must read entire artifacts even if they only need a small section. No section-level addressing beyond what `read_file` with line ranges provides.
- **No cross-feature learning** — The only mechanism for cross-feature knowledge transfer is the optional `decisions.md` file (reviewer writes, researcher reads if exists).

#### Orchestrator-to-Agent Communication

The orchestrator communicates with agents via:

1. **Prompt injection** — The orchestrator constructs a prompt containing the agent's task description, file paths, and context
2. **`runSubagent` invocation** — Each agent is dispatched as a separate subagent call
3. **Completion contract** — Agents return exactly one line: `DONE:`, `NEEDS_REVISION:`, or `ERROR:`

#### Agent-to-Agent Communication (Indirect Only)

Agents never communicate directly with each other. All communication is mediated through:

1. **Artifacts on disk** — Agent A writes `design.md`; Agent B reads `design.md` in a later step
2. **Orchestrator routing** — The orchestrator reads completion contracts and routes work to the next agent

#### Information Flow Diagram

```
initial-request.md ──→ ALL AGENTS (grounding)
                  │
research/*.md ────→ analysis.md ──→ feature.md ──→ design.md ──→ plan.md + tasks/*.md
                                       │              │              │
                                       ├──→ design_critical_review.md│
                                       │                             │
                                       ├─────────────────────→ implementer (code)
                                       │              │              │
                                       └──────────────┴──────→ verifier.md ──→ review.md
```

---

### 6. Architectural Bottlenecks

Based on the architecture analysis, the following bottlenecks are identified:

#### 6.1 Sequential Middle Pipeline (Spec → Design → Critical Review → Planning)

**Location:** Steps 2, 3, 3b, 4 of the orchestrator pipeline
**Nature:** Four sequential single-agent stages with no parallelism
**Impact:** These stages execute one after another, each blocking the next. No work can proceed until the entire chain completes.
**Current state:** There is no structural dependency preventing some of these from running in partial overlap (e.g., parts of planning could potentially begin before critical review completes), but the current architecture requires full completion of each step.

#### 6.2 Single Critical Thinker

**Location:** Step 3b, `critical-thinker.agent.md`
**Nature:** One agent reviews the entire design across all risk categories (security, scalability, maintainability, backwards compatibility, edge cases, performance, strategic risks, scope risks, complexity risks, integration risks)
**Impact:** Reviewing all categories sequentially is time-intensive. The initial request explicitly calls for splitting into 4 parallel sub-agents.
**Current state:** Single invocation with broad scope (critical-thinker.agent.md lines 63-94 define risk categories plus "beyond the categories" open-ended review).

#### 6.3 Single Verifier

**Location:** Step 6, `verifier.agent.md`
**Nature:** One agent performs build, test execution, per-task verification, and feature-level verification
**Impact:** Verifier performs 4 distinct verification activities sequentially: (1) detect build system, (2) build, (3) run test suite, (4) per-task acceptance criteria check, (5) feature-level verification. Several of these could be parallelized.
**Current state:** Single invocation (verifier.agent.md lines 66-115 define 6 sequential workflow steps).

#### 6.4 Single Reviewer

**Location:** Step 7, `reviewer.agent.md`
**Nature:** One agent performs security scanning, code quality review, convention checking, decision logging, and instruction updates
**Impact:** The reviewer handles multiple distinct concerns: (1) maintainability/readability, (2) security review (secrets + OWASP), (3) test quality, (4) architectural alignment, (5) convention compliance, (6) decisions.md updates, (7) `.github/instructions` updates.
**Current state:** Single invocation with tiered depth (reviewer.agent.md lines 37-43 define tier selection).

#### 6.5 Repeated Full Artifact Reads

**Location:** All agents
**Nature:** Every agent reads full artifacts from disk on every invocation. There is no artifact index, summary, or navigation metadata.
**Impact:** As features grow complex and artifacts become large, agents spend significant context window on re-reading large files. The initial request identifies this as problem #1: "Agents repeatedly read large artifacts unnecessarily."
**Current state:** No caching, indexing, or partial-read optimization exists. Agents are instructed to use `semantic_search` and `grep_search` for "context-efficient reading" (Operating Rule 1 across all agents), but this applies to codebase reading, not artifact reading.

#### 6.6 No Persistent Operational Memory

**Location:** Across the entire pipeline
**Nature:** No shared context system exists between agent invocations. Cross-feature knowledge is limited to the optional `decisions.md` file.
**Impact:** Agents cannot learn from prior runs. Each feature implementation starts from scratch. The initial request identifies this as problem #2: "There is no persistent operational memory shared across agents."
**Current state:** The only cross-feature mechanism is `decisions.md` (reviewer writes, researcher reads if exists — reviewer.agent.md lines 73-91).

#### 6.7 Research Phase Limited to 3 Focus Areas

**Location:** Step 1.1, orchestrator.agent.md lines 93-105
**Nature:** Research is always split into exactly 3 fixed focus areas: architecture, impact, dependencies.
**Impact:** The initial request calls for 4 parallel research agents. The current architecture hardcodes 3.
**Current state:** The orchestrator's table at lines 93-103 defines exactly 3 rows. The researcher's focus area table (researcher.agent.md lines 37-42) also defines exactly 3 areas.

#### 6.8 Verification-Review Sequential Dependency

**Location:** Steps 6-7
**Nature:** Review cannot begin until verification completes, even though some review activities (code style, naming, convention compliance) don't depend on verification results.
**Impact:** The review stage waits for the full verification loop (potentially 3 retry iterations) before starting any work.

---

### 7. Key Observations

#### 7.1 Architecture Is Purely Declarative (No Runtime Code)

Forge has **no runtime code** — no executable orchestrator, no agent framework, no message bus. The entire system consists of markdown instruction files that are interpreted by the GitHub Copilot agent runtime. This means:

- Architecture changes = editing markdown files
- There is no deployment, compilation, or infrastructure
- The "runtime" is the VS Code Copilot agent mode environment
- All "execution" happens through `runSubagent` calls in the Copilot runtime

#### 7.2 Agent Isolation Is Strongly Enforced

Each agent has explicit constraints on what files it can read and write. The implementer is explicitly forbidden from reading `plan.md` (implementer.agent.md line 19). The verifier and reviewer are explicitly read-only (verifier.agent.md lines 131-137, reviewer.agent.md lines 35-41). This strict isolation is a deliberate architectural decision (README lines 205-207).

#### 7.3 Two-Layer Verification Is a Core Design Decision

The separation of unit-level TDD (implementer) from integration-level verification (verifier) is a conscious design choice, explicitly documented and justified in the README (lines 228-232) and in `comparison-forge-vs-gem-team.md`. Upgrading to a verification cluster must preserve this two-layer model.

#### 7.4 Documentation as First-Class Communication Channel

Markdown artifacts serve dual purpose: (1) communication channel between agents, (2) full audit trail for humans. This is explicitly called out as a design decision (README lines 213-214). Any memory system must not undermine artifacts as the single source of truth.

#### 7.5 Consistent Agent Template

All agents follow an identical structural template (see Section 1 above). This consistency is a v2 achievement. Any new sub-agents (critical thinking cluster, verification cluster, review cluster) should follow this same template to maintain consistency.

#### 7.6 Error Recovery Is Multi-Layered

Error recovery operates at three levels:

1. **Tool-level:** Each agent retries transient tool failures up to 2 times (Operating Rule 2)
2. **Agent-level:** Orchestrator retries failed agents once (Global Rule 4)
3. **Pipeline-level:** Verification failure triggers replan→re-implement→re-verify loop (max 3 iterations)

These compose: worst case is 3 internal × 2 orchestrator = 6 tool calls per operation. This retry budget is explicitly documented across all agents.

#### 7.7 The Orchestrator Is the Single Point of Control

All workflow logic resides in the orchestrator. There are no distributed decisions — the orchestrator reads completion contracts and makes all routing decisions. This is explicit: "You coordinate a deterministic, end-to-end workflow" (orchestrator.agent.md line 11). Introducing parallel sub-agent clusters will require aggregation logic, which must either live in the orchestrator or in dedicated aggregator sub-agents.

#### 7.8 No `.github` Directory Present

Despite references to `.github/instructions/*.instructions.md` in the reviewer agent (reviewer.agent.md line 30) and references to `.github/agents/` in the prior analysis (analysis.md line 17), no `.github` directory exists in the current workspace. The agent files are in `NewAgentsAndPrompts/` directory. This discrepancy should be noted — the reviewer's instruction to update `.github/instructions` files may need to be updated to reflect the actual directory structure.

---

## File References

| File                                                         | Relevance                                                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| `NewAgentsAndPrompts/orchestrator.agent.md`                  | Central pipeline definition, concurrency rules, NEEDS_REVISION routing, parallel execution model |
| `NewAgentsAndPrompts/researcher.agent.md`                    | Research parallelism model, focused/synthesis modes, retrieval strategy                          |
| `NewAgentsAndPrompts/spec.agent.md`                          | Sequential stage, self-verification workflow                                                     |
| `NewAgentsAndPrompts/designer.agent.md`                      | Sequential stage, revision loop with critical thinker                                            |
| `NewAgentsAndPrompts/critical-thinker.agent.md`              | Sequential bottleneck, risk category structure (candidate for splitting)                         |
| `NewAgentsAndPrompts/planner.agent.md`                       | Task decomposition, execution waves, dependency graph, pre-mortem, mode detection                |
| `NewAgentsAndPrompts/implementer.agent.md`                   | TDD workflow, strict isolation, unit-level verification                                          |
| `NewAgentsAndPrompts/verifier.agent.md`                      | Integration verification, sequential workflow (candidate for splitting)                          |
| `NewAgentsAndPrompts/reviewer.agent.md`                      | Security review, tiered depth, decision log, instruction updates (candidate for splitting)       |
| `NewAgentsAndPrompts/documentation-writer.agent.md`          | Optional agent, task-routed                                                                      |
| `NewAgentsAndPrompts/feature-workflow.prompt.md`             | Entry-point prompt, APPROVAL_MODE variable                                                       |
| `README.md`                                                  | Architecture rationale, design decisions, v2 changelog, workflow diagram                         |
| `docs/comparison-forge-vs-gem-team.md`                       | Forge vs Gem Team architectural comparison                                                       |
| `docs/optimization-from-gem-team.md`                         | Optimization roadmap, P0-P3 priorities, what NOT to adopt                                        |
| `docs/feature/agent-improvements/analysis.md`                | Prior analysis from v2 improvements                                                              |
| `docs/feature/forge-architecture-upgrade/initial-request.md` | Grounding document for the current upgrade                                                       |

---

## Assumptions & Limitations

1. **Assumption:** The `NewAgentsAndPrompts/` directory contains the current/latest agent definitions. The README references agents at root level, but the workspace structure shows them under `NewAgentsAndPrompts/`.
2. **Assumption:** The Copilot agent runtime supports `runSubagent` for concurrent dispatch (up to 4). This is referenced but described as "platform-dependent" in the orchestrator.
3. **Limitation:** No `.github` directory exists in the workspace, so the reviewer's `.github/instructions` output path cannot be verified.
4. **Limitation:** No prior `decisions.md` file exists in any feature directory, so cross-feature learning has not been exercised yet.
5. **Assumption:** The `APPROVAL_MODE` feature is experimental and may not be supported by the current runtime (orchestrator.agent.md line 36 notes this explicitly).

---

## Open Questions

1. Should the research phase expand from 3 to 4 focus areas (per initial request), and if so, what is the 4th focus area?
2. When splitting agents into parallel sub-agents (critical thinker ×4, verifier ×4, reviewer ×4), should the aggregation logic live in the orchestrator or in dedicated aggregator agents?
3. How should the memory system interact with the existing artifact-based communication — should memory be a separate file, a section in existing artifacts, or a structured companion file?
4. The reviewer references `.github/instructions/` as an output path, but this directory doesn't exist. Should this be updated to match the actual project structure?
5. Is the `runSubagent` mechanism capable of supporting the increased parallelism (up to 4 sub-agents per cluster, potentially 16+ concurrent invocations during implementation)?

---

## Research Metadata

- **confidence_level:** high — All 11 agent/prompt files were read in full; README, comparison doc, optimization doc, and prior analysis were examined.
- **coverage_estimate:** Complete coverage of all agent definitions, orchestration flow, artifact structure, and communication patterns. All files in `NewAgentsAndPrompts/` were read. Supporting documentation was examined for architectural rationale and design decisions.
- **gaps:** No runtime behavior was observed (purely declarative analysis). The actual behavior of `runSubagent` in the Copilot runtime was not verified — findings are based on documented intent. No `.github` directory exists to verify reviewer output paths.
