# Analysis: Improve Forge Orchestrator and All Related Agents

**Summary:** Forge's 9-agent pipeline can be substantially improved by selectively adopting Gem Team patterns — TDD enforcement, pre-mortem analysis, task size limits, security-focused review, hybrid retrieval, concurrency caps, and cross-cutting operating rules — while preserving Forge's core strengths (deterministic pipeline, dedicated spec/design/critical-thinker stages, dedicated verifier, full autonomy, rich markdown artifact trail).

## Abstract

Three independent research streams (architecture, impact, dependencies) analyzed the Forge multi-agent framework against the Gem Team framework. The research identified 14 actionable improvements beyond the 10 originally proposed in the initial request, discovered 2 pre-existing bugs, mapped all cross-agent data flow dependencies, and classified every change by priority, complexity, and coupling. The improvements fall into 3 tightly-coupled change clusters and 6+ independently-adoptable changes. This document merges and organizes all findings into a single implementation reference.

---

## Scope & Purpose

**What was analyzed:** All 9 Forge agent files (`.github/agents/*.agent.md`), the workflow prompt (`.github/prompts/feature-workflow.prompt.md`), all 8 Gem Team agent files (remote, via URL), and two prior comparison documents (`docs/comparison-forge-vs-gem-team.md`, `docs/optimization-from-gem-team.md`).

**Why:** To identify improvements from Gem Team that can be adopted into Forge without breaking its core architecture, and to discover gaps the prior comparison documents missed.

**Constraints:**

- Output all improved agents to `NewAgentsAndPrompts/`, not `.github/`
- **Full permission to alter any Forge agent in any way that makes it better** — no sacred cows, no architecture constraints
- Do whatever produces the highest-quality, most robust, most effective agent system

---

## Repository Overview

### Forge Architecture

Forge is a multi-agent orchestration framework for GitHub Copilot with a **fixed 8-step deterministic pipeline**:

```
Setup → Research (parallel ×3) → Synthesize → Spec → Design → Critical Review
→ Plan → Implement (parallel waves) → Verify → Review
```

**9 agents** + 1 prompt file:

| Agent                     | Role                                                           | Primary Output                |
| ------------------------- | -------------------------------------------------------------- | ----------------------------- |
| Orchestrator              | Top-level coordinator; dispatches all agents via `runSubagent` | `initial-request.md`          |
| Researcher (focused ×3)   | Parallel codebase research by focus area                       | `research/<focus-area>.md`    |
| Researcher (synthesis)    | Merges all research partials                                   | `analysis.md`                 |
| Spec                      | Formal requirements specification                              | `feature.md`                  |
| Designer                  | Technical design document                                      | `design.md`                   |
| Critical Thinker          | Adversarial design review (unique to Forge)                    | `design_critical_review.md`   |
| Planner                   | Task decomposition with execution waves                        | `plan.md`, `tasks/*.md`       |
| Implementer (×N per wave) | Implements exactly one task                                    | Code files, task file updates |
| Verifier                  | Builds project, runs tests, verifies acceptance criteria       | `verifier.md`                 |
| Reviewer                  | Code quality review, `.github/instructions` updates            | `review.md`                   |

**Key architectural properties:**

- Linear artifact chain: each agent reads prior artifacts and writes exactly one primary output
- Orchestrator is the sole router — agents never invoke other agents
- All inter-agent communication is through distinct markdown artifact files (no shared mutable state)
- Two feedback loops: Verifier→Planner (max 3×) and Reviewer→Planner
- Completion contract: `DONE: <details>` / `ERROR: <reason>` (text-based, per-agent format varies)

### Gem Team Architecture

Gem Team is a comparable multi-agent framework with 8 agents and a **dynamic DAG-driven dispatch** model. Key structural differences:

| Dimension           | Forge                                       | Gem Team                                                               |
| ------------------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| Pipeline model      | Fixed 8-step deterministic                  | Dynamic DAG dispatch                                                   |
| State format        | Distributed markdown artifacts              | Centralized `plan.yaml` (YAML, in-place mutation)                      |
| Agent count         | 9 (spec, designer, critical-thinker unique) | 8 (chrome-tester, devops, doc-writer unique)                           |
| Human gates         | None (autonomous)                           | 3 mandatory pauses                                                     |
| Concurrency         | Unbounded per wave                          | Max 4 concurrent                                                       |
| Verification        | Dedicated verifier agent                    | Inline per-task                                                        |
| Completion contract | Text `DONE:`/`ERROR:` (2 states)            | JSON `{status, id, summary}` (3 states: success/failed/needs_revision) |
| Agent structure     | Free-form markdown headings                 | XML-like semantic tags (`<role>`, `<workflow>`, `<final_anchor>`)      |
| Shared rules        | None — each agent authored independently    | Identical operating rules block in ALL 8 agents                        |

---

## Gem Team Architecture Analysis — Key Patterns Identified

The architecture research identified **12 significant patterns** in Gem Team that the prior comparison documents either missed entirely or analyzed superficially:

### Patterns Not in Prior Comparison Documents

1. **`<final_anchor>` Anti-Drift Pattern** — Every Gem agent ends with a tag repeating core behavioral constraints and "stay as [role]". Exploits LLM recency bias to prevent mode-switching. _Forge has no equivalent._

2. **`detailed thinking on` Directive** — Present in ALL 8 Gem agents as the first instruction after the `<agent>` tag. Activates extended chain-of-thought reasoning. _Forge has no thinking mode directive._

