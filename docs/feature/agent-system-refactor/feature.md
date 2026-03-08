# Agent System Refactor — Feature Specification

## Title & Summary

**Feature:** Clean-Slate Multi-Agent Pipeline Rebuild

A full architectural rewrite of the multi-agent orchestration system, built fresh in a new directory. Reduces the system from 6,416 lines / 19 files / 9 agents / 10+ steps to ≤1,500 lines / ~10 files / 8 agents / 8 steps. Removes SQLite dependency, eliminates cross-file reference fragility, leverages VS Code native agent features, and enforces hard size limits to prevent recurring complexity spiral.

**Selected Direction:** C (Clean-Slate Rebuild)

---

## Background & Context

### Research Sources

- [research/architecture.yaml](research/architecture.yaml) — 19 findings: current system structure, 6 modern agent frameworks, DAG scheduling, VS Code agent format, simplification patterns
- [research/impact.yaml](research/impact.yaml) — Impact analysis: blast radius of changes, what works vs what's over-engineered, VS Code platform capabilities
- [research/dependencies.yaml](research/dependencies.yaml) — Dependency mapping: agent-to-reference chains, SQLite coupling, evidence gate chains, external tools
- [research/patterns.yaml](research/patterns.yaml) — Pattern analysis: completion contracts, adversarial review, TDD enforcement, industry comparisons
- [initial-request.md](initial-request.md) — Original user request

### Problem Statement

The current agent system has grown through 5 improvement cycles into a 6,416-line instruction-based system across 19 files. No improvement run has ever simplified the system — every run added complexity. The system is ~100x larger than GitHub's recommended instruction size and exceeds every comparable industry system. Key issues:

1. **Complexity spiral**: Each pipeline run adds more rules, checks, and infrastructure. None simplify.
2. **SQLite over-engineering**: 936-line sql-templates.md + SQLITE_BUSY handling + evidence gates that have never caught a real failure across 5 runs.
3. **Cross-file fragility**: § section references are the #1 source of Critical review findings.
4. **Unused platform features**: VS Code native `tools` field, `handoffs`, `agents` field all unused.
5. **Instruction dilution**: 6,400+ lines of natural language instructions risk LLM instruction-following degradation.

### Industry Context

| System                  | Architecture           | Agent Instructions     | Steps      | Evidence Mechanism                |
| ----------------------- | ---------------------- | ---------------------- | ---------- | --------------------------------- |
| **SWE-Agent**           | Single agent           | ~1 YAML config         | 1 (loop)   | JSON trajectories                 |
| **mini-SWE-Agent**      | Single agent           | ~100 lines Python      | 1 (loop)   | None                              |
| **OpenHands**           | Single agent           | SDK code               | 1-2        | Event stream                      |
| **MetaGPT**             | Multi-agent (~4 roles) | Python class defs      | ~5         | File artifacts                    |
| **AutoGen**             | Multi-agent (flexible) | ~10 lines Python/agent | Variable   | In-memory messages                |
| **CrewAI**              | Multi-agent (YAML)     | ~5-8 lines YAML/agent  | Variable   | LanceDB memory                    |
| **AgentForge**          | Multi-agent (DAG)      | ~5,000 LOC total       | DAG levels | Formal I/O contracts              |
| **Current System**      | Multi-agent (9 agents) | 6,416 lines Markdown   | 10+        | SQLite (936 lines SQL)            |
| **New System (target)** | Multi-agent (8 agents) | ≤1,500 lines Markdown  | 8          | YAML files + completion contracts |

---

## Agent Inventory

| #   | Agent            | File                  | Replaces                                | Primary Responsibility                        |
| --- | ---------------- | --------------------- | --------------------------------------- | --------------------------------------------- |
| 1   | **Orchestrator** | orchestrator.agent.md | Orchestrator                            | Pipeline coordination, DAG execution, routing |
| 2   | **Researcher**   | researcher.agent.md   | Researcher                              | Parallel web + codebase research              |
| 3   | **Architect**    | architect.agent.md    | Spec + Designer                         | Combined requirements + technical design      |
| 4   | **Planner**      | planner.agent.md      | Planner                                 | DAG task decomposition, file ownership        |
| 5   | **Implementer**  | implementer.agent.md  | Implementer                             | TDD implementation (unit tests only)          |
| 6   | **Tester**       | tester.agent.md       | Verifier + Test Runner (new)            | Dynamic testing + evidence evaluation         |
| 7   | **Reviewer**     | reviewer.agent.md     | Adversarial Reviewer                    | Multi-perspective code review                 |
| 8   | **Knowledge**    | knowledge.agent.md    | Knowledge + Instruction Optimizer (new) | Post-mortem + instruction optimization        |

