# Schemas Reference — v1.0

> **Type:** Shared reference document (not an agent definition)
> **Purpose:** Single-source-of-truth for all typed YAML schema definitions used across the 9-agent deterministic pipeline. Every agent output conforms to one of these schemas. This document defines required fields, types, allowed values, and provides one complete example per schema.
> **Source:** Derived from [design.md](../../../docs/feature/next-gen-multi-agent-system/design.md) §Decision 4, §Decision 5, §Decision 6, and §Schemas Reference.

---

## Schema Version Requirement

Every agent output MUST include `schema_version: "1.0"` in the common header. This field is required for all agent outputs and enables forward-compatible evolution. See [Schema Evolution Strategy](#schema-evolution-strategy) for version increment policy.

---

## Common Agent Output Header

All agent outputs share this common header structure. The `payload` section is agent-specific and defined by the relevant schema below.

```yaml
agent_output:
  agent: "<agent name>" # string, required — one of the 9 agent names
  instance: "<instance identifier>" # string, required — unique instance ID (e.g., "researcher-architecture")
  step: "<pipeline step>" # string, required — pipeline step number (e.g., "step-1", "step-3b")
  started_at: "<ISO8601>" # string, required — ISO 8601 timestamp
  completed_at: "<ISO8601>" # string, required — ISO 8601 timestamp
  schema_version: "1.0" # string, required — schema version
  payload:
    # Agent-specific payload defined per schema below
```

### Human-Readable Companions

For artifacts that humans need to read (design.md, feature.md, plan.md), agents produce BOTH:

1. **Primary:** Typed YAML output (machine-readable, used for routing)
2. **Companion:** Markdown file (human-readable, generated from the same data)

The orchestrator reads ONLY the typed YAML for routing decisions. The Markdown companions are for human consumption and auditability.

---

## Producer/Consumer Dependency Table

| #   | Schema                  | Producer                                          | Consumers                                          | Purpose                                                                                    |
| --- | ----------------------- | ------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| 1   | `completion-contract`   | All agents (shared header)                        | Orchestrator                                       | Three-state completion signal; embedded in every agent output                              |
| 2   | `research-output`       | Researcher                                        | Spec, Designer                                     | Structured research findings                                                               |
| 3   | `spec-output`           | Spec                                              | Designer, Planner, Implementer, Verifier           | Structured requirements + acceptance criteria                                              |
| 4   | `design-output`         | Designer                                          | Planner, Implementer, Adversarial Reviewer         | Structured design decisions + justifications                                               |
| 5   | `plan-output`           | Planner                                           | Orchestrator (risk summary), Implementer, Verifier | Task list + risk classifications + wave assignments + `overall_risk_summary`               |
| 6   | `task-schema`           | Planner                                           | Implementer, Verifier                              | Per-task spec with risk level + `relevant_context` pointers (sub-schema of `plan-output`)  |
| 7   | `implementation-report` | Implementer                                       | Verifier                                           | Baseline records + change summary + self-check + verification ledger entries               |
| 8   | `verification-report`   | Verifier                                          | Orchestrator (gate_status), Adversarial Reviewer   | Cascade results + evidence gate summary + regression analysis (complements SQL ledger)     |
| 9   | `review-findings`       | Adversarial Reviewer                              | Orchestrator (verdict routing), Knowledge Agent    | Per-perspective findings in Markdown + YAML verdict summary + SQL INSERT of verdict record |
| 10  | `knowledge-output`      | Knowledge Agent                                   | (terminal — no downstream consumers)               | Knowledge updates + decision log entries + store_memory calls                              |
| 11  | `artifact_evaluations`  | Implementer, Verifier, Adversarial Reviewer (SQL) | Knowledge Agent (SQL query)                        | SQLite table — per-artifact usefulness/clarity scores + qualitative feedback               |
| 12  | `instruction_updates`   | Knowledge Agent (SQL)                             | Orchestrator (audit query)                         | SQLite table — governed instruction file change tracking                                   |
| 13  | `e2e-contract`          | Project author / Researcher (interactive mode)    | Orchestrator (Step 0), Planner, Verifier (Tier 5)  | E2E contract schema — app lifecycle, interaction config, skills, browser tool config       |
| 14  | `e2e-skill`             | Project author / Researcher (Step 0.5)            | Verifier (Tier 5)                                  | E2E skill schema — reusable interaction procedures (test-suite, exploratory, adversarial)  |

---

## Schema 1: `completion-contract`

> Embedded in every agent output. The orchestrator reads this block for routing decisions.

### Fields

| Field                                | Type            | Required    | Constraint / Allowed Values                                                                       |
| ------------------------------------ | --------------- | ----------- | ------------------------------------------------------------------------------------------------- |
| `status`                             | string          | Yes         | Exactly one of: `DONE`, `NEEDS_REVISION`, `ERROR`                                                 |
| `summary`                            | string          | Yes         | ≤200 characters; one-line description                                                             |
| `severity`                           | string \| null  | Yes         | `Blocker`, `Critical`, `Major`, `Minor`, or `null`; required for review agents, `null` for others |
| `findings_count`                     | integer         | Yes         | ≥ 0; `0` if no findings                                                                           |
| `risk_level`                         | string \| null  | Yes         | `"🟢"`, `"🟡"`, `"🔴"`, or `null`; required for planner/reviewer, `null` for others               |
| `output_paths`                       | list of strings | Yes         | ≥ 1 entry; relative paths to output files produced                                                |
| `evidence_summary`                   | object \| null  | No          | For verification/review agents only; `null` for others                                            |
| `evidence_summary.total_checks`      | integer         | Conditional | Required when `evidence_summary` is present                                                       |
| `evidence_summary.passed`            | integer         | Conditional | Required when `evidence_summary` is present                                                       |
| `evidence_summary.failed`            | integer         | Conditional | Required when `evidence_summary` is present                                                       |
| `evidence_summary.security_blockers` | integer         | Conditional | Required when `evidence_summary` is present                                                       |

### Example

```yaml
completion:
  status: DONE
  summary: "Verified task-03: 8 checks passed, 0 failed, no regressions"
  severity: null
  findings_count: 0
  risk_level: null
  output_paths:
    - "verification-reports/task-03.yaml"
  evidence_summary:
    total_checks: 8
    passed: 8
    failed: 0
    security_blockers: 0
```

---

## Schema 2: `research-output`

> Produced by each Researcher instance (×4 parallel). One output per focus area.
> **Output path:** `research/<focus>.yaml` (e.g., `research/architecture.yaml`)
> **Format:** YAML-only (no MD companion — per DP-2 rationalization)

### Fields

| Field                           | Type            | Required | Constraint / Allowed Values                                                    |
| ------------------------------- | --------------- | -------- | ------------------------------------------------------------------------------ |
| `agent_output` (header)         | object          | Yes      | Common header (see above)                                                      |
| `payload.focus`                 | string          | Yes      | One of: `architecture`, `impact`, `dependencies`, `patterns`                   |
| `payload.findings`              | list of objects | Yes      | ≥ 1 entry                                                                      |
| `payload.findings[].id`         | string          | Yes      | Unique finding identifier (e.g., `F-1`, `F-2`)                                 |
| `payload.findings[].title`      | string          | Yes      | Short descriptive title                                                        |
| `payload.findings[].category`   | string          | Yes      | Classification of finding (e.g., `structure`, `pattern`, `risk`, `dependency`) |
| `payload.findings[].detail`     | string          | Yes      | Detailed finding description                                                   |
| `payload.findings[].evidence`   | list of strings | Yes      | ≥ 1 entry; file paths, code references, or observations                        |
| `payload.findings[].relevance`  | string          | Yes      | How this finding impacts downstream pipeline steps                             |
| `payload.summary`               | string          | Yes      | High-level summary of all findings                                             |
| `payload.source_files_examined` | list of strings | Yes      | ≥ 1 entry; files read during research                                          |
| `completion` (contract)         | object          | Yes      | See Schema 1                                                                   |

### Example

```yaml
agent_output:
  agent: "researcher"
  instance: "researcher-architecture"
  step: "step-1"
  started_at: "2026-02-26T14:31:00Z"
  completed_at: "2026-02-26T14:33:20Z"
  schema_version: "1.0"
  payload:
    focus: "architecture"
    findings:
      - id: "F-1"
        title: "Existing orchestrator uses 23 agents across 8 steps"
        category: "structure"
        detail: "The Forge orchestrator dispatches 23 unique agent definitions across an 8-step pipeline. Memory is managed via dual-layer system (shared + isolated) with ~12 merge operations per run."
        evidence:
          - ".github/agents/forge-orchestrator.agent.md"
          - ".github/agents/implementer.agent.md"
        relevance: "Establishes baseline agent count for comparison; identifies merge overhead as optimization target"
      - id: "F-2"
        title: "SQL verification ledger in Anvil uses anvil_checks table"
        category: "pattern"
        detail: "Anvil records every verification step as an INSERT into the anvil_checks SQLite table. Evidence gating uses COUNT queries. No verification exists without a corresponding SQL record."
        evidence:
          - "Anvil/anvil.agent.md"
        relevance: "Provides proven verification pattern for adoption in new pipeline"
    summary: "Forge uses 23 agents with prose-based communication. Anvil uses 1 agent with SQL evidence. Combining lean agent count with SQL verification is the key architectural opportunity."
    source_files_examined:
      - ".github/agents/forge-orchestrator.agent.md"
      - "Anvil/anvil.agent.md"
      - ".github/agents/implementer.agent.md"
completion:
  status: DONE
  summary: "Architecture research complete: 2 findings on agent structure and verification"
  severity: null
  findings_count: 2
  risk_level: null
  output_paths:
    - "research/architecture.yaml"
```

---

## Schema 3: `spec-output`

> Produced by the Spec agent. Contains structured requirements and acceptance criteria.
> **Output path:** `spec-output.yaml`
> **Companion:** `feature.md` (human-readable)

### Fields

| Field                                                | Type            | Required | Constraint / Allowed Values                                       |
| ---------------------------------------------------- | --------------- | -------- | ----------------------------------------------------------------- |
| `agent_output` (header)                              | object          | Yes      | Common header                                                     |
| `payload.feature_name`                               | string          | Yes      | Name of the feature being specified                               |
| `payload.directions`                                 | list of objects | Yes      | ≥ 1 entry; architectural directions evaluated                     |
| `payload.directions[].id`                            | string          | Yes      | Direction identifier (e.g., `A`, `B`, `C`)                        |
| `payload.directions[].name`                          | string          | Yes      | Short name                                                        |
| `payload.directions[].summary`                       | string          | Yes      | Brief description                                                 |
| `payload.common_requirements`                        | list of objects | Yes      | ≥ 1 entry                                                         |
| `payload.common_requirements[].id`                   | string          | Yes      | Requirement ID (e.g., `CR-1`)                                     |
| `payload.common_requirements[].text`                 | string          | Yes      | Requirement text                                                  |
| `payload.common_requirements[].priority`             | string          | Yes      | `must` \| `should` \| `may`                                       |
| `payload.functional_requirements`                    | list of objects | Yes      | ≥ 1 entry                                                         |
| `payload.functional_requirements[].id`               | string          | Yes      | Requirement ID (e.g., `FR-1`)                                     |
| `payload.functional_requirements[].text`             | string          | Yes      | Requirement text                                                  |
| `payload.functional_requirements[].sub_requirements` | list of objects | No       | Sub-items under a functional requirement                          |
| `payload.acceptance_criteria`                        | list of objects | Yes      | ≥ 1 entry                                                         |
| `payload.acceptance_criteria[].id`                   | string          | Yes      | Criteria ID (e.g., `AC-1`)                                        |
| `payload.acceptance_criteria[].text`                 | string          | Yes      | Acceptance criteria text                                          |
| `payload.acceptance_criteria[].test_method`          | string          | Yes      | How to verify (`inspection`, `demonstration`, `test`, `analysis`) |
| `payload.edge_cases`                                 | list of objects | No       | Edge cases and error scenarios                                    |
| `payload.constraints`                                | list of strings | No       | Platform or environment constraints                               |
| `completion` (contract)                              | object          | Yes      | See Schema 1                                                      |

### Example

```yaml
agent_output:
  agent: "spec"
  instance: "spec"
  step: "step-2"
  started_at: "2026-02-26T14:35:00Z"
  completed_at: "2026-02-26T14:40:00Z"
  schema_version: "1.0"
  payload:
    feature_name: "Next-Generation Multi-Agent Pipeline"
    directions:
      - id: "A"
        name: "Lean Pipeline"
        summary: "6-8 agents, zero-merge memory, deterministic steps"
    common_requirements:
      - id: "CR-1"
        text: "All agent outputs MUST use typed YAML schemas"
        priority: "must"
      - id: "CR-2"
        text: "Agent count MUST be between 4 and 15 unique definitions"
        priority: "must"
    functional_requirements:
      - id: "FR-1"
        text: "Pipeline orchestration"
        sub_requirements:
          - id: "FR-1.1"
            text: "The orchestrator MUST dispatch agents in a deterministic sequence"
    acceptance_criteria:
      - id: "AC-1"
        text: "Every agent output conforms to its typed YAML schema"
        test_method: "inspection"
      - id: "AC-2"
        text: "Agent count is between 4 and 15 unique definitions"
        test_method: "inspection"
    edge_cases:
      - id: "EC-1"
        text: "Agent returns malformed YAML — retry once, then ERROR"
    constraints:
      - "Windows environment, VS Code GitHub Copilot agent framework"
completion:
  status: DONE
  summary: "Specification complete: 15 CR, 9 FR groups, 15 AC"
  severity: null
  findings_count: 0
  risk_level: null
  output_paths:
    - "spec-output.yaml"
    - "feature.md"
```

---

## Schema 4: `design-output`

> Produced by the Designer agent. Contains design decisions with justification scoring.
> **Output path:** `design-output.yaml`
> **Companion:** `design.md` (human-readable)

### Fields

| Field                                                    | Type            | Required    | Constraint / Allowed Values                          |
| -------------------------------------------------------- | --------------- | ----------- | ---------------------------------------------------- |
| `agent_output` (header)                                  | object          | Yes         | Common header                                        |
| `payload.architecture`                                   | string          | Yes         | Selected architecture summary                        |
| `payload.decisions`                                      | list of objects | Yes         | ≥ 1 entry; numbered design decisions                 |
| `payload.decisions[].id`                                 | string          | Yes         | Decision identifier (e.g., `D-1`, `D-2`)             |
| `payload.decisions[].title`                              | string          | Yes         | Decision title                                       |
| `payload.decisions[].risk`                               | string          | Yes         | `"🟢"` \| `"🟡"` \| `"🔴"`                           |
| `payload.decisions[].rationale`                          | string          | Yes         | Justification for the decision                       |
| `payload.decisions[].alternatives_rejected`              | list of objects | Yes         | ≥ 1 entry; alternatives considered and rejected      |
| `payload.decisions[].alternatives_rejected[].name`       | string          | Yes         | Alternative name                                     |
| `payload.decisions[].alternatives_rejected[].reason`     | string          | Yes         | Rejection rationale                                  |
| `payload.decisions[].alternatives_rejected[].confidence` | string          | Yes         | `High` \| `Medium` \| `Low`                          |
| `payload.agent_inventory`                                | list of objects | No          | Agent definitions summary (if architecture decision) |
| `payload.pipeline_steps`                                 | list of objects | No          | Pipeline step definitions (if pipeline decision)     |
| `payload.deviation_records`                              | list of objects | No          | Intentional spec deviations with rationale           |
| `payload.deviation_records[].id`                         | string          | Conditional | e.g., `DR-1`                                         |
| `payload.deviation_records[].spec_requirement`           | string          | Conditional | Which spec item is deviated from                     |
| `payload.deviation_records[].deviation`                  | string          | Conditional | What the design does differently                     |
| `payload.deviation_records[].rationale`                  | string          | Conditional | Why the deviation is justified                       |
| `completion` (contract)                                  | object          | Yes         | See Schema 1                                         |

### Example

```yaml
agent_output:
  agent: "designer"
  instance: "designer"
  step: "step-3"
  started_at: "2026-02-26T14:42:00Z"
  completed_at: "2026-02-26T14:55:00Z"
  schema_version: "1.0"
  payload:
    architecture: "Direction A Enhanced — 9 agents, zero-merge, typed YAML, SQL verification"
    decisions:
      - id: "D-1"
        title: "Architecture Selection"
        risk: "🟡"
        rationale: "Lean pipeline from Direction A, enhanced with D's typed schemas and B's specialized verification"
        alternatives_rejected:
          - name: "Direction C (Anvil-Core)"
            reason: "Mega-agents create context window fragility"
            confidence: "High"
          - name: "Direction E (Event-Driven)"
            reason: "Over-engineering for sequential agent framework"
            confidence: "High"
      - id: "D-2"
        title: "Agent Inventory — 9 Definitions"
        risk: "🟢"
        rationale: "Minimal agent count within 4-15 range; each agent has single responsibility"
        alternatives_rejected:
          - name: "23-agent Forge model"
            reason: "Excessive orchestration overhead; CT cluster duplication"
            confidence: "High"
    deviation_records:
      - id: "DR-1"
        spec_requirement: "FR-1.6"
        deviation: "Orchestrator uses run_in_terminal for SQLite queries and Git operations"
        rationale: "Independent evidence verification requires SQL access; SQLite init must happen before any agent dispatch"
    agent_inventory:
      - name: "Orchestrator"
        role: "Lean dispatch coordinator"
        responsibilities: 5
      - name: "Researcher"
        role: "Codebase investigation"
        instances: 4
completion:
  status: DONE
  summary: "Design complete: 10 decisions, 9 agents, 2 deviation records"
  severity: null
  findings_count: 0
  risk_level: "🟡"
  output_paths:
    - "design-output.yaml"
    - "design.md"
```

---

## Schema 5: `plan-output`

> Produced by the Planner agent. Contains task decomposition, risk classifications, and wave assignments.
> **Output path:** `plan-output.yaml`
> **Companion:** `plan.md` (human-readable)
> **Sub-output:** `tasks/*.yaml` (per-task schemas — see Schema 6)

### Fields

| Field                            | Type            | Required | Constraint / Allowed Values                                     |
| -------------------------------- | --------------- | -------- | --------------------------------------------------------------- |
| `agent_output` (header)          | object          | Yes      | Common header                                                   |
| `payload.overall_risk_summary`   | string          | Yes      | `"🟢"` \| `"🟡"` \| `"🔴"`; overall feature risk assessment     |
| `payload.total_tasks`            | integer         | Yes      | ≥ 1                                                             |
| `payload.waves`                  | list of objects | Yes      | ≥ 1 entry; execution waves                                      |
| `payload.waves[].id`             | string          | Yes      | Wave identifier (e.g., `wave-1`)                                |
| `payload.waves[].tasks`          | list of strings | Yes      | ≥ 1 entry; task IDs in this wave                                |
| `payload.waves[].max_concurrent` | integer         | Yes      | ≤ 4; maximum parallel tasks in this wave                        |
| `payload.tasks`                  | list of objects | Yes      | ≥ 1 entry; task summaries (full task schemas in `tasks/*.yaml`) |
| `payload.tasks[].id`             | string          | Yes      | Task identifier (e.g., `task-01`)                               |
| `payload.tasks[].title`          | string          | Yes      | Task title                                                      |
| `payload.tasks[].size`           | string          | Yes      | `Standard` \| `Large`                                           |
| `payload.tasks[].risk`           | string          | Yes      | `"🟢"` \| `"🟡"` \| `"🔴"`                                      |
| `payload.tasks[].depends_on`     | list of strings | No       | Task IDs this task depends on; empty = no dependencies          |
| `payload.tasks[].agent`          | string          | Yes      | Assigned agent name                                             |
| `payload.dependency_graph`       | object          | No       | DAG representation of task dependencies                         |
| `completion` (contract)          | object          | Yes      | See Schema 1                                                    |

### Example

```yaml
agent_output:
  agent: "planner"
  instance: "planner"
  step: "step-4"
  started_at: "2026-02-26T15:10:00Z"
  completed_at: "2026-02-26T15:18:00Z"
  schema_version: "1.0"
  payload:
    overall_risk_summary: "🟡"
    total_tasks: 6
    waves:
      - id: "wave-1"
        tasks: ["task-01", "task-02", "task-03"]
        max_concurrent: 3
      - id: "wave-2"
        tasks: ["task-04", "task-05", "task-06"]
        max_concurrent: 3
    tasks:
      - id: "task-01"
        title: "Create schemas reference document"
        size: "Standard"
        risk: "🟢"
        depends_on: []
        agent: "implementer"
      - id: "task-02"
        title: "Define dispatch patterns and severity taxonomy"
        size: "Standard"
        risk: "🟢"
        depends_on: []
        agent: "implementer"
      - id: "task-03"
        title: "Add input validation to auth handler"
        size: "Large"
        risk: "🔴"
        depends_on: ["task-01"]
        agent: "implementer"
      - id: "task-04"
        title: "Implement orchestrator agent"
        size: "Large"
        risk: "🟡"
        depends_on: ["task-01", "task-02"]
        agent: "implementer"
      - id: "task-05"
        title: "Implement researcher agent"
        size: "Standard"
        risk: "🟢"
        depends_on: ["task-01"]
        agent: "implementer"
      - id: "task-06"
        title: "Write README with Mermaid pipeline diagram"
        size: "Standard"
        risk: "🟢"
        depends_on: ["task-04", "task-05"]
        agent: "implementer"
    dependency_graph:
      task-01: []
      task-02: []
      task-03: ["task-01"]
      task-04: ["task-01", "task-02"]
      task-05: ["task-01"]
      task-06: ["task-04", "task-05"]
completion:
  status: DONE
  summary: "Plan complete: 6 tasks in 2 waves, overall risk 🟡"
  severity: null
  findings_count: 0
  risk_level: "🟡"
  output_paths:
    - "plan-output.yaml"
    - "plan.md"
    - "tasks/task-01.yaml"
    - "tasks/task-02.yaml"
    - "tasks/task-03.yaml"
    - "tasks/task-04.yaml"
    - "tasks/task-05.yaml"
    - "tasks/task-06.yaml"
```

---

## Schema 6: `task-schema`

> Produced by the Planner as sub-output of `plan-output`. One file per task. Contains per-task specification with risk classification and `relevant_context` pointers to bound downstream agent reads.
> **Output path:** `tasks/<task-id>.yaml` (e.g., `tasks/task-03.yaml`)
>
> **Version 2.0 (breaking change):** `acceptance_criteria` changed from `list of strings` to `list of objects` with structured sub-fields `{id, text, test_method}`. This enables AC-to-test traceability and test_method propagation from spec through task to implementation. **Migration:** Planner (producer) must emit structured AC objects; Implementer and Verifier (consumers) must consume the object format. Existing string-format ACs are not forward-compatible.

### Fields

| Field                                          | Type            | Required    | Constraint / Allowed Values                                                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------- | --------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `task.id`                                      | string          | Yes         | Task identifier assigned by Planner (e.g., `task-03`)                                                                                                                                                                                                                                                                                                        |
| `task.title`                                   | string          | Yes         | Task title                                                                                                                                                                                                                                                                                                                                                   |
| `task.description`                             | string          | Yes         | Detailed description of work to be done                                                                                                                                                                                                                                                                                                                      |
| `task.agent`                                   | string          | Yes         | Assigned agent (e.g., `implementer`)                                                                                                                                                                                                                                                                                                                         |
| `task.size`                                    | string          | Yes         | `Standard` \| `Large`                                                                                                                                                                                                                                                                                                                                        |
| `task.risk`                                    | string          | Yes         | `"🟢"` \| `"🟡"` \| `"🔴"`                                                                                                                                                                                                                                                                                                                                   |
| `task.depends_on`                              | list of strings | No          | Task IDs this task depends on                                                                                                                                                                                                                                                                                                                                |
| `task.acceptance_criteria`                     | list of objects | Yes         | ≥ 1 entry; structured acceptance criteria (v2.0)                                                                                                                                                                                                                                                                                                             |
| `task.acceptance_criteria[].id`                | string          | Yes         | AC identifier from spec (e.g., `AC-1`)                                                                                                                                                                                                                                                                                                                       |
| `task.acceptance_criteria[].text`              | string          | Yes         | Testable acceptance criterion text                                                                                                                                                                                                                                                                                                                           |
| `task.acceptance_criteria[].test_method`       | string          | Yes         | `inspection` \| `demonstration` \| `test` \| `analysis`                                                                                                                                                                                                                                                                                                      |
| `task.relevant_context`                        | object          | Yes         | Pointers to specific upstream sections (bounds read amplification)                                                                                                                                                                                                                                                                                           |
| `task.relevant_context.design_sections`        | list of strings | Yes         | ≥ 1 entry; paths with section pointers into `design-output.yaml`                                                                                                                                                                                                                                                                                             |
| `task.relevant_context.spec_requirements`      | list of strings | Yes         | ≥ 1 entry; paths with section pointers into `spec-output.yaml`                                                                                                                                                                                                                                                                                               |
| `task.relevant_context.files_to_modify`        | list of objects | No          | Files to create or modify                                                                                                                                                                                                                                                                                                                                    |
| `task.relevant_context.files_to_modify[].path` | string          | Conditional | Relative file path                                                                                                                                                                                                                                                                                                                                           |
| `task.relevant_context.files_to_modify[].risk` | string          | Conditional | `"🟢"` \| `"🟡"` \| `"🔴"`                                                                                                                                                                                                                                                                                                                                   |
| `task.workflow_lane`                           | string          | Yes         | `'unit-only'` \| `'unit-integration'` \| `'full-tdd-e2e'`; default derived from risk (`🟢` → `unit-only`, `🟡` → `unit-integration`, `🔴` → `full-tdd-e2e`)                                                                                                                                                                                                  |
| `task.e2e_required`                            | boolean         | Yes         | Independent boolean: `true` when E2E contract exists AND planner assesses E2E adds verification value. Not gated on `workflow_lane` or risk level. Defaults to `true` for 🔴 tasks when contract exists; `true` or `false` for 🟢/🟡 at planner discretion; `false` when no contract exists. Can be explicitly set to `false` by planner for any risk level. |
| `task.e2e_contract_path`                       | string \| null  | Yes         | Path to discovered E2E contract; required when `e2e_required=true`; default `null`                                                                                                                                                                                                                                                                           |

### Example

```yaml
task:
  id: "task-03"
  title: "Add input validation to auth handler"
  description: "Implement server-side input validation for the authentication handler, including email format, password strength, and rate limiting checks."
  agent: "implementer"
  size: "Large"
  risk: "🔴"
  depends_on:
    - "task-01"
  acceptance_criteria:
    - id: "AC-1"
      text: "Email validation rejects malformed addresses"
      test_method: "test"
    - id: "AC-2"
      text: "Password strength check enforces minimum 12 characters"
      test_method: "test"
    - id: "AC-3"
      text: "Rate limiting returns 429 after 5 attempts per minute"
      test_method: "test"
    - id: "AC-4"
      text: "All validation errors return structured JSON error responses"
      test_method: "inspection"
  relevant_context:
    design_sections:
      - "design-output.yaml#payload.decisions[id='D-8']" # Risk classification
      - "design-output.yaml#payload.decisions[id='D-6']" # Verification architecture
    spec_requirements:
      - "spec-output.yaml#payload.functional_requirements[id='FR-4']" # Evidence gating
    files_to_modify:
      - path: "src/auth/handler.ts"
        risk: "🔴"
      - path: "src/auth/validator.ts"
        risk: "🟡"
```

---

## Schema 7: `implementation-report`

> Produced by each Implementer instance. Contains baseline records, change summary, and self-check results.
> **Output path:** `implementation-reports/<task-id>.yaml`

### Fields

| Field                                                 | Type            | Required    | Constraint / Allowed Values                                             |
| ----------------------------------------------------- | --------------- | ----------- | ----------------------------------------------------------------------- |
| `agent_output` (header)                               | object          | Yes         | Common header                                                           |
| `payload.task_id`                                     | string          | Yes         | Matches `task.id` from assigned task schema                             |
| `payload.task_type`                                   | string          | Yes         | `code` \| `documentation` \| `configuration`                            |
| `payload.baseline`                                    | object          | Yes         | Pre-implementation state capture                                        |
| `payload.baseline.ide_diagnostics`                    | object          | Yes         | IDE error/warning counts before changes                                 |
| `payload.baseline.ide_diagnostics.errors`             | integer         | Yes         | ≥ 0                                                                     |
| `payload.baseline.ide_diagnostics.warnings`           | integer         | Yes         | ≥ 0                                                                     |
| `payload.baseline.build_exit_code`                    | integer \| null | Yes         | Build exit code before changes; `null` if no build system               |
| `payload.baseline.test_summary`                       | object \| null  | Yes         | Test results before changes; `null` if no tests                         |
| `payload.baseline.test_summary.total`                 | integer         | Conditional | Total tests                                                             |
| `payload.baseline.test_summary.passed`                | integer         | Conditional | Passing tests                                                           |
| `payload.baseline.test_summary.failed`                | integer         | Conditional | Failing tests                                                           |
| `payload.changes`                                     | list of objects | Yes         | ≥ 1 entry; files created or modified                                    |
| `payload.changes[].path`                              | string          | Yes         | Relative file path                                                      |
| `payload.changes[].action`                            | string          | Yes         | `created` \| `modified` \| `deleted`                                    |
| `payload.changes[].description`                       | string          | Yes         | Brief description of the change                                         |
| `payload.self_check`                                  | object          | Yes         | Post-implementation self-verification results                           |
| `payload.self_check.ide_diagnostics`                  | object          | Yes         | IDE error/warning counts after changes                                  |
| `payload.self_check.ide_diagnostics.errors`           | integer         | Yes         | ≥ 0                                                                     |
| `payload.self_check.ide_diagnostics.warnings`         | integer         | Yes         | ≥ 0                                                                     |
| `payload.self_check.build_exit_code`                  | integer \| null | Yes         | Build exit code after changes; `null` if no build system                |
| `payload.self_check.test_summary`                     | object \| null  | Yes         | Test results after changes; `null` if no tests                          |
| `payload.self_check.self_fix_attempts`                | integer         | Yes         | 0–2; number of self-fix iterations performed                            |
| `payload.self_check.git_staged`                       | boolean         | Yes         | `true` if `git add -A` was executed after completion                    |
| `payload.verification_entries`                        | list of objects | No          | Baseline-phase SQL entries for `anvil_checks` (recorded by Implementer) |
| `payload.verification_entries[].check_name`           | string          | Conditional | Check name for SQL INSERT                                               |
| `payload.verification_entries[].phase`                | string          | Conditional | Always `baseline` for Implementer entries                               |
| `payload.verification_entries[].tool`                 | string          | Conditional | Tool used (e.g., `ide-get_diagnostics`, `dotnet build`)                 |
| `payload.verification_entries[].passed`               | boolean         | Conditional | Whether the check passed                                                |
| `payload.behavioral_coverage`                         | list of objects | Conditional | Required for `task_type='code'`; maps each AC to its verifying test     |
| `payload.behavioral_coverage[].ac_id`                 | string          | Yes         | AC identifier (e.g., `AC-1`)                                            |
| `payload.behavioral_coverage[].test_file`             | string          | Conditional | Relative path to test file; required when `status='covered'`            |
| `payload.behavioral_coverage[].test_name`             | string          | Conditional | Test function/method name; required when `status='covered'`             |
| `payload.behavioral_coverage[].status`                | string          | Yes         | `covered` \| `not_applicable`                                           |
| `payload.behavioral_coverage[].justification`         | string          | Conditional | Required when `status='not_applicable'`; explains why no test exists    |
| `payload.tdd_red_green`                               | object          | Conditional | Required for `task_type='code'` when TDD applies; records TDD cycle     |
| `payload.tdd_red_green.tests_written_first`           | boolean         | Yes         | `true` if tests were written before production code                     |
| `payload.tdd_red_green.initial_run_failures`          | integer         | Yes         | Number of test failures on initial (red) run                            |
| `payload.tdd_red_green.initial_run_exit_code`         | integer         | Yes         | Exit code of the initial (red) test run                                 |
| `payload.tdd_red_green.verify_phase`                  | object          | Conditional | Required for `task_type='code'`; VERIFY step results after GREEN phase  |
| `payload.tdd_red_green.verify_phase.get_errors_clean` | boolean         | Yes         | `true` if `get_errors` produced no new errors after GREEN phase         |
| `payload.tdd_red_green.verify_phase.typecheck_clean`  | boolean         | Yes         | `true` if type checker passed after GREEN phase                         |
| `payload.tdd_red_green.verify_phase.tests_passing`    | boolean         | Yes         | `true` if all tests pass after GREEN phase                              |
| `payload.tdd_red_green.tdd_fallback_reason`           | string \| null  | Yes         | Reason TDD was skipped; `null` when TDD was performed normally          |
| `completion` (contract)                               | object          | Yes         | See Schema 1                                                            |

> **`behavioral_coverage.status` values:**
>
> - `covered` — AC has a corresponding automated test that invokes production code. Requires `test_file` and `test_name`.
> - `not_applicable` — AC does not require an automated test. Requires `justification`. Valid reasons include:
>   - `test_method='inspection'` or `test_method='analysis'` — criterion verified by review, not automation
>   - `test_method='demonstration'` — criterion verified by runtime evidence (screenshot, log)
>   - TDD Fallback (EC-2) — `test_method='test'` but tests could not be written (no test framework, bootstrap task, or documented justification per TDD Fallback rules)

### Example

```yaml
agent_output:
  agent: "implementer"
  instance: "implementer-task-03"
  step: "step-5"
  started_at: "2026-02-26T15:25:00Z"
  completed_at: "2026-02-26T15:42:00Z"
  schema_version: "1.0"
  payload:
    task_id: "task-03"
    task_type: "code"
    baseline:
      ide_diagnostics:
        errors: 0
        warnings: 2
      build_exit_code: 0
      test_summary:
        total: 45
        passed: 45
        failed: 0
    changes:
      - path: "src/auth/handler.ts"
        action: "modified"
        description: "Added input validation middleware with email, password, and rate limit checks"
      - path: "src/auth/validator.ts"
        action: "created"
        description: "New validation utility module with email regex and password strength rules"
      - path: "tests/auth/validator.test.ts"
        action: "created"
        description: "Unit tests for email validation, password strength, and rate limiting"
    self_check:
      ide_diagnostics:
        errors: 0
        warnings: 2
      build_exit_code: 0
      test_summary:
        total: 52
        passed: 52
        failed: 0
      self_fix_attempts: 0
      git_staged: true
    verification_entries:
      - check_name: "baseline-ide-diagnostics"
        phase: "baseline"
        tool: "ide-get_diagnostics"
        passed: true
      - check_name: "baseline-build"
        phase: "baseline"
        tool: "dotnet build"
        passed: true
      - check_name: "baseline-tests"
        phase: "baseline"
        tool: "dotnet test"
        passed: true
    behavioral_coverage:
      - ac_id: "AC-1"
        test_file: "tests/auth/validator.test.ts"
        test_name: "rejects malformed email addresses"
        status: "covered"
      - ac_id: "AC-2"
        test_file: "tests/auth/validator.test.ts"
        test_name: "enforces minimum 12 character password"
        status: "covered"
      - ac_id: "AC-3"
        test_file: "tests/auth/validator.test.ts"
        test_name: "returns 429 after 5 attempts per minute"
        status: "covered"
      - ac_id: "AC-4"
        status: "not_applicable"
        justification: "test_method='inspection' — JSON response structure verified by code review"
    tdd_red_green:
      tests_written_first: true
      initial_run_failures: 3
      initial_run_exit_code: 1
completion:
  status: DONE
  summary: "Implemented task-03: input validation for auth handler, 7 new tests, 0 self-fixes"
  severity: null
  findings_count: 0
  risk_level: null
  output_paths:
    - "implementation-reports/task-03.yaml"
```

---

## Schema 8: `verification-report`

> Produced by each Verifier instance (one per task). Contains cascade results, evidence gate summary, and regression analysis. Complements the SQL ledger (`anvil_checks`) — if they disagree, SQL wins.
> **Output path:** `verification-reports/<task-id>.yaml`

### Fields

| Field                                                                   | Type            | Required    | Constraint / Allowed Values                                     |
| ----------------------------------------------------------------------- | --------------- | ----------- | --------------------------------------------------------------- |
| `agent_output` (header)                                                 | object          | Yes         | Common header                                                   |
| `payload.task_id`                                                       | string          | Yes         | Matches task being verified                                     |
| `payload.run_id`                                                        | string          | Yes         | Pipeline run identifier (ISO 8601)                              |
| `payload.evidence_gate`                                                 | object          | Yes         | Gate summary for orchestrator routing                           |
| `payload.evidence_gate.total_checks`                                    | integer         | Yes         | ≥ 1                                                             |
| `payload.evidence_gate.passed`                                          | integer         | Yes         | ≥ 0                                                             |
| `payload.evidence_gate.failed`                                          | integer         | Yes         | ≥ 0                                                             |
| `payload.evidence_gate.gate_status`                                     | string          | Yes         | `passed` \| `failed`                                            |
| `payload.findings`                                                      | list of objects | Yes         | ≥ 1 entry; individual check results                             |
| `payload.findings[].check_name`                                         | string          | Yes         | Check identifier used in SQL INSERT                             |
| `payload.findings[].tier`                                               | integer         | Yes         | 1–4; which verification tier                                    |
| `payload.findings[].phase`                                              | string          | Yes         | `baseline` \| `after`                                           |
| `payload.findings[].tool`                                               | string          | Yes         | Tool used for the check                                         |
| `payload.findings[].command`                                            | string \| null  | No          | Command executed; `null` if not applicable                      |
| `payload.findings[].exit_code`                                          | integer \| null | No          | Command exit code; `null` if not applicable                     |
| `payload.findings[].passed`                                             | boolean         | Yes         | Whether the check passed                                        |
| `payload.findings[].output_snippet`                                     | string \| null  | No          | ≤ 500 chars; first 500 characters of output                     |
| `payload.regressions`                                                   | list of objects | No          | Checks that passed in baseline but failed in after              |
| `payload.regressions[].check_name`                                      | string          | Conditional | Name of the regressed check                                     |
| `payload.regressions[].baseline_result`                                 | boolean         | Conditional | Always `true` (passed in baseline)                              |
| `payload.regressions[].after_result`                                    | boolean         | Conditional | Always `false` (failed in after)                                |
| `payload.regressions[].detail`                                          | string          | Conditional | Description of the regression                                   |
| `payload.baseline_cross_check`                                          | object \| null  | No          | Results of independent baseline verification via `git show`     |
| `payload.baseline_cross_check.method`                                   | string          | Conditional | Always `"git show pipeline-baseline-{run_id}:<path>"`           |
| `payload.baseline_cross_check.discrepancies_found`                      | boolean         | Conditional | `true` if Implementer baseline claims don't match               |
| `payload.e2e_results`                                                   | object \| null  | No          | E2E verification results; `null` when `e2e_required=false`      |
| `payload.e2e_results.suite_passed`                                      | boolean \| null | Conditional | Test suite execution result; `null` if no test-suite skills     |
| `payload.e2e_results.exploratory_passed`                                | boolean \| null | Conditional | Exploratory interaction result; `null` if no exploratory skills |
| `payload.e2e_results.adversarial_passed`                                | boolean \| null | Conditional | Adversarial interaction result; `null` if no adversarial skills |
| `payload.e2e_results.interaction_logs`                                  | list of objects | Conditional | Per-skill interaction logs                                      |
| `payload.e2e_results.interaction_logs[].skill_id`                       | string          | Yes         | Skill identifier                                                |
| `payload.e2e_results.interaction_logs[].skill_type`                     | string          | Yes         | `test-suite` \| `exploratory` \| `adversarial`                  |
| `payload.e2e_results.interaction_logs[].steps_executed`                 | integer         | Yes         | Total steps executed                                            |
| `payload.e2e_results.interaction_logs[].steps_passed`                   | integer         | Yes         | Steps that passed assertions                                    |
| `payload.e2e_results.interaction_logs[].steps_failed`                   | integer         | Yes         | Steps that failed assertions                                    |
| `payload.e2e_results.interaction_logs[].step_results`                   | list of objects | Yes         | Per-step result details                                         |
| `payload.e2e_results.interaction_logs[].step_results[].order`           | integer         | Yes         | Step execution order                                            |
| `payload.e2e_results.interaction_logs[].step_results[].action`          | string          | Yes         | Action performed                                                |
| `payload.e2e_results.interaction_logs[].step_results[].target`          | string          | Yes         | Target of the action                                            |
| `payload.e2e_results.interaction_logs[].step_results[].result`          | string          | Yes         | `passed` \| `failed` \| `skipped`                               |
| `payload.e2e_results.interaction_logs[].step_results[].actual_behavior` | string \| null  | No          | Observed behavior description                                   |
| `payload.e2e_results.interaction_logs[].step_results[].evidence_path`   | string \| null  | No          | Path to evidence file (screenshot, HAR, etc.)                   |
| `payload.e2e_results.interaction_logs[].step_results[].duration_ms`     | integer         | Yes         | Time taken for this step in milliseconds                        |
| `payload.e2e_results.artifact_paths`                                    | list of strings | Conditional | Paths to evidence files (screenshots, traces, videos)           |
| `payload.e2e_results.evidence_manifest_path`                            | string \| null  | Conditional | Path to evidence-manifest.yaml                                  |
| `completion` (contract)                                                 | object          | Yes         | See Schema 1                                                    |

### Example

```yaml
agent_output:
  agent: "verifier"
  instance: "verifier-task-03"
  step: "step-6"
  started_at: "2026-02-26T15:45:00Z"
  completed_at: "2026-02-26T15:50:00Z"
  schema_version: "1.0"
  payload:
    task_id: "task-03"
    run_id: "2026-02-26T14:30:00Z"
    evidence_gate:
      total_checks: 8
      passed: 8
      failed: 0
      gate_status: "passed"
    findings:
      - check_name: "ide-diagnostics"
        tier: 1
        phase: "after"
        tool: "ide-get_diagnostics"
        command: null
        exit_code: null
        passed: true
        output_snippet: "0 errors, 2 warnings (pre-existing)"
      - check_name: "syntax-check"
        tier: 1
        phase: "after"
        tool: "tsc --noEmit"
        command: "tsc --noEmit"
        exit_code: 0
        passed: true
        output_snippet: null
      - check_name: "build"
        tier: 2
        phase: "after"
        tool: "npm run build"
        command: "npm run build"
        exit_code: 0
        passed: true
        output_snippet: "Build completed successfully in 3.2s"
      - check_name: "lint"
        tier: 2
        phase: "after"
        tool: "eslint"
        command: "npx eslint src/auth/"
        exit_code: 0
        passed: true
        output_snippet: null
      - check_name: "tests"
        tier: 2
        phase: "after"
        tool: "jest"
        command: "npx jest --testPathPattern=auth"
        exit_code: 0
        passed: true
        output_snippet: "52 tests passed, 0 failed"
      - check_name: "import-check"
        tier: 3
        phase: "after"
        tool: "node"
        command: 'node -e "require(''./src/auth/validator'')"'
        exit_code: 0
        passed: true
        output_snippet: null
      - check_name: "smoke-execution"
        tier: 3
        phase: "after"
        tool: "node"
        command: 'node -e "const v = require(''./src/auth/validator''); console.log(v.validateEmail(''test@example.com''))"'
        exit_code: 0
        passed: true
        output_snippet: "true"
      - check_name: "baseline-ide-diagnostics"
        tier: 1
        phase: "baseline"
        tool: "ide-get_diagnostics"
        command: null
        exit_code: null
        passed: true
        output_snippet: "0 errors, 2 warnings"
    regressions: []
    baseline_cross_check:
      method: "git show pipeline-baseline-2026-02-26T14:30:00Z:src/auth/handler.ts"
      discrepancies_found: false
completion:
  status: DONE
  summary: "Verified task-03: 8 checks passed, 0 failed, no regressions"
  severity: null
  findings_count: 0
  risk_level: null
  output_paths:
    - "verification-reports/task-03.yaml"
  evidence_summary:
    total_checks: 8
    passed: 8
    failed: 0
    security_blockers: 0
```

---

## Schema 9: `review-findings`

> Produced by each Adversarial Reviewer instance (×3 parallel, one per perspective). Each reviewer produces three outputs:
>
> 1. **Markdown findings file** — detailed human-readable findings at `review-findings/<scope>-<perspective>.md`
> 2. **YAML verdict summary** — machine-readable verdict at `review-verdicts/<scope>-<perspective>.yaml`
> 3. **SQL INSERT** — verdict record into `anvil_checks` with `phase='review'`
>
> This schema defines the YAML verdict summary. The Markdown findings file is free-form prose. The SQL INSERT follows the `anvil_checks` schema defined in [SQLite Schemas](#sqlite-schemas).
>
> **Verdict File Naming Convention:**
>
> - Path: `review-verdicts/<scope>-<perspective>.yaml`
> - `scope`: `"design"` or `"code"`
> - `perspective`: `"security-sentinel"`, `"architecture-guardian"`, or `"pragmatic-verifier"`
> - Examples: `review-verdicts/design-security-sentinel.yaml`, `review-verdicts/code-pragmatic-verifier.yaml`
> - Findings follow same pattern: `review-findings/<scope>-<perspective>.md`
> - Consumers use `list_dir` on `review-verdicts/` then filter by scope prefix (e.g., `design-*` for all design verdicts)

### YAML Verdict Summary Fields

| Field                     | Type    | Required | Constraint / Allowed Values                                                     |
| ------------------------- | ------- | -------- | ------------------------------------------------------------------------------- |
| `reviewer_perspective`    | string  | Yes      | `security-sentinel` \| `architecture-guardian` \| `pragmatic-verifier`          |
| `scope`                   | string  | Yes      | `design` \| `code`                                                              |
| `verdicts`                | object  | Yes      | Per-category verdicts (one per review category)                                 |
| `verdicts.security`       | string  | Yes      | `approve` \| `needs_revision` \| `blocker`                                      |
| `verdicts.architecture`   | string  | Yes      | `approve` \| `needs_revision` \| `blocker`                                      |
| `verdicts.correctness`    | string  | Yes      | `approve` \| `needs_revision` \| `blocker`                                      |
| `overall`                 | string  | Yes      | `approve` \| `needs_revision` \| `blocker` — aggregate of per-category verdicts |
| `findings_count`          | object  | Yes      | Breakdown by severity                                                           |
| `findings_count.blocker`  | integer | Yes      | ≥ 0                                                                             |
| `findings_count.critical` | integer | Yes      | ≥ 0                                                                             |
| `findings_count.major`    | integer | Yes      | ≥ 0                                                                             |
| `findings_count.minor`    | integer | Yes      | ≥ 0                                                                             |
| `summary`                 | string  | Yes      | ≤ 500 characters; one-paragraph summary of review outcome                       |

### Review Focus Area Details

| Focus          | Scope                                                                                |
| -------------- | ------------------------------------------------------------------------------------ |
| `security`     | Injection vectors, authentication gaps, data exposure, secrets, authorization bypass |
| `architecture` | Coupling, scalability boundaries, component responsibilities, API surface area       |
| `correctness`  | Edge cases, logic errors, spec compliance, testing gaps, error handling              |

### SQL INSERT Convention for Review Records

> Each reviewer INSERTs **3 rows** — one per category (`security`, `architecture`, `correctness`) — with the `instance` column set to the reviewer's perspective ID. This 3-INSERT-per-reviewer pattern enables per-category evidence gate queries in [sql-templates.md](sql-templates.md) §6 (EG-5, EG-6).

When `phase='review'` in `anvil_checks`:

| Column           | Value                                                                                                                                                                                     |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `task_id`        | Per `task_id` convention (see below)                                                                                                                                                      |
| `phase`          | `'review'`                                                                                                                                                                                |
| `check_name`     | `'review-{scope}-{category}'` where category is `security` \| `architecture` \| `correctness` (e.g., `'review-code-security'`, `'review-code-architecture'`, `'review-code-correctness'`) |
| `instance`       | Reviewer perspective ID (e.g., `'security-sentinel'`, `'architecture-guardian'`, `'pragmatic-verifier'`)                                                                                  |
| `tool`           | `'adversarial-review'`                                                                                                                                                                    |
| `command`        | `'adversarial-review'`                                                                                                                                                                    |
| `exit_code`      | `NULL`                                                                                                                                                                                    |
| `output_snippet` | First 500 chars of summary                                                                                                                                                                |
| `passed`         | `1` if category verdict is `'approve'`, else `0`                                                                                                                                          |
| `verdict`        | The category-level verdict (`approve` \| `needs_revision` \| `blocker`)                                                                                                                   |
| `severity`       | Highest finding severity for the category (`NULL` if clean approve)                                                                                                                       |
| `round`          | Current review round (1 or 2)                                                                                                                                                             |
| `run_id`         | Current pipeline run ID                                                                                                                                                                   |

### Example (YAML Verdict Summary)

```yaml
reviewer_perspective: "security-sentinel"
scope: "design"
verdicts:
  security: "needs_revision"
  architecture: "approve"
  correctness: "approve"
overall: "needs_revision"
findings_count:
  blocker: 0
  critical: 1
  major: 3
  minor: 7
summary: "No security blockers. 1 critical auth concern in token handling requiring rotation policy. 3 major findings around input sanitization in API endpoints."
```

---

## Schema 10: `knowledge-output`

> Produced by the Knowledge Agent. Terminal schema — no downstream consumers. Contains knowledge updates, decision log entries, and store_memory calls. Non-blocking: ERROR from this agent does not halt the pipeline.
> **Output path:** `knowledge-output.yaml`
> **Sub-output:** `decisions.yaml` (append-only decision log updates)
> **Sub-output:** `evidence-bundle.md` (Step 8b — assembled evidence bundle)

### Fields

| Field                                                       | Type            | Required    | Constraint / Allowed Values                                    |
| ----------------------------------------------------------- | --------------- | ----------- | -------------------------------------------------------------- |
| `agent_output` (header)                                     | object          | Yes         | Common header                                                  |
| `payload.knowledge_updates`                                 | list of objects | Yes         | ≥ 0 entries; patterns and conventions learned                  |
| `payload.knowledge_updates[].type`                          | string          | Conditional | `convention` \| `command` \| `pattern` \| `lesson`             |
| `payload.knowledge_updates[].key`                           | string          | Conditional | Short identifier for the knowledge entry                       |
| `payload.knowledge_updates[].value`                         | string          | Conditional | The knowledge content                                          |
| `payload.knowledge_updates[].stored_via`                    | string          | Conditional | `store_memory` \| `decisions.yaml`                             |
| `payload.decision_log_entries`                              | list of objects | No          | Append-only entries for `decisions.yaml`                       |
| `payload.decision_log_entries[].id`                         | string          | Conditional | Decision identifier                                            |
| `payload.decision_log_entries[].title`                      | string          | Conditional | Decision title                                                 |
| `payload.decision_log_entries[].rationale`                  | string          | Conditional | Why this decision was made                                     |
| `payload.decision_log_entries[].confidence`                 | string          | Conditional | `High` \| `Medium` \| `Low`                                    |
| `payload.evidence_bundle`                                   | object \| null  | No          | Step 8b evidence bundle summary (assembled by Knowledge Agent) |
| `payload.evidence_bundle.overall_confidence`                | string          | Conditional | `High` \| `Medium` \| `Low`                                    |
| `payload.evidence_bundle.verification_summary`              | object          | Conditional | Aggregated pass/fail/regression counts per task                |
| `payload.evidence_bundle.review_summary`                    | object          | Conditional | Verdicts per model, issues found and fixed                     |
| `payload.evidence_bundle.rollback_command`                  | string          | Conditional | Git revert command                                             |
| `payload.evidence_bundle.blast_radius`                      | object          | Conditional | Files modified, risk classifications, regression count         |
| `payload.evidence_bundle.known_issues`                      | list of objects | Conditional | Issues with severity ratings                                   |
| `payload.pipeline_telemetry_summary`                        | object \| null  | No          | Aggregated telemetry from `pipeline_telemetry` table           |
| `payload.pipeline_telemetry_summary.total_dispatches`       | integer         | Conditional | Total agent dispatches                                         |
| `payload.pipeline_telemetry_summary.total_duration_seconds` | number          | Conditional | Wall-clock pipeline duration                                   |
| `payload.pipeline_telemetry_summary.error_count`            | integer         | Conditional | Total ERROR statuses                                           |
| `completion` (contract)                                     | object          | Yes         | See Schema 1                                                   |

### Example

```yaml
agent_output:
  agent: "knowledge-agent"
  instance: "knowledge-agent"
  step: "step-8"
  started_at: "2026-02-26T16:20:00Z"
  completed_at: "2026-02-26T16:25:00Z"
  schema_version: "1.0"
  payload:
    knowledge_updates:
      - type: "command"
        key: "build-command"
        value: "npm run build succeeds with Node 20 on this project"
        stored_via: "store_memory"
      - type: "convention"
        key: "test-pattern"
        value: "Tests use Jest with --testPathPattern for selective execution"
        stored_via: "store_memory"
      - type: "lesson"
        key: "auth-validation-pattern"
        value: "Input validation middleware should be separate from route handlers for testability"
        stored_via: "decisions.yaml"
    decision_log_entries:
      - id: "DL-001"
        title: "Validation middleware architecture"
        rationale: "Separating validation from route handlers enables unit testing without HTTP context"
        confidence: "High"
    evidence_bundle:
      overall_confidence: "High"
      verification_summary:
        tasks_verified: 6
        total_checks: 48
        passed: 47
        failed: 1
        regressions: 0
      review_summary:
        design_review:
          round: 1
          verdicts:
            {
              "gpt-5.3-codex": "approve",
              "gemini-3-pro-preview": "approve",
              "claude-opus-4.6": "approve",
            }
        code_review:
          round: 1
          verdicts:
            {
              "gpt-5.3-codex": "approve",
              "gemini-3-pro-preview": "approve",
              "claude-opus-4.6": "needs_revision",
            }
      rollback_command: "git revert --no-commit pipeline-baseline-2026-02-26T14:30:00Z..HEAD"
      blast_radius:
        files_modified: 12
        risk_red_files: 1
        risk_yellow_files: 3
        risk_green_files: 8
        regressions: 0
      known_issues:
        - description: "1 code review finding on error message verbosity"
          severity: "Minor"
    pipeline_telemetry_summary:
      total_dispatches: 26
      total_duration_seconds: 1842
      error_count: 0
completion:
  status: DONE
  summary: "Knowledge capture complete: 3 updates stored, evidence bundle assembled"
  severity: null
  findings_count: 0
  risk_level: null
  output_paths:
    - "knowledge-output.yaml"
    - "decisions.yaml"
    - "evidence-bundle.md"
```

---

## Schema 13: `e2e-contract`

> Defines the typed YAML schema for `e2e-contract.yaml` — a project-level file describing how to start the application, verify readiness, configure interaction tools, and discover skills for E2E verification.
> **Output path:** `e2e-contract.yaml` (project root) or `.e2e/contract.yaml`
> **Trust Level:** Contract structure is **project-trusted** (same trust as source code). Command executables are **semi-trusted** (validated against command allowlist D-22). Skill content (selectors, form values) is **untrusted** (sanitized before use D-25). Pipeline-generated contracts require user approval before command execution.

### Fields

#### App Lifecycle

| Field                         | Type            | Required    | Constraint / Allowed Values                                                                                                             |
| ----------------------------- | --------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `app_type`                    | string          | Yes         | `web` \| `api` \| `cli` \| `desktop` \| `library`                                                                                       |
| `start_command`               | object          | Yes         | Structured command format (D-21) — NOT a raw shell string                                                                               |
| `start_command.executable`    | string          | Yes         | Binary name; must match `tier5_command_allowlist` (D-22). No path separators, spaces, or shell metacharacters                           |
| `start_command.args`          | list of strings | Yes         | Command arguments; `{port}` placeholder validated as integer [1024, 65535]                                                              |
| `start_command.env`           | map             | No          | Environment variables; values validated against pattern allowlist (alphanumeric + common path chars, no shell metacharacters `;\|&\`$`) |
| `ready_check`                 | string          | Yes         | URL (e.g., `http://localhost:{port}/health`) or structured command object                                                               |
| `ready_timeout_ms`            | integer         | Yes         | Max wait for readiness; default `30000`                                                                                                 |
| `base_url`                    | string          | Yes         | Application base URL (e.g., `http://localhost:{port}`)                                                                                  |
| `shutdown_command`            | object \| null  | No          | Structured command format; `null` if process kill is sufficient                                                                         |
| `shutdown_command.executable` | string          | Conditional | Binary name; must match command allowlist                                                                                               |
| `shutdown_command.args`       | list of strings | Conditional | Shutdown command arguments                                                                                                              |
| `shutdown_timeout_ms`         | integer         | No          | Max wait for graceful shutdown; default `10000`                                                                                         |

#### Port Management

| Field              | Type    | Required | Constraint / Allowed Values                                         |
| ------------------ | ------- | -------- | ------------------------------------------------------------------- |
| `default_port`     | integer | Yes      | Port number; validated range [1024, 65535]                          |
| `port_env_var`     | string  | No       | Environment variable name for port override (e.g., `PORT`)          |
| `port_range_start` | integer | No       | Start of port range for parallel instances; validated [1024, 65535] |

#### Runner

| Field             | Type           | Required | Constraint / Allowed Values                            |
| ----------------- | -------------- | -------- | ------------------------------------------------------ |
| `e2e_runner`      | string         | No       | `playwright` \| `cypress` \| `custom` \| `none`        |
| `e2e_command`     | string \| null | No       | Test execution command; `null` if no pre-written suite |
| `e2e_config_path` | string \| null | No       | Path to runner configuration file                      |
| `e2e_env`         | map            | No       | Additional environment variables for test execution    |

#### Parallelism

| Field                        | Type    | Required | Constraint / Allowed Values                                                                     |
| ---------------------------- | ------- | -------- | ----------------------------------------------------------------------------------------------- |
| `supports_parallel`          | boolean | No       | Whether app supports parallel test instances; default `false`                                   |
| `requires_isolated_instance` | boolean | No       | Whether each test needs its own app instance; default `true`                                    |
| `max_concurrent_instances`   | integer | No       | Maximum parallel app instances; default `2`, max `4`                                            |
| `db_isolation_strategy`      | string  | No       | `per-instance-db` \| `transaction-rollback` \| `shared-with-locking`; default `per-instance-db` |

#### Interaction

| Field                                       | Type            | Required    | Constraint / Allowed Values                        |
| ------------------------------------------- | --------------- | ----------- | -------------------------------------------------- |
| `interaction_type`                          | string          | Yes         | `browser` \| `api` \| `cli` \| `none`              |
| `interaction_tools`                         | object          | No          | Tool configuration per interaction type            |
| `interaction_tools.browser`                 | object          | Conditional | Required when `interaction_type='browser'`         |
| `interaction_tools.browser.tool`            | string          | Conditional | `playwright` \| `puppeteer` \| `custom`            |
| `interaction_tools.browser.install_command` | string \| null  | Conditional | Installation command; `null` if pre-installed      |
| `interaction_tools.api`                     | object          | Conditional | Required when `interaction_type='api'`             |
| `interaction_tools.api.tool`                | string          | Conditional | `curl` \| `fetch` \| `custom`                      |
| `interaction_tools.api.base_headers`        | map \| null     | No          | Default HTTP headers for API interaction           |
| `interaction_tools.api.auth_setup`          | string \| null  | No          | Authentication setup command or token reference    |
| `interaction_tools.cli`                     | object          | Conditional | Required when `interaction_type='cli'`             |
| `interaction_tools.cli.tool`                | string          | Conditional | `terminal`                                         |
| `interaction_tools.cli.shell`               | string \| null  | No          | Shell to use; `null` for system default            |
| `interaction_tools.cli.working_dir`         | string \| null  | No          | Working directory for CLI commands                 |
| `readiness_checks`                          | list of objects | No          | Checks run before interaction begins               |
| `readiness_checks[].type`                   | string          | Conditional | `http` \| `tcp` \| `command`                       |
| `readiness_checks[].target`                 | string          | Conditional | URL, host:port, or command                         |
| `readiness_checks[].expect`                 | string \| null  | No          | Expected response pattern                          |
| `readiness_checks[].timeout_ms`             | integer         | Conditional | Max wait for this check; default `5000`            |
| `evidence_capture`                          | object          | No          | Evidence capture methods                           |
| `evidence_capture.screenshots`              | boolean         | No          | Capture screenshots; default `true`                |
| `evidence_capture.har_files`                | boolean         | No          | Capture HAR network logs; default `false`          |
| `evidence_capture.response_logs`            | boolean         | No          | Capture API response logs; default `true`          |
| `evidence_capture.console_logs`             | boolean         | No          | Capture browser/CLI console output; default `true` |
| `evidence_capture.video`                    | boolean         | No          | Record video; default `false`                      |
| `evidence_capture.dom_snapshots`            | boolean         | No          | Capture DOM snapshots; default `false`             |

#### Browser Tool Config (`@playwright/cli`)

| Field                    | Type   | Required    | Constraint / Allowed Values                                                    |
| ------------------------ | ------ | ----------- | ------------------------------------------------------------------------------ |
| `browser`                | object | Conditional | Required when `interaction_type='browser'`                                     |
| `browser.tool`           | string | Yes         | `"playwright-cli"` — the CLI binary from `@playwright/cli`                     |
| `browser.install`        | string | Yes         | `"npm install -g @playwright/cli@latest"` — global install command             |
| `browser.skills_install` | string | Yes         | `"playwright-cli install --skills"` — installs agent-discoverable skill guides |
| `browser.session_prefix` | string | Yes         | `"verify-{task-id}"` — session isolation via `-s=<name>` flag                  |
| `browser.config`         | string | Yes         | `".playwright/cli.config.json"` — per-project configuration path               |

#### Skills

| Field                  | Type           | Required | Constraint / Allowed Values                                                                                                                                                                                                                                                                                                                                                                                     |
| ---------------------- | -------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `skills`               | list           | No       | Inline skill objects or path references; see Schema 14 for skill format                                                                                                                                                                                                                                                                                                                                         |
| `skill_format`         | string         | No       | `'playwright-yaml'` \| `'copilot-skill'`; default `'playwright-yaml'`. Determines how skill files are interpreted: `playwright-yaml` uses structured YAML skill definitions, `copilot-skill` uses `.md` files with structured interaction descriptions (GitHub Copilot Skills format). The Verifier Tier 5 reads copilot-skill `.md` files as interaction guides and converts steps to Playwright CLI commands. |
| `skill_discovery_path` | string \| null | No       | Directory path for skill YAML files (organized by type: `test-suite/`, `exploratory/`, `adversarial/`)                                                                                                                                                                                                                                                                                                          |

#### Evidence

| Field                   | Type    | Required | Constraint / Allowed Values                                     |
| ----------------------- | ------- | -------- | --------------------------------------------------------------- |
| `screenshot_on_failure` | boolean | No       | Capture screenshot on step failure; default `true`              |
| `trace_on_failure`      | boolean | No       | Capture trace on failure; default `true`                        |
| `video_recording`       | boolean | No       | Record video during interaction; default `false`                |
| `har_capture`           | boolean | No       | Capture HAR network log; default `false` for API apps           |
| `evidence_output_dir`   | string  | No       | Output directory for evidence files; default `'.e2e-evidence/'` |

#### Trust Level

| Field         | Type   | Required | Constraint / Allowed Values                                          |
| ------------- | ------ | -------- | -------------------------------------------------------------------- |
| `trust_level` | string | No       | `project-trusted` \| `pipeline-generated`; default `project-trusted` |

### Example

```yaml
# e2e-contract.yaml — project root
app_type: "web"
start_command:
  executable: "npm"
  args: ["run", "dev", "--", "--port", "{port}"]
  env:
    NODE_ENV: "test"
ready_check: "http://localhost:{port}/health"
ready_timeout_ms: 30000
base_url: "http://localhost:{port}"
shutdown_command:
  executable: "npm"
  args: ["run", "stop"]
shutdown_timeout_ms: 10000

default_port: 3000
port_env_var: "PORT"
port_range_start: 9001

e2e_runner: "playwright"
e2e_command: "npx playwright test"
e2e_config_path: "playwright.config.ts"
e2e_env:
  CI: "true"

supports_parallel: true
requires_isolated_instance: true
max_concurrent_instances: 2
db_isolation_strategy: "per-instance-db"

interaction_type: "browser"
interaction_tools:
  browser:
    tool: "playwright"
    install_command: "npx playwright install"
readiness_checks:
  - type: "http"
    target: "http://localhost:{port}/health"
    expect: "200"
    timeout_ms: 5000
evidence_capture:
  screenshots: true
  har_files: false
  response_logs: true
  console_logs: true
  video: false
  dom_snapshots: false

browser:
  tool: "playwright-cli"
  install: "npm install -g @playwright/cli@latest"
  skills_install: "playwright-cli install --skills"
  session_prefix: "verify-{task-id}"
  config: ".playwright/cli.config.json"

skills:
  - "skills/test-suite/smoke-tests.yaml"
  - "skills/exploratory/login-flow.yaml"
  - "skills/adversarial/auth-bypass.yaml"
skill_discovery_path: "skills/"

screenshot_on_failure: true
trace_on_failure: true
video_recording: false
har_capture: false
evidence_output_dir: ".e2e-evidence/"

trust_level: "project-trusted"
```

---

## Schema 14: `e2e-skill`

> Defines the typed YAML schema for E2E skill definitions — reusable interaction procedures for test-suite execution, exploratory interaction, and adversarial testing.
> **Output path:** Skills are stored as YAML files in the directory referenced by the contract's `skill_discovery_path`, organized by type (`test-suite/`, `exploratory/`, `adversarial/`).
> **Execution model:** All steps are pre-defined — no agent improvisation (D-18). The verifier executes steps deterministically using the script-generation model (D-20).

### Skill Metadata Fields

| Field             | Type            | Required | Constraint / Allowed Values                                       |
| ----------------- | --------------- | -------- | ----------------------------------------------------------------- |
| `id`              | string          | Yes      | Unique skill identifier (e.g., `SKILL-001`)                       |
| `name`            | string          | Yes      | Human-readable skill name                                         |
| `description`     | string          | Yes      | What this skill verifies                                          |
| `type`            | string          | Yes      | `test-suite` \| `exploratory` \| `adversarial`                    |
| `interaction`     | string          | Yes      | `browser` \| `api` \| `cli` \| `test-command`                     |
| `app_type_filter` | list of strings | No       | App types this skill applies to; `['*']` for all; default `['*']` |
| `preconditions`   | list of strings | No       | Conditions that must be true before the skill runs                |
| `tags`            | list of strings | No       | Categorization tags                                               |
| `timeout_ms`      | integer         | No       | Max time for entire skill execution; default `60000`              |

### Step Fields

| Field          | Type              | Required    | Constraint / Allowed Values                                                                                                                                                                                                                                               |
| -------------- | ----------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `steps`        | list of objects   | Yes         | ≥ 1 entry; ordered interaction steps                                                                                                                                                                                                                                      |
| `order`        | integer           | Yes         | Execution sequence number                                                                                                                                                                                                                                                 |
| `action`       | string            | Yes         | Action to perform: `navigate`, `click`, `fill`, `submit`, `http_request`, `run_command`, `assert_visible`, `assert_text`, `scroll`, `hover`, `wait_for`, `send_input`, `assert_output`, `assert_exit_code`, `assert_status`, `assert_body`, `assert_header`, `screenshot` |
| `target`       | string            | Yes         | CSS selector, URL path, API endpoint, or CLI command (depends on interaction type)                                                                                                                                                                                        |
| `value`        | string \| null    | No          | Input value for `fill`/`submit`/request body actions; `null` if not applicable                                                                                                                                                                                            |
| `method`       | string \| null    | No          | HTTP method for API interactions: `GET` \| `POST` \| `PUT` \| `DELETE` \| `PATCH`                                                                                                                                                                                         |
| `headers`      | map \| null       | No          | HTTP headers for API interactions                                                                                                                                                                                                                                         |
| `expect`       | string \| null    | No          | Human-readable expected result for agent judgment                                                                                                                                                                                                                         |
| `assert`       | object \| null    | No          | Machine-checkable assertion                                                                                                                                                                                                                                               |
| `assert.type`  | string            | Conditional | `status_code` \| `text_contains` \| `element_visible` \| `element_not_visible` \| `url_matches` \| `response_body_contains` \| `exit_code_equals`                                                                                                                         |
| `assert.value` | string \| integer | Conditional | Expected value for the assertion                                                                                                                                                                                                                                          |
| `capture`      | string \| null    | No          | Evidence capture: `screenshot` \| `response` \| `har` \| `dom` \| `console`                                                                                                                                                                                               |
| `timeout_ms`   | integer           | No          | Max wait for this step; default `5000`                                                                                                                                                                                                                                    |
| `on_failure`   | string            | No          | `fail` \| `continue` \| `skip_remaining`; default `fail`                                                                                                                                                                                                                  |

### Adversarial Variation Fields

| Field                                                 | Type              | Required    | Constraint / Allowed Values                                           |
| ----------------------------------------------------- | ----------------- | ----------- | --------------------------------------------------------------------- |
| `adversarial_variations`                              | list of objects   | No          | Adversarial test variations; typically used with `type='adversarial'` |
| `adversarial_variations[].id`                         | string            | Yes         | Variation identifier                                                  |
| `adversarial_variations[].name`                       | string            | Yes         | Human-readable variation name                                         |
| `adversarial_variations[].description`                | string            | Yes         | What this variation tests                                             |
| `adversarial_variations[].severity_if_missed`         | string            | Yes         | `critical` \| `high` \| `medium` \| `low`                             |
| `adversarial_variations[].overrides`                  | list of objects   | Yes         | Step overrides for this variation                                     |
| `adversarial_variations[].overrides[].step_order`     | integer           | Yes         | Which step to override (references `steps[].order`)                   |
| `adversarial_variations[].overrides[].field`          | string            | Yes         | Step field to override (e.g., `value`, `target`, `headers`)           |
| `adversarial_variations[].overrides[].override_value` | string \| map     | Yes         | Adversarial replacement value                                         |
| `adversarial_variations[].expected_behavior`          | string            | Yes         | What SHOULD happen (e.g., `"Error message displayed, no redirect"`)   |
| `adversarial_variations[].assert`                     | object \| null    | No          | Machine-checkable assertion for expected adversarial outcome          |
| `adversarial_variations[].assert.type`                | string            | Conditional | Same types as step `assert.type`                                      |
| `adversarial_variations[].assert.value`               | string \| integer | Conditional | Expected value for the adversarial assertion                          |

### Example

```yaml
# skills/exploratory/login-flow.yaml
id: "SKILL-001"
name: "Login Flow — Happy Path + Adversarial"
description: "Navigate to login, fill credentials, verify redirect to dashboard"
type: "exploratory"
interaction: "browser"
app_type_filter: ["web"]
preconditions:
  - "App is running and accessible at base_url"
  - "Test user account exists (test@example.com / TestPass123!)"
tags: ["auth", "login", "critical-path"]
timeout_ms: 30000

steps:
  - order: 1
    action: "navigate"
    target: "/login"
    expect: "Login page loads with email and password fields"
    assert:
      type: "element_visible"
      value: "input[name='email']"
    capture: "screenshot"
    timeout_ms: 5000
    on_failure: "fail"

  - order: 2
    action: "fill"
    target: "input[name='email']"
    value: "test@example.com"
    expect: "Email field populated"
    timeout_ms: 3000
    on_failure: "fail"

  - order: 3
    action: "fill"
    target: "input[name='password']"
    value: "TestPass123!"
    expect: "Password field populated"
    timeout_ms: 3000
    on_failure: "fail"

  - order: 4
    action: "click"
    target: "button[type='submit']"
    expect: "Form submitted, redirect to dashboard"
    assert:
      type: "url_matches"
      value: "/dashboard"
    capture: "screenshot"
    timeout_ms: 10000
    on_failure: "fail"

  - order: 5
    action: "assert_visible"
    target: "[data-testid='welcome-message']"
    expect: "Welcome message displayed with user name"
    assert:
      type: "text_contains"
      value: "Welcome"
    capture: "screenshot"
    timeout_ms: 5000
    on_failure: "fail"

adversarial_variations:
  - id: "AV-001"
    name: "SQL Injection in Email"
    description: "Attempt SQL injection via email field"
    severity_if_missed: "critical"
    overrides:
      - step_order: 2
        field: "value"
        override_value: "' OR '1'='1' --"
    expected_behavior: "Error message displayed, no redirect to dashboard"
    assert:
      type: "url_matches"
      value: "/login"

  - id: "AV-002"
    name: "XSS in Email Field"
    description: "Attempt XSS injection via email field"
    severity_if_missed: "high"
    overrides:
      - step_order: 2
        field: "value"
        override_value: "<script>alert('xss')</script>"
    expected_behavior: "Input rejected with validation error, no script execution"
    assert:
      type: "element_visible"
      value: ".error-message"

  - id: "AV-003"
    name: "Empty Credentials"
    description: "Submit form with empty email and password"
    severity_if_missed: "medium"
    overrides:
      - step_order: 2
        field: "value"
        override_value: ""
      - step_order: 3
        field: "value"
        override_value: ""
    expected_behavior: "Validation error displayed, form not submitted"
    assert:
      type: "element_visible"
      value: ".validation-error"
```

---

## SQLite Schemas

> All tables reside in `verification-ledger.db` within the feature directory (per D-1 database consolidation). SQLite is the PRIMARY verification ledger. All agents use `run_in_terminal` for SQL operations. Schema initialization is centralized in Step 0 by the orchestrator.
>
> **Canonical DDL:** See [sql-templates.md](sql-templates.md) §1 for all table CREATE statements, indexes, and PRAGMA settings. Do not duplicate DDL here to prevent drift.

| Table                  | Purpose                                                                        |
| ---------------------- | ------------------------------------------------------------------------------ |
| `anvil_checks`         | Verification ledger — baseline, after-implementation, and review check results |
| `pipeline_telemetry`   | Pipeline metrics — dispatch timing, status, and retry counts per agent         |
| `artifact_evaluations` | Artifact quality — usefulness/clarity scores and qualitative feedback          |
| `instruction_updates`  | Instruction governance — Knowledge Agent modifications to instruction files    |

### `anvil_checks` Phase Semantics

| Phase      | Written By           | When                                             | `verdict`/`severity` |
| ---------- | -------------------- | ------------------------------------------------ | -------------------- |
| `baseline` | Implementer          | Before any code changes (baseline capture step)  | Always `NULL`        |
| `after`    | Verifier             | After implementation (full verification cascade) | Always `NULL`        |
| `review`   | Adversarial Reviewer | During adversarial review (verdict insertion)    | Set per review       |

### Evidence Gate Queries

> **See [sql-templates.md](sql-templates.md) §6 for all canonical evidence gate queries (EG-1 through EG-6).** Do not duplicate queries here to prevent drift. Evidence gate logic uses `instance`-based grouping and per-category `check_name` matching — do not use `LIKE`-based counting for review approval checks.

### Full Step 0 Initialization Script

> **See [sql-templates.md](sql-templates.md) §1 for the canonical DDL.** Do not duplicate DDL here to prevent drift. The orchestrator executes sql-templates.md §1 DDL via `run_in_terminal` at Step 0. All tables consolidated in `verification-ledger.db` (per D-1).

---

## `task_id` Naming Convention

> All agents MUST use this convention consistently. `task_id` appears in `anvil_checks`, `pipeline_telemetry`, task schemas, implementation reports, and verification reports.

| Scope                             | Convention                     | Example                         | When Used                                      |
| --------------------------------- | ------------------------------ | ------------------------------- | ---------------------------------------------- |
| **Per-task**                      | Planner-assigned task IDs      | `task-01`, `task-03`, `task-12` | Implementation (Step 5), Verification (Step 6) |
| **Feature-level (design review)** | `{feature_slug}-design-review` | `auth-handler-design-review`    | Adversarial Design Review (Step 3b)            |
| **Feature-level (code review)**   | `{feature_slug}-code-review`   | `auth-handler-code-review`      | Adversarial Code Review (Step 7)               |

**Rules:**

- `feature_slug` is a kebab-case identifier derived from the feature name (e.g., `next-gen-multi-agent-system`)
- Per-task IDs are created by the Planner and used consistently across all downstream agents
- Feature-level IDs are constructed by the orchestrator for review steps where the scope is the entire feature, not a single task

---

## `check_name` Naming Patterns

> `check_name` values in `anvil_checks` follow consistent patterns to enable SQL `LIKE` filtering in evidence gate queries. Agents MUST use these patterns when INSERTing records.

### Patterns

| Pattern                       | Phase      | Used By              | Description                                                                                                                                                                                                                                     |
| ----------------------------- | ---------- | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `baseline-{type}`             | `baseline` | Implementer          | Baseline capture checks (e.g., `baseline-ide-diagnostics`, `baseline-build`, `baseline-tests`)                                                                                                                                                  |
| `ide-diagnostics`             | `after`    | Verifier             | Tier 1: IDE diagnostic check                                                                                                                                                                                                                    |
| `syntax-check`                | `after`    | Verifier             | Tier 1: Syntax/parse check                                                                                                                                                                                                                      |
| `build`                       | `after`    | Verifier             | Tier 2: Build/compile                                                                                                                                                                                                                           |
| `type-check`                  | `after`    | Verifier             | Tier 2: Type checker (tsc, mypy, pyright)                                                                                                                                                                                                       |
| `lint`                        | `after`    | Verifier             | Tier 2: Linter (eslint, ruff, clippy)                                                                                                                                                                                                           |
| `tests`                       | `after`    | Verifier             | Tier 2: Test execution                                                                                                                                                                                                                          |
| `import-check`                | `after`    | Verifier             | Tier 3: Import/load test                                                                                                                                                                                                                        |
| `smoke-execution`             | `after`    | Verifier             | Tier 3: Smoke execution (throwaway script)                                                                                                                                                                                                      |
| `tier3-infeasible`            | `after`    | Verifier             | Tier 3: Recorded when Tier 3 cannot be performed                                                                                                                                                                                                |
| `readiness-{type}`            | `after`    | Verifier             | Tier 4: Operational readiness (Large tasks only). Types: `observability`, `degradation`, `secrets`                                                                                                                                              |
| `review-{scope}-{category}`   | `review`   | Adversarial Reviewer | Review verdict per category where `{scope}` is `design` or `code` and `{category}` is `security`, `architecture`, or `correctness` (e.g., `review-code-security`, `review-design-architecture`). Reviewer identity stored in `instance` column. |
| `baseline-discrepancy`        | `after`    | Verifier             | Flagged when Implementer baseline claims don't match `git show`                                                                                                                                                                                 |
| `behavioral-coverage`         | `after`    | Verifier             | Tier 2: Behavioral coverage verification — confirms each `test_method='test'` AC has a corresponding automated test that invokes production code                                                                                                |
| `runtime-wiring`              | `after`    | Verifier             | Tier 2: Runtime wiring verification — confirms new source files are imported/referenced by at least one pre-existing source file (new-file tasks only)                                                                                          |
| `tdd-compliance`              | `after`    | Verifier             | TDD cycle verified — RED-GREEN-VERIFY phases completed with evidence                                                                                                                                                                            |
| `tdd-fallback`                | `after`    | Verifier             | TDD was skipped with recorded reason (fallback documented)                                                                                                                                                                                      |
| `e2e-contract-found`          | `after`    | Verifier             | E2E contract exists at discovered path and is structurally valid                                                                                                                                                                                |
| `e2e-contract-validation`     | `after`    | Verifier             | E2E contract field-level validation result (required fields, type compatibility, port ranges)                                                                                                                                                   |
| `e2e-instance-start`          | `after`    | Verifier             | Application started on assigned port; PID recorded in `output_snippet`                                                                                                                                                                          |
| `e2e-readiness`               | `after`    | Verifier             | Application passed `ready_check` within `ready_timeout_ms`                                                                                                                                                                                      |
| `e2e-suite-execution`         | `after`    | Verifier             | Pre-written test suite execution result (Tier 5 Phase 2)                                                                                                                                                                                        |
| `e2e-exploratory`             | `after`    | Verifier             | Exploratory interaction result — all exploratory skill steps passed (Tier 5 Phase 3)                                                                                                                                                            |
| `e2e-adversarial`             | `after`    | Verifier             | Adversarial interaction aggregate result (Tier 5 Phase 4)                                                                                                                                                                                       |
| `e2e-adversarial-{variation}` | `after`    | Verifier             | Per-variation adversarial result (D-26); `{variation}` is the variation name in kebab-case                                                                                                                                                      |
| `e2e-instance-shutdown`       | `after`    | Verifier             | Application and interaction tools shut down cleanly; no orphaned processes (Tier 5 Phase 5)                                                                                                                                                     |
| `e2e-test-execution`          | `after`    | Verifier             | Composite: all E2E phases passed (suite + exploratory + adversarial). Used by evidence gate EG-10 for `full-tdd-e2e` lane verification                                                                                                          |

### SQL LIKE Filter Usage

```sql
-- All design review records for current round
WHERE check_name LIKE 'review-design-%' AND round = {current_round}

-- All code review records for current round
WHERE check_name LIKE 'review-code-%' AND round = {current_round}

-- All baseline records for a task
WHERE check_name LIKE 'baseline-%' AND phase = 'baseline'

-- All Tier 4 readiness checks
WHERE check_name LIKE 'readiness-%'

-- All E2E verification checks for a task
WHERE check_name LIKE 'e2e-%' AND phase = 'after'

-- All adversarial variation results for a task
WHERE check_name LIKE 'e2e-adversarial-%' AND phase = 'after'

-- TDD compliance checks
WHERE check_name IN ('tdd-compliance', 'tdd-fallback')
```

---

## Completion Contract Routing Matrix

> **Canonical location** (per AC-13.1, CORR-4). `global-operating-rules.md` §5 references this table — do not duplicate there.
>
> This table defines which agents return which completion statuses and what routing action the orchestrator takes. Only Adversarial Reviewer and Verifier support `NEEDS_REVISION`. All other agents either succeed (`DONE`) or fail (`ERROR`) — revision routing is handled by the orchestrator's decision table.

| Agent                | `DONE`                    | `NEEDS_REVISION`          | `ERROR`                         |
| -------------------- | ------------------------- | ------------------------- | ------------------------------- |
| Researcher           | → Spec (Step 2)           | N/A (not returned)        | Retry once, then pipeline ERROR |
| Spec                 | → Design (Step 3)         | N/A (not returned)        | Retry once, then pipeline ERROR |
| Designer             | → Design Review (Step 3b) | N/A (not returned)        | Retry once, then pipeline ERROR |
| Adversarial Reviewer | → Next step               | → Route to revision agent | Pipeline ERROR if blocker       |
| Planner              | → Approval Gate (Step 4a) | N/A (not returned)        | Retry once, then pipeline ERROR |
| Implementer          | → Verification (Step 6)   | N/A (not returned)        | Retry once, then pipeline ERROR |
| Verifier             | → Code Review (Step 7)    | → Replan (Step 4)         | Retry once, then pipeline ERROR |
| Knowledge Agent      | → Auto-Commit (Step 9)    | N/A (not returned)        | Non-blocking, proceed to commit |

### Routing Rules

1. **Retry policy:** All agents get at most 1 retry on `ERROR` (transient). Deterministic failures are not retried.
2. **NEEDS_REVISION routing:** Only Adversarial Reviewer and Verifier return this. The orchestrator routes revision to the appropriate upstream agent (Designer for design review, Implementer for code review/verification).
3. **Knowledge Agent isolation:** `ERROR` from Knowledge Agent does not halt the pipeline — it is non-blocking.
4. **Blocker escalation:** Any `verdict='blocker'` from Adversarial Reviewer escalates to pipeline `ERROR`.
5. **Review round limit:** Maximum 2 review rounds. If still `NEEDS_REVISION` after round 2, pipeline `ERROR`.

---

## Output Format Classification

> Per DP-2 (YAML+MD Output Rationalization, AC-4.1). Each artifact is classified as **YAML-only**, **YAML+MD**, or **MD-only** based on whether it has machine consumers, human consumers, or both.

### YAML-only (machine-consumed)

No MD companion file. Consumed by downstream agents or orchestrator for routing/data.

| Artifact                                     | Producer             | Consumer(s)                            |
| -------------------------------------------- | -------------------- | -------------------------------------- |
| `research/<focus>.yaml`                      | Researcher           | Spec, Designer                         |
| `tasks/<task-id>.yaml`                       | Planner              | Implementer, Verifier                  |
| `implementation-reports/<task-id>.yaml`      | Implementer          | Verifier                               |
| `verification-reports/<task-id>.yaml`        | Verifier             | Orchestrator, Adversarial Reviewer     |
| `review-verdicts/<scope>-<perspective>.yaml` | Adversarial Reviewer | Orchestrator, Designer (revision mode) |
| `knowledge-output.yaml`                      | Knowledge Agent      | (terminal — no downstream)             |
| `decisions.yaml`                             | Knowledge Agent      | (terminal — append-only log)           |

### YAML+MD (human-facing specifications)

YAML is the machine-readable primary; MD companion is for human approval gates and auditability.

| Artifact      | Producer | YAML File            | MD Companion |
| ------------- | -------- | -------------------- | ------------ |
| Specification | Spec     | `spec-output.yaml`   | `feature.md` |
| Design        | Designer | `design-output.yaml` | `design.md`  |
| Plan          | Planner  | `plan-output.yaml`   | `plan.md`    |

### MD-only (human audit trail)

No YAML companion. Narrative documents for human consumption.

| Artifact                                   | Producer             | Purpose                                    |
| ------------------------------------------ | -------------------- | ------------------------------------------ |
| `review-findings/<scope>-<perspective>.md` | Adversarial Reviewer | Detailed review narrative with code blocks |
| `evidence-bundle.md`                       | Knowledge Agent      | End-of-pipeline human audit document       |

### Net Change from Previous Design

Researcher drops 4 MD files per run (one per focus area). No other agent output format changes. Implementation reports, verification reports, and review verdicts were already YAML-only; review findings were already MD-only.

---

## Schema Evolution Strategy

### Ownership

Each schema is **owned by its producer agent**. The producer's agent definition (`.agent.md`) is the authoritative reference for its output schema. This document (`schemas.md`) provides the centralized reference, but individual agent definitions MUST be consistent.

### Change Policy

| Change Type                                          | Safety   | Consumer Impact               | Action Required                                      |
| ---------------------------------------------------- | -------- | ----------------------------- | ---------------------------------------------------- |
| **Additive** (new optional field)                    | Safe     | None                          | Add to producer; consumers ignore unknown fields     |
| **Breaking** (remove/rename/retype field)            | Unsafe   | All consumers affected        | Update ALL listed consumers in dependency table      |
| **Default change** (new required field with default) | Moderate | Existing outputs remain valid | Add to producer; consumers handle missing-as-default |

### Version Policy

- All schemas start at `schema_version: "1.0"`
- **Additive changes:** Increment minor version (e.g., `"1.0"` → `"1.1"`)
- **Breaking changes:** Increment major version (e.g., `"1.1"` → `"2.0"`)
- The `schema_version` field is REQUIRED in every agent output (in the common header)
- Consumers SHOULD check `schema_version` and warn on unexpected major versions

### Change Process

1. **Identify** the producer and all consumers from the [Producer/Consumer Dependency Table](#producerconsumer-dependency-table)
2. **For breaking changes:** Update producer definition first, then update each consumer definition
3. **Increment** `schema_version` in the schema definition
4. **Verify** consumer compatibility by running pipeline integration tests

### Organization

Schemas in this document are ordered by pipeline step:

```
research → spec → design → plan → implementation → verification → review → knowledge
```

The `completion-contract` (Schema 1) is listed first as it applies to all agents.

### Schemas Removed from v1

| Removed Schema        | Resolution                                                                                                       |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `pipeline-manifest`   | Eliminated — orchestrator tracks state in-context only                                                           |
| `approval-request`    | Inlined — approval prompt format defined in orchestrator agent definition, not a separate schema                 |
| `decision-record`     | Merged — decision records are embedded within `design-output` and `plan-output` payloads                         |
| `verification-record` | Merged — individual check records are embedded within `implementation-report` and `verification-report` payloads |
