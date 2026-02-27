# Artifact Evaluations by Implementer (Task 12)

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "implementer-12"
  source_artifact: "tasks/12-feature-workflow-prompt.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "Implementation Steps section provided clear step-by-step guidance"
    - "Acceptance Criteria mapped directly to deliverable checks"
    - "Relevant Context pointer to design.md §Decision 10 with line numbers"
  missing_information:
    - "Did not mention VS Code prompt file attribute limitations (mode/variables not supported)"
  information_not_used:
    - "Test Requirements section — TDD skipped for config-only task"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "implementer-12"
  source_artifact: "design.md"
  usefulness_score: 8
  clarity_score: 7
  useful_elements:
    - "§Decision 10 Prompt File Structure provided exact template for the prompt file"
    - "§Decision 10 Output Directory Layout confirmed all agent file paths"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "Most of design.md (Decisions 1–9, pipeline steps, Mermaid diagram) — not relevant to this task"
  inaccuracies:
    - "Prompt File Structure template specifies `mode: agent` and `variables:` in YAML frontmatter, but VS Code prompt files do not support these attributes — required adaptation to `name`/`agent`/`description`"
  impact_on_work:
    - "Minor rework to adapt YAML frontmatter from design template to actual VS Code prompt file conventions"
```

```yaml
artifact_evaluation:
  evaluator: "implementer-12"
  source_artifact: "feature.md"
  usefulness_score: 4
  clarity_score: 8
  useful_elements:
    - "AC-14 confirmed output structure expectations"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "Most functional requirements and acceptance criteria — not relevant to prompt file creation"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```