3. **Shared Operating Rules ("Base Class")** — Identical verbatim rules in ALL 8 agents: communication style, file reading limits (200 lines), tool preferences (batch edits), error categories (transient→handle, persistent→escalate). _Forge has no shared rules across agents._

4. **Evidence Storage Architecture** — Chrome-tester defines `docs/plan/{plan_id}/evidence/{task_id}/` with subfolders for screenshots, logs, network. _Not mentioned in prior docs._

5. **Memory System with Citation Verification** — References `/memories/memory-system-patterns.md` with read/create/update operations requiring `file:line` citations. _Prior docs mentioned memory but not citations._

6. **`mcp_sequential-th_sequentialthinking` for Complex Reasoning** — Planner uses a separate tool for multi-step reasoning (3+ steps). _Not in prior docs._

7. **`list_code_usages` Before Refactoring** — Implementer must check all call sites before modifying existing code. _Not in prior docs._

8. **Agent-Specific Plan Fields** — Plan tasks include fields tailored to specific agents: `tech_stack` for implementer, `review_depth` for reviewer, `validation_matrix` for chrome-tester. _Not analyzed in prior docs._

9. **Idempotency Requirement** — DevOps requires all operations to be idempotent with atomic operations. _Not in prior docs._

10. **Quality Bar Heuristic** — Reviewer uses "Would a staff engineer approve this?" as a calibration standard. _Not in prior docs._

11. **Three-State Completion (needs_revision)** — Allows orchestrator to differentiate terminal failures from recoverable issues, routing back to the same agent. _Mentioned in passing but not analyzed as architecture._

12. **Plan Lifecycle Modes** — Planner detects `initial | replan | extension` mode to handle different planning contexts. _Not analyzed as a pattern._

---

## Improvement Inventory

### P0 — Critical (Must Implement)

#### P0-1: Pre-Mortem Analysis in Planner

- **Description:** After creating task index and execution waves, planner identifies failure scenarios per task with likelihood/impact/mitigation, overall risk level, and key assumptions. Output as new section in `plan.md`.
- **Agents affected:** `planner.agent.md`
- **Dependencies:** None — independently adoptable
- **Complexity:** Low — adds new section with format specification; no other agent parses it
- **Source:** gem-planner pre_mortem + per-task failure_modes

#### P0-2: TDD Enforcement in Implementer

- **Description:** Replace implementer's workflow with: write failing tests → confirm fail → write minimal code → run `get_errors` after every edit → confirm pass → optional refactor. Remove "Do NOT build" and "Do NOT run tests" rules. Update frontmatter description.
- **Agents affected:** `implementer.agent.md` (primary), `verifier.agent.md` (role redefinition), `orchestrator.agent.md` (global rule update)
- **Dependencies:** **TIGHTLY COUPLED** — all 3 files must change simultaneously. If implementer gets TDD but verifier/orchestrator are not updated, the verifier's "sole owner of testing" identity and the orchestrator's "implementers never run tests" rule become false.
- **Complexity:** High — largest cross-agent change; requires coordinated edits to 3 files
- **Breaking change:** Orchestrator global rule "Implementers never build or run tests" must be rewritten to "Implementers perform unit-level TDD. The verifier performs integration-level verification."
- **Source:** gem-implementer TDD cycle

#### P0-3: Critical-Thinker Completion Contract (Bug Fix)

- **Description:** Add `## Completion Contract` section to critical-thinker. The orchestrator expects `DONE:`/`ERROR:` from it (Step 3b), but the agent file never defines this contract. This is a **pre-existing bug**, not a new improvement.
- **Agents affected:** `critical-thinker.agent.md`
- **Dependencies:** None
- **Complexity:** Trivial — add 3 lines

#### P0-4: Critical-Thinker Output Specification (Bug Fix)

- **Description:** Add explicit output file specification. The orchestrator references `design_critical_review.md` but the critical-thinker doesn't specify its output file, creating ambiguity.
- **Agents affected:** `critical-thinker.agent.md`
- **Dependencies:** None
- **Complexity:** Trivial — add output section

---

### P1 — High (Should Implement)

#### P1-1: Task Size Limits in Planner

- **Description:** Hard limits: max 3 files touched, max 2 dependencies, max 500 lines changed, max medium effort. Tasks exceeding any limit must be broken into subtasks.
- **Agents affected:** `planner.agent.md`
- **Dependencies:** None — independently adoptable
- **Complexity:** Low — adds new section with limit table
- **Source:** gem-planner `<task_size_limits>`

#### P1-2: Concurrency Cap in Orchestrator

- **Description:** Maximum 4 concurrent subagent invocations. Waves with >4 tasks are split into sub-waves of ≤4, dispatched sequentially.
- **Agents affected:** `orchestrator.agent.md`, `feature-workflow.prompt.md` (reinforcement)
- **Dependencies:** None — independently adoptable
- **Complexity:** Low — adds rule + sub-wave splitting logic to Step 5.2
- **Source:** gem-orchestrator max 4 concurrent

#### P1-3: Security-Focused Review

