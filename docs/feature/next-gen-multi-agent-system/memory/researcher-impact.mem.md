# Memory: researcher-impact

## Status

DONE: Comprehensive impact analysis of all 23 Forge agents, Anvil components, and merge/remove/introduce candidates completed with quantitative backing from one full pipeline run.

## Key Findings

- CT cluster (4 agents) is the primary restructuring target: high prompt duplication (~50% shared content), caused 3-iteration bottleneck (12 agent invocations for one design review), and is replaceable by Anvil's multi-model adversarial review pattern
- R cluster's r-quality, r-security, r-testing (3 agents) perform same-model-different-prompt review that Anvil achieves with genuinely different models; r-knowledge is unique and should be retained
- V cluster's v-tasks and v-feature overlap (both verify acceptance criteria at different granularity) and can merge; v-build and v-tests provide essential ground-truth signals and should stay
- Memory system creates ~20+ files with ~12 merge operations per run — the Markdown semi-structured memory pattern should be replaced with typed schemas per the article's first principle
- Proposed direction reduces agent count from 21 active to ~10–12, eliminating 9–11 agents while adding 1 new adversarial reviewer agent (parameterized by model and review scope)

## Highest Severity

N/A

## Decisions Made

(none — research only)

## Artifact Index

- research/impact.md — §Forge Agent Inventory and Value Assessment (per-agent value ratings with post-mortem scores), §Anvil Components — Impact of Incorporation (6 components with affected-file lists), §Memory System Impact (overhead quantification and typed-schema replacement), §Artifact Evaluation System Impact (cost-vs-value analysis with 3 options), §Component Removal Impact (5 removal candidates with risk assessment), §New Components Impact (4 new components with pipeline integration points), §Agent Count Impact Summary (net reduction table)
