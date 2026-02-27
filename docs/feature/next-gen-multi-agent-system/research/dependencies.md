# Research: Dependencies

## Focus Area

Dependencies — module interactions, data flow, API/interface contracts, external dependencies, integration points between Forge and Anvil systems.

## Summary

Forge uses a file-based, memory-first DAG of 23 agents with 3 cluster dispatch patterns and strict Markdown-file contracts; Anvil uses a single-agent SQL-tracked loop with subagent code-review and git-centric verification. The two systems share no communication protocol, storage format, or verification strategy, creating significant integration surface for a merged design.

---

## Findings

### 1. Forge Agent Data Flow Graph

#### 1.1 Complete Pipeline Dependency Chain

The Forge orchestrator enforces an 8-step sequential pipeline where each step depends on artifacts produced by prior steps. The critical path is:

```
Orchestrator (Step 0: Setup)
  └─→ Researcher ×4 [parallel, Pattern A] (Step 1)
        ├─ researcher-architecture → research/architecture.md + memory/researcher-architecture.mem.md
        ├─ researcher-impact       → research/impact.md       + memory/researcher-impact.mem.md
        ├─ researcher-dependencies → research/dependencies.md + memory/researcher-dependencies.mem.md
        └─ researcher-patterns     → research/patterns.md     + memory/researcher-patterns.mem.md
      └─→ Spec (Step 2)
            └─→ feature.md + memory/spec.mem.md
          └─→ Designer (Step 3)
                └─→ design.md + memory/designer.mem.md
              └─→ CT Cluster ×4 [parallel, Pattern A] (Step 3b)
                    ├─ ct-security        → ct-review/ct-security.md        + memory/ct-security.mem.md
                    ├─ ct-scalability     → ct-review/ct-scalability.md     + memory/ct-scalability.mem.md
                    ├─ ct-maintainability → ct-review/ct-maintainability.md + memory/ct-maintainability.mem.md
                    └─ ct-strategy        → ct-review/ct-strategy.md       + memory/ct-strategy.mem.md
                  └─→ Planner (Step 4)
                        └─→ plan.md + tasks/*.md + memory/planner.mem.md
                      └─→ Implementer ×N [parallel waves, ≤4/wave] (Step 5)
                            └─→ code files + test files + memory/implementer-<task-id>.mem.md
                          └─→ V Cluster [Pattern B+C] (Step 6)
                                ├─ v-build [sequential gate]  → verification/v-build.md   + memory/v-build.mem.md
                                ├─ v-tests [parallel]         → verification/v-tests.md   + memory/v-tests.mem.md
                                ├─ v-tasks [parallel]         → verification/v-tasks.md   + memory/v-tasks.mem.md
                                └─ v-feature [parallel]       → verification/v-feature.md + memory/v-feature.mem.md
                              └─→ R Cluster ×4 [parallel, Pattern A] (Step 7)
                                    ├─ r-quality   → review/r-quality.md   + memory/r-quality.mem.md
                                    ├─ r-security  → review/r-security.md  + memory/r-security.mem.md
                                    ├─ r-testing   → review/r-testing.md   + memory/r-testing.mem.md
                                    └─ r-knowledge → review/r-knowledge.md + knowledge-suggestions.md + decisions.md + memory/r-knowledge.mem.md
                                  └─→ Post-Mortem [non-blocking] (Step 8)
                                        └─→ post-mortems/<date>.md + agent-metrics/<date>-run-log.md + memory/post-mortem.mem.md
```

#### 1.2 Agent-to-Artifact Read/Write Matrix

