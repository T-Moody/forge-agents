# Research: Architecture

## Focus Area

Architecture — Repository structure, project layout, architecture patterns, agent taxonomy, orchestration mechanics, memory systems, communication patterns, and state management across both the Forge Orchestrator and Anvil Agent systems.

## Summary

Two fundamentally different multi-agent architectures coexist: Forge is a 23-agent deterministic pipeline with file-based memory, cluster dispatch patterns, and three-state completion contracts; Anvil is a single evidence-first agent with SQL-tracked verification, adversarial multi-model review, and risk-based task sizing.

---

## Findings

### 1. Repository Structure

#### 1.1 Top-Level Layout

```
OrchestratorAgents/
├── .github/
│   ├── agents/          # 23 Forge agent definitions (*.agent.md) + 2 reference docs
│   └── prompts/         # 1 prompt file (feature-workflow.prompt.md)
├── Anvil/
│   └── anvil.agent.md   # Single monolithic Anvil agent (419 lines)
├── docs/
│   └── feature/         # Per-feature documentation structure (Forge pipeline output)
│       ├── orchestrator-tool-restriction/
│       ├── self-improvement-system/
│       └── next-gen-multi-agent-system/
├── NewAgents/
│   └── .github/         # Empty — target output directory for next-gen system
├── LICENSE
└── README.md
```

#### 1.2 Forge Agent Directory (`.github/agents/`)

Contains 23 files total:

| File                                | Type             | Role                                                  |
| ----------------------------------- | ---------------- | ----------------------------------------------------- |
| `orchestrator.agent.md` (544 lines) | Agent definition | Central coordinator — dispatches all other agents     |
| `researcher.agent.md`               | Agent definition | Focused codebase research (4 parallel instances)      |
| `spec.agent.md`                     | Agent definition | Feature specification from research findings          |
| `designer.agent.md`                 | Agent definition | Technical design document creation                    |
| `planner.agent.md`                  | Agent definition | Dependency-aware task decomposition                   |
| `implementer.agent.md`              | Agent definition | TDD-based task implementation                         |
| `documentation-writer.agent.md`     | Agent definition | Documentation generation                              |
| `ct-security.agent.md`              | Agent definition | CT cluster: security & backwards compatibility        |
| `ct-scalability.agent.md`           | Agent definition | CT cluster: scalability & performance                 |
| `ct-maintainability.agent.md`       | Agent definition | CT cluster: maintainability & complexity              |
| `ct-strategy.agent.md`              | Agent definition | CT cluster: strategic risks & scope                   |
| `v-build.agent.md`                  | Agent definition | V cluster: build gate (sequential)                    |
| `v-tests.agent.md`                  | Agent definition | V cluster: test suite execution                       |
| `v-tasks.agent.md`                  | Agent definition | V cluster: per-task acceptance verification           |
| `v-feature.agent.md`                | Agent definition | V cluster: feature-level acceptance verification      |
| `r-quality.agent.md`                | Agent definition | R cluster: code quality review                        |
| `r-security.agent.md`               | Agent definition | R cluster: security scanning (pipeline blocker)       |
| `r-testing.agent.md`                | Agent definition | R cluster: test quality review                        |
| `r-knowledge.agent.md`              | Agent definition | R cluster: knowledge evolution (non-blocking)         |
| `post-mortem.agent.md`              | Agent definition | Quantitative post-mortem analysis (non-blocking)      |
| `critical-thinker.agent.md`         | Deprecated       | Superseded by CT cluster (retained for reference)     |
| `dispatch-patterns.md`              | Reference doc    | Full definitions of Pattern A/B/C                     |
| `evaluation-schema.md`              | Reference doc    | Artifact evaluation YAML schema (shared by 14 agents) |

#### 1.3 Prompt File (`.github/prompts/`)

Single file: `feature-workflow.prompt.md` — entry point prompt that binds to the orchestrator agent. Accepts `{{USER_FEATURE}}` and `{{APPROVAL_MODE}}` variables. Provides execution rules, key artifacts table, and variable schema.

#### 1.4 Anvil Directory (`Anvil/`)

Single file: `anvil.agent.md` (419 lines) — a self-contained monolithic agent definition containing all logic, workflow, verification, review, and commit procedures.

### 2. Forge Orchestrator Architecture

