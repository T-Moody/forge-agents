# Feature Specification: Next-Gen Multi-Agent System

## Title & Short Summary

A next-generation, deterministic, multi-agent system for GitHub Copilot in VS Code—merging the best of Forge Orchestrator (pipeline orchestration, parallel dispatch, cluster patterns) and Anvil (evidence-first verification, adversarial multi-model review, risk classification, SQL ledger). The system must be robust, auditable, and scalable while minimizing agent count and orchestration overhead.

**Decision note:** This specification presents **5 distinct architectural directions**. The Design Agent is responsible for selecting or synthesizing a final architecture. The Spec Agent does NOT commit to a single approach.

---

## Background & Context

### Source Systems

| System                 | Description                                                                                                                                                 | Key Strength                                                                         |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Forge Orchestrator** | 23-agent, 8-step deterministic pipeline with 3 dispatch patterns (A/B/C), 3 clusters (CT, V, R), dual-layer memory system, three-state completion contracts | Pipeline orchestration, parallelism, structured revision routing                     |
| **Anvil Agent**        | Single 419-line agent with 12-phase loop, SQL verification ledger, multi-model adversarial review, risk classification, pushback system                     | Evidence-first verification, adversarial quality assurance, per-file risk assessment |
| **Article Principles** | "Multi-agent workflows often fail" (GitHub Blog, Feb 2026)                                                                                                  | Typed schemas, action schemas, MCP enforcement, design-for-failure-first             |

### Research References

- [research/architecture.md](research/architecture.md) — Forge 8-step pipeline, Anvil 12-phase loop, dispatch patterns, memory architecture
- [research/impact.md](research/impact.md) — Agent value assessment, Anvil component impact, agent count reduction analysis (21 → 10–12)
- [research/dependencies.md](research/dependencies.md) — Data flow graphs, inter-system dependency gaps, external dependencies, article principle gap analysis
- [research/patterns.md](research/patterns.md) — 10 reusable patterns, 5 anti-patterns, cross-system comparison, failure prevention mapping

### Key Research Findings

1. **CT cluster is the primary bottleneck**: 4 same-model-different-prompt agents required 3 iterations (12 invocations) for one design review. Adversarial multi-model review (genuinely different LLMs) is strictly superior.
2. **Memory system is over-engineered**: ~20+ files, ~12 merge operations per run. Typed schemas can replace prose memory with far lower overhead.
3. **Forge and Anvil communicate incompatibly**: Markdown files (Forge) vs. SQL tables (Anvil). Neither uses typed schemas at every agent boundary.
4. **Net agent reduction achievable**: 21 active → 10–12, eliminating 9–11 agents while adding 1 parameterized adversarial reviewer.
5. **Anvil's SQL ledger is the strongest verification contract**: INSERT-before-report eliminates hallucinated verification. Forge has no equivalent.

---

## Architectural Directions

The following 5 directions represent distinct philosophies for the next-gen system. Each is self-contained with enough detail for the Design Agent to make an informed decision—or synthesize a hybrid from multiple directions.

---

### Direction A: Lean Pipeline

**Philosophy:** Aggressive consolidation. Merge everything possible. Minimize agent count, memory overhead, and orchestration complexity. Every agent must justify its existence independently.

**Agent Count:** 6–8

**Agent List:**

| #   | Agent                    | Role                                                                              |
| --- | ------------------------ | --------------------------------------------------------------------------------- |
| 1   | Orchestrator             | Pipeline coordination, dispatch, evidence gating, risk-based routing              |
| 2   | Researcher (×4 parallel) | Codebase investigation (counts as 1 definition, 4 instances)                      |
| 3   | Spec                     | Feature specification with pushback                                               |
| 4   | Designer                 | Technical design                                                                  |
| 5   | Planner                  | Task decomposition with risk classification                                       |
| 6   | Implementer              | TDD implementation + documentation (doc-writer merged in)                         |
| 7   | Verifier                 | Unified verification (build gate + tests + acceptance criteria + evidence ledger) |
| 8   | Adversarial Reviewer     | Parameterized by model and scope (design review OR code review)                   |

**Pipeline Structure:**

```
Step 0: Setup
Step 1: Research ×4 (parallel) → typed schema output
Step 2: Specification (sequential, with pushback gate)
Step 3: Design (sequential)
Step 3b: Adversarial Design Review — 1–3 models based on risk (replaces CT cluster)
Step 4: Planning (sequential, with risk classification)
Step 5: Implementation waves (≤4 concurrent, with baseline capture)
Step 6: Unified Verification (build gate → all checks, Pattern B, wrapped in replan loop)
Step 7: Adversarial Code Review — 1–3 models based on risk (replaces R cluster except r-knowledge)
Step 8: Knowledge capture (folded into orchestrator or omitted)
```

**Communication Mechanism:**

- Typed YAML schemas at every agent boundary (replacing Markdown memory files)
- Each agent returns a structured YAML output document with validated fields
- Orchestrator reads typed outputs directly — no merge subagent needed
- SQL verification ledger for all verification evidence (Anvil pattern)

**Memory/State Management:**

- No shared `memory.md` — replaced by typed agent output files that serve as the pipeline state
- No isolated `.mem.md` files — agent outputs ARE the memory
- SQL `verification_ledger` table for all verification evidence (persisted)
- VS Code `store_memory` for cross-session knowledge only
- Lifecycle: create on agent completion, read by downstream agents, no merge/prune operations

**Verification Strategy:**

- Unified verifier runs tiered cascade: IDE diagnostics → build → type check → lint → tests → smoke
- SQL ledger with INSERT-before-report (Anvil pattern)
- Baseline capture before implementation
- Evidence gates at: post-research, post-design-review, post-implementation, post-verification
- Minimum signals: 2 for standard tasks, 3 for Large/🔴

**Adversarial Review Strategy:**

- Parameterized single agent definition dispatched with different models
- Design review (Step 3b): 1 model for standard, 3 models for Large/🔴
- Code review (Step 7): 1 model for standard, 3 models for Large/🔴
- Models: gpt-5.3-codex, gemini-3-pro-preview, claude-opus-4.6
- Each verdict INSERTed into SQL ledger with `phase = 'review'`
- SQL COUNT gate before proceeding: ≥1 for standard, ≥3 for Large/🔴

**Approval Mode Integration:**

- Structured multiple-choice prompts (Anvil `ask_user` pattern)
- Gates at: post-research (approve research direction), post-planning (approve task breakdown)
- Autonomous mode skips gates; interactive mode enforces them

**Tradeoffs:**

| Pros                                                | Cons                                                                               |
| --------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Fewest agents → lowest orchestration overhead       | Unified verifier is complex (covers build, tests, acceptance criteria, evidence)   |
| No memory merge operations                          | r-knowledge capability lost unless folded into orchestrator or post-step           |
| Simplest pipeline to understand                     | Single verifier agent may hit context limits on large features                     |
| Fastest pipeline execution                          | Less specialization — adversarial reviewer must handle both design and code review |
| Typed schemas enforce correctness at every boundary | Aggressive consolidation may lose nuance from specialized agents                   |

**Risk Classification:** 🟡 — Good balance of simplicity and capability, but unified verifier complexity and loss of r-knowledge specialization are risks.

---

### Direction B: Forge-Plus

**Philosophy:** Evolutionary upgrade. Preserve Forge's proven pipeline structure. Replace underperforming components (CT, partial R) with Anvil patterns. Add typed schemas, evidence gating, and risk classification as extensions rather than replacements.

**Agent Count:** 12–14

**Agent List:**

