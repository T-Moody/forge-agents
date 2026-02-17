# Task 15: Feature Workflow Prompt Update

## Agent

implementer

## Depends On

14

## Description

Update `feature-workflow.prompt.md` with references to the memory system, cluster model, and knowledge evolution. This is a minor additive change (~10 lines) to ensure the user-facing workflow prompt reflects the upgraded architecture.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §8.2 (Feature Workflow Prompt Updates)
- `NewAgentsAndPrompts/feature-workflow.prompt.md` — current file
- `NewAgentsAndPrompts/orchestrator.agent.md` — reference for pipeline overview (updated in task 14)

## Output File

- `NewAgentsAndPrompts/feature-workflow.prompt.md` (modified in place)

## Acceptance Criteria

1. [x] Prompt mentions that `memory.md` is created and maintained across the pipeline
2. [x] Prompt references the cluster model (CT, V, R clusters replace single agents)
3. [x] Prompt mentions knowledge evolution (`knowledge-suggestions.md` produced during review)
4. [x] Prompt mentions the 4th research focus area (`patterns`)
5. [x] Changes are additive — no existing prompt content is removed
6. [x] Changes are concise (~10 lines of additions)

## Status

Complete — TDD skipped: markdown-only prompt file, no behavioral code.

## Implementation Guidance

**Approach:** Add brief references to the new capabilities in appropriate sections of the existing prompt file. Do not restructure the prompt — just add awareness of the new features.

**Specific additions (from design.md §8.2):**

1. In the pipeline overview section, add mention of:
   - Memory system: `memory.md` is maintained across pipeline steps
   - Cluster parallelization: CT, V, R steps use parallel sub-agents
   - Research: now includes 4 focus areas (architecture, dependencies, impact, patterns)

2. In the outputs section, add:
   - `memory.md` — pipeline memory (maintained by orchestrator)
   - `knowledge-suggestions.md` — knowledge evolution suggestions (review for potential improvements)
   - `decisions.md` — architectural decision log

3. Keep additions brief — the prompt is user-facing and should remain concise.

**Note:** This is a markdown prompt file — TDD does not apply.
