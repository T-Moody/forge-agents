# Memory: ct-strategy

## Status

DONE: strategy — 11 findings (2 Critical, 4 High, 4 Medium, 1 Low). Core contradictions: orchestrator cannot write pipeline manifest without violating tool restrictions; YAML schema validation is prompt-level only (no machine enforcement); SQL evidence gating inaccessible to orchestrator; zero-merge redistributes cost to downstream agents; model routing for custom agents unverified.

## Key Findings

- Pipeline manifest "direct write" contradicts orchestrator's tool restrictions (no create_file/replace_string_in_file); resolving via subagent dispatches inflates dispatch count from 18-22 to ~26-30
- Typed YAML schema validation is prompt-level (LLM-instructed field checking), not machine-enforced; provides no stronger guarantees than structured Markdown when the "parser" is an LLM
- SQL evidence gating requires COUNT queries the orchestrator cannot execute (run_in_terminal forbidden); orchestrator must trust Verifier self-reports, undermining anti-hallucination property
- "Zero-merge" eliminates explicit merge steps but distributes synthesis cost to every downstream agent reading N upstream files; late-pipeline agents (Verifier, Knowledge Agent) face O(N) file reads
- Multi-model adversarial review uses custom agent dispatch, not Anvil's platform `code-review` type; model routing for custom agents is unverified (A-2 assumption extended without evidence)

## Highest Severity

Critical

## Decisions Made

(none — CT-Strategy identifies problems, does not make decisions)

## Artifact Index

- [ct-review/ct-strategy.md](../ct-review/ct-strategy.md)
  - §Findings — 11 findings across Critical/High/Medium/Low severity
  - §Cross-Cutting Observations — security review fallback, schema versioning, scalability of zero-merge model
  - §Requirement Coverage — 19 requirements assessed with coverage status and notes
