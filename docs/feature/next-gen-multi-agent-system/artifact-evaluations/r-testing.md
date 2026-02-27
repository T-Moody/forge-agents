# Artifact Evaluations by R-Testing

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "r-testing"
  source_artifact: "design.md"
  usefulness_score: 9
  clarity_score: 8
  useful_elements:
    - "§Decision 6 (Verification Architecture) — defined 4-tier cascade, check_name conventions, and SQL evidence gates with precise queries"
    - "§Decision 7 (Adversarial Review) — disagreement resolution protocol with clear routing table"
    - "§Deviation Records DR-1 and DR-2 — explicitly documented spec deviations with rationale, enabling verification against original spec"
    - "§Evidence Bundle (Step 8b) — defined 6-component proof-of-quality structure"
  missing_information:
    - "No SQLite fallback path defined for EC-4 (SQLite unavailable) — design mandates SQL but doesn't specify degraded-mode operation"
    - "No approval timeout handling protocol for EC-2 — design defines approval gates but not timeout/resume behavior"
    - "No context window escalation protocol for EC-8 — design mentions efficient reading but no detection or recovery mechanism"
  information_not_used:
    - "N/A — none identified"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "Missing fallback paths required additional analysis to determine whether edge case gaps were design omissions vs. implementation omissions — concluded they are design-level gaps propagated into implementation"
```

```yaml
artifact_evaluation:
  evaluator: "r-testing"
  source_artifact: "feature.md"
  usefulness_score: 10
  clarity_score: 9
  useful_elements:
    - "§Edge Cases (EC-1 through EC-10) — each with Input/Condition, Expected Behavior, and Severity if Missed — directly usable as test scenario source"
    - "§Test Scenarios (TS-1 through TS-20) — mapped to Acceptance Criteria with clear verification methods"
    - "§Acceptance Criteria (AC-1 through AC-15) — testable criteria with pass/fail definitions"
    - "§Risk table — likelihood/impact/mitigation for each identified risk, enabling prioritized test focus"
  missing_information:
    - "No test scenario defined for EC-2 (approval timeout) or EC-7 (replan loop exhaustion) — these are specified as edge cases but lack corresponding TS-N entries"
    - "No test scenario for EC-8 (context window) or EC-10 (concurrency limit) — present as edge cases but not mapped to verification steps"
  information_not_used:
    - "§Architectural Directions B through E — only Direction A Enhanced was selected, so alternative directions were not relevant to testing review"
  inaccuracies:
    - "TS-2 and TS-3 reference reviewer counts of '1 for standard, 3 for high risk' but the final design always dispatches 3 reviewers (DR-2 deviation) — test scenarios need updating to match the design deviation"
  impact_on_work:
    - "TS-2/TS-3 inconsistency with DR-2 required cross-referencing design.md to determine which was authoritative — design.md DR-2 (always 3 reviewers) is the correct specification"
```