| #   | Agent                    | Role                                                                        |
| --- | ------------------------ | --------------------------------------------------------------------------- |
| 1   | Orchestrator             | Enhanced with evidence gating, risk-based dispatch, typed schema validation |
| 2   | Researcher (×4 parallel) | Unchanged from Forge                                                        |
| 3   | Spec                     | Enhanced with pushback system                                               |
| 4   | Designer                 | Unchanged from Forge                                                        |
| 5   | Planner                  | Enhanced with risk classification per file                                  |
| 6   | Implementer              | Enhanced with baseline capture, verification cascade                        |
| 7   | Documentation Writer     | Retained as separate agent                                                  |
| 8   | V-Build                  | Retained as sequential gate                                                 |
| 9   | V-Tests                  | Retained for test execution                                                 |
| 10  | V-Verify                 | Merged v-tasks + v-feature into unified acceptance verifier                 |
| 11  | Adversarial Reviewer     | New — replaces CT cluster and r-quality/r-security/r-testing                |
| 12  | R-Knowledge              | Retained for knowledge evolution                                            |
| 13  | Post-Mortem              | Retained (optional, non-blocking)                                           |

**Pipeline Structure:**

```
Step 0: Setup
Step 1: Research ×4 (Pattern A) → typed schema merge
Step 2: Specification (sequential, with pushback)
Step 3: Design (sequential)
Step 3b: Adversarial Design Review — 1–3 models (replaces CT cluster)
Step 4: Planning (sequential, with risk classification)
Step 5: Implementation waves (≤4 concurrent, with baseline capture)
Step 6: V-Cluster (Pattern B+C): V-Build gate → V-Tests + V-Verify parallel → replan loop
Step 7: Adversarial Code Review — 1–3 models (replaces r-quality/r-security/r-testing)
Step 7b: R-Knowledge (non-blocking, Pattern A single agent)
Step 8: Post-Mortem (non-blocking)
```

**Communication Mechanism:**

- Hybrid: typed YAML schemas for agent completion signals + Markdown artifacts for human-readable outputs
- Completion contracts remain three-state (DONE/NEEDS_REVISION/ERROR)
- Memory files retain Forge format but with typed headers (YAML front matter + Markdown body)
- Orchestrator reads YAML front matter for routing decisions, ignores Markdown body
- SQL verification ledger added for V-cluster evidence

**Memory/State Management:**

- Retain dual-layer memory (shared + isolated) but reduce merge operations
- Shared `memory.md` still exists but with typed YAML sections
- Isolated `.mem.md` files use YAML front matter for machine-readable fields
- Memory merge reduced: only at cluster boundaries (not after every agent)
- SQL `verification_ledger` for verification evidence
- VS Code `store_memory` for cross-session knowledge

**Verification Strategy:**

- V-Build as sequential gate (unchanged)
- V-Tests for full test suite execution (unchanged)
- V-Verify (new merged agent) for acceptance criteria verification
- Pattern B+C for V-cluster dispatch (sequential gate + parallel, wrapped in replan loop)
- SQL ledger for V-Build and V-Tests evidence (new)
- Baseline capture before implementation (new)

**Adversarial Review Strategy:**

- Single parameterized adversarial reviewer replaces CT (4 agents) and partial R cluster (3 agents)
- Design review at Step 3b: same models and counts as Direction A
- Code review at Step 7: same models and counts as Direction A
- R-Security blocker policy retained: any security blocker finding → pipeline ERROR
- R-Knowledge retained separately for knowledge evolution

**Approval Mode Integration:**

- Forge-style approval gates retained at existing positions
- Enhanced with structured multiple-choice prompts (Anvil pattern)
- Same gates as Direction A

**Tradeoffs:**

| Pros                                                     | Cons                                                                    |
| -------------------------------------------------------- | ----------------------------------------------------------------------- |
| Lowest migration risk — builds on proven Forge structure | Higher agent count → more orchestration overhead than Direction A       |
| Retains specialized V-Build/V-Tests ground-truth signals | Still has memory merge operations (reduced but not eliminated)          |
| R-Knowledge retained for knowledge evolution             | Hybrid memory (typed + prose) may cause confusion                       |
| Post-Mortem retained for pipeline learning               | Documentation Writer retained as separate agent despite low utilization |
| Familiar pipeline for maintainers                        | Doesn't fully commit to typed schemas — partial adoption                |

**Risk Classification:** 🟢 — Lowest risk. Evolutionary path preserves what works, replaces what doesn't. Higher complexity ceiling than necessary.

---

### Direction C: Anvil-Core

**Philosophy:** Anvil-first approach. Extend Anvil's single-agent evidence-first loop into a minimal multi-agent pipeline. SQL-centric state management. Every verification is an INSERT. Maximize evidence-based quality assurance at the cost of pipeline flexibility.

**Agent Count:** 4–6

**Agent List:**