- **Description:** Add security review section to reviewer: secrets/PII scanning for all reviews; OWASP Top 10 for security-sensitive changes (auth, data, payments, admin). Add tiered review depth (Full/Standard/Lightweight) determined by examining the diff content. Add explicit read-only enforcement.
- **Agents affected:** `reviewer.agent.md`
- **Dependencies:** None if reviewer self-determines depth from diff content (Approach A — recommended). Would be coupled to planner if using task metadata (Approach B — not recommended).
- **Complexity:** Medium — adds substantive new section
- **Source:** gem-reviewer `<review_criteria>`, OWASP scanning, grep_search for secrets/PII

#### P1-4: Hybrid Retrieval Strategy in Researcher

- **Description:** Prescribe retrieval methodology: (1) `semantic_search` for conceptual discovery, (2) `grep_search` for exact patterns, (3) merge + deduplicate, (4) `read_file` for detailed examination, (5) `file_search` for existence verification.
- **Agents affected:** `researcher.agent.md`
- **Dependencies:** None — independently adoptable
- **Complexity:** Low — adds new subsection to Focused Research Rules
- **Source:** gem-researcher hybrid retrieval

#### P1-5: Implementer Security Rules

- **Description:** Add: never hardcode secrets/API keys/tokens/passwords; never expose PII in logs/errors; fix security vulnerabilities before returning.
- **Agents affected:** `implementer.agent.md`
- **Dependencies:** Logically pairs with P0-2 (TDD) but independently adoptable
- **Complexity:** Low — adds rules
- **Source:** gem-implementer security rules

#### P1-6: Implementer Code Quality Principles

- **Description:** Add YAGNI/KISS/DRY guidance + `list_code_usages` before refactoring.
- **Agents affected:** `implementer.agent.md`
- **Dependencies:** None
- **Complexity:** Low — adds rules
- **Source:** gem-implementer expertise + `list_code_usages` pattern

#### P1-7: Verifier Technology-Agnostic Commands

- **Description:** Replace hardcoded `dotnet build TourneyPal.sln` and `dotnet test tests/TourneyPal.Tests` with technology-agnostic instructions: detect build system from codebase (package.json, Makefile, .sln, pom.xml, etc.), run appropriate commands.
- **Agents affected:** `verifier.agent.md`
- **Dependencies:** None (but should coordinate with P0-2 verifier role shift)
- **Complexity:** Medium — rewrites build/test workflow sections
- **Source:** Pre-existing portability issue identified during research

#### P1-8: Planner Mode Awareness

- **Description:** Add explicit mode detection at top of workflow: initial (no plan.md), replan (verifier.md with failures), or extension (existing plan with new objectives). Currently handles replan implicitly but doesn't detect modes.
- **Agents affected:** `planner.agent.md`
- **Dependencies:** None
- **Complexity:** Low
- **Source:** gem-planner mode detection (initial/replan/extension)

---

### P2 — Medium (Implement When Needed)

#### P2-1: Per-Task Agent Routing

- **Description:** Add optional `agent` field to task files (default: `implementer`). Orchestrator reads field and dispatches to specified agent.
- **Agents affected:** `planner.agent.md`, `orchestrator.agent.md`
- **Dependencies:** **TIGHTLY COUPLED** — planner + orchestrator must change together. If only planner changes, field is ignored. If only orchestrator changes, it must default to `implementer`.
- **Complexity:** Low-Medium
- **Source:** gem-planner/gem-orchestrator per-task agent assignment

#### P2-2: Optional Human Approval Gates

- **Description:** Add `{{APPROVAL_MODE}}` to prompt. When `true`: pause after research synthesis and after planning. When `false` or unset: fully autonomous (default).
- **Agents affected:** `orchestrator.agent.md`, `feature-workflow.prompt.md`
- **Dependencies:** **TIGHTLY COUPLED** — orchestrator + prompt must change together
- **Complexity:** Low-Medium
- **Source:** gem-orchestrator 3 mandatory pauses (adapted to optional)

#### P2-3: Cross-Agent Decision Log

- **Description:** Add `decisions.md` convention: reviewer appends significant architectural decisions; researcher reads it during architecture research.
- **Agents affected:** `reviewer.agent.md` (write), `researcher.agent.md` (read), `orchestrator.agent.md` (add to artifact list)
- **Dependencies:** Must define convention first, then update agents. Each agent update is independent.
- **Complexity:** Low
- **Source:** gem memory system (simplified to file-based)

#### P2-4: Designer Security Considerations

- **Description:** Add "Security Considerations" subsection to design.md: auth/authz patterns, data protection, threat model, input validation strategy.
- **Agents affected:** `designer.agent.md`
- **Dependencies:** None
- **Complexity:** Low
- **Source:** Cross-cutting pattern from gem-implementer and gem-reviewer security emphasis

#### P2-5: Designer Failure & Recovery Section

- **Description:** Add "Failure & Recovery" subsection to design.md: expected failure modes, retry/fallback strategies, graceful degradation.
- **Agents affected:** `designer.agent.md`
- **Dependencies:** Complements P0-1 (planner pre-mortem) — addresses failure at design level vs. task level
- **Complexity:** Low
- **Source:** Cross-cutting pattern

#### P2-6: Critical-Thinker Structured Risk Categories

- **Description:** Add specific risk categories to probe: security, scalability, maintainability, backwards compatibility, edge cases, performance. Add instruction to ground risks in specific technical details from design.md.
- **Agents affected:** `critical-thinker.agent.md`
- **Dependencies:** None
- **Complexity:** Low
- **Source:** Cross-cutting pattern from gem agents' structured risk handling

