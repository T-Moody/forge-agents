# Artifact Evaluations by R-Quality

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "r-quality"
  source_artifact: "design.md"
  usefulness_score: 8
  clarity_score: 7
  useful_elements:
    - "§CT Revision Summary table — clear mapping of 7 CT findings to design resolutions with section references"
    - "§Per-Agent Evaluation Targets table — definitive list of which agents evaluate which artifacts"
    - "§Agents Excluded table — explicit rationale for why v-build, r-security, r-knowledge have no evaluation step"
    - "§PostMortem vs R-Knowledge boundary table — clear scope delineation preventing overlap"
    - "§Evaluating Agent Changes template — provided a base template for all 14 agents"
  missing_information:
    - "The evaluation step template does not specify source artifact list formatting (header style, bold vs plain), leading to 5 template variants across implementers"
    - "No cross-reference check between the template's error handling text ('skip evaluation') and evaluation-schema.md Rule 4 ('write evaluation_error block') — this conflict propagated to 6 agents"
  information_not_used:
    - "§Security Considerations — not directly relevant to code quality review"
    - "§Telemetry Data Structure field-level details — reviewed for completeness but not quality-relevant"
  inaccuracies:
    - "§Evaluating Agent Changes template (line ~336) says 'skip evaluation and proceed' on failure, contradicting evaluation-schema.md Rule 4 which says 'write an evaluation_error block and proceed' — this is the root cause of the error handling behavioral split across agents"
  impact_on_work:
    - "The template conflict required tracing the error handling inconsistency back to its root cause (design vs schema disagreement) rather than just flagging agent-level differences"
```
