# Artifact Evaluation Schema — v1

> **Type:** Shared reference document (not an agent definition)
> **Purpose:** Single-source-of-truth for the artifact evaluation YAML schema used by all 14 evaluating agents and the PostMortem agent.
> **Rationale:** Centralizing the schema eliminates 14-file duplication; schema changes require updating only this file.

---

## Artifact Evaluation Schema

Each evaluating agent produces one YAML block per upstream pipeline-produced artifact it consumed. All blocks are written to:

```
docs/feature/<feature-slug>/artifact-evaluations/<agent-name>.md
```

### Schema

```yaml
artifact_evaluation:
  evaluator: "<agent-name>" # string, must match the evaluating agent's name
  source_artifact: "<relative path>" # string, path relative to feature directory
  usefulness_score: <integer 1-10> # how useful was this artifact for completing the task
  clarity_score: <integer 1-10> # how clear and well-structured was the artifact
  useful_elements:
    - "<specific section or element>" # ≥1 entry; use "N/A — none identified" if empty
  missing_information:
    - "<concrete missing detail>" # ≥1 entry; use "N/A — none identified" if empty
  information_not_used:
    - "<info read but not used>" # ≥1 entry; use "N/A — none identified" if empty
  inaccuracies:
    - "<incorrect assumption or error>" # ≥1 entry; use "N/A — none identified" if empty
  impact_on_work:
    - "<rework or clarification needed>" # ≥1 entry; use "N/A — none identified" if empty
```

### Field Reference

| Field                  | Type            | Constraint | Description                                            |
| ---------------------- | --------------- | ---------- | ------------------------------------------------------ |
| `evaluator`            | string          | Required   | Must match the evaluating agent's name                 |
| `source_artifact`      | string          | Required   | Path relative to the feature directory                 |
| `usefulness_score`     | integer         | 1–10       | How useful was this artifact for completing the task   |
| `clarity_score`        | integer         | 1–10       | How clear and well-structured was the artifact         |
| `useful_elements`      | list of strings | ≥1 entry   | Specific sections or elements that were useful         |
| `missing_information`  | list of strings | ≥1 entry   | Concrete details that were missing                     |
| `information_not_used` | list of strings | ≥1 entry   | Information read but not used                          |
| `inaccuracies`         | list of strings | ≥1 entry   | Incorrect assumptions or errors found                  |
| `impact_on_work`       | list of strings | ≥1 entry   | Rework or clarification needed due to artifact quality |

---

## Evaluation Error Fallback

If evaluation generation fails for an artifact, write this block instead:

```yaml
evaluation_error:
  evaluator: "<agent-name>"
  source_artifact: "<path>"
  error: "<description of what failed>"
```

| Field             | Type   | Constraint | Description                              |
| ----------------- | ------ | ---------- | ---------------------------------------- |
| `evaluator`       | string | Required   | Must match the evaluating agent's name   |
| `source_artifact` | string | Required   | Path relative to the feature directory   |
| `error`           | string | Required   | Description of what prevented evaluation |

---

## Rules

1. **One block per source artifact.** Each `artifact_evaluation` or `evaluation_error` block covers exactly one source artifact. Place each block in its own fenced YAML code block.

2. **List fields must have ≥1 entry.** All five list fields (`useful_elements`, `missing_information`, `information_not_used`, `inaccuracies`, `impact_on_work`) must contain at least one entry. If nothing applies, use `"N/A — none identified"`.

3. **Score range.** Both `usefulness_score` and `clarity_score` must be integers in the range 1–10 (inclusive).

4. **Non-blocking behavior.** Evaluation failure must **never** cause the agent's completion status to be ERROR. If evaluation cannot be completed, write an `evaluation_error` block and proceed. The agent returns DONE with its primary output intact.

5. **Evaluation is secondary.** The agent's primary task output always takes priority. Evaluation is performed after primary work is complete and before self-verification or memory writing.

6. **Collision avoidance.** If the evaluation file already exists (e.g., from a NEEDS_REVISION re-execution or same-feature re-run), append a sequence suffix to the filename: `<agent-name>-2.md`, `<agent-name>-3.md`, etc. No files are ever overwritten.

7. **Output file format.** The evaluation file is a Markdown file containing one or more fenced YAML code blocks:

   ````markdown
   # Artifact Evaluations by <Agent Name>

   > Schema: `.github/agents/evaluation-schema.md`

   ```yaml
   artifact_evaluation:
     evaluator: "spec"
     source_artifact: "research/architecture.md"
     usefulness_score: 8
     clarity_score: 9
     useful_elements:
       - "§Repository Structure provided clear layout"
     missing_information:
       - "N/A — none identified"
     information_not_used:
       - "§README vs. Codebase Divergences was not relevant to spec work"
     inaccuracies:
       - "N/A — none identified"
     impact_on_work:
       - "N/A — none identified"
   ```

   ```yaml
   artifact_evaluation:
     evaluator: "spec"
     source_artifact: "research/impact.md"
     usefulness_score: 7
     clarity_score: 8
     useful_elements:
       - "§Existing Agent Files Requiring Modification"
     missing_information:
       - "Edge case coverage for evaluation failures"
     information_not_used:
       - "N/A — none identified"
     inaccuracies:
       - "N/A — none identified"
     impact_on_work:
       - "Minor rework to add edge case handling for missing evaluations"
   ```
   ````