| Agent              | Reads (Primary)                                                                                                               | Reads (Selective via Artifact Index)                               | Writes                                                                                                                                         |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Orchestrator**   | All `memory/*.mem.md`, `memory.md`                                                                                            | `plan.md` (for wave parsing)                                       | `memory.md` (via subagent delegation)                                                                                                          |
| **Researcher ×4**  | `memory.md`, `initial-request.md`, codebase                                                                                   | —                                                                  | `research/<focus>.md`, `memory/researcher-<focus>.mem.md`                                                                                      |
| **Spec**           | `memory/*.mem.md` (researcher×4), `initial-request.md`, `memory.md`                                                           | `research/*.md` (sections per Artifact Index)                      | `feature.md`, `memory/spec.mem.md`, `artifact-evaluations/spec.md`                                                                             |
| **Designer**       | `memory/spec.mem.md`, `memory/researcher-*.mem.md`, `memory.md`, `initial-request.md`                                         | `feature.md`, `research/*.md`, `ct-review/ct-*.md` (revision mode) | `design.md`, `memory/designer.mem.md`, `artifact-evaluations/designer.md`                                                                      |
| **CT ×4**          | `memory.md`, `memory/designer.mem.md`, `memory/spec.mem.md`, `initial-request.md`                                             | `design.md`, `feature.md`                                          | `ct-review/ct-<focus>.md`, `memory/ct-<focus>.mem.md`, `artifact-evaluations/ct-<focus>.md`                                                    |
| **Planner**        | `memory/designer.mem.md`, `memory/spec.mem.md`, `memory.md`, `initial-request.md`                                             | `design.md`, `feature.md`, `research/*.md`, `ct-review/ct-*.md`    | `plan.md`, `tasks/*.md`, `memory/planner.mem.md`, `artifact-evaluations/planner.md`                                                            |
| **Implementer ×N** | `memory.md`, `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md`, task file                               | `feature.md`, `design.md`                                          | Code, tests, task file (status), `memory/implementer-<id>.mem.md`, `artifact-evaluations/implementer-<id>.md`                                  |
| **Doc Writer**     | `memory.md`, `memory/planner.mem.md`, `memory/designer.mem.md`, `memory/spec.mem.md`, task file                               | `feature.md`, `design.md`                                          | Doc files, task file (status), `memory/documentation-writer-<id>.mem.md`                                                                       |
| **V-Build**        | `memory.md`, `memory/planner.mem.md`, codebase                                                                                | —                                                                  | `verification/v-build.md`, `memory/v-build.mem.md`                                                                                             |
| **V-Tests**        | `memory.md`, `memory/v-build.mem.md`, `memory/planner.mem.md`, `v-build.md`, codebase                                         | —                                                                  | `verification/v-tests.md`, `memory/v-tests.mem.md`, `artifact-evaluations/v-tests.md`                                                          |
| **V-Tasks**        | `memory.md`, `memory/v-build.mem.md`, `memory/planner.mem.md`, `v-build.md`, `plan.md`, `tasks/*.md`, codebase                | —                                                                  | `verification/v-tasks.md`, `memory/v-tasks.mem.md`, `artifact-evaluations/v-tasks.md`                                                          |
| **V-Feature**      | `memory.md`, `memory/v-build.mem.md`, `memory/spec.mem.md`, `v-build.md`, `feature.md`, `design.md`, codebase                 | —                                                                  | `verification/v-feature.md`, `memory/v-feature.mem.md`, `artifact-evaluations/v-feature.md`                                                    |
| **R-Quality**      | `memory.md`, `memory/implementer-*.mem.md`, `memory/designer.mem.md`, `initial-request.md`, `design.md`, git diff, codebase   | —                                                                  | `review/r-quality.md`, `memory/r-quality.mem.md`, `artifact-evaluations/r-quality.md`                                                          |
| **R-Security**     | `memory.md`, `initial-request.md`, git diff, codebase                                                                         | —                                                                  | `review/r-security.md`, `memory/r-security.mem.md`                                                                                             |
| **R-Testing**      | `memory.md`, `memory/implementer-*.mem.md`, `memory/planner.mem.md`, `initial-request.md`, `feature.md`, git diff, codebase   | —                                                                  | `review/r-testing.md`, `memory/r-testing.mem.md`, `artifact-evaluations/r-testing.md`                                                          |
| **R-Knowledge**    | `memory.md`, `memory/implementer-*.mem.md`, `memory/planner.mem.md`, `initial-request.md`, `decisions.md`, git diff, codebase | `feature.md`, `design.md`, `plan.md` (via Artifact Index)          | `review/r-knowledge.md`, `knowledge-suggestions.md`, `decisions.md`, `memory/r-knowledge.mem.md`, `.github/instructions/*`, `.github/skills/*` |
| **Post-Mortem**    | Orchestrator telemetry (dispatch context), all `memory/*.mem.md`, `artifact-evaluations/*.md`, `memory.md`                    | —                                                                  | `post-mortems/<date>.md`, `agent-metrics/<date>-run-log.md`, `memory/post-mortem.mem.md`                                                       |

#### 1.3 Memory File Dependencies (Who Reads Whose Memory)

The memory-first protocol creates a layered dependency graph:

