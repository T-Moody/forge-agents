# Forge — Multi-Agent Development Pipeline

**A deterministic, documentation-driven agent pipeline for GitHub Copilot that takes raw feature requests and forges them into production-ready, verified implementations.**

Forge coordinates 22 specialized agents through a structured, repeatable workflow — parallel research, specification, design with cluster-based critical review, dependency-aware planning, TDD-driven wave-based implementation, cluster-based integration verification, and security-aware peer review — producing a full audit trail of artifacts at every stage.

---

## Why Forge?

Traditional AI coding assistants operate as a single agent trying to hold everything in context. This fails at scale:

| Problem                       | How Forge Solves It                                                                                                 |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Context overload**          | Each agent receives only the inputs it needs — never the full project state                                         |
| **No separation of concerns** | 22 specialized agents organized into clusters, each with a single responsibility and strict I/O contract            |
| **Sequential bottlenecks**    | Parallel research (4 concurrent agents) and wave-based parallel implementation (max 4 concurrent)                   |
| **No quality gates**          | TDD enforcement, critical-thinking design review, integration verification, and security-aware peer review          |
| **No audit trail**            | Every stage produces a markdown artifact under `docs/feature/<feature-slug>/`                                       |
| **Ad-hoc workflows**          | A deterministic, ordered pipeline with three-state completion contracts, retry logic, and structured error recovery |
| **Technology lock-in**        | Technology-agnostic verification — auto-detects build systems (npm, dotnet, cargo, maven, make, go, python, gradle) |
| **Security blind spots**      | Security thread across designer (threat modeling), implementer (no secrets/PII), and reviewer (OWASP scanning)      |

---

## Agent Roster

### Core Pipeline Agents

| Agent                  | File                                                           | Role                                                                                                                                             |
| ---------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `orchestrator`         | [orchestrator.agent.md](orchestrator.agent.md)                 | Coordinates the end-to-end workflow. Delegates all work via `runSubagent`. Never writes code or docs directly. Enforces concurrency cap (max 4). |
| `researcher`           | [researcher.agent.md](researcher.agent.md)                     | Investigates the codebase using hybrid retrieval (semantic search → grep → read). Runs ×4 in parallel across focus areas, then synthesizes.      |
| `spec`                 | [spec.agent.md](spec.agent.md)                                 | Produces a formal feature specification with structured edge cases, acceptance criteria, and self-verification.                                  |
| `designer`             | [designer.agent.md](designer.agent.md)                         | Creates a technical design document — architecture, data models, APIs, security considerations, failure & recovery analysis.                     |
| `planner`              | [planner.agent.md](planner.agent.md)                           | Builds dependency-aware plans with pre-mortem analysis, task size limits (max 3 files/500 lines), and per-task agent routing.                    |
| `implementer`          | [implementer.agent.md](implementer.agent.md)                   | Implements exactly one task using TDD — writes failing tests first, then minimal code, running `get_errors` after every edit.                    |
| `documentation-writer` | [documentation-writer.agent.md](documentation-writer.agent.md) | _(Optional)_ Generates API docs, architectural diagrams (Mermaid), and maintains code-documentation parity. Invoked via task routing.            |

### CT Cluster (Critical Thinking — Step 3b)

| Agent                | File                                                       | Role                                                                                   |
| -------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `ct-security`        | [ct-security.agent.md](ct-security.agent.md)               | Probes for security vulnerabilities and data exposure risks (parallel).                |
| `ct-scalability`     | [ct-scalability.agent.md](ct-scalability.agent.md)         | Probes for scalability bottlenecks and resource exhaustion risks (parallel).           |
| `ct-maintainability` | [ct-maintainability.agent.md](ct-maintainability.agent.md) | Probes for tight coupling, complexity, and integration risks (parallel).               |
| `ct-strategy`        | [ct-strategy.agent.md](ct-strategy.agent.md)               | Probes for strategic risks, scope risks, and fundamental approach validity (parallel). |
| `ct-aggregator`      | [ct-aggregator.agent.md](ct-aggregator.agent.md)           | Aggregates CT sub-agent findings into `design_critical_review.md`.                     |

