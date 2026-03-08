# Agent System Refactor — Technical Design (Revision 2)

## Title & Summary

**Feature:** Clean-Slate Multi-Agent Pipeline Rebuild
**Design Decision Count:** 15 architectural decisions (12 preserved from R1 + 3 new in R2)
**Revision:** Round 2 — addresses 13 Major findings from Security Sentinel, Architecture Guardian, Pragmatic Verifier
**Overall Risk:** 🟡 (business logic modification — replacing core pipeline infrastructure)

A complete architectural redesign of the multi-agent orchestration system. Reduces from 6,416 lines / 19 files / 10+ steps to ~1,100 lines / 10 files / 8 steps. Eliminates SQLite dependency, eliminates cross-file section references, leverages VS Code native agent features, and enforces hard size limits. All decisions traced to established industry practices. R2 adds trust boundary analysis, implementer command allowlist, incremental pipeline logging, pre-commit output validation, quick-fix step set specification, and failure routing for embedded design review.

---

## Revision History

| Round | Date       | Trigger                            | Changes                                                                                                                                                                                    |
| ----- | ---------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| R1    | 2026-03-08 | Initial design                     | 12 decisions, 8 agents, 8 steps, ~1,100 lines, zero SQLite                                                                                                                                 |
| R2    | 2026-03-08 | 13 Major findings from 3 reviewers | +3 decisions (D-13/D-14/D-15), trust boundary model, command allowlist, incremental logging, pre-commit validation, quick-fix steps, failure routing, conditional inputs, YAML parse gates |

### R2 Major Finding Resolutions

| Finding                                 | Source                | Resolution                                                     |
| --------------------------------------- | --------------------- | -------------------------------------------------------------- |
| SS-S-1: Implementer terminal scope      | Security Sentinel     | Command allowlist + audit in implementation report             |
| SS-S-2: fetch_webpage autonomous toggle | Security Sentinel     | Documented residual risk in D-10, URL audit trail              |
| SS-S-3: Knowledge post-review writes    | Security Sentinel     | Pre-commit git diff allowlist validation (Step 8)              |
| SS-A-1: No trust boundary analysis      | Security Sentinel     | D-14: 3-tier trust model with data flow transitions            |
| SS-C-1: Undefined YAML status behavior  | Security Sentinel     | Treat unknown status as ERROR (validation rule)                |
| SS-C-2: Quick-fix step set undefined    | Security Sentinel     | D-13: steps {1,5,6,7,8}, Code Review mandatory                 |
| AG-A-1: Orchestrator 150-line fallback  | Architecture Guardian | D-8 amended: extract to dag-dispatch.md if over                |
| AG-A-2: Tester split trigger            | Architecture Guardian | D-7 amended: >140 lines OR >20 duplicated lines                |
| AG-A-3: File ownership enforcement      | Architecture Guardian | Implementer constraint: no undeclared files, files_requested[] |
| AG-C-1: Naming convention               | Architecture Guardian | Standardized architecture-output.yaml                          |
| AG-C-2: Design review failure routing   | Architecture Guardian | Max 1 revision round for 🔴, then ERROR                        |
| PV-C-1: Architect missing research      | Pragmatic Verifier    | Conditional inputs: research optional for 🟢                   |
| PV-C-2: Audit trail on failure          | Pragmatic Verifier    | D-15: incremental pipeline-log.yaml                            |

---

## Context & Inputs

### Input Artifacts Consumed

| Artifact                   | Key Insights                                                                                           |
| -------------------------- | ------------------------------------------------------------------------------------------------------ |
| initial-request.md         | Full rewrite authority, web research for dev runs, Test Runner agent, DAG-based scheduling, strict TDD |
| spec-output.yaml           | Direction C (clean-slate), 8 agents, 8 steps, ≤1,500 lines, no SQLite, 22 acceptance criteria          |
| research/architecture.yaml | 19 findings: current system 6,416 lines, industry comparison, VS Code native features unused           |
| research/impact.yaml       | SQLite permeates 12/19 files, evidence gates never blocked, 3 system generations coexist               |
| research/dependencies.yaml | Agent-to-reference dependency graph, EG chain fragility, 4 SQLite alternatives                         |
| research/patterns.yaml     | Completion contracts proven, adversarial review catches real issues, industry trend toward simplicity  |

### Key Research Findings Driving Design

