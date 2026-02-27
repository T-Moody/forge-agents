# Artifact Evaluations by V-Feature

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "v-feature"
  source_artifact: "verification/v-build.md"
  usefulness_score: 9
  clarity_score: 10
  useful_elements:
    - "§File Manifest Verification table provided exact file counts, line counts, and section line numbers for all 14 files — eliminated need for independent file enumeration"
    - "§Structural Checks matrix confirmed Completion Contract, Self-Verification, and Anti-Drift Anchor presence across all 9 agents — directly accelerated AC-3, AC-12, CR-12 verification"
    - "Per-file line counts enabled targeted read_file calls with accurate line ranges"
  missing_information:
    - "No content-level checks — v-build confirmed structural presence of sections but not whether content within those sections is correct (e.g., severity vocabulary consistency)"
  information_not_used:
    - "§Environment section (OS, tool versions) — not relevant to feature-level acceptance criteria verification"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified; v-build data was reliable and saved significant initial discovery time"
```

```yaml
artifact_evaluation:
  evaluator: "v-feature"
  source_artifact: "feature.md"
  usefulness_score: 10
  clarity_score: 9
  useful_elements:
    - "§Acceptance Criteria (AC-1 through AC-15) with explicit Pass/Fail definitions — provided the testable criteria framework for the entire verification"
    - "§Common Requirements (CR-1 through CR-15) with precise wording — enabled exact-match verification against implementation"
    - "§Architectural Directions with tradeoff tables — provided context for understanding design deviations (e.g., DR-2 reviewer count)"
    - "§Functional Requirements (FR-1 through FR-9) — provided detailed sub-criteria for verifying AC-level claims"
  missing_information:
    - "AC-7 does not specify whether the severity taxonomy applies only to formal finding severity fields or also to informal prose references (e.g., edge case severity-if-missed in spec) — this ambiguity made the partially-met determination borderline"
  information_not_used:
    - "§Edge Cases (EC-1 through EC-10) — not directly verifiable at the agent-definition level; these describe runtime behavior"
    - "§Test Scenarios (TS-1 through TS-20) — not verifiable without a running pipeline"
    - "§User Stories (US-1 through US-3) — descriptive flows, not testable acceptance criteria"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified; the acceptance criteria were clear and testable"
```