**Agent count: 8** (down from 9 current + 2 proposed = 11 before pushback)

### Merge Rationale

- **Spec + Designer → Architect**: MetaGPT's Architect role handles both. Eliminates a handoff and reduces pipeline length. Requirements and design are tightly coupled — one agent produces both with full context.
- **Verifier + Test Runner → Tester**: Splits the current Verifier's dual nature (static evidence evaluation + dynamic test execution) into a single coherent agent that does both. Eliminates the need for a separate Test Runner agent.
- **Knowledge + Instruction Optimizer → Knowledge**: Both operate post-pipeline on accumulated data. Merging prevents a separate agent for a non-critical step.

---

## Pipeline Flow

```
Step 1: Setup
  │  Git baseline, feature directory, approval mode, web research toggle
  │
Step 2: Research (parallel, 2-4 instances)
  │  [🟢 simple: SKIP this step]
  │  Gate: ≥2 researchers complete
  │
Step 3: Architecture (single agent)
  │  Combined spec + design output
  │  [🔴 complex: embedded design review sub-phase with 2-3 Reviewers]
  │
Step 4: Planning (single agent)
  │  DAG task decomposition with file ownership
  │  [Interactive: approval gate]
  │
Step 5: Implementation (parallel waves, DAG-ordered)
  │  TDD: RED → GREEN per task
  │  Concurrency: ≤4 per wave, no file ownership overlap
  │
Step 6: Testing (sequential dynamic, parallel static)
  │  Start app → integration/E2E/API tests → stop app → evidence eval
  │  [Failure: loop back to Step 5, max 3 cycles]
  │
Step 7: Code Review (parallel, 2-3 instances)
  │  MANDATORY for all risk levels
  │  Gate: ≥2 approve + 0 blockers
  │  [Failure: loop back to Step 5 with findings, max 2 review rounds]
  │
Step 8: Completion
     Knowledge extraction → instruction recommendations → git commit
```

### Risk-Based Scaling

| Risk        | Research        | Design Review | Testing            | Code Review   | Max Dispatches |
| ----------- | --------------- | ------------- | ------------------ | ------------- | -------------- |
| 🟢 Simple   | Skip            | Skip          | Unit only (static) | 2 reviewers   | ~8             |
| 🟡 Standard | 2-3 researchers | Skip          | Unit + integration | 2-3 reviewers | ~15            |
| 🔴 Complex  | 4 researchers   | 2-3 reviewers | Full E2E + live    | 3 reviewers   | ~25            |

---

## State Management

### Approach: YAML File-Based Evidence

**Rationale**: SQLite evidence gates have never caught a failure across 5 pipeline runs (research impact F-14). Completion contracts already contain the routing signals. No other agent framework uses SQLite for evidence tracking (research architecture F-16). Completion-contract-based routing is industry-standard (AutoGen, LangGraph, MetaGPT all use structured return values).

**Implementation**:

1. **Routing**: Orchestrator reads completion blocks from agent output YAML files. `status: DONE` → proceed. `status: NEEDS_REVISION` → re-dispatch with context. `status: ERROR` → halt.
2. **Evidence**: File existence in the feature directory serves as evidence. Research complete = research/_.yaml files exist. Implementation complete = implementation-reports/task-_.yaml files exist.
3. **Audit trail**: The Knowledge agent produces an evidence-bundle.yaml at completion summarizing: all dispatch events, timing, findings, and outcomes.
4. **Verification**: Tester reads implementation reports (YAML files) to verify TDD compliance and test results. No SQL queries needed.

### What This Eliminates (~1,100 lines removed)

- sql-templates.md (936 lines) — entirely eliminated
- evaluation-schema.md (84 lines) — subsumed into Tester/Knowledge agent instructions
- SQLITE_BUSY handling in global rules — eliminated
- Evidence gate queries (EG-1 through EG-10) — replaced by file-existence checks
- sql_escape() sanitization rules — eliminated (no SQL)
- run_in_terminal for sqlite3 — eliminated (no SQLite CLI dependency)

---

## File Format Requirements