#### P2-7: Researcher Confidence Metadata

- **Description:** Add optional metadata fields to research output: `confidence_level` (high/medium/low), `coverage_estimate`, `gaps` with impact assessment.
- **Agents affected:** `researcher.agent.md`
- **Dependencies:** None
- **Complexity:** Low
- **Source:** gem-researcher `research_metadata`

#### P2-8: Planner Implementation Specification

- **Description:** Add optional "Implementation Specification" section to plan.md: code_structure, affected_areas, component_details, integration_points. Gives implementers clearer architectural guidance.
- **Agents affected:** `planner.agent.md`
- **Dependencies:** May overlap with `design.md` — should reference design.md rather than duplicate
- **Complexity:** Low
- **Source:** gem-planner implementation_specification

#### P2-9: Documentation Writer Agent (New)

- **Description:** New optional agent for API docs (OpenAPI/Swagger), architectural diagrams (Mermaid), code-documentation parity. Not in core pipeline — invoked only when planner assigns documentation tasks via `agent` field.
- **Agents affected:** New file: `documentation-writer.agent.md`
- **Dependencies:** Requires P2-1 (per-task agent routing) to be routed by the orchestrator
- **Complexity:** Medium — full new agent definition
- **Source:** gem-documentation-writer

---

### P3 — Low (Nice-to-Have)

#### P3-1: Self-Reflection Steps

- **Description:** Before returning, agents verify: output addresses the task, no obvious omissions, "would a senior engineer approve?" Fix issues before returning. Apply to implementer and reviewer; optionally extend to others.
- **Agents affected:** `implementer.agent.md`, `reviewer.agent.md`
- **Dependencies:** P0-2 (TDD) should be implemented first for implementer so self-reflection can reference the TDD cycle
- **Complexity:** Low
- **Source:** gem-implementer reflect step

#### P3-2: Context-Efficient File Reading Guidance

- **Description:** Add to agents that read the codebase: "Prefer semantic search, file outlines, and targeted line-range reads; limit to ~200 lines per `read_file` call."
- **Agents affected:** `researcher.agent.md`, `planner.agent.md`, `critical-thinker.agent.md`, `verifier.agent.md`, `reviewer.agent.md`
- **Dependencies:** None
- **Complexity:** Trivial — adds guidance line to 5 agents
- **Source:** Shared Gem operating rule

#### P3-3: Quality Bar Heuristic for Reviewer

- **Description:** Add "Apply the standard: Would a staff engineer approve this code?" as reviewer quality calibration.
- **Agents affected:** `reviewer.agent.md`
- **Dependencies:** None
- **Complexity:** Trivial
- **Source:** gem-reviewer quality bar

#### P3-4: Spec Edge Case Structure

- **Description:** Each edge case should have clear input/condition, expected behavior, and severity if missed. Add verification step: ensure all acceptance criteria are testable.
- **Agents affected:** `spec.agent.md`
- **Dependencies:** None
- **Complexity:** Low
- **Source:** Cross-cutting pattern

#### P3-5: Designer Self-Verification

- **Description:** Before returning, verify design addresses all requirements from feature.md and every acceptance criterion has a clear implementation path.
- **Agents affected:** `designer.agent.md`
- **Dependencies:** None
- **Complexity:** Trivial
- **Source:** Inspired by gem-implementer reflection pattern

#### P3-6: Anti-Drift `<final_anchor>` Pattern

- **Description:** Add a final instruction block at the end of each agent repeating core behavioral constraints and "stay as [role]" to prevent mode-switching in long conversations. Exploits LLM recency bias.
- **Agents affected:** All 9 agents
- **Dependencies:** None — can be applied uniformly
- **Complexity:** Low — adds ~3 lines per agent
- **Source:** gem `<final_anchor>` tag (present in ALL 8 Gem agents)

#### P3-7: `detailed thinking on` Directive

- **Description:** Add chain-of-thought activation directive at the start of each agent's instructions. Effect depends on underlying model.
- **Agents affected:** All 9 agents
- **Dependencies:** None
- **Complexity:** Trivial — adds 1 line per agent
- **Source:** ALL 8 Gem agents include this

#### P3-8: Planner YAGNI/KISS/DRY Guidance

- **Description:** Add "Prefer simpler decompositions. Avoid unnecessary tasks. Each task should deliver tangible, testable value."
- **Agents affected:** `planner.agent.md`
- **Dependencies:** None
- **Complexity:** Trivial
- **Source:** gem-planner principles

---

## Per-Agent Change Summary

### 1. orchestrator.agent.md

| Change                                                                                                  | Priority | Cluster         |
| ------------------------------------------------------------------------------------------------------- | -------- | --------------- |
| Update global rule: "Implementers perform unit-level TDD; verifier does integration-level verification" | P0       | Cluster A (TDD) |
| Add concurrency cap: max 4 concurrent subagent invocations                                              | P1       | Independent     |
| Add sub-wave splitting logic to Step 5.2                                                                | P1       | Independent     |
| Add per-task agent routing to Step 5.2                                                                  | P2       | Cluster B       |
| Add `decisions.md` to artifact list (optional)                                                          | P2       | Independent     |
| Add conditional approval pause logic (if APPROVAL_MODE)                                                 | P2       | Cluster C       |
| Add anti-drift anchor                                                                                   | P3       | Independent     |

### 2. researcher.agent.md