| #   | Agent                | Role                                                                         |
| --- | -------------------- | ---------------------------------------------------------------------------- |
| 1   | Orchestrator         | Lightweight coordinator — SQL-based state machine, dispatch, evidence gating |
| 2   | Researcher-Spec      | Combined research + specification (Anvil's Understand+Survey+Plan)           |
| 3   | Designer-Planner     | Combined design + task decomposition with risk classification                |
| 4   | Implementer          | Full Anvil loop per task: baseline → implement → verify → evidence bundle    |
| 5   | Adversarial Reviewer | Multi-model review (same as Anvil's Step 5c)                                 |
| 6   | Knowledge Agent      | Optional — cross-session learning (extends Anvil's Learn step)               |

**Pipeline Structure:**

```
Step 0: Setup (SQL tables created: pipeline_state, verification_ledger, decisions)
Step 1: Research-Spec — understand codebase, produce specification (with pushback)
Step 2: Designer-Planner — design + task decomposition + risk classification
Step 2b: Adversarial Design Review — 1–3 models
Step 3: Implementation loop (per task):
  3a: Implementer — baseline capture → implement → tiered verification → evidence bundle
  3b: Adversarial Code Review — 1–3 models per task
  3c: Evidence gate — SQL COUNT check before proceeding to next task
Step 4: Knowledge capture (optional, non-blocking)
```

**Communication Mechanism:**

- SQL tables as the primary communication medium between agents
- `pipeline_state` table tracks pipeline progress, agent outputs, routing decisions
- `verification_ledger` table (Anvil's `anvil_checks`) for all verification evidence
- `decisions` table for architectural decisions (replaces decisions.md)
- Minimal file artifacts: feature.md, design.md, plan.md generated as exports from SQL state
- Agent outputs are SQL INSERTs, not file writes

**Memory/State Management:**

- SQL-centric: all pipeline state in SQLite tables
- No shared `memory.md` — replaced by `pipeline_state` SQL table
- No isolated `.mem.md` files — replaced by SQL records
- Cross-session: `session_store` SQL (Anvil pattern) + VS Code `store_memory`
- Lifecycle: SQL transactions for atomicity, SELECT for reads, no merge/prune pattern

**Verification Strategy:**

- Full Anvil verification cascade per task: IDE diagnostics → build → type check → lint → tests → smoke
- SQL ledger with INSERT-before-report
- Baseline capture before every task
- Evidence gates at every transition (SQL COUNT checks)
- Per-task verification (not feature-level) — catches issues earlier
- Automatic revert on 2 failed fix attempts (Anvil pattern)

**Adversarial Review Strategy:**

- Same as Direction A but applied per-task, not per-feature
- Design review once for the whole feature
- Code review per implementation task
- Fix-and-re-review cycle: max 2 adversarial rounds per task
- Remaining findings after 2 rounds → Low confidence, presented as known issues

**Approval Mode Integration:**

- SQL-backed approval tracking
- Structured `ask_user` prompts at: post-spec (approve direction), post-plan (approve tasks)
- Pushback system from Anvil integrated into Researcher-Spec agent

**Tradeoffs:**

| Pros                                                     | Cons                                                                                   |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Fewest agents — maximum consolidation                    | Agent definitions are very complex (each does multiple Forge agents' work)             |
| SQL-native state is queryable, auditable, tamper-evident | Requires SQLite availability in every runtime environment                              |
| Per-task verification catches issues earlier             | No specialized research phase — combined Researcher-Spec may produce weaker research   |
| Automatic revert on failure                              | Pipeline is less parallelizable — per-task sequential loop is inherently serial        |
| Evidence-first everything                                | Large feature.md/design.md may not fit single agent context windows                    |
| Full Anvil verification rigor                            | Requires significant SQL infrastructure not present in current VS Code agent framework |

**Risk Classification:** 🔴 — Highest risk. SQL dependency may not be satisfiable in all environments. Agent consolidation into 4–6 mega-agents creates context window risks and reduces specialization. Per-task sequential loop limits parallelism.

---

### Direction D: Hybrid Typed

**Philosophy:** Article-first approach. Typed schemas at every boundary. Action schemas constraining every agent output. MCP-style enforcement where possible. Cherry-pick the best patterns from both systems with formal contracts at every interface.

**Agent Count:** 8–10

**Agent List:**

| #   | Agent                    | Role                                                                                             |
| --- | ------------------------ | ------------------------------------------------------------------------------------------------ |
| 1   | Orchestrator             | Schema-validating coordinator — validates every agent output against typed schema before routing |
| 2   | Researcher (×4 parallel) | Typed output: YAML research schema with required fields                                          |
| 3   | Spec                     | Typed output: YAML requirements schema                                                           |
| 4   | Designer                 | Typed output: YAML design schema with decision justifications                                    |
| 5   | Planner                  | Typed output: YAML task schema with risk classifications                                         |
| 6   | Implementer              | Typed output: YAML implementation report + code changes                                          |
| 7   | Verifier                 | Typed output: verification evidence (SQL or structured YAML)                                     |
| 8   | Adversarial Reviewer     | Typed output: review verdict schema per model                                                    |
| 9   | R-Knowledge              | Typed output: knowledge evolution schema                                                         |
| 10  | Post-Mortem              | Optional — typed metrics schema                                                                  |

**Pipeline Structure:**

```
Step 0: Setup (schema definitions loaded, validation rules active)
Step 1: Research ×4 (parallel) → validated YAML research outputs
Step 2: Specification → validated YAML requirements
Step 3: Design → validated YAML design
Step 3b: Adversarial Design Review → validated verdict schemas
Step 4: Planning → validated YAML task schemas with risk levels
Step 5: Implementation waves → validated implementation reports
Step 6: Verification → validated evidence records (SQL or YAML)
Step 7: Adversarial Code Review → validated verdict schemas
Step 7b: R-Knowledge → validated knowledge updates
Step 8: Post-Mortem → validated metrics (optional)
```

**Communication Mechanism:**

- Every agent boundary has a defined YAML schema (input AND output)
- Orchestrator validates every agent output against its schema before proceeding
- Schema violation → retry once → ERROR (treated as contract failure per article)
- Schemas define: required fields, types, allowed values, minimum entries
- Example output contract: `{ status: DONE|NEEDS_REVISION|ERROR, severity: Critical|High|Medium|Low, findings: [{id, category, description, affected_files, recommendation}] }`
- Human-readable Markdown artifacts generated as exports from typed data

**Memory/State Management:**

- Pipeline state = ordered collection of validated YAML outputs
- No memory merge — downstream agents read upstream typed outputs directly
- Orchestrator maintains a typed pipeline manifest (list of completed steps, output paths, validation status)
- Verification evidence in SQL where available, structured YAML fallback
- Cross-session: VS Code `store_memory`
- Decision log: typed YAML append-only

**Verification Strategy:**

- Verifier agent produces typed evidence records
- SQL ledger preferred, structured YAML fallback for environments without SQLite
- Baseline capture before implementation
- Evidence gates: orchestrator validates evidence record counts and structure
- Tiered cascade from Anvil with formal schema for each tier's output

**Adversarial Review Strategy:**

- Same model/count rules as Direction A
- Each reviewer's verdict must conform to a review verdict schema
- Orchestrator validates verdict schema before accepting
- Disagreement handling: majority rules for standard issues; any security blocker escalates regardless

**Approval Mode Integration:**

- Typed approval schema: `{ gate: string, options: [{id, label, description}], selected: string }`
- Enforced at: post-research, post-planning
- Autonomous mode auto-selects default option

**Tradeoffs:**

| Pros                                                             | Cons                                                                               |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Maximum reliability — schema violations caught at every boundary | Schema design is complex upfront work                                              |
| Machine-parseable pipeline state — no prose parsing              | Typed YAML schemas add verbosity to every agent definition                         |
| SQL or YAML verification — works with or without SQLite          | Schema evolution requires coordinated updates across agents                        |
| Clear contracts enable agent-level unit testing                  | Over-engineering risk — some outputs (like research findings) resist strict typing |
| Article principles fully implemented                             | More agent definitions than Direction A with similar total work                    |
| Both Forge AND Anvil best patterns preserved                     | Orchestrator becomes a schema validation engine — adds complexity                  |

**Risk Classification:** 🟡 — Good theoretical foundation. Schema design complexity and potential over-engineering of research/spec outputs are the main risks. Fallback strategy (SQL or YAML) mitigates environment dependency.

---

### Direction E: Event-Driven

**Philosophy:** Replace fixed pipeline with an event-driven state machine. Agents react to state transitions rather than following a predetermined sequence. Maximum flexibility for complex features — pipeline steps can be reordered, skipped, or repeated based on runtime conditions.

**Agent Count:** 8–12

**Agent List:**

| #   | Agent                        | Role                                                                                 |
| --- | ---------------------------- | ------------------------------------------------------------------------------------ |
| 1   | Orchestrator (State Machine) | Event router — maintains state machine, dispatches agents based on state transitions |
| 2   | Researcher (×4 parallel)     | Triggered by `RESEARCH_REQUESTED` event                                              |
| 3   | Spec                         | Triggered by `RESEARCH_COMPLETE` event                                               |
| 4   | Designer                     | Triggered by `SPEC_COMPLETE` event                                                   |
| 5   | Planner                      | Triggered by `DESIGN_APPROVED` event                                                 |
| 6   | Implementer                  | Triggered by `TASK_READY` events (one per task)                                      |
| 7   | Verifier                     | Triggered by `IMPLEMENTATION_COMPLETE` events                                        |
| 8   | Adversarial Reviewer         | Triggered by `REVIEW_REQUESTED` events (design or code)                              |
| 9   | R-Knowledge                  | Triggered by `PIPELINE_COMPLETE` event                                               |
| 10  | Post-Mortem                  | Triggered by `PIPELINE_COMPLETE` event (optional)                                    |

**Pipeline Structure:**

```
State Machine (not fixed pipeline):

INIT → RESEARCHING → SPECIFYING → DESIGNING → REVIEWING_DESIGN →
  → PLANNING → [IMPLEMENTING → VERIFYING → REVIEWING_CODE]* →
  → CAPTURING_KNOWLEDGE → COMPLETE

Transitions:
  RESEARCHING → SPECIFYING: when ≥3 of 4 researchers complete
  SPECIFYING → DESIGNING: on spec DONE
  DESIGNING → REVIEWING_DESIGN: on design DONE
  REVIEWING_DESIGN → PLANNING: on review DONE (no critical findings)
  REVIEWING_DESIGN → DESIGNING: on review NEEDS_REVISION
  PLANNING → IMPLEMENTING: on plan DONE (per-task events emitted)
  IMPLEMENTING → VERIFYING: on task implementation DONE
  VERIFYING → REVIEWING_CODE: on verification DONE
  REVIEWING_CODE → IMPLEMENTING: on review NEEDS_REVISION (next task or fix)
  REVIEWING_CODE → CAPTURING_KNOWLEDGE: when all tasks complete
  Any → ERROR: on unrecoverable failure
```

**Communication Mechanism:**

- Event-based: agents emit typed events, orchestrator routes based on state machine rules
- Event schema: `{ event_type: string, source_agent: string, payload: typed_data, timestamp: ISO8601 }`
- Event log: append-only record of all state transitions (replaces memory.md)
- Agent outputs attached as event payloads (typed)
- Orchestrator reads event log for routing decisions

**Memory/State Management:**

- Event log IS the memory — complete, append-only, queryable
- No separate memory files — event payloads contain all agent outputs
- SQL verification ledger for verification evidence (separate from event log)
- State machine state persisted in orchestrator context
- Cross-session: VS Code `store_memory` for knowledge

**Verification Strategy:**

- Same tiered cascade as Direction A
- Verification triggered per-task (event-driven: each `IMPLEMENTATION_COMPLETE` triggers verification)
- SQL ledger for evidence
- Evidence gates expressed as state machine transition guards

**Adversarial Review Strategy:**

- Same model/count rules as Direction A
- Review triggered by events: `DESIGN_COMPLETE` for design review, `VERIFICATION_COMPLETE` for code review
- Flexible: can re-review at any point by emitting a `REVIEW_REQUESTED` event
- Supports iterative review-fix cycles naturally through state machine loops

**Approval Mode Integration:**

- Approval gates are state machine transition guards
- `APPROVAL_REQUIRED` events pause the state machine
- User response resumes with `APPROVAL_GRANTED` or `APPROVAL_DENIED` event
- Autonomous mode auto-grants

**Tradeoffs:**

| Pros                                                           | Cons                                                                     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Maximum flexibility — pipeline can adapt to runtime conditions | Most complex orchestrator implementation                                 |
| Natural support for iterative review-fix cycles                | State machine bugs can cause infinite loops or deadlocks                 |
| Event log provides complete audit trail                        | Event-driven model is unfamiliar — harder to debug and maintain          |
| Agents are truly decoupled — only coupled to event schemas     | More complex testing — state machine combinatorics                       |
| Supports future extensions without pipeline restructuring      | Overhead of event serialization/deserialization                          |
| Per-task verification with natural parallelism                 | Risk of over-engineering for a system that runs sequentially in practice |

**Risk Classification:** 🔴 — Highest architectural complexity. State machine bugs are hard to diagnose. The system runs in GitHub Copilot's agent framework which is fundamentally sequential (one orchestrator, one conversation) — event-driven flexibility may not be realizable. Over-engineering risk is high.

---

## Architectural Direction Comparison Matrix

| Dimension               | A: Lean Pipeline | B: Forge-Plus | C: Anvil-Core         | D: Hybrid Typed     | E: Event-Driven |
| ----------------------- | ---------------- | ------------- | --------------------- | ------------------- | --------------- |
| Agent count             | 6–8              | 12–14         | 4–6                   | 8–10                | 8–12            |
| Migration risk          | Medium           | Low           | High                  | Medium              | High            |
| Orchestrator complexity | Low              | Medium        | Low                   | Medium–High         | High            |
| Agent complexity        | Medium           | Low           | High                  | Medium              | Low             |
| Schema enforcement      | Medium           | Low–Medium    | SQL-native            | High                | Medium–High     |
| Verification rigor      | High             | Medium–High   | Highest               | High                | High            |
| Parallelism potential   | High             | High          | Low (per-task serial) | High                | High            |
| Memory overhead         | Minimal          | Medium        | Minimal (SQL)         | Low                 | Low (event log) |
| Environment dependency  | Low              | Low           | High (SQLite)         | Low (YAML fallback) | Low             |
| Extensibility           | Medium           | Medium        | Low                   | High                | Highest         |
| Risk rating             | 🟡               | 🟢            | 🔴                    | 🟡                  | 🔴              |

---

## Common Requirements

The following requirements apply to **ALL** architectural directions. Any chosen direction MUST satisfy every requirement in this section.

### CR-1: Deterministic Pipeline Execution

The system MUST execute a deterministic pipeline: given the same input (initial request + codebase state), the pipeline MUST follow the same sequence of steps and dispatch the same agents in the same order. Non-determinism from LLM outputs is acceptable; non-determinism from pipeline routing is not.

### CR-2: Three-State Completion Contracts

Every agent MUST return exactly one of:

- `DONE:` — success, proceed to next step
- `NEEDS_REVISION:` — addressable issues found; orchestrator routes to appropriate agent
- `ERROR:` — unrecoverable failure; retry once per Global Rule 4, then escalate

The orchestrator MUST NOT return `NEEDS_REVISION` — it handles all revision routing internally.

### CR-3: Parallel Subagent Dispatch

The system MUST support parallel subagent dispatch via `runSubagent` with a concurrency cap of 4 concurrent subagents per dispatch wave. Sub-wave partitioning MUST be used when tasks exceed the concurrency cap.

### CR-4: Adversarial Multi-Model Review

For Large OR 🔴 changes:

- Three reviewers dispatched in parallel using: `gpt-5.3-codex`, `gemini-3-pro-preview`, `claude-opus-4.6`
- Same review prompt, different models
- Each verdict recorded independently (SQL INSERT or typed schema)
- Must have ≥3 review records before proceeding

For Standard changes:

- One reviewer using `gpt-5.3-codex`
- Must have ≥1 review record before proceeding

Maximum 2 adversarial rounds. After the second round, remaining findings presented as known issues with `Confidence: Low`.

### CR-5: Evidence Gating

The pipeline MUST enforce evidence gates at critical transitions. Progression MUST be blocked if evidence is insufficient:

- Post-implementation: build passes + test results recorded (not self-reported)
- Post-verification: minimum verification signal count met (2 for standard, 3 for Large/🔴)
- Post-review: review verdict count met (1 for standard, 3 for Large/🔴)

Evidence gates MUST be machine-checkable (SQL COUNT or typed record count), not prose assertions.

### CR-6: Risk Classification

Every proposed file change MUST be classified:

- 🟢 Additive: new tests, docs, config, comments
- 🟡 Business logic: modifying logic, function signatures, DB queries, UI state
- 🔴 Critical: auth/crypto/payments, data deletion, schema migrations, concurrency, public API

Any 🔴 file MUST escalate the associated task to Large verification depth. Risk classification MUST be recorded and MUST drive verification depth and reviewer count.

### CR-7: Justification Scoring

Architectural decisions MUST include justification scores. Each decision MUST document: the alternatives considered, the selection rationale, and a confidence level (High/Medium/Low) with a concrete definition of what the confidence rating means.

### CR-8: Structured Multiple-Choice Approval Mode

The system MUST support two modes:

- **Autonomous:** Pipeline runs without user interaction. Approval gates auto-proceed.
- **Interactive:** At designated approval gates, the system presents structured multiple-choice prompts with: a clear question, ≥2 labeled options, a description of each option's implications.

### CR-9: No "Pending Step" Pattern

The system MUST NOT implement any "pending step" pattern. Every pipeline step either executes or is skipped based on deterministic conditions. No steps wait indefinitely for external input outside of designated approval gates.

### CR-10: Output Directory Structure

All generated agent definitions and prompts MUST be written to `NewAgents/.github/`. A README with a final Mermaid pipeline diagram MUST be created at `NewAgents/`.

### CR-11: Self-Verification Before Return

Every agent MUST verify its own output against its input requirements before returning its completion contract. Self-verification failures MUST be addressed before returning, or reported as part of the completion signal.

### CR-12: Anti-Drift Anchors

Every agent definition MUST include an anti-drift anchor: an identity reminder at the end of the agent definition that prevents context window drift in long conversations.

### CR-13: Unified Severity Taxonomy

The system MUST use a single severity taxonomy across all agents and pipeline stages:

- **Blocker** — prevents pipeline progression
- **Critical** — must be addressed before pipeline completion
- **Major** — should be addressed, but pipeline can proceed
- **Minor** — informational, no action required

No agent may use a different severity vocabulary (e.g., no separate High/Medium/Low scale for one cluster and Blocker/Major/Minor for another).

### CR-14: Pushback System

The system MUST include a pushback mechanism (at the specification or pre-pipeline stage) that evaluates request quality before execution. Pushback MUST surface:

- Implementation concerns: tech debt, duplication, complexity, simpler alternatives
- Requirements concerns: conflicts, symptom vs. root cause, dangerous edge cases, implicit assumptions

Pushback MUST be presented with structured choices and MUST NOT proceed until acknowledged (in interactive mode) or logged (in autonomous mode).

### CR-15: Bounded Retry Budget

The system MUST enforce bounded retries at all levels:

- Agent-level: max 2 internal retries for transient errors
- Orchestrator-level: max 1 retry per agent dispatch
- Revision loops: max 3 iterations for verification-replan cycles, max 1 for design revision
- Adversarial review: max 2 rounds

These compose to a worst-case bound. Deterministic failures MUST NOT be retried.

---

## Functional Requirements

### FR-1: Agent Lifecycle Management

- **FR-1.1:** The orchestrator MUST initialize all pipeline state before dispatching any agent.
- **FR-1.2:** The orchestrator MUST dispatch agents via `runSubagent` with appropriate parameters (agent type, model selection for adversarial review, input file paths).
- **FR-1.3:** The orchestrator MUST capture and process every agent's completion contract (DONE/NEEDS_REVISION/ERROR).
- **FR-1.4:** The orchestrator MUST route NEEDS_REVISION results to the appropriate upstream agent based on a defined routing table.
- **FR-1.5:** The orchestrator MUST terminate the pipeline on unrecoverable ERROR after retry exhaustion.
- **FR-1.6:** The orchestrator tool restrictions MUST be enforced: allowed tools are `agent`, `agent/runSubagent`, `memory`, `read_file`, `list_dir`. All file creation/modification MUST be delegated to subagents.

### FR-2: Pipeline Execution

- **FR-2.1:** The pipeline MUST support both autonomous and interactive execution modes.
- **FR-2.2:** In autonomous mode, the pipeline MUST run to completion without user prompts (approval gates auto-proceed).
- **FR-2.3:** In interactive mode, the pipeline MUST pause at designated approval gates and present structured multiple-choice options.
- **FR-2.4:** The pipeline MUST enforce step ordering — no step may execute before its prerequisites are complete.
- **FR-2.5:** The pipeline MUST support parallel dispatch within steps (e.g., 4 researchers in parallel, 3 adversarial reviewers in parallel).
- **FR-2.6:** Parallel dispatch MUST use the reusable dispatch pattern: dispatch ≤4 → wait for all → retry errors once → evaluate results (≥2 outputs required for clusters).

### FR-3: State Management

- **FR-3.1:** Pipeline state MUST be persisted in a structured format (typed YAML, SQL, or equivalent) — not unstructured prose.
- **FR-3.2:** Verification evidence MUST be persisted in a machine-queryable format (SQL preferred, structured YAML fallback).
- **FR-3.3:** State MUST survive individual agent failures — a failed agent MUST NOT corrupt pipeline state.
- **FR-3.4:** State reads by the orchestrator MUST NOT require a separate subagent dispatch (the orchestrator reads state directly).
- **FR-3.5:** Cross-session knowledge MUST be stored via VS Code `store_memory` for facts that should persist across pipeline runs.

### FR-4: Verification and Quality Assurance

- **FR-4.1:** Verification MUST use a tiered cascade: Tier 1 (IDE diagnostics, syntax) → Tier 2 (build, type check, lint, tests) → Tier 3 (import/load, smoke execution).
- **FR-4.2:** Verification MUST capture baseline state before implementation changes.
- **FR-4.3:** Verification MUST record every check as a structured evidence record (SQL INSERT or typed schema entry).
- **FR-4.4:** Verification MUST enforce minimum signal counts: ≥2 for standard tasks, ≥3 for Large/🔴 tasks.
- **FR-4.5:** Zero verification MUST NEVER be acceptable, regardless of task size.
- **FR-4.6:** Regressions MUST be detected by comparing baseline to post-implementation evidence.
- **FR-4.7:** After 2 failed fix attempts for a verification check, the system MUST revert changes for that check and record the failure.

### FR-5: Adversarial Review Mechanism

- **FR-5.1:** Adversarial review MUST be applied at the design stage (reviewing the design document).
- **FR-5.2:** Adversarial review MUST be applied at the code stage (reviewing implemented code via `git diff --staged`).
- **FR-5.3:** Reviewer count MUST be determined by risk classification: 1 for standard, 3 for Large/🔴.
- **FR-5.4:** For 3-reviewer dispatch, each reviewer MUST use a different LLM model (gpt-5.3-codex, gemini-3-pro-preview, claude-opus-4.6).
- **FR-5.5:** All review verdicts MUST be recorded as structured evidence (SQL INSERT or typed schema).
- **FR-5.6:** Evidence gate MUST verify verdict count before proceeding: ≥1 for standard, ≥3 for Large/🔴.
- **FR-5.7:** If real issues are found, the system MUST fix and re-review (max 2 rounds).
- **FR-5.8:** After 2 rounds, remaining findings MUST be presented as known issues with `Confidence: Low`.

### FR-6: Memory and Inter-Agent Communication

- **FR-6.1:** Agent outputs MUST be structured with at minimum: status (completion contract), severity (unified taxonomy), and key findings.
- **FR-6.2:** The system MUST support a memory-first reading pattern: agents read summaries/indexes before full artifacts.
- **FR-6.3:** The system MUST NOT require a separate "memory merge subagent" — structured outputs should be directly readable by the orchestrator.
- **FR-6.4:** Architectural decisions MUST be recorded in an append-only decision log with justification.

### FR-7: Error Handling and Recovery

- **FR-7.1:** The system MUST categorize errors as transient (retry) or deterministic (do not retry).
- **FR-7.2:** Transient errors (network timeout, tool unavailable) MUST be retried up to 2 times.
- **FR-7.3:** Deterministic errors (file not found, permission denied, schema violation) MUST NOT be retried.
- **FR-7.4:** Non-blocking agents (knowledge capture, post-mortem) MUST NOT cause pipeline failure on ERROR.
- **FR-7.5:** Critical security findings MUST halt the pipeline regardless of retry budget.
- **FR-7.6:** Partial pipeline state MUST be preserved on failure for potential resumption or diagnosis.

### FR-8: Approval Mode Interactions

- **FR-8.1:** The system MUST present approval requests as structured multiple-choice with: question, ≥2 options, descriptions.
- **FR-8.2:** Approval gates MUST be placed at: post-research (approve direction) and post-planning (approve tasks), at minimum.
- **FR-8.3:** In autonomous mode, approval gates MUST auto-select a default option and log the auto-selection.
- **FR-8.4:** Approval responses MUST be recorded in pipeline state for auditability.

### FR-9: Output File Structure

- **FR-9.1:** Agent definitions MUST be written to `NewAgents/.github/agents/`.
- **FR-9.2:** Prompt/entry-point files MUST be written to `NewAgents/.github/prompts/`.
- **FR-9.3:** A README MUST be created at `NewAgents/README.md` with a Mermaid diagram of the final pipeline.
- **FR-9.4:** Reference documents (dispatch patterns, evaluation schema, etc.) MUST be co-located with agent definitions.

---

## Non-Functional Requirements

### NFR-1: Determinism

Given the same initial request and codebase state, the pipeline MUST follow the same agent dispatch sequence and routing decisions. LLM output non-determinism is inherent and accepted; pipeline routing non-determinism is not.

### NFR-2: Auditability

Every pipeline decision MUST be traceable: which agent produced which finding, what severity was assigned, what routing decision followed. The complete decision chain from input to output MUST be reconstructable from pipeline state and evidence records.

### NFR-3: Scalability

The system MUST handle complex multi-file features (10+ tasks, 50+ files) without exceeding context window limits for any single agent. Task decomposition and wave-based execution MUST prevent combinatorial context explosion.

### NFR-4: Robustness

The system MUST handle: individual agent failures without pipeline corruption, missing or malformed agent outputs (schema validation), tool unavailability (graceful degradation), and partial pipeline completion (preservable state).

### NFR-5: Context Efficiency

The system MUST minimize context window usage: agents MUST read summaries/indexes before full artifacts, agent definition prompts MUST be as concise as possible, and unnecessary intermediate artifacts MUST NOT be generated.

### NFR-6: Maintainability

Each agent MUST be independently updateable: changing one agent's behavior MUST NOT require changing other agents, provided the agent's output schema remains compatible. Agent definitions MUST be modular with clear input/output contracts.

### NFR-7: Security

Security findings at any pipeline stage MUST be able to halt the pipeline. Security analysis MUST be integrated into the adversarial review phase (not a separate optional step). The security blocker policy (security finding → pipeline ERROR) MUST be enforced regardless of architectural direction.

---

## Constraints & Assumptions

### Constraints

- **C-1:** Must work within VS Code GitHub Copilot agent framework — agents are dispatched via `runSubagent`, not as standalone processes.
- **C-2:** Must use `runSubagent` for all parallel dispatch — no external process management.
- **C-3:** Three specific model types for adversarial review: `gpt-5.3-codex`, `gemini-3-pro-preview`, `claude-opus-4.6`. Model routing depends on platform support (see A-2).
- **C-4:** No "pending step" pattern — every step either executes or is skipped deterministically.
- **C-5:** System correctness takes priority over human readability of intermediate artifacts.
- **C-6:** Maximum 4 concurrent subagent invocations per dispatch wave.
- **C-7:** Output destination: `NewAgents/.github/` for agent definitions and prompts, `NewAgents/` for README.
- **C-8:** Agent definitions are Markdown files (`.agent.md`) — this is the VS Code Copilot agent file format.

### Assumptions

- **A-1:** SQLite is available in the agent runtime environment for the verification ledger. If not, a structured YAML fallback MUST be provided.
- **A-2:** The VS Code Copilot runtime supports dispatching `code-review` subagents to specific LLM models (model routing). This is platform-dependent and MUST be verified during implementation.
- **A-3:** The `ask_user` tool is available for interactive approval gates.
- **A-4:** The `ide-get_diagnostics` tool is available for IDE diagnostic checks.
- **A-5:** Git operations are available via `run_in_terminal` for staging, diffing, committing, and reverting.
- **A-6:** Both Forge and Anvil source files remain available for reference during implementation.
- **A-7:** The agent framework supports 419+ line agent definitions (Anvil is 419 lines, Forge orchestrator is 544 lines).

---

## Acceptance Criteria

Each criterion is testable with a clear pass/fail definition.

### AC-1: Architectural Completeness

**Pass:** The design document selects (or synthesizes from) one of the 5 directions and provides complete specifications for: every agent's role, every pipeline step, every dispatch pattern, every communication schema, every evidence gate, and every approval gate.
**Fail:** Any of the above elements is missing or undefined.

### AC-2: Agent Count Within Range

**Pass:** The final agent count (unique definitions, not instances) is ≤15 and ≥4.
**Fail:** Agent count exceeds 15 (over-engineering) or is below 4 (insufficient specialization).

### AC-3: Completion Contracts Universal

**Pass:** Every agent definition includes a completion contract section specifying which of the three states (DONE/NEEDS_REVISION/ERROR) it can return, with routing rules for each.
**Fail:** Any agent lacks a completion contract or uses non-standard states.

### AC-4: Adversarial Review Functional

**Pass:** The system dispatches 3 different-model reviewers in parallel for Large/🔴 changes, records all verdicts as structured evidence, enforces the COUNT gate (≥3 verdicts), and limits to 2 adversarial rounds.
**Fail:** Any of: wrong reviewer count, same-model reviewers, unrecorded verdicts, missing COUNT gate, or unbounded rounds.

### AC-5: Evidence Gating Enforced

**Pass:** Evidence gates exist at post-implementation and post-review transitions. Gates are machine-checkable (COUNT query or typed record count). Pipeline progression is blocked when evidence is insufficient.
**Fail:** Gates are prose-based, gates are missing at required transitions, or pipeline proceeds despite insufficient evidence.

### AC-6: Risk Classification Integrated

**Pass:** Every file change is classified 🟢/🟡/🔴. Risk classification drives verification depth (standard vs. Large). 🔴 files escalate to Large verification with 3 reviewers. Classification is recorded in pipeline state.
**Fail:** Classification is missing, does not drive verification depth, or 🔴 files are not escalated.

### AC-7: Unified Severity Taxonomy

**Pass:** All agents use the same 4-level severity taxonomy (Blocker/Critical/Major/Minor). No agent uses an alternative vocabulary.
**Fail:** Any agent uses a different severity vocabulary.

### AC-8: Typed Communication

**Pass:** Agent outputs use structured formats (YAML, SQL, or equivalent) with defined fields. The orchestrator can read agent outputs programmatically without parsing prose.
**Fail:** Agent communication relies on unstructured Markdown prose for routing decisions.

### AC-9: Deterministic Routing

**Pass:** Given a fixed set of agent outputs, the orchestrator always makes the same routing decision. Routing logic is documented as a deterministic decision table.
**Fail:** Routing depends on order of reads, timing, or undocumented heuristics.

### AC-10: Approval Mode Working

**Pass:** In interactive mode, the system presents structured multiple-choice prompts at designated gates. In autonomous mode, gates auto-proceed with logged selections.
**Fail:** Prompts are free-form, gates are missing, or autonomous mode blocks.

### AC-11: Pipeline Completes End-to-End

**Pass:** The system can execute from initial request to final output (agent definitions + README + Mermaid diagram) without manual intervention in autonomous mode.
**Fail:** Pipeline hangs, crashes, or requires manual intervention outside of designated approval gates.

### AC-12: Self-Verification Implemented

**Pass:** Every agent definition includes a self-verification step that checks its output against its input requirements before returning.
**Fail:** Any agent lacks self-verification.

### AC-13: Pushback System Functional

**Pass:** The system evaluates request quality, surfaces implementation and requirements concerns with structured choices, and blocks execution until acknowledged (interactive) or logged (autonomous).
**Fail:** Pushback is missing, unstructured, or does not block execution.

### AC-14: Output Structure Correct

**Pass:** Agent definitions are in `NewAgents/.github/agents/`, prompts in `NewAgents/.github/prompts/`, README with Mermaid diagram in `NewAgents/README.md`.
**Fail:** Files are in wrong locations or README is missing.

### AC-15: Bounded Retries Enforced

**Pass:** All retry loops have documented maximum iteration counts. No infinite loop is possible in any revision/retry path.
**Fail:** Any revision or retry path lacks a documented maximum or can loop indefinitely.

---

## Edge Cases & Error Handling

### EC-1: Adversarial Review Models Disagree

- **Input/Condition:** Three adversarial reviewers return conflicting verdicts (e.g., 2 approve, 1 finds critical issues).
- **Expected Behavior:** Any security blocker finding from ANY model escalates to pipeline ERROR regardless of majority. For non-security findings: majority rules — if 2/3 approve, proceed with the dissenting findings logged as known issues. If 2/3 find issues, route to NEEDS_REVISION.
- **Severity if Missed:** Critical — conflicting reviews left unresolved could allow flawed code to proceed or unnecessarily block valid code.

### EC-2: Approval Mode Interaction Timeout

- **Input/Condition:** Interactive mode is active, but the user does not respond to an approval prompt within a reasonable time (e.g., VS Code session times out, user walks away).
- **Expected Behavior:** The pipeline MUST NOT proceed automatically on timeout. Pipeline state MUST be preserved so the user can resume. The orchestrator MUST log the timeout and present the same approval prompt on resume.
- **Severity if Missed:** High — auto-proceeding on timeout defeats the purpose of interactive approval; losing state forces full restart.

### EC-3: Subagent Produces Invalid Output Format

- **Input/Condition:** A subagent returns output that does not conform to its defined typed schema (missing fields, wrong types, malformed YAML/SQL).
- **Expected Behavior:** The orchestrator MUST detect the schema violation, retry the agent dispatch once (treating it as a transient error). If the second attempt also fails schema validation, the orchestrator MUST treat it as ERROR and follow the error escalation path.
- **Severity if Missed:** Critical — unvalidated outputs propagate through the pipeline, causing downstream agent failures or incorrect routing decisions.

### EC-4: SQL Ledger Becomes Corrupted or Unavailable

- **Input/Condition:** The SQLite database becomes corrupted mid-pipeline (disk error, concurrent access), or SQLite is not available in the runtime environment.
- **Expected Behavior:** If SQL becomes unavailable mid-pipeline: fall back to structured YAML evidence records for remaining verification steps. Log the fallback. If SQL was never available: use YAML throughout and log at pipeline start. Previously recorded SQL evidence MUST NOT be lost — attempt to read existing records before falling back.
- **Severity if Missed:** High — loss of verification evidence undermines the entire evidence-gating strategy. Silent fallback without logging hides the degradation.

### EC-5: Partial Failure Mid-Pipeline

- **Input/Condition:** The pipeline fails at Step 5 (implementation) after research, spec, design, and planning have completed successfully.
- **Expected Behavior:** All completed step outputs MUST be preserved on disk. The pipeline MUST record the failure point in pipeline state. On restart, the orchestrator MUST detect the partial state and offer to resume from the failure point (not restart from scratch). Completed steps MUST NOT be re-executed.
- **Severity if Missed:** Critical — losing completed work forces O(hours) of re-execution. No resumability makes the system fragile for complex features.

### EC-6: Researcher Returns Insufficient Data

- **Input/Condition:** Fewer than 2 of 4 parallel researchers return DONE (e.g., 2 return ERROR, 1 returns incomplete data).
- **Expected Behavior:** The pipeline MUST treat this as cluster ERROR per Pattern A rules (≥2 outputs required). Retry each failed researcher once. If still <2 after retries, present available findings to the user with a warning that research is incomplete, and offer to proceed or abort (in interactive mode). In autonomous mode, proceed with available findings and log the gap.
- **Severity if Missed:** Medium — proceeding with insufficient research may produce a weaker spec, but the pipeline can still function. Failing to log the gap hides the quality reduction.

### EC-7: Verification Replan Loop Exhaustion

- **Input/Condition:** The verification-replan loop (Pattern C) reaches its maximum iteration count (3) without achieving DONE.
- **Expected Behavior:** The pipeline MUST proceed with the verification findings documented in the artifacts. The completion contract MUST include a warning that verification is incomplete. The evidence bundle MUST separate passed and failed checks. Confidence MUST be rated Low.
- **Severity if Missed:** Medium — blocking indefinitely is worse than proceeding with documented issues, but silently proceeding without Low confidence rating misleads users.

### EC-8: Agent Context Window Exceeded

- **Input/Condition:** A complex feature generates enough upstream artifacts that a downstream agent's context window is exceeded.
- **Expected Behavior:** The system MUST use the memory-first reading pattern (summaries/indexes before full artifacts). Agents MUST read selectively using artifact indexes. If context is still exceeded, the agent MUST request the orchestrator to provide a summarized context (reduced artifacts). This MUST NOT silently truncate input.
- **Severity if Missed:** High — context truncation causes agents to miss critical information, producing incomplete or incorrect outputs.

### EC-9: Model Routing Unavailable

- **Input/Condition:** The runtime does not support dispatching `code-review` subagents to specific LLM models (model routing is platform-dependent).
- **Expected Behavior:** Fall back to same-model review with different prompts (emphasizing different review concerns per reviewer). Log the fallback. Reduce the quality guarantee — present review results with a warning that genuine multi-model diversity was not achieved.
- **Severity if Missed:** Medium — same-model review is weaker than multi-model but still provides value. Failing without fallback is worse than degraded operation.

### EC-10: Concurrent Subagent Limit Hit

- **Input/Condition:** A dispatch step requires more than 4 concurrent subagents (e.g., 8 implementation tasks ready simultaneously).
- **Expected Behavior:** Partition into sub-waves of ≤4. Execute sub-waves sequentially. Each sub-wave's outputs are available to subsequent sub-waves. State is updated between sub-waves.
- **Severity if Missed:** Low — the platform may enforce its own concurrency limit, but without explicit sub-wave partitioning, dispatch may fail or behave unpredictably.

---

## User Stories / Flows

### US-1: Standard Feature Development (Autonomous Mode)

1. User invokes the pipeline with a feature request
2. System runs pushback evaluation — no concerns raised
3. 4 researchers investigate codebase in parallel
4. Spec agent produces feature specification
5. Designer produces technical design
6. 1 adversarial reviewer evaluates design (standard risk) → DONE
7. Planner decomposes into tasks with risk classification (all 🟢/🟡)
8. Implementer executes tasks in waves (≤4 concurrent), with baseline capture and verification per task
9. Unified verification confirms build, tests, acceptance criteria with SQL evidence
10. 1 adversarial code reviewer evaluates → DONE
11. Knowledge captured, pipeline completes
12. Output: agent definitions in `NewAgents/.github/`, README with Mermaid diagram

### US-2: High-Risk Feature (Interactive Mode)

1. User invokes pipeline with a feature touching auth/crypto files
2. System runs pushback evaluation — flags 🔴 files, recommends Large scope
3. [Approval gate] User confirms proceeding as Large
4. 4 researchers investigate in parallel
5. Spec agent produces feature specification
6. Designer produces technical design
7. 3 adversarial reviewers (multi-model) evaluate design → 1 finds security concern → NEEDS_REVISION
8. Designer revises → 3 reviewers re-evaluate → all approve
9. [Approval gate] User approves planning direction
10. Planner decomposes with risk classifications (some 🔴)
11. Implementer executes with full verification cascade per task
12. 3 adversarial code reviewers per task → findings fixed
13. Knowledge captured, pipeline completes with High confidence

### US-3: Pipeline Failure and Recovery

1. User invokes pipeline. Research + spec + design complete successfully
2. Implementation fails on task 3 of 6 due to build error after 2 fix attempts
3. System reverts task 3 changes, records failure in evidence ledger
4. System preserves state: tasks 1–2 complete, task 3 failed, tasks 4–6 pending
5. System presents failure with: completed work, failure evidence, and option to resume or abort
6. User resumes — system picks up from task 3 (replanned), executes tasks 3–6

---

## Test Scenarios

### TS-1: Completion Contract Compliance

- **Verifies:** AC-3
- **Test:** For each agent definition, verify it includes a completion contract section specifying DONE/NEEDS_REVISION/ERROR behavior.

### TS-2: Adversarial Review — Standard Risk

- **Verifies:** AC-4
- **Test:** Simulate a standard-risk feature. Verify exactly 1 reviewer is dispatched at design review and code review stages. Verify verdict is recorded as structured evidence. Verify COUNT gate passes with ≥1.

### TS-3: Adversarial Review — High Risk (🔴)

- **Verifies:** AC-4
- **Test:** Simulate a feature with 🔴 files. Verify 3 different-model reviewers dispatched in parallel. Verify all 3 verdicts recorded. Verify COUNT gate requires ≥3.

### TS-4: Evidence Gating Block

- **Verifies:** AC-5
- **Test:** Simulate a verification step that produces insufficient evidence (e.g., 1 signal when 2 required). Verify the pipeline blocks at the gate and does not proceed.

### TS-5: Risk Classification Escalation

- **Verifies:** AC-6
- **Test:** Simulate a task that modifies both 🟢 and 🔴 files. Verify the entire task is escalated to Large. Verify 3 reviewers are dispatched.

### TS-6: Unified Severity Taxonomy

- **Verifies:** AC-7
- **Test:** Inspect every agent definition. Verify all severity references use Blocker/Critical/Major/Minor vocabulary.

### TS-7: Typed Output Schema

- **Verifies:** AC-8
- **Test:** For each agent, verify the output section defines a structured schema (YAML or SQL) with named fields, types, and constraints.

### TS-8: Deterministic Routing

- **Verifies:** AC-9
- **Test:** Provide the orchestrator with a fixed set of mock agent outputs. Run routing logic twice. Verify identical routing decisions both times.

### TS-9: Approval Mode — Interactive

- **Verifies:** AC-10
- **Test:** In interactive mode, verify each approval gate presents structured multiple-choice options. Verify pipeline pauses until response. Verify response is recorded.

### TS-10: Approval Mode — Autonomous

- **Verifies:** AC-10
- **Test:** In autonomous mode, verify approval gates auto-proceed. Verify auto-selection is logged.

### TS-11: End-to-End Pipeline

- **Verifies:** AC-11
- **Test:** Run the complete pipeline in autonomous mode on a sample feature request. Verify all output files are generated in correct locations.

### TS-12: Self-Verification

- **Verifies:** AC-12
- **Test:** For each agent definition, verify it includes a self-verification step in its workflow.

### TS-13: Pushback System

- **Verifies:** AC-13
- **Test:** Submit a request with known issues (scope too large, conflicting requirements). Verify pushback is raised with structured choices.

### TS-14: Output File Locations

- **Verifies:** AC-14
- **Test:** After pipeline completion, verify: `NewAgents/.github/agents/` contains agent definitions, `NewAgents/.github/prompts/` contains prompt files, `NewAgents/README.md` exists and contains a Mermaid diagram.

### TS-15: Retry Budget Enforcement

- **Verifies:** AC-15
- **Test:** Simulate an agent that always returns ERROR. Verify it is retried exactly once (not more). Verify the pipeline escalates after retry exhaustion.

### TS-16: Adversarial Disagreement Handling

- **Verifies:** EC-1
- **Test:** Simulate 3 reviewers: 2 approve, 1 finds a non-security issue. Verify pipeline proceeds with the dissenting finding logged. Then simulate 1 reviewer finding a security blocker with 2 approvals. Verify pipeline halts.

### TS-17: Schema Violation Handling

- **Verifies:** EC-3
- **Test:** Simulate an agent returning malformed output (missing required YAML field). Verify the orchestrator detects the violation, retries once, and escalates to ERROR on second failure.

### TS-18: SQL Fallback

- **Verifies:** EC-4
- **Test:** Simulate SQLite unavailability at pipeline start. Verify the system uses YAML evidence records throughout and logs the fallback.

### TS-19: Partial Pipeline Recovery

- **Verifies:** EC-5
- **Test:** Simulate a pipeline that completes steps 0–4 and fails at step 5. Verify all completed outputs are preserved. Restart the pipeline. Verify it detects partial state and offers to resume from step 5.

### TS-20: Model Routing Fallback

- **Verifies:** EC-9
- **Test:** Simulate model routing unavailability. Verify the system falls back to same-model-different-prompt review and logs the degradation.

---

## Dependencies & Risks

### Dependencies

| Dependency                                      | Type        | Impact if Unavailable                                |
| ----------------------------------------------- | ----------- | ---------------------------------------------------- |
| VS Code Copilot agent framework (`runSubagent`) | Platform    | System cannot function — hard dependency             |
| Model routing for adversarial review            | Platform    | Degrades to same-model review (EC-9 fallback)        |
| SQLite in agent runtime                         | Environment | Falls back to YAML (EC-4 fallback)                   |
| `ask_user` tool                                 | Platform    | Interactive mode non-functional — autonomous only    |
| `ide-get_diagnostics` tool                      | Platform    | Tier 1 verification degraded                         |
| Git operations via `run_in_terminal`            | Environment | Staging, diffing, committing, reverting unavailable  |
| Context7 MCP tools                              | External    | Documentation lookup degraded — non-critical         |
| Source Forge agent files (`.github/agents/`)    | Reference   | Implementation references — needed during build only |
| Source Anvil file (`Anvil/anvil.agent.md`)      | Reference   | Implementation references — needed during build only |

### Risks

| Risk                                                | Likelihood | Impact                                             | Mitigation                                                                         |
| --------------------------------------------------- | ---------- | -------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Model routing not supported by platform             | Medium     | High — adversarial review degrades to single-model | Same-model-different-prompt fallback (EC-9)                                        |
| SQLite unavailable in all target environments       | Low–Medium | Medium — verification evidence less robust         | YAML fallback with structured schemas (EC-4)                                       |
| Context window limits for complex features          | Medium     | High — agents produce incomplete outputs           | Memory-first reading, artifact indexes, selective reads (EC-8)                     |
| Typed schema design takes longer than expected      | Medium     | Medium — delays implementation                     | Start with minimal schemas, iterate                                                |
| Agent consolidation loses specialized capability    | Low        | Medium — quality regression in specific dimensions | Keep R-Knowledge separate; allow architectural directions to retain specialization |
| Orchestrator complexity exceeds context window      | Low–Medium | High — orchestrator itself fails                   | Keep orchestrator focused on routing only; delegate all work to subagents          |
| Concurrency behavior varies across VS Code versions | Low        | Low — sub-wave partitioning handles it             | Explicit concurrency cap enforcement (CR-3)                                        |