### Agent File Structure (.agent.md)

```yaml
---
name: agent-name
description: One-line description for VS Code
tools:
  - read_file
  - grep_search
  # ... only tools this agent needs
agents: []  # empty for non-orchestrator agents
---

# Agent Name

## Role
One paragraph: who you are, what you do, what you never do.

## Inputs
List of input files/artifacts this agent consumes.

## Workflow
Numbered steps (5-10 steps max).

## Output Schema
Inline YAML example of the output this agent produces.

## Constraints
Bullet list of hard rules and boundaries.
```

**Key differences from current system:**

- `tools` field provides platform-enforced tool restriction (replaces 105-line tool-access-matrix.md)
- Output schema co-located in agent file (replaces 1,422-line schemas.md)
- No § cross-references to external documents
- No anti-drift anchor needed (short enough that instruction dilution is not a risk)
- No self-verification checklist (short instructions + platform tool enforcement makes this unnecessary)

### Completion Contract (inline in each agent)

```yaml
completion:
  status: "DONE" # DONE | NEEDS_REVISION | ERROR
  summary: "≤200 chars"
  output_paths:
    - "path/to/output.yaml"
```

---

## Functional Requirements Detail

### FR-1: Pipeline Structure (8 Steps)

The pipeline consolidates the current 10+ steps into 8 by:

- Merging Spec + Design into Architecture (1 step instead of 3: spec, design, design-review)
- Merging Verification into Testing (1 step instead of 2)
- Merging Knowledge + Instruction Optimization + Commit into Completion (1 step instead of 3)
- Setup remains standalone (git initialization is a prerequisite)

Industry precedent: MetaGPT uses ~5 steps for a comparable pipeline. Anthropic recommends "finding the simplest solution possible, and only increasing complexity when needed."

### FR-6: DAG Task Decomposition with File Ownership

The Planner declares tasks with explicit dependencies and file ownership:

```yaml
tasks:
  - id: "add-health-endpoint"
    description: "Add /health endpoint with build info"
    depends_on: []
    files:
      - "Controllers/HealthController.cs"
      - "Tests/HealthControllerTests.cs"
    risk_level: "🟢"

  - id: "add-logging-middleware"
    description: "Add structured logging middleware"
    depends_on: []
    files:
      - "Middleware/LoggingMiddleware.cs"
      - "Tests/LoggingMiddlewareTests.cs"
    risk_level: "🟡"

  - id: "integrate-health-logging"
    description: "Wire health endpoint into logging pipeline"
    depends_on: ["add-health-endpoint", "add-logging-middleware"]
    files:
      - "Program.cs"
    risk_level: "🟢"
```

The Orchestrator computes parallel groups from the DAG:

- **Group 1**: `add-health-endpoint` + `add-logging-middleware` (no dependency, no file overlap → parallel)
- **Group 2**: `integrate-health-logging` (depends on Group 1 → sequential)

File ownership validation: if two tasks with no dependency edge declare the same file, the Orchestrator adds a dependency edge (one waits for the other). This prevents merge conflicts.

---

## Non-Functional Requirements

| ID    | Requirement             | Target                                  | Rationale                                                                                                                |
| ----- | ----------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| NFR-1 | Total system size       | ≤1,500 lines                            | GitHub recommends "2 pages"; current system is 6,416 lines. 1,500 allows 8 × ~150-line agents + ~300 lines shared rules. |
| NFR-2 | Per-agent size          | ≤150 lines                              | CrewAI: 5-8 lines/agent, AutoGen: ~10 lines/agent. 150 lines is generous but prevents bloat.                             |
| NFR-3 | Shared reference docs   | ≤2 files, ≤150 lines each               | Eliminates the 3,882-line reference doc burden. Rules inlined or co-located.                                             |
| NFR-4 | Cross-file § references | Zero                                    | #1 source of Critical findings. Eliminated by design.                                                                    |
| NFR-5 | External dependencies   | git only (+ project tools)              | Removes sqlite3 CLI dependency. Playwright etc. are project-specific, not system dependencies.                           |
| NFR-6 | Pipeline dispatches     | ≤25 for 🔴, ≤15 for 🟡, ≤8 for 🟢       | Current system: 34-56 dispatches per run. Target 50-75% reduction.                                                       |
| NFR-7 | Traceability            | Every pattern traced to industry source | User requirement: "Use ESTABLISHED practices only."                                                                      |

