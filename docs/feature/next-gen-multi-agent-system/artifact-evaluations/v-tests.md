# Artifact Evaluations by V-Tests

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "v-tests"
  source_artifact: "verification/v-build.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "§File Manifest Verification table with per-file line counts and section locations — enabled targeted reads instead of scanning all 14 files"
    - "§Structural Checks matrix confirming Completion Contract, Self-Verification, and Anti-Drift Anchor presence — reduced redundant verification effort"
    - "Build system detection (static Markdown artifacts, no compilation) — set correct expectation for test methodology"
  missing_information:
    - "No indication of which severity vocabulary each agent uses — had to read each file to discover the findings_count taxonomy mismatch in run 1"
  information_not_used:
    - "§Environment section (OS, tool versions) — not relevant for content quality inspection"
    - "feature-workflow.prompt.md and README.md entries — outside scope of 9-agent cross-cutting verification"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — v-build.md accurately oriented the verification approach; no rework needed in either run"
```
