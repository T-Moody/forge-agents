# Task 06: CT Sub-Agents — Maintainability + Strategy

**Status: DONE**

## Agent

implementer

## Depends On

none

## Description

Create two new Critical Thinking cluster sub-agent files: `ct-maintainability.agent.md` and `ct-strategy.agent.md`. Same structural pattern as Task 05. Both are parallel agents that read `memory.md` but do NOT write to it.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §3.1 (CT Sub-Agent Definitions), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — CT-AC-1 through CT-AC-9
- `NewAgentsAndPrompts/critical-thinker.agent.md` — current file as structural reference

## Output File

- `NewAgentsAndPrompts/ct-maintainability.agent.md`
- `NewAgentsAndPrompts/ct-strategy.agent.md`

## Acceptance Criteria

1. Both files follow v2 template: YAML frontmatter, role statement, inputs/outputs, 5 operating rules + Rule 6 (memory-first), workflow, output spec, completion contract, anti-drift anchor
2. **ct-maintainability** covers: maintainability concerns, complexity risks, integration risks. Key question: "Will a new dev understand this in 6 months?" "What's the coupling surface?"
3. **ct-strategy** covers: strategic risks, scope risks, edge cases, fundamental approach validity. Key question: "Is this the right approach at all?" "What did the design quietly drop?"
4. Both include the adversarial mindset section from the current critical-thinker
5. Both list inputs: `memory.md` (FIRST), `initial-request.md`, `design.md`, `feature.md`
6. Both output to `docs/feature/<feature-slug>/ct-review/ct-<focus>.md`
7. Both include mandatory "Cross-Cutting Observations" section in output format
8. Completion contract: `DONE:` / `ERROR:` only
9. Both do NOT write to `memory.md` (parallel agents)
10. Output follows standardized format from design.md §3.1

## Implementation Guidance

Follow the same structural pattern as Task 05 (ct-security and ct-scalability). The only differences are the focus-specific content:

**ct-maintainability:**

- Risk categories: Maintainability concerns, Complexity risks, Integration risks
- Probe: tight coupling, missing abstractions, unclear boundaries, violation of project conventions, complexity proportional to problem
- Key questions: "Will a new developer understand this in 6 months?", "What's the coupling surface?", "Could 80% of the value be delivered with 20% of the design?"

**ct-strategy:**

- Risk categories: Strategic risks, Scope risks, Edge cases, Fundamental approach validity
- Probe: long-term maintenance burden, future options foreclosed, gold-plating vs corner-cutting, scope alignment with initial request
- Key questions: "Is this the right approach at all?", "What did the design quietly drop?", "What's the worst thing that could happen if this design ships as-is?"
- This is the broadest sub-agent — it explicitly asks whether the fundamental approach is correct

**Both use the same standardized output format from design.md §3.1** (see Task 05 for the template).

**Note:** These are markdown agent definition files — TDD does not apply. Create the files directly.