| Change                                    | Priority | Cluster     |
| ----------------------------------------- | -------- | ----------- |
| Add hybrid retrieval strategy subsection  | P1       | Independent |
| Add confidence metadata fields (optional) | P2       | Independent |
| Add `decisions.md` read convention        | P2       | Independent |
| Add context-efficient reading guidance    | P3       | Independent |
| Add anti-drift anchor                     | P3       | Independent |

### 3. spec.agent.md

| Change                                   | Priority | Cluster     |
| ---------------------------------------- | -------- | ----------- |
| Add structured edge case format guidance | P3       | Independent |
| Add self-verification step               | P3       | Independent |
| Add anti-drift anchor                    | P3       | Independent |

### 4. designer.agent.md

| Change                                              | Priority | Cluster     |
| --------------------------------------------------- | -------- | ----------- |
| Add "Security Considerations" to design.md contents | P2       | Independent |
| Add "Failure & Recovery" to design.md contents      | P2       | Independent |
| Add self-verification step                          | P3       | Independent |
| Add anti-drift anchor                               | P3       | Independent |

### 5. critical-thinker.agent.md

| Change                                                       | Priority | Cluster     |
| ------------------------------------------------------------ | -------- | ----------- |
| **Add completion contract (DONE:/ERROR:)** (bug fix)         | P0       | Independent |
| **Add output file specification** (bug fix)                  | P0       | Independent |
| Add structured risk categories                               | P2       | Independent |
| Add "ground risks in specific technical details" instruction | P2       | Independent |
| Add context-efficient reading guidance                       | P3       | Independent |
| Add anti-drift anchor                                        | P3       | Independent |

### 6. planner.agent.md

| Change                                              | Priority | Cluster     |
| --------------------------------------------------- | -------- | ----------- |
| Add pre-mortem analysis section + plan.md format    | P0       | Independent |
| Add task size limits section                        | P1       | Independent |
| Add mode awareness (initial/replan/extension)       | P1       | Independent |
| Add `agent` field to task file requirements         | P2       | Cluster B   |
| Add implementation specification section (optional) | P2       | Independent |
| Add YAGNI/KISS/DRY guidance                         | P3       | Independent |
| Add context-efficient reading guidance              | P3       | Independent |
| Add anti-drift anchor                               | P3       | Independent |

### 7. implementer.agent.md

**Most significantly changed agent.**

| Change                                                                                                           | Priority | Cluster         |
| ---------------------------------------------------------------------------------------------------------------- | -------- | --------------- |
| Replace workflow with TDD cycle (write tests → confirm fail → write code → get_errors → confirm pass → refactor) | P0       | Cluster A (TDD) |
| Remove "Do NOT build" and "Do NOT run tests" rules                                                               | P0       | Cluster A (TDD) |
| Update frontmatter description                                                                                   | P0       | Cluster A (TDD) |
| Add `get_errors` after every edit rule                                                                           | P0       | Cluster A (TDD) |
| Add security rules (no secrets/PII, fix vulnerabilities)                                                         | P1       | Independent     |
| Add YAGNI/KISS/DRY + `list_code_usages` before refactoring                                                       | P1       | Independent     |
| Add self-reflection step                                                                                         | P3       | Independent     |
| Add anti-drift anchor                                                                                            | P3       | Independent     |

### 8. verifier.agent.md

| Change                                                                                       | Priority | Cluster         |
| -------------------------------------------------------------------------------------------- | -------- | --------------- |
| Redefine role from "sole owner of build/test" to "integration-level verifier"                | P0       | Cluster A (TDD) |
| Add note: "Implementers have already verified unit tests; focus on cross-task interactions"  | P0       | Cluster A (TDD) |
| Replace hardcoded `dotnet build TourneyPal.sln` with technology-agnostic instructions        | P1       | Independent     |
| Replace hardcoded `dotnet test tests/TourneyPal.Tests` with technology-agnostic instructions | P1       | Independent     |
| Add context-efficient reading guidance                                                       | P3       | Independent     |
| Add anti-drift anchor                                                                        | P3       | Independent     |

### 9. reviewer.agent.md

| Change                                                                           | Priority | Cluster     |
| -------------------------------------------------------------------------------- | -------- | ----------- |
| Add security review section (secrets/PII + OWASP for security-sensitive changes) | P1       | Independent |
| Add tiered review depth (Full/Standard/Lightweight)                              | P2       | Independent |
| Add explicit read-only enforcement                                               | P1       | Independent |
| Add `decisions.md` write convention                                              | P2       | Independent |
| Add quality bar heuristic                                                        | P3       | Independent |
| Add self-reflection step                                                         | P3       | Independent |
| Add context-efficient reading guidance                                           | P3       | Independent |
| Add anti-drift anchor                                                            | P3       | Independent |

### 10. feature-workflow.prompt.md

| Change                                            | Priority | Cluster     |
| ------------------------------------------------- | -------- | ----------- |
| Add concurrency cap rule (reinforce orchestrator) | P1       | Independent |
| Add `{{APPROVAL_MODE}}` optional approval gates   | P2       | Cluster C   |
| Add per-task agent routing rule                   | P2       | Cluster B   |

### 11. documentation-writer.agent.md (NEW — Optional)

| Change                                                                                           | Priority |
| ------------------------------------------------------------------------------------------------ | -------- |
| Create new agent: API docs, diagrams, parity enforcement, coverage matrix, read-only source code | P2       |
| Requires P2-1 (per-task agent routing)                                                           | —        |

