# Task 05: CT Sub-Agents — Security + Scalability

## Agent

implementer

## Depends On

none

## Description

Create two new Critical Thinking cluster sub-agent files: `ct-security.agent.md` and `ct-scalability.agent.md`. Each follows the v2 agent template, inherits the adversarial mindset from the current critical-thinker, and focuses on a specific subset of risk categories. Both are parallel agents that read `memory.md` but do NOT write to it.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §3.1 (CT Sub-Agent Definitions), §8.1 (Memory Protocol)
- `docs/feature/forge-architecture-upgrade/feature.md` — CT-AC-1 through CT-AC-9
- `NewAgentsAndPrompts/critical-thinker.agent.md` — current file as structural reference for mindset/workflow

## Output File

- `NewAgentsAndPrompts/ct-security.agent.md`
- `NewAgentsAndPrompts/ct-scalability.agent.md`

## Acceptance Criteria

1. Both files follow v2 template: YAML frontmatter, role statement, inputs/outputs, 5 operating rules + Rule 6 (memory-first), workflow, output spec, completion contract, anti-drift anchor
2. **ct-security** covers: security vulnerabilities, backwards compatibility risks. Key question: "What can be exploited?" "What breaks for existing users?"
3. **ct-scalability** covers: scalability bottlenecks, performance implications. Key question: "What happens at 10× load?" "Where are the N+1 patterns?"
4. Both include the adversarial mindset section from the current critical-thinker
5. Both list inputs: `initial-request.md`, `design.md`, `feature.md`, `memory.md` (memory FIRST)
6. Both output to `docs/feature/<feature-slug>/ct-review/ct-<focus>.md`
7. Both include mandatory "Cross-Cutting Observations" section in their output format
8. Completion contract: `DONE:` / `ERROR:` only (no `NEEDS_REVISION` — only the aggregator decides)
9. Both do NOT write to `memory.md` (parallel agents — findings go to intermediate output files)
10. Each sub-agent's output follows the standardized format from design.md §3.1

## Implementation Guidance

**Structure (both files identical in structure, different in focus):**

```
---
name: ct-security (or ct-scalability)
description: "<focus-specific description>"
---

# CT-Security (or CT-Scalability) Agent

You are the **CT-Security** (or CT-Scalability) agent...
[Role statement — inherit adversarial mindset from critical-thinker.agent.md]
[Scoped to specific risk categories]
You NEVER write code, designs, plans, or specifications.
You NEVER propose solutions — you identify problems.

## Inputs
- docs/feature/<feature-slug>/memory.md (read first — operational memory)
- docs/feature/<feature-slug>/initial-request.md
- docs/feature/<feature-slug>/design.md
- docs/feature/<feature-slug>/feature.md

## Outputs
- docs/feature/<feature-slug>/ct-review/ct-<focus>.md

## Operating Rules
[5 standard rules from v2 template + Rule 6 memory-first reading]

## Mindset
[Inherit adversarial mindset from critical-thinker.agent.md — identical across all CT sub-agents]

## Risk Categories
[Focus-specific subset — see design.md §3.1 for exact categories]

## Workflow
1. Read `memory.md` for orientation
2. Read `initial-request.md` — is the design solving the right problem?
3. Read `design.md` — probe for risks in assigned categories
4. Read `feature.md` — check requirement coverage
5. Dig into the codebase to verify design claims
6. Write findings to ct-review/ct-<focus>.md using standardized format
7. Self-verification: confirm all findings are specific and grounded

## Output Format
[Standardized sub-agent output format from design.md §3.1]

## Completion Contract
- DONE: <focus-area> — <one-line summary of findings>
- ERROR: <reason>

## Anti-Drift Anchor
```

**Key differentiation:**

- **ct-security:** Risk categories = Security vulnerabilities + Backwards compatibility. Probe authentication/authorization gaps, data exposure, injection vectors, insecure defaults, breaking changes.
- **ct-scalability:** Risk categories = Scalability bottlenecks + Performance implications. Probe single points of failure, unbounded growth, missing pagination, N+1 patterns, expensive operations.

**Output format (from design.md §3.1):**

```markdown
# Critical Review: <Focus Area>

## Findings

### [Severity: Critical/High/Medium/Low] Finding Title

- **What:** Specific risk description
- **Where:** File, component, or design section
- **Likelihood:** High / Medium / Low
- **Impact:** High / Medium / Low
- **Assumption at risk:** What underlying assumption this challenges

## Cross-Cutting Observations

- Observation with reference to which other sub-agent's scope it belongs in

## Requirement Coverage

| Requirement | Coverage Status | Notes |
```

**Note:** These are markdown agent definition files — TDD does not apply. Create the files directly.

## Status

**Complete.** TDD skipped: markdown agent definition files — no behavioral code.

### Acceptance Criteria Checklist

- [x] 1. Both files follow v2 template: YAML frontmatter, role statement, inputs/outputs, 5 operating rules + Rule 6 (memory-first), workflow, output spec, completion contract, anti-drift anchor
- [x] 2. ct-security covers: security vulnerabilities, backwards compatibility risks
- [x] 3. ct-scalability covers: scalability bottlenecks, performance implications
- [x] 4. Both include the adversarial mindset section from the current critical-thinker
- [x] 5. Both list inputs: memory.md (FIRST), initial-request.md, design.md, feature.md
- [x] 6. Both output to docs/feature/<feature-slug>/ct-review/ct-<focus>.md
- [x] 7. Both include mandatory "Cross-Cutting Observations" section in output format
- [x] 8. Completion contract: DONE/ERROR only (no NEEDS_REVISION)
- [x] 9. Both do NOT write to memory.md (parallel agents)
- [x] 10. Each sub-agent's output follows the standardized format from design.md §3.1
