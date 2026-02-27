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

| #   | Schema                  | Producer                   | Consumers                                          | Purpose                                                                                   |
| --- | ----------------------- | -------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| 1   | `completion-contract`   | All agents (shared header) | Orchestrator                                       | Three-state completion signal; embedded in every agent output                             |
| 2   | `research-output`       | Researcher                 | Spec, Designer                                     | Structured research findings                                                              |
| 3   | `spec-output`           | Spec                       | Designer, Planner, Implementer, Verifier           | Structured requirements + acceptance criteria                                             |
| 4   | `design-output`         | Designer                   | Planner, Implementer, Adversarial Reviewer         | Structured design decisions + justifications                                              |
| 5   | `plan-output`           | Planner                    | Orchestrator (risk summary), Implementer, Verifier | Task list + risk classifications + wave assignments + `overall_risk_summary`              |
| 6   | `task-schema`           | Planner                    | Implementer, Verifier                              | Per-task spec with risk level + `relevant_context` pointers (sub-schema of `plan-output`) |
| 7   | `implementation-report` | Implementer                | Verifier                                           | Baseline records + change summary + self-check + verification ledger entries              |
| 8   | `verification-report`   | Verifier                   | Orchestrator (gate_status), Adversarial Reviewer   | Cascade results + evidence gate summary + regression analysis (complements SQL ledger)    |
| 9   | `review-findings`       | Adversarial Reviewer       | Orchestrator (verdict routing), Knowledge Agent    | Per-model findings in Markdown + YAML verdict summary + SQL INSERT of verdict record      |
| 10  | `knowledge-output`      | Knowledge Agent            | (terminal — no downstream consumers)               | Knowledge updates + decision log entries + store_memory calls                             |

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
> **Companion:** `research/<focus>.md` (human-readable)

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
    - "research/architecture.md"
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

### Fields

| Field                                          | Type            | Required    | Constraint / Allowed Values                                        |
| ---------------------------------------------- | --------------- | ----------- | ------------------------------------------------------------------ |
| `task.id`                                      | string          | Yes         | Task identifier assigned by Planner (e.g., `task-03`)              |
| `task.title`                                   | string          | Yes         | Task title                                                         |
| `task.description`                             | string          | Yes         | Detailed description of work to be done                            |
| `task.agent`                                   | string          | Yes         | Assigned agent (e.g., `implementer`)                               |
| `task.size`                                    | string          | Yes         | `Standard` \| `Large`                                              |
| `task.risk`                                    | string          | Yes         | `"🟢"` \| `"🟡"` \| `"🔴"`                                         |
| `task.depends_on`                              | list of strings | No          | Task IDs this task depends on                                      |
| `task.acceptance_criteria`                     | list of strings | Yes         | ≥ 1 entry; testable acceptance criteria                            |
| `task.relevant_context`                        | object          | Yes         | Pointers to specific upstream sections (bounds read amplification) |
| `task.relevant_context.design_sections`        | list of strings | Yes         | ≥ 1 entry; paths with section pointers into `design-output.yaml`   |
| `task.relevant_context.spec_requirements`      | list of strings | Yes         | ≥ 1 entry; paths with section pointers into `spec-output.yaml`     |
| `task.relevant_context.files_to_modify`        | list of objects | No          | Files to create or modify                                          |
| `task.relevant_context.files_to_modify[].path` | string          | Conditional | Relative file path                                                 |
| `task.relevant_context.files_to_modify[].risk` | string          | Conditional | `"🟢"` \| `"🟡"` \| `"🔴"`                                         |

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
    - "Email validation rejects malformed addresses"
    - "Password strength check enforces minimum 12 characters"
    - "Rate limiting returns 429 after 5 attempts per minute"
    - "All validation errors return structured JSON error responses"
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