1. **Complexity spiral**: 5 improvement runs, all additive, none simplifying. System grew from ~5,381 → 6,416 lines.
2. **Evidence gates never fired**: 10 SQL evidence gates across 5 runs, zero blocks. The safety net never caught anything.
3. **Cross-file §references = #1 Critical finding source**: Every pipeline run produces Critical findings from broken cross-references.
4. **Industry comparison**: SWE-Agent achieves SOTA in ~100 lines. CrewAI agents are 5-8 lines. Current system is 100x GitHub's recommended size.
5. **VS Code native features unused**: `tools` YAML field, `agents` field, `handoffs` — all available but not leveraged.
6. **Completion contracts already work**: Zero routing failures across 5 runs. The primary routing mechanism is proven.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                      │
│  Pipeline coordinator, DAG dispatch, routing         │
│  agents: [researcher, architect, planner,            │
│           implementer, tester, reviewer, knowledge]  │
└────────┬──────┬──────┬──────┬──────┬──────┬──────┬──┘
         │      │      │      │      │      │      │
    ┌────┴┐ ┌──┴──┐ ┌─┴──┐ ┌┴───┐ ┌┴──┐ ┌─┴──┐ ┌─┴──┐
    │Rsrch│ │Archt│ │Plan│ │Impl│ │Test│ │Revw│ │Know│
    │×2-4 │ │ ×1  │ │ ×1 │ │×1-4│ │ ×1 │ │×2-3│ │ ×1 │
    └─────┘ └─────┘ └────┘ └────┘ └────┘ └────┘ └────┘

    Step 2    Step 3  Step 4  Step 5  Step 6  Step 7  Step 8

    Routing: Completion contracts (DONE/NEEDS_REVISION/ERROR)
    Evidence: YAML file existence in docs/feature/<slug>/
    Tool access: VS Code native `tools` YAML frontmatter
    State: Zero SQLite — YAML files only
```

### Design Principles

1. **Completion-contract-only routing** — The orchestrator reads `status: DONE/NEEDS_REVISION/ERROR` from YAML output files. No SQL queries, no evidence gates. (AutoGen, LangGraph, MetaGPT all use structured returns)
2. **Platform-enforced tool restrictions** — VS Code's `tools` field in YAML frontmatter restricts what each agent can use. Replaces the 105-line tool-access-matrix.md. (VS Code custom agent docs)
3. **Co-located schemas** — Each agent defines its own output schema inline. No central schemas.md. (Eliminates 1,422 lines)
4. **Zero cross-file §references** — All rules inlined or co-located. Only reference: global-rules.md by filename, never by section number. (Removes #1 Critical finding source)
5. **Hard size limits** — ≤150 lines/agent, ≤1,500 total. Prevents the complexity spiral. (Anthropic: "simplest solution possible")
6. **DAG-based execution** — Planner declares dependencies, orchestrator computes parallel groups at dispatch time. (GitHub Actions `needs:`, Airflow DAG)

---

## Agent Inventory (8 Agents)

| #   | Agent            | Replaces                    | Line Budget | Responsibilities                                                                                    |
| --- | ---------------- | --------------------------- | ----------- | --------------------------------------------------------------------------------------------------- |
| 1   | **Orchestrator** | Orchestrator                | 150         | Pipeline coordination, DAG dispatch, routing, git ops, R2: pipeline-log.yaml, pre-commit validation |
| 2   | **Researcher**   | Researcher                  | 100         | Parallel web + codebase research                                                                    |
| 3   | **Architect**    | Spec + Designer             | 140         | Combined requirements + technical design, R2: conditional inputs for 🟢                             |
| 4   | **Planner**      | Planner                     | 130         | DAG task decomposition, file ownership                                                              |
| 5   | **Implementer**  | Implementer                 | 130         | TDD implementation, unit tests only, R2: command audit + file ownership                             |
| 6   | **Tester**       | Verifier + Test Runner      | 140         | Dynamic testing + evidence evaluation, R2: command allowlist audit                                  |
| 7   | **Reviewer**     | Adversarial Reviewer        | 110         | Multi-perspective code review, R2: command + URL audit                                              |
| 8   | **Knowledge**    | Knowledge + Inst. Optimizer | 100         | Post-mortem + evidence bundle (reads pipeline-log.yaml) + instruction recommendations               |

**Total agent lines: ~1,000** | **global-rules.md: ~100** | **Grand total: ~1,100** (budget remaining: ~400)

### Merge Rationale

- **Spec + Designer → Architect**: MetaGPT uses a single Architect role for both requirements and design. Eliminates one pipeline handoff and the context loss between Spec and Designer. Requirements and design decisions are tightly coupled — the same agent should hold both. (D-2)
- **Verifier + Test Runner → Tester**: The Verifier currently has dual responsibilities (static evidence evaluation + dynamic test execution in Tier 5). Making this a single agent with two modes (static/dynamic) is cleaner than splitting it. The orchestrator controls which mode runs. (D-7)
- **Knowledge + Instruction Optimizer → Knowledge**: Both operate post-pipeline on accumulated data. Instruction optimization is a natural extension of post-mortem analysis. The Knowledge agent writes recommendations to a file for human review — never modifies .agent.md files directly. (D-2)

---

## Agent File Format

### YAML Frontmatter (D-3)

Every `.agent.md` file uses this structure:

```yaml
---
name: <agent-name>
description: <one-line description for VS Code>
tools:
  - read_file
  - grep_search
  # ... only tools this agent needs (platform-enforced)
agents: [] # empty for non-orchestrator agents
---
```

The orchestrator additionally includes:

```yaml
agents:
  - researcher
  - architect
  - planner
  - implementer
  - tester
  - reviewer
  - knowledge
