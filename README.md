# Forge — Multi-Agent Development Pipeline

**A deterministic, documentation-driven agent pipeline for GitHub Copilot that takes raw feature requests and forges them into production-ready, verified implementations.**

Forge coordinates 9 specialized agents through a structured, repeatable workflow — parallel research, specification, design with critical review, dependency-aware planning, wave-based implementation, verification, and peer review — producing a full audit trail of artifacts at every stage.

---

## Why Forge?

Traditional AI coding assistants operate as a single agent trying to hold everything in context. This fails at scale:

| Problem                       | How Forge Solves It                                                                                   |
| ----------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Context overload**          | Each agent receives only the inputs it needs — never the full project state                           |
| **No separation of concerns** | 9 specialized agents, each with a single responsibility and strict I/O contract                       |
| **Sequential bottlenecks**    | Parallel research (3 concurrent agents) and wave-based parallel implementation                        |
| **No quality gates**          | Critical-thinking design review, automated verification, and peer code review built into the pipeline |
| **No audit trail**            | Every stage produces a markdown artifact under `docs/feature/<feature-slug>/`                         |
| **Ad-hoc workflows**          | A deterministic, ordered pipeline with retry logic and error recovery                                 |

---

## Agent Roster

| Agent              | File                                                   | Role                                                                                                                                               |
| ------------------ | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `orchestrator`     | [orchestrator.agent.md](orchestrator.agent.md)         | Coordinates the end-to-end workflow. Delegates all work via `runSubagent`. Never writes code or docs directly.                                     |
| `researcher`       | [researcher.agent.md](researcher.agent.md)             | Investigates the existing codebase — architecture, impact areas, and dependencies. Runs in parallel across focus areas, then synthesizes findings. |
| `spec`             | [spec.agent.md](spec.agent.md)                         | Produces a formal feature specification with functional/non-functional requirements, constraints, and acceptance criteria.                         |
| `designer`         | [designer.agent.md](designer.agent.md)                 | Creates a technical design document — architecture, data models, APIs, tradeoffs, and testing strategy.                                            |
| `critical-thinker` | [critical-thinker.agent.md](critical-thinker.agent.md) | Challenges assumptions in the design by asking probing questions. Forces reconsideration of risky decisions before implementation begins.          |
| `planner`          | [planner.agent.md](planner.agent.md)                   | Builds a dependency-aware implementation plan with execution waves for parallel dispatch. Produces `plan.md` and individual task files.            |
| `implementer`      | [implementer.agent.md](implementer.agent.md)           | Implements exactly one task per invocation. Writes code and tests but never builds or runs them.                                                   |
| `verifier`         | [verifier.agent.md](verifier.agent.md)                 | Sole owner of build and test execution. Builds the project, runs the test suite, and produces a per-task verification report.                      |
| `reviewer`         | [reviewer.agent.md](reviewer.agent.md)                 | Performs a peer-style review of the git diff — maintainability, naming, test quality, architectural alignment.                                     |

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
 ┌────────────────┬────────────────────────┬────────────────────┐
 │  researcher    │     researcher         │    researcher      │
 │  (architecture)│     (impact)           │    (dependencies)  │
 └───────┬────────┘────────┬───────────────┘────────┬───────────┘
         └────────────────┬┘────────────────────────┘
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
              ┌───────────────────────┐
              │  3b. CRITICAL REVIEW  │
              │  critical-thinker     │
              └───────────┬───────────┘
                          ▼  (loop back to design if issues found)
              ┌───────────────────────┐
              │  4. PLANNING          │
              │  planner → plan.md    │
              │           + tasks/*.md│
              └───────────┬───────────┘
                          ▼
 ┌────────────────┬───────────────────┬────────────────────────┐
 │  implementer   │   implementer     │   implementer          │
 │  (task A)      │   (task B)        │   (task C)             │
 └───────┬────────┘───────┬───────────┘───────┬────────────────┘
         └───────────────┬┘───────────────────┘
                         ▼
              ┌───────────────────────┐
              │  6. VERIFICATION      │
              │  verifier             │
              │  → verifier.md        │
              └───────────┬───────────┘
                          ▼  (retry loop on failure, max 3)
              ┌───────────────────────┐
              │  7. FINAL REVIEW      │
              │  reviewer → review.md │
              └───────────────────────┘
```

### Stages at a Glance

| #   | Stage          | Agent(s)                  | Parallelism       | Output                          |
| --- | -------------- | ------------------------- | ----------------- | ------------------------------- |
| 0   | Setup          | orchestrator              | —                 | `initial-request.md`            |
| 1   | Research       | researcher ×3 + synthesis | 3 concurrent      | `research/*.md` → `analysis.md` |
| 2   | Specification  | spec                      | —                 | `feature.md`                    |
| 3   | Design         | designer                  | —                 | `design.md`                     |
| 3b  | Design Review  | critical-thinker          | —                 | `design_critical_review.md`     |
| 4   | Planning       | planner                   | —                 | `plan.md` + `tasks/*.md`        |
| 5   | Implementation | implementer ×N            | Per-wave parallel | Code + tests                    |
| 6   | Verification   | verifier                  | —                 | `verifier.md`                   |
| 7   | Review         | reviewer                  | —                 | `review.md`                     |

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
├── design.md                   # Technical design
├── design_critical_review.md   # Critical thinking review of design
├── plan.md                     # Dependency graph + execution waves
├── tasks/
│   ├── 01-task-description.md  # Individual implementation tasks
│   ├── 02-task-description.md
│   └── ...
├── verifier.md                 # Build + test + per-task verification report
└── review.md                   # Peer review findings
```

---

## Key Design Decisions

### Deterministic Pipeline

Every feature follows the same ordered stages. The orchestrator does not improvise — it executes a fixed workflow with defined inputs and outputs at each step.

### Strict Agent Isolation

Agents receive only the files they need. The implementer never sees `plan.md`. The verifier never modifies code. The orchestrator never writes code or documentation.

### Documentation as First-Class Output

Every stage produces a markdown artifact. These artifacts serve as both the communication channel between agents and a full audit trail for humans.

### Critical Thinking Gate

Before planning begins, the critical-thinker agent challenges the design for assumptions, gaps, and risky decisions. This catches architectural issues before any code is written.

### Wave-Based Parallel Execution

The planner produces a dependency graph. The orchestrator groups independent tasks into waves and dispatches all tasks within a wave concurrently. Waves execute sequentially; tasks within a wave execute in parallel.

### Verification Ownership

Implementers write code and tests but never execute them. The verifier is the sole agent that builds the project and runs tests, ensuring a single consistent verification environment.

### Error Recovery Loops

- If verification fails, only the failing tasks are re-planned and re-implemented (up to 3 retries).
- If the final review finds blocking concerns, the orchestrator loops back to planning.
- If a parallel research agent fails, it is retried once before the step is marked failed.

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
├── orchestrator.agent.md          # Workflow coordinator
├── researcher.agent.md            # Codebase investigation
├── spec.agent.md                  # Feature specification
├── designer.agent.md              # Technical design
├── critical-thinker.agent.md      # Design review / assumption challenge
├── planner.agent.md               # Dependency-aware task planning
├── implementer.agent.md           # Single-task code implementation
├── verifier.agent.md              # Build, test, and verification
├── reviewer.agent.md              # Peer-style code review
├── feature-workflow.prompt.md     # Entry-point prompt
└── README.md
```

---

## Completion Contracts

Every agent returns exactly one line when finished:

```
DONE: <summary>
ERROR: <reason>
```

The orchestrator uses these signals to decide whether to proceed, retry, or loop back. This contract is enforced across all agents.

---

## License

This project is licensed under the [MIT License](LICENSE).

---

**Forge** — From requirement to reviewed implementation, in one deterministic pipeline.
