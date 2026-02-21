# Memory: researcher-patterns

## Status

DONE: Comprehensive patterns analysis complete — 16 pattern categories documented across all 19 active agent files

## Key Findings

- All agents follow identical chatagent format: YAML frontmatter (name/description), 6 standard Operating Rules, Workflow, Completion Contract, Anti-Drift Anchor — any new agent (PostMortem) must replicate this structure exactly
- No YAML/JSON structured output exists anywhere in the codebase — all inter-agent communication is Markdown-only; the proposed artifact_evaluation YAML would be the first structured data format in agent outputs
- Error handling is copy-paste identical across all 19 agents (retry 2x transient, no retry deterministic, 6 max total calls) with one exception: R-Security uses `severity: blocker` instead of `severity: critical`
- KE-SAFE-\* numbered safety constraints in R-Knowledge (file boundaries, suggestion completeness/categorization, safety filter, append-only decisions) establish the pattern for constraining agents that propose changes
- Self-verification is implemented in spec, designer, implementer, all CT sub-agents, and R-quality/R-security — always as penultimate workflow step before writing memory; the non-blocking R-Knowledge pattern is the model for a PostMortem agent

## Highest Severity

N/A

## Decisions Made

None

## Artifact Index

- research/patterns.md — §Agent File Format Pattern (canonical structure for new agents)
- research/patterns.md — §Isolated Memory File Format Pattern (memory file structure and naming variations)
- research/patterns.md — §Self-Verification Patterns (per-agent verification approaches)
- research/patterns.md — §Knowledge-Suggestions.md Pattern (structured suggestion format for human review)
- research/patterns.md — §Safety Constraint Filter Pattern (KE-SAFE-\* system for constraining modification proposals)
- research/patterns.md — §Error Handling Patterns (uniform error handling and retry budgets)
- research/patterns.md — §YAML/Structured Output Patterns (no existing YAML output — Markdown only)
- research/patterns.md — §Naming Conventions (file naming, directory structure, memory file naming)
- research/patterns.md — §Completion Contract Patterns (two-state vs three-state vs custom)
- research/patterns.md — §Non-Blocking Agent Pattern (R-Knowledge as model for PostMortem)
