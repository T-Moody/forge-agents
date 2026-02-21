# Artifact Evaluations by V-Feature

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "v-feature"
  source_artifact: "verification/v-build.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "§Structural Validation Performed — YAML frontmatter validation confirmed tools field syntax"
    - "§Prohibited Tool References Check — pre-verified grep results with line numbers and context classification"
    - "§Warnings — trailing empty code fences flagged as cosmetic (useful to note but non-blocking)"
  missing_information:
    - "N/A — none identified"
  information_not_used:
    - "§Environment section (OS/PowerShell version) — not relevant to feature-level acceptance criteria verification"
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
  clarity_score: 8
  useful_elements:
    - "§Acceptance Criteria — clear pass/fail definitions for each AC enabled direct verification"
    - "§Phase 1 Scope — explicit in-scope/deferred table with rationale prevented verification of out-of-scope ACs"
    - "§AC-12 Phase 1 Interpretation — blockquote clarified what to verify for AC-12 when memory.md is preserved"
    - "§Functional Requirements — FR-1 through FR-6 provided implementation detail behind each AC"
  missing_information:
    - "AC-3 pass criterion ('returns zero matches') is overly strict — should note that prohibited tools necessarily appear in prohibition contexts. The Phase 1 Scope section could include an AC-3 interpretation note similar to AC-12's."
  information_not_used:
    - "§Edge Cases (EC-1 through EC-8) — not relevant to Phase 1 verification scope"
    - "§Test Scenarios — not verifiable in Phase 1 (no runtime tests)"
    - "§Dependencies & Risks — informational context only"
  inaccuracies:
    - "AC-3 strict pass criterion conflicts with design intent — the design (§1.3) explicitly adds prohibited tool names to Operating Rule 5 and Anti-Drift Anchor, but AC-3 says zero matches anywhere"
  impact_on_work:
    - "Required pragmatic interpretation of AC-3 rather than strict literal reading — noted as cross-cutting observation in verification report"
```