#### 2.1 Pipeline Overview — 8-Step Process

The Forge pipeline runs a deterministic 8-step sequence. No step may be skipped.

```
Step 0:  Setup → memory.md + initial-request.md
Step 1:  Research ×4 (parallel, Pattern A) → merge memories
Step 2:  Specification (sequential) → merge memory
Step 3:  Design (sequential) → merge memory
Step 3b: CT Cluster ×4 (parallel, Pattern A) → orchestrator CT evaluation → merge
Step 4:  Planning (sequential) → merge memory
Step 5:  Implementation waves (≤4 per sub-wave, parallel) → merge between waves
Step 6:  Verification — V Cluster (Pattern B+C) → merge
Step 7:  Final Review — R Cluster ×4 (parallel, Pattern A) → merge
Step 8:  Post-Mortem (sequential, non-blocking) → merge
```

#### 2.2 Agent Taxonomy

**Sequential (single-instance) agents:**

- Spec → produces `feature.md`
- Designer → produces `design.md`
- Planner → produces `plan.md` + `tasks/*.md`
- Implementer → one instance per task (parallel within waves)
- Documentation Writer → one instance per task (parallel within waves)
- Post-Mortem → produces run-log + post-mortem report

**Parallel cluster agents (dispatched as groups):**

| Cluster                | Agents                                                       | Dispatch Pattern                                                          | Pipeline Step |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------- | ------------- |
| Research               | researcher ×4 (architecture, impact, dependencies, patterns) | Pattern A (Fully Parallel)                                                | Step 1.1      |
| CT (Critical Thinking) | ct-security, ct-scalability, ct-maintainability, ct-strategy | Pattern A (Fully Parallel)                                                | Step 3b       |
| V (Verification)       | v-build (gate), v-tests, v-tasks, v-feature                  | Pattern B (Sequential Gate + Parallel) wrapped in Pattern C (Replan Loop) | Step 6        |
| R (Review)             | r-quality, r-security, r-testing, r-knowledge                | Pattern A (Fully Parallel)                                                | Step 7        |

#### 2.3 Orchestrator Role and Constraints

The orchestrator (544 lines) is the central coordinator with strict constraints:

- **Tool restrictions:** Allowed: `agent`, `agent/runSubagent`, `memory`, `read_file`, `list_dir`. Explicitly forbidden: `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`, `grep_search`, `semantic_search`, `file_search`, `get_errors`.
- **Never writes files directly** — all file creation/modification delegated to subagents via `runSubagent`.
- **Sole writer to shared `memory.md`** — delegates merge operations to subagents.
- **Evaluates cluster results directly** by reading sub-agent isolated memory files (no aggregator agents).
- **Manages memory lifecycle:** init, merge, prune, invalidate, validate.
- **Accumulates telemetry** in its context window during Steps 1–7, dispatches to PostMortem at Step 8.
- **Concurrency cap:** Maximum 4 concurrent subagent invocations per wave.

#### 2.4 Dispatch Patterns

Three reusable dispatch patterns defined in `dispatch-patterns.md`:

**Pattern A — Fully Parallel:**

1. Dispatch ≤4 sub-agents concurrently
2. Wait for all to return
3. Retry errors once per Global Rule 4
4. If ≥2 outputs available → orchestrator reads isolated memories for decision
5. If <2 outputs → cluster ERROR

Used by: Research (Step 1.1), CT cluster (Step 3b), R cluster (Step 7).

**Pattern B — Sequential Gate + Parallel:**

1. Dispatch gate agent (V-Build) sequentially
2. If gate ERROR → retry once, if persistent → skip parallel, cluster ERROR
3. If gate DONE → dispatch remaining ≤3 in parallel
4. Orchestrator reads isolated memories for decision

Used by: V cluster (Step 6).

**Pattern C — Replan Loop (wraps Pattern B):**

1. Run V cluster (Pattern B)
2. If DONE → continue
3. If NEEDS_REVISION → invoke planner (replan mode) with V memory paths → re-run implementation and verification
4. Maximum 3 iterations
5. If still not DONE after 3 → proceed with findings documented in V artifacts

Used by: Verification-Replan cycle (Steps 5–6 iteration).

#### 2.5 Three-State Completion Contracts

Every subagent returns exactly one of:

- `DONE:` — success, proceed
- `NEEDS_REVISION:` — addressable issues found; orchestrator routes to appropriate agent
- `ERROR:` — unrecoverable failure; retry once, then escalate

Not all agents use all three states:

- **DONE/ERROR only:** Researcher, Spec, Designer, Planner, Implementer, Documentation Writer, CT sub-agents, V-Build, Post-Mortem
- **DONE/NEEDS_REVISION/ERROR:** V-Tests, V-Tasks, V-Feature, R-Quality, R-Security, R-Testing
- **DONE/ERROR only (non-blocking):** R-Knowledge, Post-Mortem

The orchestrator NEVER returns `NEEDS_REVISION` itself — it handles all revision routing internally.

#### 2.6 NEEDS_REVISION Routing Table

| Source                | Routes To                                            | Max Loops | Escalation                                                     |
| --------------------- | ---------------------------------------------------- | --------- | -------------------------------------------------------------- |
| CT cluster evaluation | Designer → full CT re-run                            | 1         | Proceed with warning; forward findings as planning constraints |
| V cluster evaluation  | Planner (REPLAN mode) → Implementers → full V re-run | 3         | Proceed with findings in V artifacts                           |
| R cluster evaluation  | Implementer(s) for affected tasks                    | 1         | Escalate to planner for full replan                            |
| R-Security ERROR      | Retry once → Planner if persistent                   | 1         | Halt pipeline                                                  |

#### 2.7 Cluster Decision Logic

The orchestrator evaluates cluster results directly (no aggregator agents) by reading sub-agent isolated memory files:

**CT Cluster Decision Flow:**

1. Read 4 CT memory files → count available (≥2 required)
2. Extract `Highest Severity` + `Medium Severity` from each
3. Any Critical/High/Medium → NEEDS_REVISION (route to designer with individual CT artifact paths)
4. All Low → DONE

**V Cluster Decision Flow:**

1. Read 4 V memory files → extract `Status` from each
2. Apply decision table (V-Build is the critical gate; any NEEDS_REVISION triggers replan)
3. 2+ missing/ERROR → cluster ERROR

**R Cluster Decision Flow (strict priority order):**

1. R-Security override FIRST: missing → ERROR; ERROR → ERROR; Blocker → ERROR
2. Read remaining memories → count available non-knowledge (≥2 required)
3. R-Knowledge is NON-BLOCKING
4. Any non-knowledge Major severity → NEEDS_REVISION
5. Otherwise → DONE

### 3. Forge Memory Architecture

#### 3.1 Dual-Layer Memory System

```
docs/feature/<feature-slug>/
├── memory.md                          # Shared memory (orchestrator sole writer)
└── memory/                            # Agent-isolated memory directory
    ├── researcher-architecture.mem.md # One per agent invocation
    ├── researcher-impact.mem.md
    ├── spec.mem.md
    ├── designer.mem.md
    ├── planner.mem.md
    ├── implementer-<task-id>.mem.md   # One per implementation task
    ├── ct-security.mem.md
    ├── ct-scalability.mem.md
    ├── v-build.mem.md
    ├── v-tests.mem.md
    ├── r-quality.mem.md
    ├── r-security.mem.md
    ├── r-knowledge.mem.md
    ├── post-mortem.mem.md
    └── ...
```

**Shared `memory.md`** — Orchestrator sole writer. Contains:

- Artifact Index (persistent — never pruned)
- Recent Decisions (pruned at checkpoints)
- Lessons Learned (persistent — never pruned)
- Recent Updates (pruned at checkpoints)

**Isolated `memory/<agent>.mem.md`** — Written by each agent. Standard format:

- Status: DONE/ERROR with one-line summary
- Key Findings: ≤5 bullet points
- Highest Severity: varies by agent taxonomy
- Decisions Made: ≤2 sentences each
- Artifact Index: file paths with §Section pointers and relevance notes

#### 3.2 Memory Lifecycle

