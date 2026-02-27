# Artifact Evaluations by Implementer (Task 11)

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "implementer-11"
  source_artifact: "tasks/11-orchestrator-agent.md"
  usefulness_score: 10
  clarity_score: 10
  useful_elements:
    - "§Implementation Steps provided exact line ranges for design.md sections"
    - "§Acceptance Criteria — all 17 criteria were specific and verifiable"
    - "§Relevant Context from design.md — pinpointed every section needed"
    - "§Completion Checklist — mapped 1:1 to acceptance criteria"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Test Requirements — TDD not applicable to markdown agent definitions"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "implementer-11"
  source_artifact: "design.md"
  usefulness_score: 10
  clarity_score: 9
  useful_elements:
    - "§Decision 2 Orchestrator detail — exact tool lists, 5 responsibilities, DR-1 constraint"
    - "§Decision 3 Pipeline Overview — complete Steps 0–9 with gate conditions"
    - "§Orchestrator Decision Table — all routing conditions in tabular form"
    - "§Decision 6 Evidence Gate SQL queries — all 7 templates with filter criteria"
    - "§Decision 9 Approval Mode — structured multiple-choice prompt examples"
    - "§Decision 8 Escalation Paths — retry budgets and escalation logic"
    - "§Pipeline State Model — in-context state fields and EC-5 recovery"
    - "§Deviation Records DR-1 — run_in_terminal justification and constraints"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Decision 1 Architecture Selection scoring — not needed for implementation"
    - "§Data Storage Strategy duplicate schema definitions — used §Decision 6 as canonical per planner guidance"
  inaccuracies:
    - "§Data Storage §anvil_checks schema output_snippet limit is 2000 while §Decision 6 says 500 — used 500 (§Decision 6 canonical)"
    - "§Data Storage §anvil_checks severity enum uses lowercase ('blocker','critical','high','medium','low') while §Decision 6 uses title-case ('Blocker','Critical','Major','Minor') — used §Decision 6 canonical values"
  impact_on_work:
    - "Dual schema definitions required reconciliation — added ~5 minutes to implementation"
```

```yaml
artifact_evaluation:
  evaluator: "implementer-11"
  source_artifact: "feature.md"
  usefulness_score: 4
  clarity_score: 8
  useful_elements:
    - "§Functional Requirements FR-1.6 — context for DR-1 deviation"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Architectural Directions — selection already made in design.md"
    - "§Common Requirements — implemented via design.md decisions"
    - "§Acceptance Criteria — implemented via task-level criteria"
  inaccuracies:
    - "FR-1.6 tool restriction is superseded by DR-1 deviation — acknowledged in design.md"
  impact_on_work:
    - "N/A — none identified"
```