### V Cluster (Verification — Step 6)

| Agent          | File                                           | Role                                                                  |
| -------------- | ---------------------------------------------- | --------------------------------------------------------------------- |
| `v-build`      | [v-build.agent.md](v-build.agent.md)           | Sequential gate: build system detection and compilation verification. |
| `v-tests`      | [v-tests.agent.md](v-tests.agent.md)           | Full test suite execution and analysis (parallel).                    |
| `v-tasks`      | [v-tasks.agent.md](v-tasks.agent.md)           | Per-task acceptance criteria verification (parallel).                 |
| `v-feature`    | [v-feature.agent.md](v-feature.agent.md)       | Feature-level acceptance criteria verification (parallel).            |
| `v-aggregator` | [v-aggregator.agent.md](v-aggregator.agent.md) | Aggregates V sub-agent outputs into `verifier.md`.                    |

### R Cluster (Review — Step 7)

| Agent          | File                                           | Role                                                                                 |
| -------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| `r-quality`    | [r-quality.agent.md](r-quality.agent.md)       | Code quality, readability, and maintainability review (parallel).                    |
| `r-security`   | [r-security.agent.md](r-security.agent.md)     | Security review — OWASP scanning, secrets/PII detection (parallel).                  |
| `r-testing`    | [r-testing.agent.md](r-testing.agent.md)       | Test quality and coverage adequacy review (parallel).                                |
| `r-knowledge`  | [r-knowledge.agent.md](r-knowledge.agent.md)   | Knowledge evolution — captures reusable patterns, maintains decision log (parallel). |
| `r-aggregator` | [r-aggregator.agent.md](r-aggregator.agent.md) | Aggregates R sub-agent findings into `review.md`.                                    |

### Deprecated

| Agent              | File                                                   | Role                                                                                                                      |
| ------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| `critical-thinker` | [critical-thinker.agent.md](critical-thinker.agent.md) | _(Deprecated)_ Replaced by the CT cluster (ct-security, ct-scalability, ct-maintainability, ct-strategy + ct-aggregator). |

---

## Workflow

The orchestrator drives the following deterministic pipeline:

```
 ┌──────────────────────────────────────────────────────────────┐
 │                     USER FEATURE REQUEST                     │
 └─────────────────────────────┬────────────────────────────────┘
                               ▼
                    ┌──────────────────────┐
                    │    0. SETUP          │
                    │  initial-request.md  │
                    └──────────┬───────────┘
                               ▼
 ┌──────────────┬──────────────┬──────────────┬─────────────────┐
 │  researcher  │  researcher  │  researcher  │   researcher    │
 │ (architecture│  (impact)    │(dependencies)│   (patterns)    │
 └──────┬───────┘──────┬───────┘──────┬───────┘───────┬─────────┘
        └─────────────┬┘──────────────┘───────────────┘
                          ▼
              ┌───────────────────────┐
              │  researcher (synth)   │
              │  → analysis.md        │
              └───────────┬───────────┘
                          ▼
              ┌───────────────────────┐
              │  2. SPECIFICATION     │
              │  spec → feature.md    │
              └───────────┬───────────┘
                          ▼
              ┌───────────────────────┐
              │  3. DESIGN            │
              │  designer → design.md │
              └───────────┬───────────┘
                          ▼
 ┌──────────────┬──────────────┬──────────────┬─────────────────┐
 │  ct-security │ct-scalability│ct-maintain.  │  ct-strategy    │
 └──────┬───────┘──────┬───────┘──────┬───────┘───────┬─────────┘
        └─────────────┬┘──────────────┘───────────────┘
                      ▼
              ┌───────────────────────┐
              │  3b. CT AGGREGATOR    │
              │  → design_critical_   │
              │    review.md          │
              └───────────┬───────────┘
                          ▼  (loop back to design if issues found)
              ┌───────────────────────┐
              │  4. PLANNING          │
              │  planner → plan.md    │
              │  (pre-mortem + tasks) │
              └───────────┬───────────┘
                          ▼
 ┌────────────────┬───────────────────┬────────────────────────┐
 │  implementer   │   implementer     │   doc-writer           │
 │  (task A, TDD) │   (task B, TDD)   │   (task C, if routed)  │
 └───────┬────────┘───────┬───────────┘───────┬────────────────┘
         └───────────────┬┘───────────────────┘
                         ▼        (max 4 concurrent per sub-wave)
              ┌───────────────────────┐
              │  6. V-BUILD (gate)    │
              └───────────┬───────────┘
                          ▼
     ┌────────────┬───────────────┬──────────────┐
     │  v-tests   │   v-tasks     │  v-feature   │
     └──────┬─────┘───────┬───────┘──────┬───────┘
            └────────────┬┘──────────────┘
                         ▼
              ┌───────────────────────┐
              │  V-AGGREGATOR         │
              │  → verifier.md        │
              └───────────┬───────────┘
                          ▼  (retry loop on failure, max 3)
 ┌──────────────┬──────────────┬──────────────┬─────────────────┐
 │  r-quality   │  r-security  │  r-testing   │  r-knowledge    │
 └──────┬───────┘──────┬───────┘──────┬───────┘───────┬─────────┘
        └─────────────┬┘──────────────┘───────────────┘
                      ▼
              ┌───────────────────────┐
              │  7. R-AGGREGATOR      │
              │  → review.md          │
              └───────────────────────┘
```