---

## Cross-Cutting Concerns

These are patterns that should be applied uniformly across all agents:

### 1. Shared Communication Style (Not Prioritized for Adoption)

Gem applies identical communication rules to ALL agents: "Output ONLY the requested deliverable. Zero explanation, zero preamble, zero commentary." Forge agents have no communication style directives. This could reduce token waste but may conflict with scenarios where explanation is valuable. **Consider as optional P3 enhancement** — evaluate whether it helps or hinders Forge's artifact quality.

### 2. Context-Efficient File Reading (P3)

"Prefer semantic search, file outlines, and targeted line-range reads; limit to ~200 lines per `read_file` call." Applies to 5 agents that read the codebase: researcher, planner, critical-thinker, verifier, reviewer.

### 3. Security as a Thread (P1)

Security is distributed across 3 agents (not just the reviewer):

- **Designer** (P2): Security considerations in design.md
- **Implementer** (P1): No secrets/PII, fix vulnerabilities
- **Reviewer** (P1): OWASP scanning, secrets detection

### 4. Anti-Drift Anchoring (P3)

Add `<final_anchor>` at end of every agent repeating core constraints and "stay as [role]" to prevent mode-switching. Applies to all 9 agents.

### 5. `detailed thinking on` Directive (P3)

Add chain-of-thought activation as first instruction in every agent. Applies to all 9 agents.

### 6. Error Handling Categories (Not Prioritized)

Gem categorizes errors: transient→handle, persistent→escalate, security→immediate action, missing context→blocked. Forge has no error handling guidance beyond DONE/ERROR. **Not recommended for current scope** — would require significant changes to the completion contract model. Consider for future evolution.

### 7. Completion Contract Consistency (P0 — Bug Fix)

Critical-thinker is the ONLY agent without a `DONE:`/`ERROR:` completion contract. Must be fixed.

---

## Change Dependencies & Ordering

### Dependency Graph

```
Independent (any order, any time):
  ├── P0-1  Pre-mortem analysis (planner only)
  ├── P0-3  Critical-thinker completion contract (bug fix)
  ├── P0-4  Critical-thinker output specification (bug fix)
  ├── P1-1  Task size limits (planner only)
  ├── P1-2  Concurrency cap (orchestrator only)
  ├── P1-3  Security-focused review (reviewer only, Approach A)
  ├── P1-4  Hybrid retrieval (researcher only)
  ├── P1-5  Implementer security rules
  ├── P1-6  Implementer code quality principles
  ├── P1-7  Verifier tech-agnostic commands
  ├── P1-8  Planner mode awareness
  └── All P2 independent items + all P3 items

Cluster A — MUST coordinate (3 files):
  P0-2 TDD enforcement
    ├── implementer.agent.md  (primary: new TDD workflow)
    ├── verifier.agent.md     (role redefinition to integration-level)
    └── orchestrator.agent.md (global rule: implementers do TDD)

Cluster B — MUST coordinate (2 files):
  P2-1 Per-task agent routing
    ├── planner.agent.md      (add agent field to task format)
    └── orchestrator.agent.md (read agent field, dispatch accordingly)

Cluster C — MUST coordinate (2 files):
  P2-2 Human approval gates
    ├── orchestrator.agent.md       (conditional pause logic)
    └── feature-workflow.prompt.md  (APPROVAL_MODE variable)

Sequential:
  P3-1 Self-reflection (implementer) → should follow P0-2 (TDD)
  P2-9 Documentation writer → requires P2-1 (per-task routing)
  P1-1 Task size limits → ideally before P2-1 (smaller tasks benefit routing)
```

### Recommended Implementation Order

**Phase 1 — P0 (Critical)**

1. P0-3 + P0-4: Fix critical-thinker bugs (completion contract + output spec)
2. P0-1: Pre-mortem analysis in planner
3. P0-2: TDD enforcement — update **implementer + verifier + orchestrator together**

**Phase 2 — P1 (High)** 4. P1-4: Hybrid retrieval in researcher 5. P1-1: Task size limits in planner 6. P1-8: Planner mode awareness 7. P1-2: Concurrency cap in orchestrator 8. P1-5 + P1-6: Implementer security rules + code quality principles 9. P1-3: Security-focused review in reviewer 10. P1-7: Verifier technology-agnostic commands

**Phase 3 — P2 (Medium)** 11. P2-4 + P2-5: Designer security + failure/recovery sections 12. P2-6 + P2-7: Critical-thinker risk categories + researcher confidence metadata 13. P2-1: Per-task agent routing (planner + orchestrator together) 14. P2-2: Optional approval gates (orchestrator + prompt together) 15. P2-3: Cross-agent decision log 16. P2-8: Planner implementation specification 17. P2-9: Documentation writer agent (after P2-1)

**Phase 4 — P3 (Polish)** 18. P3-1: Self-reflection (implementer + reviewer) 19. P3-2: Context-efficient file reading (5 agents) 20. P3-6 + P3-7: Anti-drift anchors + thinking directive (all agents) 21. P3-3 + P3-4 + P3-5 + P3-8: Minor enhancements

### Breaking Change Analysis

