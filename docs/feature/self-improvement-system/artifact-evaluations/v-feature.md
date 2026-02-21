# Artifact Evaluations by V-Feature

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "v-feature"
  source_artifact: "verification/v-build.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "§File Existence Verification provided complete inventory of all 2 new + 16 modified + 5 excluded files with task mappings"
    - "§Structural Content Checks gave 12 targeted checks confirming key implementation integration points"
    - "§Format Integrity YAML frontmatter check across all 21 agent files provided rapid format validation"
    - "§Warnings section identified the post-mortem.agent.md code fence format issue — useful context for feature-level verification"
  missing_information:
    - "No content-level verification of evaluation step wording consistency across the 14 agents — only existence was confirmed"
  information_not_used:
    - "§Environment details (OS, PowerShell version) — not relevant to feature-level acceptance criteria verification"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```

```yaml
artifact_evaluation:
  evaluator: "v-feature"
  source_artifact: "feature.md"
  usefulness_score: 10
  clarity_score: 9
  useful_elements:
    - "§Acceptance Criteria (AC-1 through AC-14) provided clear, testable pass/fail definitions for every criterion"
    - "§Functional Requirements (FR-1 through FR-8) gave detailed specifications that enabled precise verification against implementation"
    - "§Edge Cases & Error Handling defined expected behaviors that informed regression-check scope"
    - "§Test Scenarios (TS-1 through TS-15) provided verification procedures mapped to ACs"
  missing_information:
    - "AC-5 text does not reflect the CT-approved design revision (retaining read tools) — required external instruction to know which version to verify against"
  information_not_used:
    - "§User Stories / Flows — not directly needed for acceptance criteria verification (covered by ACs and TSs)"
    - "§Dependencies & Risks — informational context not directly used in pass/fail verification"
  inaccuracies:
    - "AC-5 literal text specifies [agent, agent/runSubagent, memory] only, which conflicts with the CT-approved design revision that retains read tools — the AC text is outdated relative to the design"
  impact_on_work:
    - "Required special handling of AC-5: verified against design revision rather than literal AC-5 text, per orchestrator instruction noting the CT cluster resolution"
```