| Action          | When                         | Description                                                                                            |
| --------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------ |
| Initialize      | Step 0                       | Create empty `memory.md` template                                                                      |
| Merge           | After each agent/cluster     | Read isolated memory → merge Key Findings, Decisions, Artifact Index into `memory.md`                  |
| Prune           | After Steps 1.1m, 2m, 4m     | Remove stale Recent Decisions/Updates older than 2 phases; preserve Artifact Index and Lessons Learned |
| Extract Lessons | Between implementation waves | Extract issue/resolution entries from implementer memories                                             |
| Invalidate      | Before revision dispatch     | Mark affected entries with `[INVALIDATED — <reason>]`                                                  |
| Clean           | After revision completes     | Remove remaining `[INVALIDATED]` entries                                                               |
| Validate        | After each agent             | Check isolated memory file exists; log warning if not (non-blocking)                                   |

#### 3.3 Memory-First Reading Pattern

All downstream agents follow this protocol:

1. Read `memory.md` FIRST for orientation
2. Read upstream agent isolated memory files for key findings + artifact indexes
3. Use Artifact Index §Section pointers for targeted reads of full artifacts
4. Avoid reading full artifacts unless necessary

This reduces context window usage by having agents read compact summaries before selectively reading detailed sections.

### 4. Anvil Agent Architecture

#### 4.1 Single-Agent Monolithic Design

Anvil is a single 419-line agent definition that handles the entire development workflow. Unlike Forge's 23-agent distributed model, Anvil is one agent with a multi-phase loop. It uses subagents only for adversarial code review (not for orchestration).

#### 4.2 The Anvil Loop (12 phases)

```
0.  Boost         — Rewrite user prompt into precise specification (silent)
0b. Git Hygiene   — Check dirty state, branch, worktree (silent)
1.  Understand    — Parse goal, criteria, assumptions, open questions (silent)
1b. Recall        — Query session_store SQL for relevant history (Medium/Large)
2.  Survey        — Search codebase for reuse opportunities (silent)
3.  Plan          — Plan changes with risk levels (silent for Medium, shown for Large)
3b. Baseline      — Capture pre-change state in verification ledger (Medium/Large)
4.  Implement     — Write code following existing patterns
5.  Verify        — The Forge: IDE diagnostics, verification cascade, adversarial review
6.  Learn         — Store confirmed facts via store_memory
7.  Present       — Show results with evidence bundle
8.  Commit        — Auto-commit with rollback instructions
```

#### 4.3 Task Sizing and Risk Classification

**Three task sizes with escalating verification:**

| Size   | Criteria                                                                    | Verification                                                                       | Reviewers                                                     |
| ------ | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Small  | Typo, rename, config, one-liner                                             | Quick verify (5a + 5b only) — no ledger, no adversarial review, no evidence bundle | None (exception: 🔴 files escalate to Large with 3 reviewers) |
| Medium | Bug fix, feature addition, refactor                                         | Full Anvil Loop with SQL ledger                                                    | 1 adversarial reviewer                                        |
| Large  | New feature, multi-file architecture, auth/crypto/payments, OR any 🔴 files | Full Anvil Loop + `ask_user` at Plan step                                          | 3 parallel multi-model reviewers                              |

**Risk classification per file:**

- 🟢 Additive: new tests, docs, config, comments
- 🟡 Business logic: modifying logic, function signatures, DB queries, UI state
- 🔴 Critical: auth/crypto/payments, data deletion, schema migrations, concurrency, public API

Any 🔴 file automatically escalates to Large regardless of task size.

#### 4.4 Verification Architecture — The Forge

**SQL Verification Ledger (`anvil_checks`):**