| Change                 | What Breaks                                                           | Severity              | Mitigation                                                                       |
| ---------------------- | --------------------------------------------------------------------- | --------------------- | -------------------------------------------------------------------------------- |
| P0-2 TDD               | Orchestrator global rule "Implementers never run tests" becomes false | **HIGH**              | Update all 3 files simultaneously                                                |
| P0-2 TDD               | Verifier "sole owner" identity contradicted                           | **MEDIUM**            | Redefine verifier role                                                           |
| P2-1 Per-task routing  | Orchestrator Step 5.2 assumes all tasks → implementer                 | **HIGH** (if partial) | Update planner + orchestrator together; default to `implementer` if field absent |
| Contract format change | Orchestrator parses `DONE:`/`ERROR:` text                             | **CRITICAL**          | Do NOT change contract format — keep DONE:/ERROR:                                |

All other improvements are **backward compatible** — they add new sections, rules, or guidance without changing existing contracts or formats.

---

## What NOT to Adopt

| Feature                                             | Why Not                                                                                                                                                                                     |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **YAML state file (plan.yaml)**                     | Markdown artifacts are core to Forge — more readable, diffable, consistent. YAML adds schema complexity without benefit for a fixed pipeline.                                               |
| **Dynamic DAG dispatch**                            | Forge's deterministic pipeline is a core strength — predictable, debuggable. Dynamic dispatch is a fundamental architecture change with marginal benefit.                                   |
| **Mandatory human pauses**                          | Forge's autonomous operation is deliberate. Optional gates (APPROVAL_MODE) sufficient.                                                                                                      |
| **Memory system (MCP-based)**                       | Requires MCP memory tools not universally available. File-based `decisions.md` is a simpler alternative.                                                                                    |
| **`disable-model-invocation: true`**                | Forge achieves same result via prose instructions. Platform-specific mechanism.                                                                                                             |
| **Tool activation calls**                           | Gem-specific environment pattern (`activate_web_interaction`, etc.). Not applicable.                                                                                                        |
| **JSON completion contracts**                       | Forge's `DONE:`/`ERROR:` is simpler, human-readable, consistent. Switching requires atomic change across all 10 files — high risk, low marginal benefit.                                    |
| **Chrome Tester as core agent**                     | Requires Chrome DevTools MCP. Too specialized for core pipeline. Optional P3 at best.                                                                                                       |
| **DevOps agent**                                    | Too infrastructure-specific (Docker, K8s, CI/CD). Adds complexity + approval gates that conflict with Forge's autonomous model.                                                             |
| **YAML research output format**                     | Markdown is more readable, consistent with Forge's artifact chain. YAML schema is verbose.                                                                                                  |
| **`reference_cache`**                               | Gem-specific tool integration for WCAG standards. Not applicable to Forge.                                                                                                                  |
| **`manage_todos`**                                  | External task tracking integration. Forge's plan.md + task files serve this purpose.                                                                                                        |
| **Three-state completion contract**                 | Would require rewriting all 9 agents + orchestrator parsing simultaneously. `needs_revision` is useful but the migration cost is too high for current scope. Consider for future evolution. |
| **XML-like semantic tags (`<role>`, `<workflow>`)** | May improve LLM prompt parsing but is an unvalidated hypothesis. Forge's markdown headings are more human-readable. Test separately if desired.                                             |

---

## Strengths to Evaluate

These Forge features are current strengths that should be **evaluated rather than assumed sacrosanct** — keep what works, improve or replace what doesn't:

1. **Fixed deterministic 8-step pipeline** — predictable workflow, easy to reason about and debug
2. **Dedicated specification stage** — formal `feature.md` requirements document before design (Gem lacks this)
3. **Dedicated design stage** — deep technical design with tradeoffs in `design.md` (Gem lacks this)
4. **Critical-thinker gate** — adversarial design review before planning (unique to Forge, no Gem equivalent)
5. **Dedicated verifier agent** — single authoritative integration verification point with comprehensive report (Gem has no dedicated verifier)
6. **Research synthesis step** — dedicated merging of parallel research partials prevents duplication (Gem lacks this)
7. **Strict agent isolation** — clear I/O boundaries per agent; implementer firewalled from plan.md
8. **Full autonomy by default** — runs end-to-end without human intervention
9. **Rich markdown artifact trail** — 10+ human-readable documents per feature
10. **`DONE:`/`ERROR:` completion contracts** — consistent, simple, text-based agent communication
11. **Markdown throughout** — all artifacts are markdown, not YAML/JSON
12. **Execution wave model** — groups of parallel tasks with dependency ordering
13. **Individual task files** — separate `tasks/*.md` rather than single monolithic plan file
14. **Orchestrator as sole router** — no agent-to-agent direct communication
15. **Verifier→Planner and Reviewer→Planner feedback loops** — structured error recovery with bounded retries

---

## Known Issues Found

### Pre-Existing Bugs

1. **Critical-thinker missing completion contract** — The orchestrator expects `DONE:`/`ERROR:` from the critical-thinker (Step 3b), but the agent file has no `## Completion Contract` section. This means the agent may produce output in an unexpected format, potentially causing the orchestrator to misinterpret results or hang.

2. **Critical-thinker missing output specification** — The orchestrator references `design_critical_review.md` as the critical-thinker's output, but the critical-thinker agent file never specifies this output file name or format. The agent has no `## Outputs` section.

### Pre-Existing Limitations

3. **Verifier hardcoded to .NET** — `dotnet build TourneyPal.sln` and `dotnet test tests/TourneyPal.Tests` are hardcoded in the verifier, making it non-portable to other technology stacks.

