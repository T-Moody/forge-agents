# Artifact Evaluations by Implementer (Task 03)

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "implementer-03"
  source_artifact: "tasks/03-researcher-agent.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "§Acceptance Criteria provided a clear checklist of exactly what to include"
    - "§Implementation Steps gave a step-by-step guide matching the design.md sections"
    - "§Relevant Context from design.md provided line-number pointers to key sections"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "N/A — none identified"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "implementer-03"
  source_artifact: "design.md"
  usefulness_score: 9
  clarity_score: 8
  useful_elements:
    - "§Decision 2 Agent Detail provided exact inputs, outputs, tools, and completion contract for Researcher"
    - "§Decision 10 Agent Definition Structure provided the template to follow"
    - "§Decision 3 Step 1 defined the dispatch pattern and gate conditions"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Decision 6 Verification Architecture — not relevant to Researcher definition"
    - "§Decision 7 Adversarial Review — not relevant to Researcher definition"
    - "§Deviation Records — not relevant to Researcher definition"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "implementer-03"
  source_artifact: "feature.md"
  usefulness_score: 3
  clarity_score: 7
  useful_elements:
    - "N/A — feature.md was not needed for this agent definition task"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "Not read — task spec and design.md provided all necessary context"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```