```

### Body Structure (6 sections)

Every agent body follows this consistent structure:

1. **Role** — One paragraph: who you are, what you do, what you never do.
2. **Inputs** — Bullet list of input files/artifacts this agent consumes.
3. **Workflow** — Numbered steps (5-10 max). The core of the agent's behavior.
4. **Output Schema** — Inline YAML example of the output format (co-located, not referenced).
5. **Constraints** — Bullet list of hard rules and boundaries.
6. **Anti-Drift Anchor** — One paragraph reinforcing identity and boundaries.

---

## Per-Agent Tool Access

| Tool                         | Orch | Rsrch | Archt | Plan | Impl | Test | Revw | Know |
| ---------------------------- | ---- | ----- | ----- | ---- | ---- | ---- | ---- | ---- |
| read_file                    | ✅   | ✅    | ✅    | ✅   | ✅   | ✅   | ✅   | ✅   |
| create_file                  | ✅   | ✅    | ✅    | ✅   | ✅   | ✅   | ✅   | ✅   |
| replace_string_in_file       | ✅   |       |       |      | ✅   |      |      |      |
| multi_replace_string_in_file |      |       |       |      | ✅   |      |      |      |
| list_dir                     | ✅   | ✅    | ✅    | ✅   | ✅   | ✅   | ✅   | ✅   |
| grep_search                  | ✅   | ✅    | ✅    | ✅   | ✅   | ✅   | ✅   | ✅   |
| semantic_search              |      | ✅    | ✅    | ✅   | ✅   | ✅   | ✅   | ✅   |
| file_search                  | ✅   | ✅    | ✅    | ✅   | ✅   | ✅   | ✅   | ✅   |
| run_in_terminal              | ✅   |       |       |      | ✅   | ✅   |      |      |
| get_terminal_output          | ✅   |       |       |      | ✅   | ✅   |      |      |
| get_errors                   |      |       |       |      | ✅   | ✅   | ✅   |      |
| get_changed_files            |      |       |       |      |      |      | ✅   |      |
| fetch_webpage                |      | ✅    | ✅    |      | ✅   |      |      |      |
| vscode_askQuestions          | ✅   |       |       |      |      |      |      |      |
| memory                       |      |       |       |      |      |      |      | ✅   |

Key restrictions:

- **Implementer**: Has `run_in_terminal` but instruction-constrained to build/test commands. All commands logged in `implementation_report.commands_executed[]`. Reviewer audits against allowlist. (R2: D-14 Tier 2)
- **Tester**: Has `run_in_terminal` for app lifecycle management and test suite execution. Audits implementer commands against allowlist. (R2)
- **Reviewer**: Read-only analysis tools + `get_errors` for static diagnostics + `get_changed_files` for diff review. R2: Audits implementer commands and fetch_webpage URL trails.
- **Knowledge**: Has `memory` tool for storing learned facts. Has `create_file` for output files only — MUST NOT create/modify .agent.md files. R2: Output restricted to allowlist, validated by orchestrator pre-commit.

---

## Pipeline Design (8 Steps)

### Step 1: Setup

**Agent:** Orchestrator (directly, no dispatch)

1. Create feature directory: `docs/feature/<slug>/`
2. Create git baseline tag: `pipeline-baseline-<run_id>`
3. Interactive mode: ask `approval_mode` (interactive/autonomous) and `web_research` (yes/no)
4. Classify feature risk: 🟢 (simple) / 🟡 (standard) / 🔴 (complex)
5. R2: Initialize `pipeline-log.yaml` with run metadata (run_id, feature_slug, risk, approval_mode)

**Outputs:** Feature directory created, initial-request.md stored, pipeline-log.yaml initialized.

### Step 2: Research

**Agent:** Researcher × 2-4 (parallel dispatch)

| Risk | Researchers | Focus Areas                                  |
| ---- | ----------- | -------------------------------------------- |
| 🟢   | SKIP        | —                                            |
| 🟡   | 2-3         | architecture, impact/dependencies, patterns  |
| 🔴   | 4           | architecture, impact, dependencies, patterns |

**Gate:** ≥2 researchers return `status: DONE`
**Approval Gate:** Interactive mode — user approves research before proceeding.
**Outputs:** `research/*.yaml` files.

### Step 3: Architecture

**Agent:** Architect × 1

The Architect reads all research outputs + initial-request.md and produces:

- Functional requirements with IDs and acceptance criteria
- Architectural direction selection with alternatives analysis
- Key technical decisions with rationale

**R2 Conditional Inputs:** For 🟢 features (Step 2 skipped), Architect gracefully handles missing research by using initial-request.md + codebase analysis (grep_search, semantic_search) as sole inputs.

**🔴 Embedded Design Review:** For complex features, orchestrator dispatches 2-3 Reviewers to review the architecture output before proceeding. Review gate: ≥2 approve + 0 blockers. **R2 Failure Routing:** If design review fails, Architect is re-dispatched with findings (max 1 revision round). If 2nd round fails → pipeline ERROR.

**R2 Naming Convention:** Output file is `architecture-output.yaml` (not `design-output.yaml`), reflecting the merged Spec+Designer → Architect identity.

**Outputs:** `architecture-output.yaml`, `architecture.md`

### Step 4: Planning

**Agent:** Planner × 1

The Planner reads architecture output and produces a DAG task graph:

```yaml
tasks:
  - id: "add-health-endpoint"
    description: "Add /health endpoint"
    depends_on: []
    files: ["Controllers/HealthController.cs", "Tests/HealthControllerTests.cs"]
    risk: "🟢"

  - id: "add-logging"
    description: "Add structured logging"
    depends_on: []
    files: ["Services/LoggingService.cs", "Tests/LoggingServiceTests.cs"]
    risk: "🟡"

  - id: "wire-health-logging"
    description: "Wire health into logging"
    depends_on: ["add-health-endpoint", "add-logging"]
    files: ["Program.cs"]
    risk: "🟢"
```

**Approval Gate:** Interactive mode — user approves plan before implementation.
**Outputs:** `plan-output.yaml`

### Step 5: Implementation (DAG-Ordered Parallel Waves)

**Agent:** Implementer × 1-4 (parallel, DAG-driven)

Orchestrator DAG dispatch algorithm:

```
1. Read task list from plan-output.yaml
2. Build dependency graph
3. LOOP until all tasks done or error:
   a. Find READY tasks (all depends_on satisfied)
   b. For each ready task, check file overlap with other ready tasks
   c. If overlap: defer one task (implicit serialization)
   d. Dispatch up to 4 non-overlapping ready tasks in parallel
   e. Wait for completion contracts
   f. DONE → mark task satisfied
   g. NEEDS_REVISION → re-dispatch with context (max 3 per task)
   h. ERROR → halt pipeline
```

Each implementer follows strict TDD:

1. **RED**: Write failing unit tests
2. **GREEN**: Write minimal production code to pass
3. **Report**: Produce implementation report YAML

**Outputs:** `implementation-reports/task-<id>.yaml` per task.

### Step 6: Testing

**Agent:** Tester × 1 (singleton for dynamic mode)

**Phase 1 — Static Evaluation:**

- Read all implementation reports
- Verify TDD compliance (tests exist, were run, pass)
- Check no regressions in existing test suites
- R2: Audit `commands_executed[]` against implementer command allowlist

**Phase 2 — Dynamic Testing (🟡/🔴 only):**

- Start the application
- Run integration tests
- Run E2E tests (if skill files present)
- Run API contract tests
- Perform live validation
- Stop the application

**Failure:** Returns `NEEDS_REVISION` → orchestrator routes back to Step 5 with error context. Max 3 total implementation-testing cycles.

**Outputs:** `test-report.yaml`

### Step 7: Code Review (MANDATORY)

**Agent:** Reviewer × 2-3 (parallel dispatch)

Each reviewer is assigned a perspective:

- **Security**: Injection, access control, data exposure, secrets
- **Architecture**: Patterns, coupling, separation of concerns, conventions
- **Correctness**: Logic errors, edge cases, test coverage, spec compliance

**Gate:** ≥2 reviewers approve AND 0 blocker findings.
**Failure:** Routes back to Step 5 with findings for fix. Max 2 review rounds total.
**NEVER skippable** — mandatory for all risk levels.

**Outputs:** `review-verdicts/<perspective>.yaml`, `review-findings/<perspective>.md`

### Step 8: Completion

**Agent:** Knowledge × 1 + Orchestrator

Knowledge agent produces:

- `evidence-bundle.yaml` — reads pipeline-log.yaml, enriches with dispatch log, timing, outcomes
- `knowledge-output.yaml` — lessons learned, metrics
- `instruction-recommendations.md` — proposed instruction improvements for human review

Orchestrator performs:

- R2: Pre-commit validation — verifies git diff only contains files in Knowledge output allowlist + feature directory
- `git add .` + `git commit -m "feat(<slug>): <summary>"`

**Outputs:** Evidence bundle, knowledge output, git commit.

---

### Risk-Based Scaling Summary

| Step                | 🟢 Simple      | 🟡 Standard          | 🔴 Complex                    |
| ------------------- | -------------- | -------------------- | ----------------------------- |
| 2. Research         | SKIP           | 2-3 researchers      | 4 researchers                 |
| 3. Architecture     | Architect only | Architect only       | Architect + 2-3 reviewers     |
| 5. Implementation   | Standard       | Standard             | Standard                      |
| 6. Testing          | Static only    | Static + integration | Full dynamic (E2E, API, live) |
| 7. Code Review      | 2 reviewers    | 2-3 reviewers        | 3 reviewers                   |
| **Est. Dispatches** | **~6-8**       | **~12-15**           | **~20-25**                    |

---

## Data Models & Schemas

### Completion Contract (all agents)

Defined once in global-rules.md, referenced by all agents:

```yaml
completion:
  status: "DONE" | "NEEDS_REVISION" | "ERROR"
  summary: "≤200 characters describing outcome"
  output_paths:
    - "relative/path/to/output.yaml"
```

### Agent Output Wrapper (all agents)

```yaml
agent_output:
  agent: "<agent-name>"
  started_at: "<ISO8601>"
  completed_at: "<ISO8601>"
  payload:
    # Agent-specific content (defined inline per agent)
completion:
  status: "DONE"
  summary: "..."
  output_paths: [...]
```

### Task Schema (planner output)

```yaml
tasks:
  - id: "<task-id>"
    description: "<what to implement>"
    depends_on: []              # list of task IDs
    files:                      # file ownership declaration
      - "path/to/file.ext"
    risk: "🟢" | "🟡" | "🔴"
```

### Implementation Report (implementer output)

```yaml
task_id: "<task-id>"
files_modified: ["path/file.ext"]
files_requested: [] # R2: if NEEDS_REVISION for more files
tests_written: ["path/test.ext"]
test_results:
  passed: <N>
  failed: <N>
tdd_compliance: true | false
commands_executed: # R2: terminal command audit trail
  - "dotnet test ..."
  - "dotnet build ..."
```

### Test Report (tester output)

```yaml
test_categories:
  unit_tests: { passed: N, failed: N }
  integration_tests: { passed: N, failed: N, skipped: N }
  e2e_tests: { passed: N, failed: N, skipped: N }
tdd_verification:
  compliant_tasks: N
  non_compliant_tasks: N
command_allowlist_audit:          # R2: audit implementer commands
  violations: []
  clean: true | false
verdict: "pass" | "fail"
```

### Review Verdict (reviewer output)

```yaml
perspective: "security" | "architecture" | "correctness"
findings:
  - severity: "blocker" | "critical" | "major" | "minor"
    description: "<finding description>"
    file: "<affected file>"
    recommendation: "<fix suggestion>"
verdict: "approve" | "request-changes"
```

---

## State Management (D-4, D-15)

### Approach: YAML File-Based Evidence

**R2 Naming Convention:** Output files use agent-name-output.yaml pattern: `architecture-output.yaml` (not design-output.yaml), `plan-output.yaml`, `test-report.yaml`.

**Routing:** Orchestrator reads completion blocks from agent output YAML files. `status: DONE` → proceed. `status: NEEDS_REVISION` → re-dispatch with context. `status: ERROR` → halt pipeline. R2: Any unknown/malformed status value → treat as ERROR.

**Evidence:** File existence in the feature directory serves as evidence. R2: For critical gates (Research ≥2, Code Review ≥2), orchestrator also validates YAML parseability before trusting status.

| Step           | Evidence Check                                   |
| -------------- | ------------------------------------------------ |
| Research       | `research/*.yaml` files exist (≥2)               |
| Architecture   | `architecture-output.yaml` exists                |
| Planning       | `plan-output.yaml` exists                        |
| Implementation | `implementation-reports/task-{id}.yaml` per task |
| Testing        | `test-report.yaml` exists                        |
| Code Review    | `review-verdicts/*.yaml` files exist (≥2)        |

### Audit Trail (R2 — D-15)

**Two-layer design:**

1. **Layer 1 — Incremental (pipeline-log.yaml):** Orchestrator appends after every dispatch. Survives pipeline crashes.
2. **Layer 2 — Enriched (evidence-bundle.yaml):** Knowledge agent reads pipeline-log.yaml at Step 8 and adds analysis, lessons, and recommendations.

```yaml
# pipeline-log.yaml schema
run_id: "<ISO8601>"
feature_slug: "<slug>"
risk: "🟢" | "🟡" | "🔴"
approval_mode: "interactive" | "autonomous"
started_at: "<ISO8601>"
dispatches:
  - step: <N>
    agent: "<agent-name>"
    instance: "<instance-id>"
    started_at: "<ISO8601>"
    completed_at: "<ISO8601>"
    status: "DONE" | "NEEDS_REVISION" | "ERROR"
    output_paths: ["..."]
```

### What This Eliminates

| Eliminated Component    | Lines Removed | Reason                                     |
| ----------------------- | ------------- | ------------------------------------------ |
| sql-templates.md        | 936           | No SQL needed                              |
| schemas.md              | 1,422         | Schemas co-located in agents               |
| evaluation-schema.md    | 84            | Subsumed into Tester                       |
| e2e-integration.md      | 671           | Subsumed into Tester skills                |
| dispatch-patterns.md    | 236           | DAG algorithm inline in Orchestrator       |
| tool-access-matrix.md   | 105           | Replaced by YAML frontmatter `tools`       |
| severity-taxonomy.md    | 128           | Risk definitions inline in global-rules.md |
| review-perspectives.md  | 118           | Perspective assignment inline in Reviewer  |
| context7-integration.md | 29            | One-line mention in relevant agents        |
| **Total eliminated**    | **3,729**     |                                            |

---

## Security Considerations

### Trust Boundary Model (R2 — D-14)

| Tier                            | Agents                                              | Capabilities                                                          | Key Restriction                                               |
| ------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Tier 1 (Full Trust)**         | Orchestrator                                        | run_in_terminal (git), agent dispatch, file edit, pipeline-log writes | Sole dispatcher; pre-commit validates Knowledge output        |
| **Tier 2 (Standard)**           | Implementer, Tester                                 | run_in_terminal (scoped to build/test), file edit (scoped)            | Implementer: declared files only. Tester: test execution only |
| **Tier 3 (Read-Only + Create)** | Researcher, Architect, Planner, Reviewer, Knowledge | read_file, create_file (new only), fetch_webpage (some)               | Cannot modify existing code                                   |

**Data Flow:** Orchestrator → Researcher (dispatch) → Architect (file) → Planner (file) → Implementer (file, tier escalation 3→2) → Tester (file, same tier) → Reviewer (file, tier de-escalation 2→3) → Knowledge (file) → Orchestrator (pre-commit gate, tier escalation 3→1)

### Tool Access Security

- **Current system**: Instruction-based tool restrictions. Acknowledged "accepted risk (residual threat level: Medium)."
- **New system**: VS Code native `tools` YAML frontmatter provides platform-enforced restrictions. If an agent's tools list doesn't include `run_in_terminal`, the platform prevents the agent from using it. This upgrades enforcement from Medium-risk instruction-based to Low-risk platform-enforced.

### Implementer Command Allowlist (R2 — SS-S-1)

Implementer terminal commands are instruction-scoped. Expected patterns: `dotnet build`, `dotnet test`, `npm run build`, `npm test`, `cargo build`, `cargo test`, `go build`, `go test`, `pytest`, `git diff`. All commands logged in `implementation_report.commands_executed[]`. Tester and Reviewer audit against this allowlist — unexpected commands trigger a Major finding.

### R2: File Ownership Enforcement (AG-A-3)

Implementer MUST NOT modify files not declared in `task.files[]`. If additional files are needed, implementer returns `NEEDS_REVISION` with `files_requested[]`. Orchestrator/Planner reviews and re-dispatches with updated ownership.

### R2: Pre-Commit Output Validation (SS-S-3)

At Step 8, orchestrator validates git diff against Knowledge output allowlist before committing. Only evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md, and files in docs/feature/<slug>/ are permitted. Unexpected files trigger commit rejection and ERROR status.

### Remaining Instruction-Level Restrictions

- **Implementer `run_in_terminal` scope**: Has terminal access but constrained to build/test commands by instructions. The platform can't distinguish `dotnet test` from `dotnet run`. R2: Mitigated by command logging + reviewer audit.
- **Knowledge file write scope**: Has `create_file` but constrained to output files only. Cannot modify `.agent.md` files by instruction. R2: Mitigated by pre-commit validation.

### Web Research Security

- `fetch_webpage` requires user approval per invocation in interactive mode (VS Code platform behavior)
- In autonomous mode, web research is disabled by the `web_research_enabled` parameter
- R2: **Residual risk accepted** — in autonomous mode, instruction toggle is sole guard. Agents MUST log all fetch_webpage URLs in output. Reviewer audits URL trails. (D-10 autonomous_mode_residual_risk)
- No unsupervised web access possible in interactive mode

---

## Failure & Recovery

### Pipeline Failure Modes

| Failure Mode                           | Detection                        | Recovery                                               | Max Retries |
| -------------------------------------- | -------------------------------- | ------------------------------------------------------ | ----------- |
| Agent returns malformed YAML           | Orchestrator parse failure       | Retry once with same agent                             | 1           |
| R2: Agent returns unknown status       | Orchestrator validation          | Treat as ERROR (EC-6)                                  | 0           |
| All researchers fail                   | <2 DONE contracts                | Halt with ERROR                                        | 0           |
| Task DAG has cycle                     | Orchestrator cycle detection     | Return to Planner with NEEDS_REVISION                  | 1           |
| File ownership collision at runtime    | Tester/Reviewer detects conflict | Re-run affected task                                   | 1           |
| Tests fail after implementation        | Tester returns NEEDS_REVISION    | Route to Step 5 (re-implement)                         | 3 cycles    |
| Code review fails                      | Review gate not met              | Route to Step 5 with findings                          | 2 rounds    |
| Build failure during testing           | Tester returns NEEDS_REVISION    | Route to Step 5 with build error                       | 3 cycles    |
| R2: Embedded design review fails (🔴)  | Review gate not met              | Re-dispatch Architect (max 1 round), then ERROR (EC-7) | 1           |
| R2: Knowledge writes outside allowlist | Pre-commit git diff check        | Commit rejected, pipeline ERROR (EC-8)                 | 0           |

### Feedback Loop Limits (from global-rules.md)

- **Implementation-Testing loop**: Max 3 cycles (Step 5 → Step 6 → Step 5). R2: Counter resets to 0 at start of each Code Review round.
- **Code Review rounds**: Max 2 rounds (Step 7 → Step 5 → Step 6 → Step 7)
- **R2: Design Review loop (🔴 only)**: Max 1 round (Step 3 → Reviewers → Step 3). Failure after 1 round → ERROR.
- **Per-task retries**: Max 3 re-dispatches for a single task
- **Transient error retries**: Max 2 retries (network, tool unavailable)
- **Deterministic failures**: Never retry (file not found, permission denied)

---

## Non-Functional Requirements

| NFR                          | Target              | Design Approach                                              |
| ---------------------------- | ------------------- | ------------------------------------------------------------ |
| Total size ≤1,500 lines      | ~1,100 lines        | Per-agent budgets, co-located schemas, one shared doc        |
| Per-agent ≤150 lines         | 100-150 per agent   | 6-section body structure, concise instructions               |
| ≤2 shared reference docs     | 1 (global-rules.md) | Everything else inlined or in YAML frontmatter               |
| Zero § cross-references      | 0                   | Co-located schemas, platform tool enforcement                |
| git-only external dependency | git                 | SQLite removed, playwright-cli is project-specific           |
| ≤25 dispatches for 🔴        | ~20-25              | 8 steps, parallel research/review, DAG implementation        |
| Industry traceability        | 15 decisions traced | Each decision cites SWE-Agent, MetaGPT, AutoGen, OWASP, etc. |

---

## Migration & Backwards Compatibility

### Migration Strategy

1. **Phase 1 — Build**: Create all new system files in `v2/.github/agents/` staging directory
2. **Phase 2 — Verify**: Run acceptance criteria tests against staging directory (AC-1 through AC-22)
3. **Phase 3 — Promote**: Archive existing `.github/agents/` → `.github/agents-archive/`, copy `v2/.github/agents/` → `.github/agents/`
4. **Phase 4 — Cleanup**: Verify VS Code discovers new agents. Archive NewAgents/ and Forge/ directories.

### copilot-instructions.md Update

The existing `.github/copilot-instructions.md` needs these modifications:

- **Remove**: SQL safety rule (#9 — sql_escape, stdin piping)
- **Remove**: References to verification-ledger.db
- **Update**: Anti-hallucination rule (#5) — change from "INSERT into SQLite BEFORE reporting" to "produce output YAML BEFORE reporting"
- **Keep unchanged**: Terminal-only testing, no file-redirect, retry policy, schema version, no runtime modification, bounded loops, output paths

### What Remains Unchanged

- `docs/feature/` directory structure (all existing feature documentation preserved)
- Git repository structure (no branch changes)
- Anvil agent (independent system, unaffected)
- Project-specific tooling (playwright-cli, dotnet, npm — these are project concerns, not system concerns)

---

## Testing Strategy

### Acceptance Criteria Verification

The 22 acceptance criteria from the spec map to these test categories:

**Automated Tests (AC-1, AC-2, AC-8, AC-18):**

```powershell
# AC-1: Total size ≤1,500 lines
(Get-ChildItem v2/.github/agents/ -Recurse *.agent.md,*.md | Get-Content | Measure-Object -Line).Lines

# AC-2: Per-agent ≤150 lines
Get-ChildItem v2/.github/agents/*.agent.md | ForEach-Object { "$($_.Name): $((Get-Content $_ | Measure-Object -Line).Lines)" }

# AC-8: No SQLite references
Select-String -Path v2/.github/agents/*.md -Pattern "sqlite|verification-ledger|sql-templates" -CaseSensitive:$false

# AC-18: No § cross-references
Select-String -Path v2/.github/agents/*.md -Pattern "§"
```

**Structural Inspection (AC-3 through AC-7, AC-10 through AC-17, AC-19 through AC-22):**
Verified by Tester agent reading the agent files and checking structural properties.

### Implementation Testing

Each implementation task follows TDD:

1. Implementer writes agent file content
2. Implementer verifies line count ≤150 (unit test equivalent)
3. Implementer verifies YAML frontmatter structure
4. Tester verifies cross-agent consistency

---

## Tradeoffs & Alternatives Considered

### Key Tradeoffs

| Tradeoff                                     | Chose                         | Over                                            | Reason                                             |
| -------------------------------------------- | ----------------------------- | ----------------------------------------------- | -------------------------------------------------- |
| 8 agents                                     | Moderate specialization       | 5 agents (over-merged) or 11 (over-specialized) | Sweet spot per spec, preserves all proven patterns |
| YAML evidence                                | Simplicity, zero dependencies | SQLite queryability                             | Evidence gates never fired in 5 runs               |
| Inline schemas                               | Co-location, zero cross-refs  | Central schemas.md                              | Eliminates #1 Critical finding source              |
| Instruction-level run_in_terminal constraint | Simplicity                    | Separate agent variants                         | Caught by mandatory code review                    |
| Single global-rules.md                       | DRY for universal rules       | Zero shared docs (too duplicative) or 2+ docs   | 6 rules × 8 agents = too much repetition to inline |

### Architectural Patterns Used (Anthropic taxonomy)

1. **Prompt Chaining** — Sequential pipeline steps (Research → Architecture → Plan → ...)
2. **Routing** — Risk-based scaling (🟢/🟡/🔴 determines step depth)
3. **Parallelization** — Researcher ×4, Reviewer ×3, Implementer ×4
4. **Orchestrator-Workers** — Hub-and-spoke with orchestrator as coordinator
5. **Evaluator-Optimizer** — Implementation → Testing → Re-implementation loop

All 5 patterns from Anthropic's guide are present but now serve the simplified 8-step pipeline instead of the previous 10+ step monolith with 936 lines of SQL infrastructure.

---

## Implementation Checklist & Deliverables

### File Inventory (11 new files)

| #   | File                                            | Lines | Purpose                                            |
| --- | ----------------------------------------------- | ----- | -------------------------------------------------- |
| 1   | `v2/.github/agents/orchestrator.agent.md`       | ~150  | Pipeline coordination, DAG dispatch                |
| 2   | `v2/.github/agents/researcher.agent.md`         | ~100  | Parallel web + codebase research                   |
| 3   | `v2/.github/agents/architect.agent.md`          | ~140  | Combined spec + design                             |
| 4   | `v2/.github/agents/planner.agent.md`            | ~130  | DAG task decomposition                             |
| 5   | `v2/.github/agents/implementer.agent.md`        | ~130  | TDD implementation                                 |
| 6   | `v2/.github/agents/tester.agent.md`             | ~140  | Dynamic testing + evidence evaluation              |
| 7   | `v2/.github/agents/reviewer.agent.md`           | ~110  | Adversarial code review                            |
| 8   | `v2/.github/agents/knowledge.agent.md`          | ~100  | Post-mortem + instruction optimization             |
| 9   | `v2/.github/agents/global-rules.md`             | ~100  | Cross-cutting rules                                |
| 10  | `v2/.github/prompts/feature-workflow.prompt.md` | ~30   | Full pipeline entry point                          |
| 11  | `v2/.github/prompts/quick-fix.prompt.md`        | ~25   | Simplified pipeline — steps {1,5,6,7,8} (R2: D-13) |

**Total: ~1,155 lines** (within 1,500 budget, ~345 margin)

### Modified Files

| File                              | Change                                         |
| --------------------------------- | ---------------------------------------------- |
| `.github/copilot-instructions.md` | Remove SQLite rules, update anti-hallucination |

### Implementation Order Recommendation

1. **Foundation**: global-rules.md (defines completion contract all agents use)
2. **Core**: orchestrator.agent.md (pipeline coordinator — everything else depends on it)
3. **Input agents**: researcher.agent.md, architect.agent.md (produce inputs for planning)
4. **Execution agents**: planner.agent.md, implementer.agent.md (DAG planning + TDD implementation)
5. **Verification agents**: tester.agent.md, reviewer.agent.md (quality assurance)
6. **Completion**: knowledge.agent.md (post-pipeline analysis)
7. **Entry points**: feature-workflow.prompt.md, quick-fix.prompt.md
8. **Promotion**: Archive old system, copy new system to .github/agents/

---

## Decisions Cross-Reference

Every decision in this document corresponds 1:1 to a decision in design-output.yaml:

| ID       | Title                                                                             | Risk   | Confidence |
| -------- | --------------------------------------------------------------------------------- | ------ | ---------- |
| D-1      | System Directory Structure — staging with promotion                               | 🟢     | High       |
| D-2      | Agent Inventory — 8 agents via strategic merges                                   | 🟡     | High       |
| D-3      | Agent File Format — YAML frontmatter + 6-section body                             | 🟢     | High       |
| D-4      | State Management — YAML file-based, zero SQLite + incremental logging (R2)        | 🟡     | High       |
| D-5      | Pipeline Structure — 8 steps with risk-based scaling                              | 🟢     | High       |
| D-6      | DAG Task Execution — dependency-driven dispatch + parallel failure semantics (R2) | 🟡     | High       |
| D-7      | Tester Agent — dual-mode with singleton constraint + split trigger (R2)           | 🟡     | Medium     |
| D-8      | Line Budget Strategy — hard caps with margin + orchestrator fallback (R2)         | 🟢     | High       |
| D-9      | Shared Reference Documents — single global-rules.md + schema evolution ack (R2)   | 🟢     | High       |
| D-10     | Web Research Integration — parameter-based toggle + autonomous risk (R2)          | 🟢     | Medium     |
| D-11     | Orchestrator Routing — completion contract + file existence + YAML parse (R2)     | 🟢     | High       |
| D-12     | Cross-File Reference Elimination — co-location                                    | 🟢     | High       |
| **D-13** | **Quick-Fix Step Set — {1,5,6,7,8} (R2 NEW)**                                     | **🟢** | **High**   |
| **D-14** | **Trust Boundary Model — 3-tier trust (R2 NEW)**                                  | **🟡** | **Medium** |
| **D-15** | **Incremental Pipeline Logging — pipeline-log.yaml (R2 NEW)**                     | **🟢** | **High**   |

**Medium confidence decisions to revisit:**

- **D-7 (Tester dual-mode)**: If the dual-mode Tester can't fit in 150 lines, split into Tester + Verifier (9 agents). R2: Split trigger — >140 lines OR >20 lines duplicated across workflows.
- **D-10 (Web research toggle)**: The instruction-level toggle for fetch_webpage is acceptable given VS Code's per-invocation approval. R2: Autonomous residual risk documented — agents must log URLs, reviewer audits.
- **D-14 (Trust boundary model)**: Trust tiers are instruction-enforced, not sandboxed. Mitigated by YAML frontmatter + Code Review + pre-commit validation. If violations detected, consider per-agent subprocess sandboxing (future).