| Field                                         | Type            | Required    | Constraint / Allowed Values                                             |
| --------------------------------------------- | --------------- | ----------- | ----------------------------------------------------------------------- |
| `agent_output` (header)                       | object          | Yes         | Common header                                                           |
| `payload.task_id`                             | string          | Yes         | Matches `task.id` from assigned task schema                             |
| `payload.task_type`                           | string          | Yes         | `code` \| `documentation` \| `configuration`                            |
| `payload.baseline`                            | object          | Yes         | Pre-implementation state capture                                        |
| `payload.baseline.ide_diagnostics`            | object          | Yes         | IDE error/warning counts before changes                                 |
| `payload.baseline.ide_diagnostics.errors`     | integer         | Yes         | ≥ 0                                                                     |
| `payload.baseline.ide_diagnostics.warnings`   | integer         | Yes         | ≥ 0                                                                     |
| `payload.baseline.build_exit_code`            | integer \| null | Yes         | Build exit code before changes; `null` if no build system               |
| `payload.baseline.test_summary`               | object \| null  | Yes         | Test results before changes; `null` if no tests                         |
| `payload.baseline.test_summary.total`         | integer         | Conditional | Total tests                                                             |
| `payload.baseline.test_summary.passed`        | integer         | Conditional | Passing tests                                                           |
| `payload.baseline.test_summary.failed`        | integer         | Conditional | Failing tests                                                           |
| `payload.changes`                             | list of objects | Yes         | ≥ 1 entry; files created or modified                                    |
| `payload.changes[].path`                      | string          | Yes         | Relative file path                                                      |
| `payload.changes[].action`                    | string          | Yes         | `created` \| `modified` \| `deleted`                                    |
| `payload.changes[].description`               | string          | Yes         | Brief description of the change                                         |
| `payload.self_check`                          | object          | Yes         | Post-implementation self-verification results                           |
| `payload.self_check.ide_diagnostics`          | object          | Yes         | IDE error/warning counts after changes                                  |
| `payload.self_check.ide_diagnostics.errors`   | integer         | Yes         | ≥ 0                                                                     |
| `payload.self_check.ide_diagnostics.warnings` | integer         | Yes         | ≥ 0                                                                     |
| `payload.self_check.build_exit_code`          | integer \| null | Yes         | Build exit code after changes; `null` if no build system                |
| `payload.self_check.test_summary`             | object \| null  | Yes         | Test results after changes; `null` if no tests                          |
| `payload.self_check.self_fix_attempts`        | integer         | Yes         | 0–2; number of self-fix iterations performed                            |
| `payload.self_check.git_staged`               | boolean         | Yes         | `true` if `git add -A` was executed after completion                    |
| `payload.verification_entries`                | list of objects | No          | Baseline-phase SQL entries for `anvil_checks` (recorded by Implementer) |
| `payload.verification_entries[].check_name`   | string          | Conditional | Check name for SQL INSERT                                               |
| `payload.verification_entries[].phase`        | string          | Conditional | Always `baseline` for Implementer entries                               |
| `payload.verification_entries[].tool`         | string          | Conditional | Tool used (e.g., `ide-get_diagnostics`, `dotnet build`)                 |
| `payload.verification_entries[].passed`       | boolean         | Conditional | Whether the check passed                                                |
| `completion` (contract)                       | object          | Yes         | See Schema 1                                                            |

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

| Field                                              | Type            | Required    | Constraint / Allowed Values                                 |
| -------------------------------------------------- | --------------- | ----------- | ----------------------------------------------------------- |
| `agent_output` (header)                            | object          | Yes         | Common header                                               |
| `payload.task_id`                                  | string          | Yes         | Matches task being verified                                 |
| `payload.run_id`                                   | string          | Yes         | Pipeline run identifier (ISO 8601)                          |
| `payload.evidence_gate`                            | object          | Yes         | Gate summary for orchestrator routing                       |
| `payload.evidence_gate.total_checks`               | integer         | Yes         | ≥ 1                                                         |
| `payload.evidence_gate.passed`                     | integer         | Yes         | ≥ 0                                                         |
| `payload.evidence_gate.failed`                     | integer         | Yes         | ≥ 0                                                         |
| `payload.evidence_gate.gate_status`                | string          | Yes         | `passed` \| `failed`                                        |
| `payload.findings`                                 | list of objects | Yes         | ≥ 1 entry; individual check results                         |
| `payload.findings[].check_name`                    | string          | Yes         | Check identifier used in SQL INSERT                         |
| `payload.findings[].tier`                          | integer         | Yes         | 1–4; which verification tier                                |
| `payload.findings[].phase`                         | string          | Yes         | `baseline` \| `after`                                       |
| `payload.findings[].tool`                          | string          | Yes         | Tool used for the check                                     |
| `payload.findings[].command`                       | string \| null  | No          | Command executed; `null` if not applicable                  |
| `payload.findings[].exit_code`                     | integer \| null | No          | Command exit code; `null` if not applicable                 |
| `payload.findings[].passed`                        | boolean         | Yes         | Whether the check passed                                    |
| `payload.findings[].output_snippet`                | string \| null  | No          | ≤ 500 chars; first 500 characters of output                 |
| `payload.regressions`                              | list of objects | No          | Checks that passed in baseline but failed in after          |
| `payload.regressions[].check_name`                 | string          | Conditional | Name of the regressed check                                 |
| `payload.regressions[].baseline_result`            | boolean         | Conditional | Always `true` (passed in baseline)                          |
| `payload.regressions[].after_result`               | boolean         | Conditional | Always `false` (failed in after)                            |
| `payload.regressions[].detail`                     | string          | Conditional | Description of the regression                               |
| `payload.baseline_cross_check`                     | object \| null  | No          | Results of independent baseline verification via `git show` |
| `payload.baseline_cross_check.method`              | string          | Conditional | Always `"git show pipeline-baseline-{run_id}:<path>"`       |
| `payload.baseline_cross_check.discrepancies_found` | boolean         | Conditional | `true` if Implementer baseline claims don't match           |
| `completion` (contract)                            | object          | Yes         | See Schema 1                                                |

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