```sql
CREATE TABLE IF NOT EXISTS anvil_checks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id TEXT NOT NULL,
    phase TEXT NOT NULL CHECK(phase IN ('baseline', 'after', 'review')),
    check_name TEXT NOT NULL,
    tool TEXT NOT NULL,
    command TEXT,
    exit_code INTEGER,
    output_snippet TEXT,
    passed INTEGER NOT NULL CHECK(passed IN (0, 1)),
    ts DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Key rule: Every verification step is an INSERT. The Evidence Bundle is a SELECT. If the INSERT didn't happen, the verification didn't happen.

**Verification Cascade (three tiers, defense in depth):**

| Tier                                        | Checks                                     | When                                            |
| ------------------------------------------- | ------------------------------------------ | ----------------------------------------------- |
| Tier 1 (Always)                             | IDE diagnostics, syntax/parse check        | Every task                                      |
| Tier 2 (If tooling exists)                  | Build/compile, type checker, linter, tests | Dynamically detected                            |
| Tier 3 (When Tiers 1-2 lack runtime signal) | Import/load test, smoke execution script   | Fallback when no runtime verification available |

Minimum signals: 2 for Medium, 3 for Large. Zero verification is never acceptable.

**Phases recorded in SQL:**

- `baseline` — pre-change state (Step 3b)
- `after` — post-change verification (Step 5)
- `review` — adversarial review verdicts (Step 5c)

#### 4.5 Adversarial Review System

**Medium (no 🔴 files):** 1 reviewer subagent

- Model: `gpt-5.3-codex`
- Reviews `git diff --staged`
- Looks for: bugs, security vulnerabilities, logic errors, race conditions, edge cases, missing error handling, architectural violations

**Large OR 🔴 files:** 3 parallel reviewers (same prompt, different models)

- `gpt-5.3-codex`
- `gemini-3-pro-preview`
- `claude-opus-4.6`
- Each verdict INSERTed with `phase = 'review'`

Gates: Must have ≥1 review row (Medium) or ≥3 review rows (Large) before presenting.
Max 2 adversarial rounds — if issues persist after 2 rounds, remaining findings are presented as known issues with `Confidence: Low`.

#### 4.6 Pushback System

Anvil evaluates requests before executing. Two categories:

**Implementation concerns:** Tech debt, duplication, unnecessary complexity, simpler approaches, scope too large/vague.

**Requirements concerns:** Conflicts with existing behavior, solving symptom not root cause, edge cases producing dangerous behavior, implicit assumptions.

Protocol: Show `⚠️ Anvil pushback` callout → `ask_user` with choices ("Proceed as requested" / "Do it your way instead" / "Let me rethink this") → Do NOT implement until user responds.

#### 4.7 Session/Recall System

Anvil uses SQL (`session_store` database) for cross-session memory:

- Queries `sessions` + `session_files` tables for past file modifications
- Queries `search_index` (FTS) for past problems (regressions, bugs, failures, reverts)
- Uses recall results to inform planning and avoid repeated mistakes

#### 4.8 Evidence Bundle

Generated from SQL query at the end of verification. Contains:

- Baseline vs. after comparison (regression detection)
- All verification results with commands and exit codes
- Adversarial review verdicts per model
- Confidence level (High/Medium/Low with precise definitions)
- Blast radius analysis
- Rollback command

#### 4.9 Confidence Levels (rigorous definitions)

- **High:** All tiers passed, no regressions, reviewers found zero issues or only fixed issues. Would merge without reading diff.
- **Medium:** Most checks passed but gaps exist (no test coverage for changed path, reviewer concern addressed but uncertain, unverified blast radius). Human should skim.
- **Low:** A check failed that couldn't be fixed, unverifiable assumptions, or reviewer issue that can't be disproved. MUST state what would raise it.

### 5. Documentation Structure (Forge Pipeline Output)

Each feature produces a standardized directory:

```
docs/feature/<feature-slug>/
├── initial-request.md          # User's original feature request
├── memory.md                   # Shared operational memory
├── memory/                     # Agent-isolated memory files
│   └── *.mem.md
├── research/                   # Step 1 outputs (4 parallel research artifacts)
│   ├── architecture.md
│   ├── impact.md
│   ├── dependencies.md
│   └── patterns.md
├── feature.md                  # Step 2 output: feature specification
├── design.md                   # Step 3 output: technical design
├── ct-review/                  # Step 3b outputs: CT cluster
│   ├── ct-security.md
│   ├── ct-scalability.md
│   ├── ct-maintainability.md
│   └── ct-strategy.md
├── plan.md                     # Step 4 output: implementation plan
├── tasks/                      # Step 4 output: individual task files
│   └── NN-description.md
├── verification/               # Step 6 outputs: V cluster
│   ├── v-build.md
│   ├── v-tests.md
│   ├── v-tasks.md
│   └── v-feature.md
├── review/                     # Step 7 outputs: R cluster
│   ├── r-quality.md
│   ├── r-security.md
│   ├── r-testing.md
│   ├── r-knowledge.md
│   └── knowledge-suggestions.md
├── decisions.md                # Architectural decision log (R-Knowledge writes, append-only)
├── artifact-evaluations/       # Structured evaluations from consuming agents
│   └── <agent-name>.md
├── agent-metrics/              # Pipeline telemetry
│   └── <run-date>-run-log.md
└── post-mortems/               # Post-mortem analysis
    └── <run-date>-post-mortem.md