---

## Constraints & Assumptions

### Constraints

- Windows environment, VS Code with GitHub Copilot extension
- Local developer workstation execution only
- VS Code Copilot `runSubagent` API for agent dispatch
- Max 4 concurrent subagent dispatches (VS Code runtime limitation)
- GitHub Copilot `.agent.md` file format
- `copilot-instructions.md` remains as repository-wide global rules (immutable at runtime)

### Assumptions

- VS Code Copilot continues supporting the `tools` YAML frontmatter field for agent tool restriction
- The `runSubagent` API continues to execute multiple calls in the same reasoning step in parallel
- The existing `copilot-instructions.md` global rules (terminal-only testing, no file-redirect, retry policy, anti-hallucination) carry forward to the new system
- Prior system directories (NewAgents/, Forge/, .github/agents/) will be archived or removed after the new system is validated

---

## Acceptance Criteria (Full)

| ID    | Criterion                     | Pass/Fail Definition                                                                      | Method                             |
| ----- | ----------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------- | ----------------------------------- | ---- |
| AC-1  | Total size ≤1,500 lines       | `(Get-ChildItem -Recurse _.agent.md,_.md                                                  | Get-Content                        | Measure-Object -Line).Lines ≤ 1500` | test |
| AC-2  | Per-agent ≤150 lines          | Each .agent.md file: `(Get-Content file                                                   | Measure-Object -Line).Lines ≤ 150` | test                                |
| AC-3  | Exactly 8 .agent.md files     | Count of \*.agent.md files = 8                                                            | inspection                         |
| AC-4  | ≤2 shared reference docs      | Count of non-.agent.md Markdown files ≤ 2                                                 | inspection                         |
| AC-5  | YAML frontmatter with tools   | Every .agent.md has `tools:` in frontmatter                                               | inspection                         |
| AC-6  | Orchestrator lists all agents | Orchestrator frontmatter `agents:` contains 7 agent names                                 | inspection                         |
| AC-7  | Non-orchestrator agents: []   | All other agents have `agents: []`                                                        | inspection                         |
| AC-8  | No SQLite references          | `grep -ri "sqlite\|verification-ledger\|sql-templates" *.agent.md *.md` returns 0 matches | test                               |
| AC-9  | YAML-based routing            | Orchestrator workflow describes reading YAML files, not SQL queries                       | inspection                         |
| AC-10 | 8 pipeline steps              | Orchestrator documents exactly 8 named steps                                              | inspection                         |
| AC-11 | Code review mandatory         | Orchestrator Step 7 has no skip condition regardless of risk                              | inspection                         |
| AC-12 | DAG task schema               | Planner output includes depends_on and files per task                                     | inspection                         |
| AC-13 | DAG-based dispatch            | Orchestrator dispatches by dependency satisfaction, not fixed waves                       | inspection                         |
| AC-14 | No file ownership overlap     | Orchestrator prevents same-file tasks in same parallel group                              | inspection                         |
| AC-15 | Implementer limited tools     | Implementer tools list excludes app lifecycle tools                                       | inspection                         |
| AC-16 | Tester has run_in_terminal    | Tester tools list includes run_in_terminal                                                | inspection                         |
| AC-17 | Knowledge no agent writes     | Knowledge cannot modify .agent.md files                                                   | inspection                         |
| AC-18 | No § cross-references         | `grep -r "§" *.agent.md *.md` returns 0 matches in new system                             | test                               |
| AC-19 | Researcher has fetch_webpage  | Researcher tools list includes fetch_webpage                                              | inspection                         |
| AC-20 | New directory                 | New system in separate directory from existing systems                                    | inspection                         |
| AC-21 | Industry traceability         | Major decisions cite industry sources                                                     | analysis                           |
| AC-22 | Completion contracts          | Every agent output includes completion block                                              | inspection                         |

---

## Edge Cases & Error Handling

