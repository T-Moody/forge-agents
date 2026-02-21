# Artifact Evaluations by V-Tasks

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "v-tasks"
  source_artifact: "verification/v-build.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "§Structural Validation Performed — YAML frontmatter validation, prohibited tool references check with exact line numbers"
    - "§Warnings — trailing empty code fences noted as cosmetic, preventing false alarm during task verification"
    - "§Build Artifacts — confirmed file line counts useful for scope validation"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Environment details (OS, PowerShell version) — not relevant to per-task acceptance criteria verification"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "v-tasks"
  source_artifact: "plan.md"
  usefulness_score: 8
  clarity_score: 9
  useful_elements:
    - "§Ordered Task Index — clear mapping of tasks to files and design sections"
    - "§Success Criteria — 5-AC scope with AC-to-design-section mapping enabled targeted verification"
    - "§Execution Waves — dependency graph clarified task sequencing for verification order"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Pre-Mortem Analysis (referenced in planner memory) — not directly needed for acceptance criteria verification"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "v-tasks"
  source_artifact: "tasks/01-tool-restriction-prose.md"
  usefulness_score: 9
  clarity_score: 10
  useful_elements:
    - "§Acceptance Criteria — precise, testable criteria with AC mapping (AC-1, AC-2, AC-3 partial, AC-11 partial, AC-12)"
    - "§Test Requirements — T-1 through T-5 plus negative tests T-N1/T-N2 gave exact grep patterns to verify"
    - "§Completion Checklist — pre-checked boxes confirmed implementer self-verification"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Implementation Steps — not needed for verification (verification checks outputs, not process)"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "v-tasks"
  source_artifact: "tasks/02-memory-disambiguation-merge-wording.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "§Acceptance Criteria — clear enumeration of all 8 merge steps and 4 disambiguation fixes"
    - "§Test Requirements — T-8 through T-13 with exact search patterns for automated verification"
    - "§Findings — proactive documentation of the 3 remaining 'no subagent invocation' matches in cluster headers, preventing false-positive flagging"
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
  evaluator: "v-tasks"
  source_artifact: "tasks/03-feature-workflow-update.md"
  usefulness_score: 8
  clarity_score: 9
  useful_elements:
    - "§Acceptance Criteria — 4 clear testable criteria covering content, excluded tools, and no-other-modification guarantee"
    - "§In-scope/Out-of-scope — explicit boundaries prevented scope confusion"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Implementation Steps — not needed for verification"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "v-tasks"
  source_artifact: "tasks/04-feature-spec-phase1-scope.md"
  usefulness_score: 8
  clarity_score: 9
  useful_elements:
    - "§Acceptance Criteria — precise counts (5 in-scope, 10 deferred) and specific AC-12 interpretation requirement"
    - "§Test Requirements — count-based verification criteria (5 in-scope, 10 deferred) were straightforward to validate"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Implementation Steps — not needed for verification"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```