```

### 6. Architectural Patterns

#### 6.1 Communication Mechanism

**Forge:** File-based communication via known deterministic paths.

- Agents produce artifacts at well-defined paths (e.g., `research/architecture.md`)
- Agents produce isolated memory files at `memory/<agent>.mem.md`
- The orchestrator reads known paths only — no discovery tools needed
- Downstream agents use Artifact Index §Section pointers for targeted reads
- No direct agent-to-agent communication — everything flows through the file system via orchestrator coordination

**Anvil:** Mixed communication:

- SQL database (`anvil_checks`) for verification ledger — structured, queryable, tamper-evident
- SQL database (`session_store`) for cross-session recall
- `git diff --staged` for passing code changes to review subagents
- `ask_user` for interactive user communication
- `store_memory` for cross-session fact persistence
- File system for code changes

#### 6.2 State Management

**Forge state mechanisms:**

- File system: All pipeline artifacts (Markdown files at known paths)
- `memory.md`: Shared operational memory (orchestrator sole writer)
- `memory/<agent>.mem.md`: Agent-isolated memory (one per agent invocation)
- Orchestrator context window: Telemetry accumulation during pipeline (Steps 1–7)
- VS Code `memory` tool: Cross-session knowledge store for codebase facts

**Anvil state mechanisms:**

- SQL (`anvil_checks`): Verification ledger with phases (baseline, after, review)
- SQL (`session_store`): Session history with FTS search_index
- Git: Working tree state, branch management, staged changes
- VS Code `store_memory`: Cross-session fact persistence
- File system: Code changes only

#### 6.3 Gating Mechanisms

**Forge gating:**

- Three-state completion contract: DONE / NEEDS_REVISION / ERROR
- Cluster decision logic (CT, V, R) with severity-based thresholds
- Pattern C replan loop (max 3 iterations)
- R-Security pipeline blocker (Blocker severity halts entire pipeline)
- Approval gates (optional, after research and after planning)
- Concurrency cap (max 4 concurrent subagents)

**Anvil gating:**

- SQL-enforced gates: "If you have zero rows in anvil_checks with phase='baseline', you skipped this step. Go back."
- Minimum verification signals: 2 for Medium, 3 for Large
- Review INSERT gate: Review-phase rows must exist before presenting
- Evidence Bundle gate: After-phase rows must meet minimum count
- Pushback system: halts execution until user responds
- Revert-on-failure: after 2 fix attempts, revert changes via `git checkout HEAD -- {files}`

#### 6.4 Parallelism Organization

**Forge parallelism:**

- Concurrency cap: ≤4 concurrent subagent invocations
- Waves with more than 4 tasks split into sub-waves of ≤4
- Pattern A: All agents in parallel (CT, R, Research)
- Pattern B: Gate → remainder in parallel (V cluster)
- Implementation waves: independent tasks in parallel, dependent tasks in sequence
- Orchestrator manages all parallel dispatch and synchronization

**Anvil parallelism:**

- Limited: only adversarial review uses parallelism (3 review subagents for Large/🔴)
- Everything else is sequential within the single agent loop
- Uses `explore` subagents for code survey but not for orchestration

#### 6.5 Error Handling Patterns

**Forge:**

- Retry budget: 3 internal attempts × 2 orchestrator-level attempts = 6 max
- Transient errors: retry up to 2 times; deterministic errors: do not retry
- Cluster minimum: ≥2 outputs for decision-making (except V-Build which is binary)
- R-Knowledge and Post-Mortem are non-blocking
- Memory failure is non-blocking (agents fall back to direct artifact reads)

**Anvil:**

- Max 2 fix attempts per check failure → revert changes if unfixable
- Max 2 adversarial review rounds → present remaining issues with Confidence: Low
- Tier 3 infeasibility is acceptable if documented via INSERT
- Never present broken code — revert rather than show failures
- "When stuck after 2 attempts, explain what failed and ask for help"

#### 6.6 Artifact Evaluation System (Forge-specific)

14 evaluating agents produce structured YAML evaluations of upstream artifacts they consumed:

```yaml
artifact_evaluation:
  evaluator: "<agent-name>"
  source_artifact: "<relative path>"
  usefulness_score: <1-10>
  clarity_score: <1-10>
  useful_elements: [...]
  missing_information: [...]
  information_not_used: [...]
  inaccuracies: [...]
  impact_on_work: [...]