| Reader Agent | Memory Files Read                                                                                                            |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Orchestrator | ALL `memory/*.mem.md` (for cluster decision logic + merge)                                                                   |
| Spec         | `researcher-architecture.mem.md`, `researcher-impact.mem.md`, `researcher-dependencies.mem.md`, `researcher-patterns.mem.md` |
| Designer     | `spec.mem.md`, `researcher-*.mem.md`, `ct-*.mem.md` (revision mode)                                                          |
| CT ×4        | `designer.mem.md`, `spec.mem.md`                                                                                             |
| Planner      | `designer.mem.md`, `spec.mem.md`, `ct-*.mem.md` (if present), `v-*.mem.md` (replan mode)                                     |
| Implementer  | `planner.mem.md`, `designer.mem.md`, `spec.mem.md`                                                                           |
| V-Build      | `planner.mem.md`                                                                                                             |
| V-Tests      | `v-build.mem.md`, `planner.mem.md`                                                                                           |
| V-Tasks      | `v-build.mem.md`, `planner.mem.md`                                                                                           |
| V-Feature    | `v-build.mem.md`, `spec.mem.md`                                                                                              |
| R-Quality    | `implementer-*.mem.md`, `designer.mem.md`                                                                                    |
| R-Testing    | `implementer-*.mem.md`, `planner.mem.md`                                                                                     |
| R-Knowledge  | `implementer-*.mem.md`, `planner.mem.md`                                                                                     |
| R-Security   | (no upstream memories explicitly; reads `memory.md` for orientation)                                                         |
| Post-Mortem  | ALL `memory/*.mem.md`                                                                                                        |

#### 1.4 Tight vs. Loose Coupling

**Tightly coupled:**

- Orchestrator ↔ ALL agents (orchestrator is the sole coordinator; every agent depends on being dispatched by it and returning to it)
- V-Build → V-Tests/V-Tasks/V-Feature (sequential gate; downstream agents cannot start without V-Build DONE)
- Planner ↔ V cluster (replan loop creates bidirectional dependency; V outputs feed back into planner)
- R-Security → Pipeline (R-Security Blocker halts entire pipeline)
- `memory.md` → All agents (every agent reads shared memory for orientation)

**Loosely coupled:**