| ID    | Input/Condition                            | Expected Behavior                                                   | Severity if Missed |
| ----- | ------------------------------------------ | ------------------------------------------------------------------- | ------------------ |
| EC-1  | Agent returns malformed YAML               | Retry once, then ERROR + halt                                       | Critical           |
| EC-2  | All researchers fail (0/4)                 | ERROR + halt. Cannot proceed without research.                      | Critical           |
| EC-3  | Planner produces cyclic DAG                | Return plan with NEEDS_REVISION + cycle description                 | High               |
| EC-4  | Undeclared file conflict in parallel tasks | Tester/Reviewer catches merge conflict. Rerun affected task.        | High               |
| EC-5  | Code review fails after 2 rounds           | ERROR + halt. Manual intervention required.                         | High               |
| EC-6  | Web research disabled but needed           | Researcher completes with available info, notes gap in output       | Medium             |
| EC-7  | >4 tasks ready to execute                  | Partition into sub-waves of ≤4, sequential between, parallel within | Medium             |
| EC-8  | 🟢 simple feature                          | Skip Research, Architect works from initial-request only            | Low                |
| EC-9  | Tester can't start app (build failure)     | NEEDS_REVISION with build error → loop back to Implementer (max 3)  | High               |
| EC-10 | Agent file exceeds 150-line limit          | Knowledge agent flags in recommendations, Reviewer catches          | Medium             |

---

## User Stories / Flows

### Story 1: Standard Feature Implementation (🟡)

1. Developer invokes pipeline with feature request
2. **Setup**: Orchestrator creates git baseline, asks approval mode + web research → interactive, yes
3. **Research**: 3 researchers investigate architecture, impact, dependencies → ≥2 complete
4. **Approval gate**: Developer reviews research summary, approves
5. **Architecture**: Architect produces requirements + design from research
6. **Planning**: Planner decomposes into 5 tasks with DAG dependencies
7. **Approval gate**: Developer reviews plan, approves
8. **Implementation**: Wave 1 (3 tasks parallel) → Wave 2 (2 tasks parallel, depend on Wave 1)
9. **Testing**: Tester runs integration tests, verifies TDD compliance, all pass
10. **Code Review**: 3 reviewers assess → 2 approve, 1 requests changes (0 blockers) → pass gate
11. **Completion**: Knowledge summarizes pipeline, commits code

### Story 2: Simple Bug Fix (🟢)

1. Developer invokes pipeline with bug report
2. **Setup**: Git baseline → autonomous mode, no web research
3. **Research**: Skipped (🟢 simple)
4. **Architecture**: Architect produces minimal spec from initial request
5. **Planning**: Single task, no dependencies
6. **Implementation**: One implementer, TDD fix
7. **Testing**: Tester verifies unit tests pass, no integration tests needed
8. **Code Review**: 2 reviewers assess → both approve
9. **Completion**: Knowledge summarizes, commits

### Story 3: Complex Architectural Change (🔴)

1. Developer invokes pipeline → interactive, web research enabled
2. **Setup**: Git baseline
3. **Research**: 4 researchers with web research → all 4 complete
4. **Approval gate**: Developer reviews research
5. **Architecture**: Architect produces comprehensive spec + design
6. **Embedded design review**: 3 Reviewers assess design → iterate if needed
7. **Planning**: 12 tasks across 4 DAG levels, file ownership declared
8. **Approval gate**: Developer reviews plan
9. **Implementation**: 4 waves (4+4+3+1 tasks)
10. **Testing**: Full E2E + integration + live validation
11. **Code Review**: 3 reviewers, Round 1 finds 1 Critical → fix → Round 2 all approve
12. **Completion**: Knowledge captures extensive post-mortem, commits

---

## Test Scenarios

| #    | Scenario                                            | Verifies         | Expected                                               |
| ---- | --------------------------------------------------- | ---------------- | ------------------------------------------------------ |
| T-1  | Count all .agent.md files in new system             | AC-3             | Exactly 8                                              |
| T-2  | Measure line count per .agent.md                    | AC-2             | Each ≤150                                              |
| T-3  | Sum total lines of all system files                 | AC-1             | ≤1,500                                                 |
| T-4  | grep for "sqlite" in new system                     | AC-8             | 0 matches                                              |
| T-5  | grep for "§" in new system                          | AC-18            | 0 matches                                              |
| T-6  | Parse YAML frontmatter of each .agent.md            | AC-5, AC-6, AC-7 | tools present, orchestrator has agents, others have [] |
| T-7  | Verify orchestrator mentions 8 named steps          | AC-10            | 8 steps                                                |
| T-8  | Check orchestrator for "skip" conditions on Step 7  | AC-11            | No skip for code review                                |
| T-9  | Inspect Planner output schema for DAG fields        | AC-12            | depends_on + files present                             |
| T-10 | Inspect Implementer tools list                      | AC-15            | No app lifecycle tools                                 |
| T-11 | Inspect Tester tools list                           | AC-16            | run_in_terminal present                                |
| T-12 | Verify new system is in separate directory          | AC-20            | Different from NewAgents/, Forge/, .github/            |
| T-13 | Verify completion block in all agent output schemas | AC-22            | All agents define it                                   |
| T-14 | Verify Researcher has fetch_webpage                 | AC-19            | Present in tools list                                  |

