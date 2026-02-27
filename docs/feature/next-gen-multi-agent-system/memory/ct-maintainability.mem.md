# Memory: ct-maintainability

## Status

DONE: Identified 10 findings (3 High, 5 Medium, 2 Low). Primary concerns: orchestrator god-agent risk with 9+ responsibilities, 14 schemas in one file with no evolution strategy, prompt-level validation undermining the typed communication foundation. Design is comprehensive but complexity is front-loaded into the orchestrator and Verifier agents.

## Key Findings

- **[High]** Orchestrator accumulates 9 distinct responsibilities (state, validation, gating, routing, risk assessment, approval, retry, telemetry, SQL detection) — god-agent risk exceeding the existing 544-line Forge orchestrator
- **[High]** 14 typed YAML schemas in one schemas.md file with no versioning strategy, no dependency graph, no evolution process, no change authority — highest-coupling artifact in the system
- **[High]** Prompt-level schema validation is the enforcement layer for the entire typed communication foundation — the LLM orchestrator "checking required fields" is the same unreliable enforcement layer the typed schemas were supposed to replace
- **[Medium]** Unified Verifier hides 4-agent complexity (3-tier cascade × 2 tracks × baseline/after × acceptance criteria) behind claims of "bounded scope (tool execution, not reasoning)" — regression detection and acceptance criteria verification ARE reasoning
- **[Medium]** Adversarial Reviewer parameterization (2 scopes × 3 models × 3 risk levels = 18 configs) unites fundamentally different tasks (design review vs. code review) in a single prompt

## Highest Severity

High

## Decisions Made

(none — CT agents identify problems, not solutions)

## Artifact Index

- [ct-review/ct-maintainability.md](../ct-review/ct-maintainability.md)
  - §Finding 1: Orchestrator God-Agent — 9 responsibilities, coupling to all agents via routing table
  - §Finding 2: Schema Evolution — 14 schemas, no versioning, no dependency graph
  - §Finding 3: Prompt-Level Validation — foundational typed-communication assumption undermined
  - §Finding 4: Unified Verifier — 4 agents behind single interface, hidden state machine
  - §Finding 5: Adversarial Reviewer — 18-config parameterization, design≠code review
  - §Finding 6: Pipeline Manifest Growth — shared state scaling problem in new format
  - §Finding 7: Dual-Track SQL/YAML — doubled maintenance surface, no parity mechanism
  - §Finding 8: Agent Independence — orchestrator coupling limits NFR-6 compliance
  - §Finding 9: Companion Drift — YAML/Markdown inconsistency risk
  - §Finding 10: No Agent Playbook — no documented add/remove process
  - §Cross-Cutting Observations — scalability (state size), security (verdict parsing), strategy (design doc size, test execution)
  - §Requirement Coverage — CR-5 and NFR-5/NFR-6 rated At Risk