> Produced by each Adversarial Reviewer instance (×3 parallel). Each reviewer produces three outputs:
>
> 1. **Markdown findings file** — detailed human-readable findings at `review-findings/<scope>-<model>.md`
> 2. **YAML verdict summary** — machine-readable verdict at `review-verdicts/<scope>.yaml`
> 3. **SQL INSERT** — verdict record into `anvil_checks` with `phase='review'`
>
> This schema defines the YAML verdict summary. The Markdown findings file is free-form prose. The SQL INSERT follows the `anvil_checks` schema defined in [SQLite Schemas](#sqlite-schemas).

### YAML Verdict Summary Fields

| Field                     | Type    | Required | Constraint / Allowed Values                                                         |
| ------------------------- | ------- | -------- | ----------------------------------------------------------------------------------- |
| `reviewer_model`          | string  | Yes      | Model identifier (e.g., `gpt-5.3-codex`, `gemini-3-pro-preview`, `claude-opus-4.6`) |
| `review_focus`            | string  | Yes      | `security` \| `architecture` \| `correctness`                                       |
| `scope`                   | string  | Yes      | `design` \| `code`                                                                  |
| `verdict`                 | string  | Yes      | `approve` \| `needs_revision` \| `blocker`                                          |
| `findings_count`          | object  | Yes      | Breakdown by severity                                                               |
| `findings_count.blocker`  | integer | Yes      | ≥ 0                                                                                 |
| `findings_count.critical` | integer | Yes      | ≥ 0                                                                                 |
| `findings_count.major`    | integer | Yes      | ≥ 0                                                                                 |
| `findings_count.minor`    | integer | Yes      | ≥ 0                                                                                 |
| `summary`                 | string  | Yes      | ≤ 500 characters; one-paragraph summary of review outcome                           |

### Review Focus Area Details

| Focus          | Scope                                                                                |
| -------------- | ------------------------------------------------------------------------------------ |
| `security`     | Injection vectors, authentication gaps, data exposure, secrets, authorization bypass |
| `architecture` | Coupling, scalability boundaries, component responsibilities, API surface area       |
| `correctness`  | Edge cases, logic errors, spec compliance, testing gaps, error handling              |

### SQL INSERT Convention for Review Records

When `phase='review'` in `anvil_checks`:

| Column           | Value                                                              |
| ---------------- | ------------------------------------------------------------------ |
| `task_id`        | Per `task_id` convention (see below)                               |
| `phase`          | `'review'`                                                         |
| `check_name`     | `'review-{scope}-{model}'` (e.g., `'review-design-gpt-5.3-codex'`) |
| `tool`           | `'adversarial-review'`                                             |
| `command`        | `'adversarial-review'`                                             |
| `exit_code`      | `NULL`                                                             |
| `output_snippet` | First 500 chars of summary                                         |
| `passed`         | `1` if `verdict='approve'`, else `0`                               |
| `verdict`        | The reviewer's verdict                                             |
| `severity`       | Highest finding severity (`NULL` if clean approve)                 |
| `round`          | Current review round (1 or 2)                                      |
| `run_id`         | Current pipeline run ID                                            |

### Example (YAML Verdict Summary)

```yaml
reviewer_model: "gpt-5.3-codex"
review_focus: "security"
scope: "design"
verdict: "approve"
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

## SQLite Schemas

> SQLite is the PRIMARY verification ledger. All agents use `run_in_terminal` for SQL operations. The database files are created in the feature directory. Schema initialization is centralized in Step 0 by the orchestrator.

### Initialization (Step 0 — Orchestrator via `run_in_terminal`)

```sql
-- MANDATORY: Set before any table creation or data operations
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;
```

**Why these are mandatory:**

- `journal_mode=WAL` — allows concurrent reads with one writer; prevents `SQLITE_BUSY` errors when up to 4 agents write in parallel
- `busy_timeout=5000` — queues concurrent writers for up to 5 seconds instead of failing immediately

### `anvil_checks` Table (Verification Ledger)

> **Canonical definition** — from design.md §Decision 6. If this differs from §Data Storage, §Decision 6 takes precedence.
> **Database file:** `verification-ledger.db` in the feature directory.

```sql
CREATE TABLE IF NOT EXISTS anvil_checks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    phase TEXT NOT NULL CHECK(phase IN ('baseline', 'after', 'review')),
    check_name TEXT NOT NULL,
    tool TEXT NOT NULL,
    command TEXT,
    exit_code INTEGER,
    output_snippet TEXT CHECK(LENGTH(output_snippet) <= 500),
    passed INTEGER NOT NULL CHECK(passed IN (0, 1)),
    verdict TEXT CHECK(verdict IN ('approve', 'needs_revision', 'blocker')),
    severity TEXT CHECK(severity IN ('Blocker', 'Critical', 'Major', 'Minor')),
    round INTEGER NOT NULL DEFAULT 1,
    ts DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

#### Column Reference

| Column           | Type     | Nullable | Constraint / Notes                                                                         |
| ---------------- | -------- | -------- | ------------------------------------------------------------------------------------------ |
| `id`             | INTEGER  | No       | Auto-increment primary key                                                                 |
| `run_id`         | TEXT     | No       | Pipeline run identifier (ISO 8601 timestamp, e.g., `2026-02-26T14:30:00Z`)                 |
| `task_id`        | TEXT     | No       | See [task_id Naming Convention](#task_id-naming-convention)                                |
| `phase`          | TEXT     | No       | `baseline` \| `after` \| `review`                                                          |
| `check_name`     | TEXT     | No       | See [check_name Naming Patterns](#check_name-naming-patterns)                              |
| `tool`           | TEXT     | No       | Tool or command used for the check                                                         |
| `command`        | TEXT     | Yes      | Full command string; `NULL` for non-command checks (e.g., IDE diagnostics)                 |
| `exit_code`      | INTEGER  | Yes      | Command exit code; `NULL` for non-command checks and review records                        |
| `output_snippet` | TEXT     | Yes      | ≤ 500 characters; first 500 chars of output (Anvil truncation rule)                        |
| `passed`         | INTEGER  | No       | `0` (failed) or `1` (passed); for reviews: `1` if `verdict='approve'`, else `0`            |
| `verdict`        | TEXT     | Yes      | `approve` \| `needs_revision` \| `blocker`; `NULL` for `phase='baseline'`/`'after'`        |
| `severity`       | TEXT     | Yes      | `Blocker` \| `Critical` \| `Major` \| `Minor`; `NULL` for baseline/after and clean approve |
| `round`          | INTEGER  | No       | Review round (1 or 2); defaults to `1`; used to prevent stale record accumulation          |
| `ts`             | DATETIME | No       | Auto-set to `CURRENT_TIMESTAMP`                                                            |

#### Phase Semantics

| Phase      | Written By           | When                                             | `verdict`/`severity` |
| ---------- | -------------------- | ------------------------------------------------ | -------------------- |
| `baseline` | Implementer          | Before any code changes (baseline capture step)  | Always `NULL`        |
| `after`    | Verifier             | After implementation (full verification cascade) | Always `NULL`        |
| `review`   | Adversarial Reviewer | During adversarial review (verdict insertion)    | Set per review       |

#### Required Indexes

```sql
-- Composite indexes for evidence gate query performance
CREATE INDEX IF NOT EXISTS idx_anvil_task_phase ON anvil_checks(task_id, phase);
CREATE INDEX IF NOT EXISTS idx_anvil_run_round ON anvil_checks(run_id, round);
```

#### Key Evidence Gate Queries

```sql
-- Baseline exists check (before Step 6)
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id = '{run_id}' AND task_id = '{task_id}' AND phase = 'baseline';
-- Must be > 0

-- Verification sufficient check (after Step 6)
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id = '{run_id}' AND task_id = '{task_id}' AND phase = 'after' AND passed = 1;
-- Must be >= 2 (Standard) or >= 3 (Large/🔴)

-- Design review approval check (after Step 3b)
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id = '{run_id}' AND task_id = '{task_id}' AND phase = 'review'
    AND check_name LIKE 'review-design-%' AND round = {current_round} AND verdict = 'approve';
-- Must be >= 2 (majority of 3)

-- Design review security blocker check (after Step 3b)
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id = '{run_id}' AND task_id = '{task_id}' AND phase = 'review'
    AND check_name LIKE 'review-design-%' AND round = {current_round} AND verdict = 'blocker';
-- Must be = 0 (any blocker → pipeline ERROR)

-- Code review approval check (after Step 7)
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id = '{run_id}' AND task_id = '{task_id}' AND phase = 'review'
    AND check_name LIKE 'review-code-%' AND round = {current_round} AND verdict = 'approve';
-- Must be >= 2 (majority of 3)

-- Code review security blocker check (after Step 7)
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id = '{run_id}' AND task_id = '{task_id}' AND phase = 'review'
    AND check_name LIKE 'review-code-%' AND round = {current_round} AND verdict = 'blocker';
-- Must be = 0 (any blocker → pipeline ERROR)

-- Review completion check (all 3 submitted for current round)
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id = '{run_id}' AND task_id = '{task_id}' AND phase = 'review'
    AND check_name LIKE 'review-{scope}-%' AND round = {current_round} AND verdict IS NOT NULL;
-- Must be >= 3 (all reviewers submitted)
```

### `pipeline_telemetry` Table

> **Database file:** `pipeline-telemetry.db` in the feature directory.

```sql
CREATE TABLE IF NOT EXISTS pipeline_telemetry (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT NOT NULL,
    step TEXT NOT NULL,
    agent TEXT NOT NULL,
    instance TEXT,
    started_at TEXT NOT NULL,
    completed_at TEXT,
    status TEXT CHECK(status IN ('DONE', 'NEEDS_REVISION', 'ERROR', 'running')),
    dispatch_count INTEGER DEFAULT 1,
    retry_count INTEGER DEFAULT 0,
    notes TEXT,
    ts DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

#### Column Reference

| Column           | Type     | Nullable | Constraint / Notes                                                        |
| ---------------- | -------- | -------- | ------------------------------------------------------------------------- |
| `id`             | INTEGER  | No       | Auto-increment primary key                                                |
| `run_id`         | TEXT     | No       | Pipeline run identifier (ISO 8601)                                        |
| `step`           | TEXT     | No       | Pipeline step (e.g., `step-1`, `step-5`)                                  |
| `agent`          | TEXT     | No       | Agent name                                                                |
| `instance`       | TEXT     | Yes      | Instance identifier (e.g., `researcher-architecture`); `NULL` if singular |
| `started_at`     | TEXT     | No       | ISO 8601 timestamp                                                        |
| `completed_at`   | TEXT     | Yes      | ISO 8601 timestamp; `NULL` if `status='running'`                          |
| `status`         | TEXT     | Yes      | `DONE` \| `NEEDS_REVISION` \| `ERROR` \| `running`                        |
| `dispatch_count` | INTEGER  | No       | Number of times this agent was dispatched (≥ 1)                           |
| `retry_count`    | INTEGER  | No       | Number of retries (≥ 0)                                                   |
| `notes`          | TEXT     | Yes      | Free-form notes (e.g., error details)                                     |
| `ts`             | DATETIME | No       | Auto-set to `CURRENT_TIMESTAMP`                                           |

#### Required Indexes

```sql
CREATE INDEX IF NOT EXISTS idx_telemetry_step ON pipeline_telemetry(step);
```

#### Key Telemetry Queries

```sql
-- Total dispatch count for the pipeline run
SELECT COUNT(*) FROM pipeline_telemetry WHERE run_id = '{run_id}';

-- Average step duration
SELECT step, AVG(JULIANDAY(completed_at) - JULIANDAY(started_at)) * 86400 AS avg_seconds
FROM pipeline_telemetry WHERE run_id = '{run_id}' GROUP BY step;

-- Failed steps
SELECT step, agent, status, notes FROM pipeline_telemetry
WHERE run_id = '{run_id}' AND status = 'ERROR';
```

### Full Step 0 Initialization Script

**Verification Ledger** (`verification-ledger.db`):

```sql
-- Execute via orchestrator run_in_terminal on verification-ledger.db at Step 0
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;

CREATE TABLE IF NOT EXISTS anvil_checks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    phase TEXT NOT NULL CHECK(phase IN ('baseline', 'after', 'review')),
    check_name TEXT NOT NULL,
    tool TEXT NOT NULL,
    command TEXT,
    exit_code INTEGER,
    output_snippet TEXT CHECK(LENGTH(output_snippet) <= 500),
    passed INTEGER NOT NULL CHECK(passed IN (0, 1)),
    verdict TEXT CHECK(verdict IN ('approve', 'needs_revision', 'blocker')),
    severity TEXT CHECK(severity IN ('Blocker', 'Critical', 'Major', 'Minor')),
    round INTEGER NOT NULL DEFAULT 1,
    ts DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_anvil_task_phase ON anvil_checks(task_id, phase);
CREATE INDEX IF NOT EXISTS idx_anvil_run_round ON anvil_checks(run_id, round);
```

**Pipeline Telemetry** (`pipeline-telemetry.db`):

```sql
-- Execute via orchestrator run_in_terminal on pipeline-telemetry.db at Step 0
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;

CREATE TABLE IF NOT EXISTS pipeline_telemetry (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT NOT NULL,
    step TEXT NOT NULL,
    agent TEXT NOT NULL,
    instance TEXT,
    started_at TEXT NOT NULL,
    completed_at TEXT,
    status TEXT CHECK(status IN ('DONE', 'NEEDS_REVISION', 'ERROR', 'running')),
    dispatch_count INTEGER DEFAULT 1,
    retry_count INTEGER DEFAULT 0,
    notes TEXT,
    ts DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_telemetry_step ON pipeline_telemetry(step);
```

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

| Pattern                 | Phase      | Used By              | Description                                                                                        |
| ----------------------- | ---------- | -------------------- | -------------------------------------------------------------------------------------------------- |
| `baseline-{type}`       | `baseline` | Implementer          | Baseline capture checks (e.g., `baseline-ide-diagnostics`, `baseline-build`, `baseline-tests`)     |
| `ide-diagnostics`       | `after`    | Verifier             | Tier 1: IDE diagnostic check                                                                       |
| `syntax-check`          | `after`    | Verifier             | Tier 1: Syntax/parse check                                                                         |
| `build`                 | `after`    | Verifier             | Tier 2: Build/compile                                                                              |
| `type-check`            | `after`    | Verifier             | Tier 2: Type checker (tsc, mypy, pyright)                                                          |
| `lint`                  | `after`    | Verifier             | Tier 2: Linter (eslint, ruff, clippy)                                                              |
| `tests`                 | `after`    | Verifier             | Tier 2: Test execution                                                                             |
| `import-check`          | `after`    | Verifier             | Tier 3: Import/load test                                                                           |
| `smoke-execution`       | `after`    | Verifier             | Tier 3: Smoke execution (throwaway script)                                                         |
| `tier3-infeasible`      | `after`    | Verifier             | Tier 3: Recorded when Tier 3 cannot be performed                                                   |
| `readiness-{type}`      | `after`    | Verifier             | Tier 4: Operational readiness (Large tasks only). Types: `observability`, `degradation`, `secrets` |
| `review-design-{model}` | `review`   | Adversarial Reviewer | Design review verdict per model (e.g., `review-design-gpt-5.3-codex`)                              |
| `review-code-{model}`   | `review`   | Adversarial Reviewer | Code review verdict per model (e.g., `review-code-claude-opus-4.6`)                                |
| `baseline-discrepancy`  | `after`    | Verifier             | Flagged when Implementer baseline claims don't match `git show`                                    |

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
```

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