### Stages at a Glance

| #   | Stage          | Agent(s)                    | Parallelism                | Output                          |
| --- | -------------- | --------------------------- | -------------------------- | ------------------------------- |
| 0   | Setup          | orchestrator                | —                          | `initial-request.md`            |
| 1   | Research       | researcher ×4 + synthesis   | 4 concurrent               | `research/*.md` → `analysis.md` |
| 2   | Specification  | spec                        | —                          | `feature.md`                    |
| 3   | Design         | designer                    | —                          | `design.md`                     |
| 3b  | Design Review  | CT cluster ×4 + aggregator  | 4 concurrent (Pattern A)   | `design_critical_review.md`     |
| 4   | Planning       | planner                     | —                          | `plan.md` + `tasks/*.md`        |
| 5   | Implementation | implementer/doc-writer ×N   | Per-wave, max 4 concurrent | Code + tests                    |
| 6   | Verification   | V cluster (gate + ×3 + agg) | Pattern B + C              | `verifier.md`                   |
| 7   | Review         | R cluster ×4 + aggregator   | 4 concurrent (Pattern A)   | `review.md`                     |

---

## What's New in v2

Forge v2 incorporates the best patterns from the [Gem Team](https://github.com/mubaidr/gem-team) multi-agent framework while preserving Forge's core architecture:

| Improvement                      | Agent(s) Affected                   | Description                                                                                                                   |
| -------------------------------- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **TDD Enforcement**              | implementer, verifier, orchestrator | Implementers follow red-green-refactor: write failing tests → minimal code → verify. Verifier refocused to integration-level. |
| **Pre-Mortem Analysis**          | planner                             | Identifies failure scenarios per task (likelihood/impact/mitigation) before execution.                                        |
| **Task Size Limits**             | planner                             | Max 3 files, 500 lines, 2 dependencies, medium effort per task. Larger tasks auto-decomposed.                                 |
| **Concurrency Cap**              | orchestrator                        | Max 4 concurrent subagent invocations. Waves >4 tasks split into sub-waves.                                                   |
| **Security Thread**              | designer, implementer, reviewer     | Security considerations in design, no-secrets rules in implementation, OWASP scanning in review.                              |
| **Hybrid Retrieval**             | researcher                          | Prescribed strategy: semantic search → grep → merge/dedup → read file → verify existence.                                     |
| **Technology-Agnostic Verifier** | verifier                            | Auto-detects build system. No hardcoded project references.                                                                   |
| **Per-Task Agent Routing**       | planner, orchestrator               | Tasks can specify which agent executes them (e.g., `documentation-writer`).                                                   |
| **Three-State Completion**       | all agents                          | `DONE:` / `NEEDS_REVISION:` / `ERROR:` — allows revision routing without full failure.                                        |
| **Anti-Drift Anchors**           | all agents                          | Final instruction block in every agent prevents LLM mode-switching.                                                           |
| **Shared Operating Rules**       | all agents                          | Consistent context-reading, error-handling, and communication guidelines across all agents.                                   |
| **Optional Approval Gates**      | orchestrator, prompt                | `{{APPROVAL_MODE}}` enables human pauses after research and planning (experimental).                                          |
| **Self-Reflection**              | implementer, reviewer               | "Would a staff engineer approve this?" check before returning.                                                                |
| **Documentation Writer**         | _(new agent)_                       | Optional agent for API docs, Mermaid diagrams, and code-documentation parity.                                                 |
| **Critical-Thinker Bug Fixes**   | critical-thinker                    | Added missing completion contract and output file specification.                                                              |

---

## Architecture: Why Forge Uses Task-Routed Implementers

Forge and [Gem Team](https://github.com/mubaidr/gem-team) take fundamentally different approaches to dispatching work during the implementation phase:

- **Gem Team** runs different agent _types_ in parallel during execution — implementers, devops agents, chrome testers, reviewers, and documentation writers all execute concurrently as peers, each assigned to tasks matching their specialty via `plan.yaml`.
- **Forge** routes all execution-phase tasks through a small set of **task agents** (primarily `implementer`, with `documentation-writer` as an optional second) while keeping **verification** and **review** as dedicated post-implementation pipeline stages.

**Why Forge's approach is deliberate:**

1. **Separation of concerns over role multiplication.** When a reviewer runs _during_ implementation (Gem's model), it reviews individual tasks in isolation. Forge's reviewer runs _after_ all implementation completes, reviewing the full git diff — catching cross-task interactions, naming inconsistencies, and architectural drift that per-task reviews miss.

2. **Integration verification requires the complete picture.** Gem's inline verification (each agent self-verifies) catches unit-level issues but can miss integration failures when separately-implemented components don't compose correctly. Forge's dedicated verifier builds the entire project and runs the full test suite after each wave, providing a single authoritative integration checkpoint.

3. **TDD at implementation, not scattered across agents.** Forge's implementer now follows strict TDD (red-green-refactor), so unit-level verification happens at the right moment — during code writing. This eliminates the need for separate verification agents per task while still catching issues early.

4. **Fewer agent types = fewer coordination failures.** Each additional agent type in the execution phase adds another completion contract, another set of routing rules, and another failure mode for the orchestrator to handle. Forge keeps the execution phase simple (implementer + optional doc-writer) and invests complexity where it has higher leverage: pre-implementation (spec → design → critical review → pre-mortem planning) and post-implementation (integration verification → security-aware review).

5. **DevOps and browser testing are environment-specific.** Gem's `gem-devops` (Docker, K8s, CI/CD) and `gem-chrome-tester` (Chrome DevTools) require specific infrastructure that many projects don't have. Rather than building these into the core pipeline, Forge's per-task agent routing (`agent` field in task files) allows adding specialized agents when needed without burdening every project with unused capabilities.

6. **The reviewer is a quality gate, not a task worker.** Forge's reviewer operates as a final quality gate with tiered depth (Full/Standard/Lightweight), OWASP scanning, and the authority to send the orchestrator back to planning. Running review as an implementation-phase task (Gem's model) reduces it to just another worker, losing its architectural role as a gate.

In short: Forge front-loads intelligence (spec, design, critical review, pre-mortem) and back-loads verification (integration build, full test suite, security review), keeping the implementation phase focused on writing correct code with TDD. Gem distributes intelligence across more agent types during execution, which adds flexibility at the cost of coordination complexity and weaker integration verification.

---

## Artifact Structure

All workflow artifacts are written to a deterministic directory layout:

```
docs/feature/<feature-slug>/
├── initial-request.md          # Original user request (grounding doc)
├── research/
│   ├── architecture.md         # Codebase structure, patterns, conventions
│   ├── impact.md               # Affected files, modules, components
│   └── dependencies.md         # Module interactions, data flow, APIs
├── analysis.md                 # Synthesized research (single source of truth)
├── feature.md                  # Formal specification
├── design.md                   # Technical design (incl. security & failure analysis)
├── design_critical_review.md   # Adversarial review of design
├── plan.md                     # Dependency graph + execution waves + pre-mortem
├── tasks/
│   ├── 01-task-description.md  # Individual tasks (with agent routing, size limits)
│   ├── 02-task-description.md
│   └── ...
├── verifier.md                 # Integration build + test + per-task verification
├── review.md                   # Security-aware peer review findings
└── decisions.md                # Architectural decision log (accumulated across features)
```

---

## Key Design Decisions

### Deterministic Pipeline

Every feature follows the same ordered stages. The orchestrator does not improvise — it executes a fixed workflow with defined inputs and outputs at each step.

### Strict Agent Isolation

Agents receive only the files they need. The implementer never sees `plan.md`. The verifier and reviewer never modify code. The orchestrator never writes code or documentation.

### Documentation as First-Class Output

Every stage produces a markdown artifact. These artifacts serve as both the communication channel between agents and a full audit trail for humans.

### Critical Thinking Gate

Before planning begins, the critical-thinker agent challenges the design across structured risk categories (security, scalability, maintainability, backwards compatibility, edge cases, performance). This catches architectural issues before any code is written.

### TDD-Driven Implementation

Implementers follow strict red-green-refactor: write failing tests first, then minimal production code, running `get_errors` after every file edit. This catches issues at the earliest possible moment while maintaining separation from integration verification.

### Wave-Based Parallel Execution

The planner produces a dependency graph with task size limits (max 3 files, 500 lines, 2 dependencies per task). The orchestrator groups independent tasks into waves and dispatches up to 4 concurrent agents per sub-wave.

### Two-Layer Verification

Implementers verify their own tasks via unit-level TDD. The verifier then performs integration-level verification — full build, full test suite, cross-task acceptance criteria — catching issues that only surface when independently-implemented components combine.

### Security Thread

Security is distributed across three agents rather than concentrated in one:

- **Designer** — threat modeling, auth/authz patterns, data protection
- **Implementer** — never hardcode secrets/PII, fix vulnerabilities before returning
- **Reviewer** — OWASP Top 10 scanning, secrets/PII regex detection, tiered review depth

### Pre-Mortem Risk Analysis

Before execution begins, the planner identifies potential failure scenarios per task — with likelihood, impact, and mitigation strategies — catching execution risks at planning time rather than discovering them during verification.

### Error Recovery Loops

- If verification fails, only the failing tasks are re-planned and re-implemented (up to 3 retries).
- If the final review finds blocking concerns, the orchestrator loops back to planning.
- If a parallel research agent fails, it is retried once before the step is marked failed.
- `NEEDS_REVISION:` allows routing back to the originating agent without full failure/retry.

### Per-Task Agent Routing

The planner can assign tasks to specific agents via the `agent` field (default: `implementer`). This enables the optional `documentation-writer` for docs tasks without hardcoding specialized agents into the core pipeline.

---

## Getting Started

### Prerequisites

- [VS Code](https://code.visualstudio.com/) with [GitHub Copilot](https://github.com/features/copilot) (agent mode)
- A workspace with the Forge agent files at the root

### Installation

1. Clone this repository into your project root (or copy the `.agent.md` and `.prompt.md` files):

```bash
git clone https://github.com/YOUR_USERNAME/forge-agents.git .forge
# Copy agent files to project root
cp .forge/*.agent.md .forge/*.prompt.md .
```

Or simply copy all `.agent.md` and `.prompt.md` files into your workspace root.

2. Open VS Code with GitHub Copilot agent mode enabled.

3. Invoke the **Feature Workflow** prompt, or directly invoke the `orchestrator` agent with your feature request.

### Usage

**Quick Start — Feature Workflow Prompt:**

In Copilot Chat, type:

```
/feature-workflow Build a two-factor authentication system for user login
```

The format is `/feature-workflow` followed by your feature description. Copilot will invoke the Feature Workflow prompt, which delegates to the orchestrator agent to run the full pipeline.

**Direct Agent Invocation:**

Alternatively, invoke the `orchestrator` agent directly and describe the feature you want built.

**What Happens Next:**

The orchestrator will:

1. Capture your request in `initial-request.md`
2. Dispatch parallel research agents to investigate the codebase
3. Walk through specification → design → critical review → planning → implementation → verification → review
4. Produce all artifacts under `docs/feature/<feature-slug>/`
5. Return a summary of what was built and where to find the detailed artifacts

---

## Project Layout

```
forge-agents/
├── orchestrator.agent.md          # Workflow coordinator (concurrency cap, agent routing, approval gates)
├── researcher.agent.md            # Codebase investigation (hybrid retrieval, ×4 parallel + synthesis)
├── spec.agent.md                  # Feature specification (structured edge cases, self-verification)
├── designer.agent.md              # Technical design (security considerations, failure analysis)
├── planner.agent.md               # Task planning (pre-mortem, size limits, agent routing, mode detection)
├── implementer.agent.md           # TDD code implementation (red-green-refactor, security rules)
├── documentation-writer.agent.md  # Optional: API docs, diagrams, parity enforcement
├── critical-thinker.agent.md      # (Deprecated — replaced by CT cluster)
│
├── ct-security.agent.md           # CT cluster: security risk analysis
├── ct-scalability.agent.md        # CT cluster: scalability analysis
├── ct-maintainability.agent.md    # CT cluster: maintainability analysis
├── ct-strategy.agent.md           # CT cluster: strategic risk analysis
├── ct-aggregator.agent.md         # CT cluster: aggregates sub-agent outputs
│
├── v-build.agent.md               # V cluster: build gate (sequential)
├── v-tests.agent.md               # V cluster: test suite verification
├── v-tasks.agent.md               # V cluster: per-task acceptance criteria
├── v-feature.agent.md             # V cluster: feature-level requirements
├── v-aggregator.agent.md          # V cluster: aggregates sub-agent outputs
│
├── r-quality.agent.md             # R cluster: code quality review
├── r-security.agent.md            # R cluster: security review (OWASP)
├── r-testing.agent.md             # R cluster: test coverage review
├── r-knowledge.agent.md           # R cluster: knowledge evolution
├── r-aggregator.agent.md          # R cluster: aggregates sub-agent outputs
│
├── dispatch-patterns.md           # Reference: cluster dispatch patterns (A, B)
├── feature-workflow.prompt.md     # Entry-point prompt
└── README.md
```

---

## Completion Contracts

Every agent returns exactly one line when finished:

```
DONE: <summary>
NEEDS_REVISION: <what needs to change>
ERROR: <reason>
```

- **`DONE:`** — Task completed successfully. Orchestrator proceeds to next step.
- **`NEEDS_REVISION:`** — Output needs improvement but isn't a failure. Orchestrator routes back to the originating agent (not a full retry).
- **`ERROR:`** — Task failed. Orchestrator retries once, then reports failure.

Every agent includes an **anti-drift anchor** at the end of its definition — a final instruction block that reinforces the agent's role and behavioral constraints, exploiting LLM recency bias to prevent mode-switching during long conversations.

---

## License

This project is licensed under the [MIT License](LICENSE).

---

**Forge v2** — From requirement to tested, reviewed, and secured implementation, in one deterministic pipeline.