4. **Inconsistent DONE: formats** — Each agent has a different detail format for the DONE: line (researcher returns focus-area and summary, planner returns task/wave counts, verifier returns test counts, etc.). While the orchestrator handles these, the inconsistency could cause parsing issues in edge cases.

5. **No verifier read-only enforcement** — The verifier does not explicitly say "do not modify code." It should be read-only (verification only), but this is not stated.

---

## Open Questions

### Architecture & Conventions

1. **Should Forge agents include `<final_anchor>` anti-drift blocks?** The architecture research identified this as a significant Gem pattern, but its actual impact on LLM behavior is unvalidated. Consider testing on a single agent first.

2. **Should Forge adopt shared operating rules across all agents?** Gem duplicates identical rules in every agent. Forge could either (a) duplicate rules in each agent file, or (b) use a shared include/reference. Option (a) follows Gem's pattern exactly; option (b) depends on whether the `chatagent` format supports includes.

3. **Does the `chatagent` format support `disable-model-invocation` and `user-invocable` frontmatter fields?** If so, the orchestrator should adopt `disable-model-invocation: true`.

4. **Does `detailed thinking on` provide measurable improvement?** This is model-dependent and unvalidated.

### TDD Implementation

5. **TDD fallback for non-testable projects** — What should the implementer do if the project has no test framework or the task doesn't warrant tests? Should there be a `skip_tdd` flag in task files?

6. **Verifier scope after TDD** — Should the verifier still run the full test suite (including unit tests already passed by implementers), or target integration/e2e tests only? Running all tests is safer but potentially redundant.

### Agent Interactions

7. **Verifier technology detection** — Should the verifier use heuristics (scan for package.json, .sln, Makefile) or should the planner/designer specify build/test commands in plan.md or design.md?

8. **Critical-thinker output file name** — The orchestrator references `design_critical_review.md` but has this been validated? Is the file name correct?

9. **Decision log lifecycle** — If `decisions.md` is adopted, who writes to it (reviewer?), who reads it (researcher?), and how is it maintained across multiple features?

### Implementation Details

10. **APPROVAL_MODE implementation** — How should `{{APPROVAL_MODE}}` be passed in VS Code Copilot's prompt system? Does the template support conditional logic?

11. **Implementation specification overlap** — Gem planner's `implementation_specification` (code_structure, affected_areas, component_details) may overlap with Forge's `design.md`. Should it be a simplified reference rather than duplication?

12. **Documentation-writer trigger timing** — If adopted, should it run post-verification or as part of implementation waves via the `agent` field?

13. **Task size limits calibration** — Gem uses max 3 files / 500 lines / 2 dependencies. Are these appropriate for Forge's typical projects?

14. **Concurrency cap value** — Gem uses 4. Is this appropriate for the target environment, or should it be configurable?

15. **Self-reflection scope** — Should self-reflection be added to ALL agents or only implementer and reviewer as currently proposed?

16. **Gem communication style adoption** — "Output ONLY the requested deliverable. Zero explanation." Should this be adopted? It reduces tokens but may reduce artifact quality where explanation is valuable.

17. **`list_code_usages` availability** — How widely available is this tool, and should it be a MUST or SHOULD for the implementer?

---

## Appendix / Sources

### Partial Research Files Synthesized

| File                                                 | Focus Area   | Key Contribution                                                                                                                                                               |
| ---------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [research/architecture.md](research/architecture.md) | Architecture | 12 Gem patterns not in prior comparisons; structural analysis of agent definitions, isolation mechanisms, state management, error handling, shared rules, completion contracts |
| [research/impact.md](research/impact.md)             | Impact       | Per-agent change specifications with priorities; cross-cutting improvements; what NOT to adopt; Forge strengths to preserve; new agent assessment                              |
| [research/dependencies.md](research/dependencies.md) | Dependencies | Complete data flow diagram; 3 tightly-coupled change clusters identified; implementation ordering constraints; breaking change analysis; contract dependency mapping           |

### Context Documents Referenced

| File                                                                          | Role                                                                                                    |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| [docs/comparison-forge-vs-gem-team.md](../../comparison-forge-vs-gem-team.md) | Prior agent-by-agent comparison (confirmed largely accurate; several gaps identified by researchers)    |
| [docs/optimization-from-gem-team.md](../../optimization-from-gem-team.md)     | Prior implementation guidance for 10 improvements (confirmed valid; additional improvements discovered) |

### Conflicts Between Partial Analyses

No material contradictions were found between the three research streams. The analyses were complementary:

- Architecture provided structural patterns and identified what Forge lacks
- Impact translated those patterns into specific per-agent changes with priorities
- Dependencies mapped which changes couple together and defined safe implementation ordering

Minor alignment notes:

- Architecture identified the three-state completion contract (`needs_revision`) as a significant pattern worth adopting. Impact and Dependencies both concluded it should NOT be adopted due to migration risk. **Resolution: Not adopted in current scope; noted for future evolution.**
- Architecture raised the question of XML-like semantic tags. Impact did not propose adopting them. **Resolution: Not adopted; filed as open question for separate testing.**
- Impact recommended P2 for per-task agent routing; Dependencies confirmed this prioritization but emphasized the tight coupling between planner and orchestrator. **Resolution: P2 with explicit cluster coordination requirement.**