---

## Dependencies & Risks

### Dependencies

| Dependency                                 | Type          | Risk if Unavailable                                                       |
| ------------------------------------------ | ------------- | ------------------------------------------------------------------------- |
| VS Code Copilot with `tools` field support | Platform      | Cannot enforce tool restrictions natively; fall back to instruction-based |
| VS Code `runSubagent` API                  | Platform      | Cannot dispatch agents; entire system non-functional                      |
| git CLI                                    | External tool | Cannot create baselines or commits                                        |
| Project-specific build tools               | External tool | Tester cannot run tests; graceful degradation (skip dynamic tests)        |

### Risks

| Risk                                                  | Likelihood | Impact | Mitigation                                                                                                                                          |
| ----------------------------------------------------- | ---------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| 150-line limit too restrictive for complex agents     | Medium     | Medium | Budget is generous (CrewAI agents are 5-8 lines). Monitor during implementation; if Orchestrator needs more, allow up to 200 for orchestrator only. |
| Completion-contract routing misses subtle failures    | Low        | High   | 5 prior runs had zero routing failures with this mechanism. Tester provides secondary verification of implementation quality.                       |
| VS Code removes/changes `tools` field behavior        | Low        | Medium | Fall back to instruction-based tool restriction. Document the fallback.                                                                             |
| New system doesn't handle existing project structures | Medium     | Medium | Preserve `copilot-instructions.md` global rules and prompt files that are project-generic.                                                          |
| Loss of audit trail without SQLite                    | Low        | Low    | Knowledge agent produces evidence-bundle.yaml. YAML files in feature directory are git-tracked. Audit trail exists in file system + git history.    |
| Complexity creep in future runs                       | High       | High   | Hard size limits (AC-1, AC-2) are acceptance criteria, not guidelines. Any violation is a review finding. Knowledge agent monitors and flags.       |

---

## Scope

### In Scope

- 8 new agent definition files (.agent.md)
- 1-2 shared reference documents (global-rules.md, optional conventions.md)
- 1-2 prompt files for pipeline invocation
- Feature directory structure convention
- Completion contract schema (inline in agents)
- DAG task schema (inline in Planner)
- copilot-instructions.md updates to reference new system

### Out of Scope

- Migration of existing pipeline runs or their artifacts
- Removal of NewAgents/, Forge/, or .github/agents/ directories (post-validation cleanup)
- Changes to VS Code or GitHub Copilot platform
- Project-specific test infrastructure (Playwright skills, E2E contracts)
- Anvil single-agent system (preserved as independent alternative)

---

## Design Decision Traceability

| Decision                               | Industry Source                                                                 |
| -------------------------------------- | ------------------------------------------------------------------------------- |
| 8-step pipeline (not 12)               | Anthropic: "finding the simplest solution possible"                             |
| Completion-contract-only routing       | AutoGen: structured message-based routing; LangGraph: node return values        |
| YAML file-based state (not SQLite)     | All frameworks: SWE-Agent JSON, OpenHands events, MetaGPT files, AutoGen memory |
| VS Code `tools` field for restrictions | VS Code custom agents docs: native tool enforcement                             |
| DAG task decomposition                 | GitHub Actions `needs:`, Airflow task DAGs, AgentForge Theorem 1                |
| Merge Spec+Designer → Architect        | MetaGPT: Architect role handles both                                            |
| ≤150 lines/agent                       | CrewAI: 5-8 lines, AutoGen: ~10 lines, GitHub: "2 pages"                        |
| 3-perspective adversarial review       | Anthropic parallelization "voting" pattern; proven value (6+ Critical catches)  |
| TDD enforcement (RED-GREEN)            | Current system proven pattern; no comparable framework does this                |
| File ownership in Planner              | Airflow: task resource declarations, Dagster: asset-centric dependencies        |
| Hub-and-spoke orchestration            | Current system proven pattern; AutoGen Group Chat Manager                       |
