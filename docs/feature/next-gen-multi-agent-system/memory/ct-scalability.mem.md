# Memory: ct-scalability

## Status

DONE: Identified 10 scalability/performance findings (3 High, 5 Medium, 2 Low). Primary risks: Verifier context window explosion at scale, orchestrator monotonic context growth, YAML ledger O(N) evidence gating, and dispatch count understatement (18-22 claim only valid for 6-task features; 39-91 for 20-task features).

## Key Findings

- Dispatch count 18-22 only valid for 6-task features; 20-task features generate 39-91 dispatches depending on replan loops
- Unified Verifier accumulates 32+ tool call outputs per wave (build logs, test results, diagnostics) in a single context window — most likely agent to hit context limits
- Orchestrator context grows monotonically across all pipeline steps with no shedding mechanism; inherits same problem as Forge's 544-line orchestrator
- YAML verification ledger fallback creates O(tasks × records) N+1 pattern for evidence gate evaluation at scale
- Zero-merge memory model eliminates merge dispatches but introduces read-amplification: every Implementer reads full design-output.yaml and spec-output.yaml regardless of task scope

## Highest Severity

High

## Decisions Made

(none — review agent, no decisions to make)

## Artifact Index

- ct-review/ct-scalability.md
  - §Findings — 10 findings (3 High, 5 Medium, 2 Low) covering dispatch scaling, Verifier context, orchestrator context, YAML ledger N+1, upstream YAML read amplification, concurrency wall, adversarial review budget, replan scope ambiguity, manifest growth, and ledger wave re-reads
  - §Cross-Cutting Observations — 3 observations for maintainability, security, and strategy scopes
  - §Requirement Coverage — 7 requirements mapped with coverage status and gaps
