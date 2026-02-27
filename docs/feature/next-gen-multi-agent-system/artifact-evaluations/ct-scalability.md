# Artifact Evaluations by CT-Scalability

> Schema: `.github/agents/evaluation-schema.md`

```yaml
artifact_evaluation:
  evaluator: "ct-scalability"
  source_artifact: "design.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "§Decision 3: Pipeline Structure — Dispatch Count Analysis provided concrete numbers to challenge"
    - "§Decision 6: Verification Architecture — SQL/YAML schemas and evidence gating mechanism clearly defined"
    - "§Decision 5: Memory System Redesign — Quantified old vs. new comparison (12 merges → 0) enabled read-amplification analysis"
    - "§Pipeline Manifest Schema — Full YAML schema enabled growth analysis"
    - "§Failure & Recovery — Explicit retry budgets and worst-case bounds enabled total dispatch calculation"
  missing_information:
    - "No dispatch count analysis for 10+ task features despite NFR-3 requiring it"
    - "No context window budget estimation for any agent (bytes/tokens consumed per step)"
    - "No definition of Planner 'replan mode' scope — full vs. targeted replanning is unspecified"
    - "No mechanism for EC-8 'request orchestrator for summarized context' — referenced in feature.md but not designed"
    - "No analysis of Verifier tool output accumulation (build logs, test output sizes)"
  information_not_used:
    - "§Decision 9: Approval Mode — not relevant to scalability analysis"
    - "§Security Considerations — reviewed but outside primary scope"
    - "§Migration & Backwards Compatibility — not relevant (greenfield)"
  inaccuracies:
    - "§Decision 6 rates Verifier risk as mitigated by 'bounded scope (tool execution, not reasoning)' — tool output accumulation IS the unbounded element, not reasoning complexity"
    - "§Decision 3 presents 18-22 dispatches as representative when it only applies to 6-task features; the design's own requirements mandate 10+ task support"
  impact_on_work:
    - "Had to manually compute dispatch counts for 20-task scenarios since the design only analyzed 6-task"
    - "Had to infer YAML ledger growth patterns since no analysis of verification record counts at scale was provided"
```

```yaml
artifact_evaluation:
  evaluator: "ct-scalability"
  source_artifact: "feature.md"
  usefulness_score: 8
  clarity_score: 8
  useful_elements:
    - "§NFR-3: Scalability — explicit requirement for 10+ tasks, 50+ files provided the benchmark for stress-testing the design"
    - "§NFR-5: Context Efficiency — requirement for summaries-before-full-artifacts provided the lens for evaluating zero-merge model tradeoffs"
    - "§EC-8: Agent Context Window Exceeded — identified the gap between requirement and design's coverage"
    - "§EC-10: Concurrent Subagent Limit Hit — confirmed sub-wave partitioning was required"
    - "§Direction C: Anvil-Core cons — 'Large feature.md/design.md may not fit single agent context windows' directly relevant to the same risk in the selected design"
  missing_information:
    - "No quantitative scalability targets (e.g., max tasks, max files, max pipeline duration)"
    - "No context window token budget specifications — requirements say 'MUST NOT exceed' but give no size guidance"
  information_not_used:
    - "§User Stories / Flows — used the design's sequence description instead"
    - "§Architectural Directions B, C, E — rejected directions not analyzed for scalability"
    - "§Test Scenarios — outside scalability scope"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "N/A — none identified"
```