```

Schema centralized in `evaluation-schema.md`. Evaluations are secondary (non-blocking). PostMortem agent aggregates these into quantitative metrics.

#### 6.7 Agent Definition File Structure (Forge)

Each agent definition follows a consistent structure:

```markdown
---
name: <agent-name>
description: <one-line description>
---

# <Agent Name> Workflow

[Role statement — what the agent does and critical constraints]
[Anti-drift anchors — "You NEVER..." statements]

## Inputs

[Explicit list with read-order: memory-first, then selective]

## Outputs

[Explicit list of files the agent may write]

## Operating Rules

[6 standard rules: context-efficient reading, error handling, output discipline,
file boundaries, tool preferences, memory-first reading]

## Workflow

[Numbered procedural steps]

## Completion Contract

[DONE: / NEEDS_REVISION: / ERROR: format]

## Anti-Drift Anchor

[Restates critical constraints to prevent role confusion]
```

Key structural conventions:

- Every agent has an explicit Inputs and Outputs section
- Every agent has Operating Rules that follow a standardized 6-rule template
- Every agent has a Completion Contract defining valid return states
- Every agent has an Anti-Drift Anchor as the final section
- Memory-first reading is enforced in every agent
- File boundary restrictions are explicit per agent

---

## File References

| File/Directory                                                                               | Rationale                                                                                                                   |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| [.github/agents/orchestrator.agent.md](.github/agents/orchestrator.agent.md)                 | Central pipeline coordinator (544 lines) — defines all 8 steps, cluster decision logic, memory lifecycle, dispatch patterns |
| [.github/agents/dispatch-patterns.md](.github/agents/dispatch-patterns.md)                   | Reference doc for Pattern A/B/C dispatch logic                                                                              |
| [.github/agents/evaluation-schema.md](.github/agents/evaluation-schema.md)                   | Shared YAML schema for artifact evaluations (used by 14 agents)                                                             |
| [.github/prompts/feature-workflow.prompt.md](.github/prompts/feature-workflow.prompt.md)     | Entry point prompt binding orchestrator to feature workflow                                                                 |
| [.github/agents/researcher.agent.md](.github/agents/researcher.agent.md)                     | Research agent (4 parallel instances per focus area)                                                                        |
| [.github/agents/spec.agent.md](.github/agents/spec.agent.md)                                 | Specification agent                                                                                                         |
| [.github/agents/designer.agent.md](.github/agents/designer.agent.md)                         | Design agent                                                                                                                |
| [.github/agents/planner.agent.md](.github/agents/planner.agent.md)                           | Planning agent with task decomposition, wave organization, pre-mortem                                                       |
| [.github/agents/implementer.agent.md](.github/agents/implementer.agent.md)                   | TDD implementation agent                                                                                                    |
| [.github/agents/documentation-writer.agent.md](.github/agents/documentation-writer.agent.md) | Documentation generation agent                                                                                              |
| [.github/agents/ct-security.agent.md](.github/agents/ct-security.agent.md)                   | CT cluster: security focus                                                                                                  |
| [.github/agents/ct-scalability.agent.md](.github/agents/ct-scalability.agent.md)             | CT cluster: scalability focus                                                                                               |
| [.github/agents/ct-maintainability.agent.md](.github/agents/ct-maintainability.agent.md)     | CT cluster: maintainability focus                                                                                           |
| [.github/agents/ct-strategy.agent.md](.github/agents/ct-strategy.agent.md)                   | CT cluster: strategy focus                                                                                                  |
| [.github/agents/v-build.agent.md](.github/agents/v-build.agent.md)                           | V cluster: build gate                                                                                                       |
| [.github/agents/v-tests.agent.md](.github/agents/v-tests.agent.md)                           | V cluster: test execution                                                                                                   |
| [.github/agents/v-tasks.agent.md](.github/agents/v-tasks.agent.md)                           | V cluster: per-task acceptance verification                                                                                 |
| [.github/agents/v-feature.agent.md](.github/agents/v-feature.agent.md)                       | V cluster: feature-level acceptance verification                                                                            |
| [.github/agents/r-quality.agent.md](.github/agents/r-quality.agent.md)                       | R cluster: code quality review                                                                                              |
| [.github/agents/r-security.agent.md](.github/agents/r-security.agent.md)                     | R cluster: security review (pipeline blocker)                                                                               |
| [.github/agents/r-testing.agent.md](.github/agents/r-testing.agent.md)                       | R cluster: test quality review                                                                                              |
| [.github/agents/r-knowledge.agent.md](.github/agents/r-knowledge.agent.md)                   | R cluster: knowledge evolution (non-blocking)                                                                               |
| [.github/agents/post-mortem.agent.md](.github/agents/post-mortem.agent.md)                   | Post-mortem quantitative analysis (non-blocking)                                                                            |
| [.github/agents/critical-thinker.agent.md](.github/agents/critical-thinker.agent.md)         | Deprecated — superseded by CT cluster                                                                                       |
| [Anvil/anvil.agent.md](Anvil/anvil.agent.md)                                                 | Monolithic Anvil agent (419 lines) — full loop, verification, adversarial review                                            |
| [docs/feature/orchestrator-tool-restriction/](docs/feature/orchestrator-tool-restriction/)   | Example of complete Forge pipeline output structure                                                                         |
| [docs/feature/self-improvement-system/](docs/feature/self-improvement-system/)               | Another example of Forge pipeline output structure                                                                          |

---

## Assumptions & Limitations

1. **Agent runtime assumed:** Analysis assumes VS Code Copilot Agent mode with `runSubagent` capability. The actual invocation mechanism and context passing model are platform-dependent.
2. **Anvil SQL databases:** The `anvil_checks` and `session_store` databases are assumed to be SQLite managed by VS Code extensions or the agent runtime. The exact storage mechanism is not specified in the agent file.
3. **Model availability:** Anvil's multi-model adversarial review references specific models (`gpt-5.3-codex`, `gemini-3-pro-preview`, `claude-opus-4.6`). Availability may vary by environment.
4. **Git dependency:** Both systems assume git is available and the workspace is a git repository. Anvil has deeper git integration (branch management, worktree detection, auto-commit).
5. **Deprecated agent:** `critical-thinker.agent.md` is retained for reference but superseded by the CT cluster. It represents an earlier non-parallel design.
6. **No runtime validation:** Forge agent definitions are Markdown — there is no schema validation or type-checking of agent inputs/outputs at runtime. Compliance depends on prompt adherence.

---

## Open Questions

1. **Context window limits:** How do the 23 Forge agents handle context window exhaustion? The orchestrator accumulates telemetry for the full run — at what scale does this exceed the context window?
2. **SubAgent invocation model:** What is the exact mechanism for `runSubagent`? Does each subagent get a fresh context, or does it inherit the orchestrator's context? This affects memory design significantly.
3. **SQL runtime for Anvil:** What provides the SQLite runtime for `anvil_checks` and `session_store`? Is this a VS Code extension, MCP tool, or built-in capability?
4. **Concurrent memory writes:** Forge's memory architecture assumes no concurrent writes to `memory.md`. But can two implementer subagents in the same wave write to their respective `memory/implementer-<task-id>.mem.md` files simultaneously without conflict?
5. **Anvil subagent model:** Anvil uses `code-review` subagents — what is the contract for these? Are they defined somewhere or are they inline agent definitions?

---

## Research Metadata

- **confidence_level:** high — All 23 Forge agent files, both reference documents, the prompt file, and the full Anvil agent were read in their entirety.
- **coverage_estimate:** ~95% of architecturally relevant content examined. All agent definitions read fully. Both reference docs and the prompt file read fully. Documentation structure verified via directory listings and existing feature directories.
- **gaps:**
  - Did not examine the content of existing feature pipeline outputs (e.g., `docs/feature/orchestrator-tool-restriction/design.md`) — these would show how the architecture performs in practice. Impact: may miss practical issues with the documented architecture.
  - Did not examine `.github/instructions/` or `.github/skills/` directories (if they exist) — these are referenced by R-Knowledge but may not be present. Impact: minimal for architecture research.
  - Did not verify whether `runSubagent` is a VS Code built-in or requires specific extension. Impact: low for architecture documentation, relevant for implementation.