- Research agents ×4 (fully parallel, no mutual dependencies)
- CT ×4 (fully parallel, no mutual dependencies)
- R ×4 (fully parallel, no mutual dependencies; R-Knowledge is non-blocking)
- Post-Mortem (non-blocking; its failure doesn't affect pipeline)

#### 1.5 Critical Path

The longest sequential dependency chain:

```
Setup → Researcher (any 1) → Spec → Designer → CT (any 1) → [CT revision loop max 1] → Planner → Implementer (wave 1, task 1) → V-Build → V (any 1) → [Replan loop max 3: Planner → Implementer → V-Build → V] → R (any 1) → [R revision max 1] → Post-Mortem
```

Minimum sequential steps: 12 agent invocations (no revision loops). Maximum: 12 + 1 (CT revision) + 3×4 (V replan loops) + 1 (R revision) = 26 agent invocations.

### 2. Anvil Internal Data Flow

#### 2.1 Phase Dependency Chain

Anvil operates as a single agent with an internal loop (not multi-agent orchestration):

```
0. Boost (prompt refinement) → boosted_prompt
  └─→ 0b. Git Hygiene → clean git state
    └─→ 1. Understand → goal, acceptance_criteria, assumptions
      └─→ 1b. Recall (SQL query: session_store) → past_session_context
        └─→ 2. Survey (codebase search) → reuse_opportunities, blast_radius
          └─→ 3. Plan (internal, shown for Large) → file_changes, risk_levels
            └─→ 3b. Baseline Capture (SQL INSERT: anvil_checks, phase='baseline') → baseline_state
              └─→ 4. Implement → code_changes, test_changes
                └─→ 5. Verify ("The Forge")
                  ├─→ 5a. IDE Diagnostics (ide-get_diagnostics)
                  ├─→ 5b. Verification Cascade (Tier 1→2→3)
                  ├─→ 5c. Adversarial Review (1 or 3 code-review subagents)
                  ├─→ 5d. Operational Readiness (Large only)
                  └─→ 5e. Evidence Bundle (SQL SELECT: anvil_checks)
                    └─→ 6. Learn (store_memory)
                      └─→ 7. Present (evidence bundle to user)
                        └─→ 8. Commit (git add, git commit)
```

#### 2.2 Anvil's Verification Cascade Sequential Dependencies

```
Tier 1 (Always):
  ├─ IDE diagnostics (ide-get_diagnostics)
  └─ Syntax/parse check

Tier 2 (If tooling exists — run ALL applicable, not just first):
  ├─ Build/compile
  ├─ Type checker
  ├─ Linter
  └─ Tests

Tier 3 (Required when Tiers 1-2 produce no runtime verification):
  ├─ Import/load test
  └─ Smoke execution
```

Within each tier, checks are independent. Between tiers, there's a sequential dependency: Tier 3 runs only if Tiers 1-2 produce no runtime signal. All checks INSERT into `anvil_checks` SQL table.

The adversarial review (5c) has its own gating: verification ledger must have INSERT rows before review can proceed. Evidence bundle (5e) requires minimum verification signals (≥2 for Medium, ≥3 for Large) before presentation.

#### 2.3 Anvil External Dependencies

| Dependency                         | Usage                                                                                          | Required?                                               |
| ---------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| **Git**                            | Status check, branch management, stash, add, commit, diff (staged), revert, worktree detection | Yes — integral to Anvil loop steps 0b, 5c, 7, 8         |
| **IDE Diagnostics**                | `ide-get_diagnostics` for error detection on changed + importing files                         | Yes — Step 5a is always required                        |
| **SQL (SQLite via session_store)** | `anvil_checks` ledger for verification tracking; `session_store` for session recall            | Yes for Medium/Large — verification ledger is mandatory |
| **Subagent: code-review**          | Adversarial review with specified models (1 for Medium, 3 for Large)                           | Yes for Medium/Large                                    |
| **store_memory**                   | VS Code cross-session knowledge persistence                                                    | Yes — Step 6 Learn                                      |
| **ask_user**                       | Interactive prompts for pushback, plan confirmation, disambiguation                            | Yes — core to pushback system                           |
| **report_intent**                  | Progress reporting (minimal output mode)                                                       | Yes — used in steps 0-3b                                |
| **Context7**                       | `context7-resolve-library-id`, `context7-query-docs` for documentation lookup                  | Optional — used when unsure about libraries             |
| **MCP tools**                      | Fetch GitHub issues/PRs referenced in requests                                                 | Optional — conditional on request content               |
| **run_in_terminal**                | Build commands, test commands, git operations                                                  | Yes — verification cascade                              |

### 3. Inter-System Dependencies for Merging

#### 3.1 Communication Protocols

| Aspect                    | Forge                                                                                         | Anvil                                                                                                             | Overlap/Conflict                                                                           |
| ------------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Primary data exchange** | Markdown files on disk (`research/*.md`, `feature.md`, `design.md`, etc.)                     | Internal state within single agent context + SQL `anvil_checks` table                                             | **Conflict:** Forge uses file-based message passing; Anvil uses in-context state + SQL     |
| **Agent communication**   | File-mediated via `runSubagent`; each agent reads upstream files and writes downstream files  | No inter-agent communication (single agent); subagents (code-review) receive prompt context                       | **Conflict:** Forge's file protocol has no Anvil equivalent                                |
| **Completion signaling**  | 3-state contract: `DONE:` / `NEEDS_REVISION:` / `ERROR:` as return strings                    | No explicit signaling — single loop, internal gating via SQL ledger checks                                        | **Conflict:** Forge requires explicit completion contract; Anvil has implicit flow control |
| **Memory system**         | `memory.md` (shared, orchestrator-managed) + `memory/<agent>.mem.md` (isolated per agent)     | `store_memory` (VS Code cross-session) + `session_store` SQL (session recall) + `anvil_checks` SQL (verification) | **Partial overlap:** Both persist knowledge across sessions; formats completely different  |
| **Routing/decision**      | Orchestrator reads isolated `.mem.md` files to make cluster decisions (CT/V/R decision flows) | Single agent makes all decisions internally; gate checks via SQL COUNT queries                                    | **Conflict:** Distributed decision-making (Forge) vs. centralized (Anvil)                  |

#### 3.2 Memory System Overlap

| Forge Memory                                                    | Anvil Equivalent                                              | Compatibility      |
| --------------------------------------------------------------- | ------------------------------------------------------------- | ------------------ |
| `memory.md` (shared pipeline state)                             | No equivalent — Anvil has no shared state file                | **Gap**            |
| `memory/<agent>.mem.md` (isolated, ≤5 bullet key findings)      | No equivalent — Anvil doesn't produce memory summaries        | **Gap**            |
| Artifact Index (section pointers for targeted reads)            | No equivalent — Anvil reads sequentially from its own context | **Gap**            |
| `store_memory` (VS Code cross-session knowledge)                | `store_memory` (same mechanism in Step 6: Learn)              | **Direct overlap** |
| No SQL equivalent                                               | `anvil_checks` (verification evidence ledger)                 | **Gap in Forge**   |
| No SQL equivalent                                               | `session_store` (session recall for history)                  | **Gap in Forge**   |
| `decisions.md` (append-only architectural decision log)         | No equivalent                                                 | **Gap in Anvil**   |
| `knowledge-suggestions.md` (human-review improvement proposals) | No equivalent (Anvil uses `store_memory` directly)            | **Gap in Anvil**   |

#### 3.3 Verification Strategy Overlap

| Forge Verification (V Cluster)               | Anvil Verification (The Forge, Step 5)                          | Overlap                                                                               |
| -------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| V-Build: build/compile gate (sequential)     | Tier 2: Build/compile check                                     | **Functionally identical**                                                            |
| V-Tests: full test suite execution           | Tier 2: Tests (full suite or relevant subset)                   | **Functionally identical**                                                            |
| V-Tasks: per-task acceptance criteria        | No equivalent — Anvil verifies as single unit                   | **Gap** — Anvil doesn't decompose into tasks                                          |
| V-Feature: feature-level acceptance criteria | Not explicit — closest is evidence bundle verification          | **Partial gap**                                                                       |
| Pattern C: Replan loop (max 3 iterations)    | Fix and re-run (max 2 attempts per check), then revert          | **Different philosophy:** Forge replans from scratch; Anvil fixes in place or reverts |
| No equivalent                                | Tier 1: IDE diagnostics (always required)                       | **Gap in Forge** — Forge doesn't mandate IDE diagnostics during verification          |
| No equivalent                                | Tier 3: Import/load test + smoke execution                      | **Gap in Forge** — Forge has no runtime smoke testing                                 |
| No equivalent                                | 5c: Adversarial review (code-review subagents)                  | **Gap in Forge V cluster** — this maps to R cluster instead                           |
| No equivalent                                | 5d: Operational readiness (observability, degradation, secrets) | **Gap in Forge**                                                                      |
| No equivalent                                | 5e: Evidence bundle (SQL-backed proof)                          | **Gap in Forge** — Forge uses narrative verification reports                          |
| No equivalent                                | Baseline capture (before/after comparison)                      | **Gap in Forge** — Forge doesn't capture pre-change baseline                          |

#### 3.4 CT Cluster vs. Anvil Adversarial Review

| Forge CT Cluster (Step 3b)                                                  | Anvil Adversarial Review (Step 5c)                         | Mapping                                                                          |
| --------------------------------------------------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Runs on design document (pre-implementation)                                | Runs on implemented code (post-implementation)             | **Different timing** — CT is design-phase, Anvil review is code-phase            |
| 4 specialized sub-agents (security, scalability, maintainability, strategy) | 1-3 code-review sub-agents (same prompt, different models) | **Different approach:** specialized focus areas vs. model diversity for coverage |
| Produces detailed findings per focus area with severity ratings             | Produces verdicts INSERTed into SQL ledger                 | **Different output format**                                                      |
| NEEDS_REVISION routes back to designer                                      | Issues found → fix → re-run both verification AND review   | **Similar revision loop concept**                                                |
| Max 1 revision loop                                                         | Max 2 adversarial rounds                                   | **Similar bounded retry**                                                        |

#### 3.5 Data Format Compatibility

| Format           | Forge Usage                                                                                                     | Anvil Usage                                                            |
| ---------------- | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| **Markdown**     | ALL artifacts (research, feature, design, plan, tasks, verification, review, memory, evaluations, post-mortems) | Presentation output (evidence bundle), documentation                   |
| **SQL (SQLite)** | Not used                                                                                                        | `anvil_checks` (verification ledger), `session_store` (session recall) |
| **YAML**         | Artifact evaluation schema (embedded in Markdown fenced blocks)                                                 | Not used                                                               |
| **JSON**         | Not used                                                                                                        | Not used                                                               |
| **Git diff**     | Used by R cluster agents (review based on git diff context)                                                     | Used by adversarial reviewers (`git diff --staged`)                    |

### 4. External Dependencies

#### 4.1 VS Code Extension APIs

| API/Tool                               | Used By                                             | Purpose                                      |
| -------------------------------------- | --------------------------------------------------- | -------------------------------------------- |
| `agent/runSubagent`                    | Forge orchestrator                                  | Dispatch subagents                           |
| `memory` (VS Code cross-session store) | Forge orchestrator, Anvil (as `store_memory`)       | Persist codebase facts across sessions       |
| `ide-get_diagnostics`                  | Anvil (Step 5a)                                     | Get IDE-reported errors/warnings for files   |
| `ask_user`                             | Anvil (pushback, plan confirmation, disambiguation) | Interactive multiple-choice prompts          |
| `report_intent`                        | Anvil (Steps 0-3b)                                  | Progress reporting in minimal-output mode    |
| `code-review` subagent type            | Anvil (Step 5c)                                     | Adversarial code review with model selection |

#### 4.2 Git Operations Required

| Operation                         | Used By                          | Context                             |
| --------------------------------- | -------------------------------- | ----------------------------------- |
| `git status --porcelain`          | Anvil (Step 0b)                  | Dirty state check                   |
| `git rev-parse --abbrev-ref HEAD` | Anvil (Step 0b)                  | Branch check                        |
| `git rev-parse --show-toplevel`   | Anvil (Step 0b)                  | Worktree detection                  |
| `git add -A`                      | Anvil (Steps 5c, 8)              | Stage changes for review/commit     |
| `git commit -m`                   | Anvil (Step 8)                   | Auto-commit with generated message  |
| `git diff --staged`               | Anvil (Step 5c), Forge R cluster | View staged changes for review      |
| `git checkout HEAD -- {files}`    | Anvil (Step 5b)                  | Revert changes on failure           |
| `git stash push -m`               | Anvil (Step 0b)                  | Stash uncommitted changes           |
| `git checkout -b`                 | Anvil (Step 0b)                  | Create feature branch               |
| `git rev-parse HEAD`              | Anvil (Step 8)                   | Capture pre-commit SHA for rollback |
| `git revert HEAD`                 | Anvil (Step 8)                   | Rollback guidance                   |

Forge does **not** directly use git operations — it delegates all codebase interaction to subagents. The implementer agent uses `run_in_terminal` for builds/tests but git operations are not explicitly defined in Forge agent specs.

#### 4.3 Model/LLM Requirements

| Model                  | Used By            | Purpose                                                                          |
| ---------------------- | ------------------ | -------------------------------------------------------------------------------- |
| `gpt-5.3-codex`        | Anvil (Step 5c)    | Adversarial code review (Medium: sole reviewer; Large: one of three)             |
| `gemini-3-pro-preview` | Anvil (Step 5c)    | Adversarial code review (Large tasks: one of three)                              |
| `claude-opus-4.6`      | Anvil (Step 5c)    | Adversarial code review (Large tasks: one of three)                              |
| (unspecified/default)  | Forge (all agents) | No model constraints specified — agents use whatever model the platform provides |

#### 4.4 Tool Requirements by Agent Category

**Forge Read-Only Agents** (researcher, spec, designer, CT×4, planner, V×4, R×4, post-mortem, doc-writer):

- `read_file`, `semantic_search`, `grep_search`, `file_search` (discovery)
- `run_in_terminal` (V-Build, V-Tests for build/test execution)
- `create_file`, `replace_string_in_file` (for writing output artifacts)

**Forge Orchestrator** (restricted tool set):

- `read_file`, `list_dir` (reading only)
- `agent/runSubagent` (dispatch)
- `memory` (VS Code cross-session store — NOT pipeline files)

**Forge Implementer** (code-writing agent):

- `read_file`, `semantic_search`, `grep_search` (discovery)
- `create_file`, `replace_string_in_file`, `multi_replace_string_in_file` (code edits)
- `get_errors` (after every file edit)
- `run_in_terminal`, `get_terminal_output` (test execution)
- `list_code_usages` (refactoring impact analysis, with `grep_search` fallback)

**Anvil** (single agent, full tool access):

- All of the above, plus:
- `ask_user` (interactive prompts)
- `report_intent` (progress reporting)
- `ide-get_diagnostics` (IDE error detection)
- `store_memory` (cross-session persistence)
- `code-review` subagent type with model selection
- `context7-resolve-library-id`, `context7-query-docs` (documentation lookup via MCP)
- SQL execution (for `anvil_checks` and `session_store`)

#### 4.5 MCP Tools Referenced

| MCP Tool                                                        | Used By                    | Purpose                                |
| --------------------------------------------------------------- | -------------------------- | -------------------------------------- |
| Context7 (`context7-resolve-library-id`, `context7-query-docs`) | Anvil                      | Library/framework documentation lookup |
| GitHub MCP (unspecified)                                        | Anvil (Step 1: Understand) | Fetch referenced GitHub issues/PRs     |

Forge does not reference any MCP tools explicitly — its agents use standard VS Code extension tools.

### 5. Article Design Principles as Dependency Constraints

The article "Multi-agent workflows often fail. Here's how to engineer ones that don't" prescribes constraints that affect the dependency architecture:

#### 5.1 Typed Schemas at Every Agent Boundary

**Current Forge state:** Agent boundaries use untyped Markdown files. The only typed schema is the artifact evaluation YAML (defined in `evaluation-schema.md`). Memory files have a loose format (Status, Key Findings ≤5 bullets, Highest Severity, Decisions Made, Artifact Index) but it's not formally validated.

**Current Anvil state:** The SQL `anvil_checks` table has a typed schema with CHECK constraints (`phase IN ('baseline', 'after', 'review')`, `passed IN (0, 1)`). This is the strongest typed contract in either system.

**Gap:** Neither system has typed schemas at EVERY agent boundary. Forge's completion contract (`DONE:/NEEDS_REVISION:/ERROR:`) is a minimal typed output, but the primary artifacts (Markdown files) have no schema validation.

#### 5.2 Action Schemas Constraining Agent Outputs

**Current Forge state:** Each agent has a defined Outputs section listing allowed files. The orchestrator enforces completion contracts (3-state: DONE/NEEDS_REVISION/ERROR). Agent definitions include "File boundaries" rules and "Anti-Drift Anchors." However, there's no runtime enforcement — these are prompt-level constraints.

**Current Anvil state:** Anvil constrains its own behavior via internal gates (SQL COUNT checks before proceeding). Task sizing (Small/Medium/Large) constrains the scope of actions.

**Gap:** Neither system has a formal action schema that constrains the structure of what an agent CAN return — only what it SHOULD return via prompt instructions.

#### 5.3 MCP Enforcement Layer

**Current state:** Neither Forge nor Anvil uses MCP as an enforcement layer for tool contracts. Forge restricts the orchestrator's tool access via prompt instructions (`Tool access (restricted)`), but this is not machine-enforced. Anvil references MCP tools (Context7, GitHub) as optional utilities, not as an enforcement mechanism.

**Gap:** The article recommends MCP as a contract enforcement layer with explicit input/output schemas validated before execution. Neither system implements this.

#### 5.4 Distributed Systems Design

**Current Forge state:** Implements several distributed systems principles:

- State isolation (isolated `.mem.md` files per agent)
- Ordering (8-step sequential pipeline with cluster dispatch patterns)
- Retry with bounded budget (3 internal × 2 orchestrator = 6 max)
- Graceful degradation (non-blocking agents like R-Knowledge, Post-Mortem)
- Logging intermediate state (telemetry context tracking, memory lifecycle)
- Validation at boundaries (cluster decision flows with severity thresholds)

**Current Anvil state:** Implements some distributed systems principles:

- State tracking (SQL ledger provides audit trail)
- Bounded retries (max 2 attempts per check, max 2 adversarial rounds)
- Failure handling (revert on failure: `git checkout HEAD -- {files}`)
- Evidence-based verification (SQL SELECT, not self-reported claims)

**Gap:** Forge lacks Anvil's SQL-backed audit trail. Anvil lacks Forge's state isolation between phases.

---

## File References

| File/Folder                                                                                  | Rationale                                                                                                                                            |
| -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| [.github/agents/orchestrator.agent.md](.github/agents/orchestrator.agent.md)                 | Central pipeline coordinator; defines all dispatch patterns, cluster decision logic, memory lifecycle, tool restrictions, and NEEDS_REVISION routing |
| [.github/agents/dispatch-patterns.md](.github/agents/dispatch-patterns.md)                   | Defines Pattern A (parallel), Pattern B (sequential gate + parallel), Pattern C (replan loop), and memory-first pattern                              |
| [.github/agents/researcher.agent.md](.github/agents/researcher.agent.md)                     | First pipeline step; 4 parallel instances with focus areas; reads codebase, writes research + isolated memory                                        |
| [.github/agents/spec.agent.md](.github/agents/spec.agent.md)                                 | Consumes 4 researcher memories + research files; produces feature.md                                                                                 |
| [.github/agents/designer.agent.md](.github/agents/designer.agent.md)                         | Consumes spec + researcher memories; produces design.md; revision mode reads CT memories                                                             |
| [.github/agents/ct-security.agent.md](.github/agents/ct-security.agent.md)                   | CT cluster member; reads designer + spec memories + design.md + feature.md                                                                           |
| [.github/agents/ct-scalability.agent.md](.github/agents/ct-scalability.agent.md)             | CT cluster member; same input pattern as ct-security                                                                                                 |
| [.github/agents/ct-maintainability.agent.md](.github/agents/ct-maintainability.agent.md)     | CT cluster member; same input pattern                                                                                                                |
| [.github/agents/ct-strategy.agent.md](.github/agents/ct-strategy.agent.md)                   | CT cluster member; broadest scope — challenges fundamental approach                                                                                  |
| [.github/agents/planner.agent.md](.github/agents/planner.agent.md)                           | Consumes designer + spec memories; produces plan.md + task files; replan mode consumes V memories                                                    |
| [.github/agents/implementer.agent.md](.github/agents/implementer.agent.md)                   | Consumes task file + upstream memories; writes code + tests; TDD workflow with get_errors                                                            |
| [.github/agents/documentation-writer.agent.md](.github/agents/documentation-writer.agent.md) | Alternative task executor; same memory inputs as implementer; writes documentation only                                                              |
| [.github/agents/v-build.agent.md](.github/agents/v-build.agent.md)                           | Sequential gate in V cluster; detects build system; all V agents depend on its output                                                                |
| [.github/agents/v-tests.agent.md](.github/agents/v-tests.agent.md)                           | Runs full test suite; reads v-build.md for build context                                                                                             |
| [.github/agents/v-tasks.agent.md](.github/agents/v-tasks.agent.md)                           | Per-task acceptance criteria verification; reads plan.md + tasks/\*.md                                                                               |
| [.github/agents/v-feature.agent.md](.github/agents/v-feature.agent.md)                       | Feature-level acceptance criteria; reads feature.md + design.md                                                                                      |
| [.github/agents/r-quality.agent.md](.github/agents/r-quality.agent.md)                       | Code quality review; reads implementer memories + git diff                                                                                           |
| [.github/agents/r-security.agent.md](.github/agents/r-security.agent.md)                     | Security scanning; pipeline-blocking on Blocker severity                                                                                             |
| [.github/agents/r-testing.agent.md](.github/agents/r-testing.agent.md)                       | Test quality review; reads implementer + planner memories                                                                                            |
| [.github/agents/r-knowledge.agent.md](.github/agents/r-knowledge.agent.md)                   | Knowledge evolution; non-blocking; writes to .github/instructions/ and .github/skills/                                                               |
| [.github/agents/post-mortem.agent.md](.github/agents/post-mortem.agent.md)                   | Quantitative analysis of pipeline run; non-blocking; reads orchestrator telemetry                                                                    |
| [.github/agents/evaluation-schema.md](.github/agents/evaluation-schema.md)                   | Shared YAML schema for artifact evaluations; referenced by 14 evaluating agents                                                                      |
| [.github/agents/critical-thinker.agent.md](.github/agents/critical-thinker.agent.md)         | DEPRECATED — superseded by CT cluster; retained for reference                                                                                        |
| [.github/prompts/feature-workflow.prompt.md](.github/prompts/feature-workflow.prompt.md)     | Entry point prompt; configures orchestrator with USER_FEATURE and APPROVAL_MODE variables                                                            |
| [Anvil/anvil.agent.md](Anvil/anvil.agent.md)                                                 | Complete single-agent system; defines 9-step internal loop, SQL ledger, adversarial review, pushback, git hygiene, evidence bundle                   |

---

## Assumptions & Limitations

1. **Assumption:** Agent definitions in `.github/agents/` accurately reflect runtime behavior. Prompt-level constraints (tool restrictions, file boundaries) are assumed to be respected, though they are not machine-enforced.
2. **Assumption:** The `session_store` SQL database referenced by Anvil (Step 1b: Recall) exists as a VS Code extension feature or plugin — its exact implementation/provider is not documented in the agent definition.
3. **Assumption:** The `code-review` subagent type referenced by Anvil is a platform-provided agent type with model selection capability, not a custom agent definition in this repo.
4. **Limitation:** Could not verify runtime dispatch behavior — analysis is based entirely on agent definition files (static analysis of Markdown specifications).
5. **Limitation:** The deprecated `critical-thinker.agent.md` is included for completeness but not analyzed in depth.
6. **Assumption:** Forge's `memory` tool (VS Code cross-session store) and Anvil's `store_memory` are the same underlying mechanism.

---

## Open Questions

1. **SQL runtime:** How is the `anvil_checks` SQLite database provisioned at runtime? Is it per-workspace, per-session, or global? This affects whether a merged system can share verification data across pipeline runs.
2. **Session store provider:** What VS Code extension or platform feature provides the `session_store` SQL database used by Anvil's Recall step? Is this available in all environments where Forge runs?
3. **Code-review model selection:** Can VS Code's `runSubagent` mechanism specify the LLM model for a subagent invocation? If not, Anvil's multi-model adversarial review pattern cannot be directly ported to Forge's dispatch mechanism.
4. **Tool restriction enforcement:** Is there a platform mechanism to enforce tool access restrictions (as specified in Forge's orchestrator), or is it entirely prompt-based? The article recommends MCP enforcement.
5. **MCP availability:** Which MCP tools (Context7, GitHub) are available in the target deployment environment? This affects which Anvil features can be retained.
6. **Concurrent subagent model limits:** The orchestrator caps at 4 concurrent subagents. Is this a platform limitation or a design choice? The article discusses parallelization tradeoffs.

---

## Research Metadata

- **confidence_level:** high — all 23 Forge agent definitions and the complete Anvil agent definition were read and analyzed; all dispatch patterns and cluster decision flows were mapped.
- **coverage_estimate:** ~95% of the relevant codebase was examined. All agent definition files, dispatch patterns, evaluation schema, and the feature workflow prompt were read. The only files not examined in full were the existing orchestrator-tool-restriction feature artifacts (examined peripherally for decisions.md patterns).
- **gaps:** (1) Runtime behavior verification — analysis is static (Markdown spec analysis), not dynamic. (2) Platform capabilities (model selection in subagent dispatch, MCP enforcement, SQL availability) are inferred, not confirmed. (3) The `NewAgents/` output directory structure was not analyzed as it's empty. Impact: downstream agents may need to independently verify platform capabilities before committing to merged architecture decisions.
