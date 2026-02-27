# Artifact Evaluations by CT-Maintainability

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "ct-maintainability"
  source_artifact: "design.md"
  usefulness_score: 9
  clarity_score: 8
  useful_elements:
    - "§Decision 2 Agent Inventory with detailed inputs/outputs/tools per agent enabled precise analysis of orchestrator coupling surface and per-agent complexity"
    - "§Decision 4 Schema Validation Protocol's explicit acknowledgment that validation is 'prompt-level, not runtime-enforced' enabled identification of the foundational enforcement gap"
    - "§Decision 6 Evidence Gating Mechanism table with both SQL and YAML columns enabled analysis of dual-track maintenance burden"
    - "§Orchestrator Decision Table (15+ rows) made the routing coupling explicit and analyzable"
    - "§Pipeline Manifest Schema provided concrete YAML structure to analyze growth characteristics"
    - "§Decision 10 Agent Definition Structure template enabled assessment of agent-level maintainability patterns"
  missing_information:
    - "No schema dependency graph showing which agents produce vs. consume each of the 14 schemas — had to reconstruct from agent detail sections"
    - "No sizing estimate for the pipeline manifest (expected line count for typical vs. complex features) — had to calculate from dispatch count analysis"
    - "No discussion of schema evolution strategy or versioning beyond 'schema_version: 1.0' — a critical gap for a system built on 14 typed schemas"
    - "No process documentation for adding/removing agents from the pipeline"
  information_not_used:
    - "§Security Considerations Authentication/Authorization section — N/A for maintainability review"
    - "§Migration & Backwards Compatibility — greenfield build, no migration concerns to analyze"
    - "§Approval Mode example YAML payloads — detailed but not relevant to structural maintainability analysis"
  inaccuracies:
    - "§Decision 6 claims Verifier has 'bounded scope (tool execution, not reasoning)' but regression detection (baseline comparison) and acceptance criteria verification against spec are reasoning tasks, not tool execution"
    - "§Decision 2 claims risk 🟢 for agent inventory because 'each agent has a single responsibility' but the Verifier has at least 4 internal responsibilities (build gate, test execution, acceptance criteria verification, evidence ledger management)"
  impact_on_work:
    - "The 1588-line design document required extensive reading to identify coupling surfaces — a design-section-to-agent mapping or reading guide would have accelerated review"
    - "Missing schema dependency graph required cross-referencing agent detail sections to understand which schemas create coupling — this analysis took significant time"
```

```yaml
artifact_evaluation:
  evaluator: "ct-maintainability"
  source_artifact: "feature.md"
  usefulness_score: 7
  clarity_score: 9
  useful_elements:
    - "§Common Requirements CR-1 through CR-15 provided clear testable constraints to evaluate design compliance against"
    - "§NFR-6 Maintainability requirement ('independently updateable agents') provided the key standard for evaluating agent coupling"
    - "§NFR-5 Context Efficiency requirement helped frame the pipeline manifest growth concern"
    - "§Acceptance Criteria AC-1 through AC-15 with pass/fail definitions enabled requirement coverage assessment"
  missing_information:
    - "No explicit requirement about schema evolution or versioning — given the typed-schema-first architecture, this is a significant gap"
    - "No requirement addressing the complexity threshold for individual agents (e.g., maximum prompt length, maximum internal state transitions) — the Verifier complexity finding would be easier to ground with such a requirement"
  information_not_used:
    - "§Architectural Directions B, C, E details — designer already selected and synthesized; direction-specific analysis was not needed for maintainability review"
    - "§User Stories US-1 through US-3 — narrative format did not add maintainability-relevant information beyond what CR/FR/NFR provided"
    - "§Test Scenarios TS-1 through TS-20 — testing detail not directly relevant to maintainability structure review"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "The absence of a schema evolution requirement meant the High finding about schema maintainability (Finding 2) had to be argued from first principles rather than grounded in a specific requirement gap"
```
