# Design: Comprehensive Forge Agent Improvements

**Summary:** This document specifies the exact structure, content, and interaction contracts for all 11 deliverable files (9 rewritten agents + 1 rewritten workflow prompt + 1 new documentation-writer agent). Every agent file is rebuilt from scratch following a canonical structure standard, incorporating 30+ improvements from Gem Team analysis while preserving Forge's core architectural strengths.

**Design Goals:**

1. Establish a uniform agent file structure that every agent follows
2. Eliminate all pre-existing bugs (critical-thinker missing contract/output, verifier hardcoded commands)
3. Adopt high-value Gem patterns (TDD, pre-mortem, task size limits, security thread, hybrid retrieval, concurrency cap, anti-drift anchors)
4. Ensure zero contradictions between any two agents
5. Make every agent file self-contained and portable across any technology stack

---

## Context & Inputs

### Source Documents

| Document                                             | Role                                                                                        |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| [initial-request.md](initial-request.md)             | Original user request: improve all Forge agents with full rewrite permission                |
| [analysis.md](analysis.md)                           | Merged analysis from 3 research streams: 14 improvements, 2 bug fixes, 3 change clusters    |
| [feature.md](feature.md)                             | 12 FR groups (55+ individual FRs), 15 acceptance criteria, 10 edge cases, 15 test scenarios |
| [research/architecture.md](research/architecture.md) | 12 Gem patterns not in prior comparisons                                                    |
| [research/impact.md](research/impact.md)             | Per-agent change specs with priorities                                                      |
| [research/dependencies.md](research/dependencies.md) | Data flow, coupling clusters, implementation ordering                                       |

### Governing Constraint

> The user has granted **full permission to alter any Forge agent in any way that makes it better**. No sacred cows. The goal is the best possible agent package.

### Architecture Preserved

These Forge core properties are **retained** (validated as genuine strengths):

1. Fixed deterministic 8-step pipeline
2. Markdown artifact chain (all inter-agent communication via markdown files)
3. Orchestrator as sole router (agents never invoke other agents)
4. Three-state text completion contracts: `DONE:` / `ERROR:` / `NEEDS_REVISION:` (see rationale in Tradeoff T2)
5. Dedicated spec, design, and critical-thinker stages
6. Dedicated verifier agent
7. Research synthesis step
8. Full autonomy by default
9. Execution wave model with individual task files
10. Verifier→Planner and Reviewer→Planner feedback loops with bounded retries
11. Lightweight revision paths via `NEEDS_REVISION:` for reviewer→implementer and critical-thinker→designer loops

---

## Agent File Structure Standard

Every agent file MUST follow this canonical structure. Sections are in a fixed order. Optional sections are marked; all others are mandatory.

### Frontmatter

```yaml
---
name: <agent-name>
description: <one-line role description>
---
```

**Rules:**

- `name` must match the filename stem (e.g., `orchestrator` for `orchestrator.agent.md`)
- `description` must be a single sentence, technology-agnostic, and must not reference project-specific names

### Section Order (Canonical)

Every agent file uses the following sections in this exact order. Not all agents have all sections — the table below marks which are universal vs. agent-specific.

| #   | Section Heading                 | Universal?        | Notes                                                           |
| --- | ------------------------------- | ----------------- | --------------------------------------------------------------- |
| 1   | `# <Agent Name> Agent Workflow` | Yes               | Title — uses the pattern `# <Role> Agent Workflow`              |
| 2   | Role preamble paragraph         | Yes               | 2-4 sentences: who you are, what you do, what you never do      |
| 3   | `## Inputs`                     | Yes               | Exactly what files/data this agent receives                     |
| 4   | `## Outputs`                    | Yes               | Exactly what files this agent produces                          |
| 5   | `## Global Rules`               | Orchestrator only | Orchestrator-specific system-wide rules                         |
| 6   | `## Operating Rules`            | Yes               | Cross-cutting rules (shared block — see Cross-Cutting Patterns) |
| 7   | `## Workflow`                   | Yes               | Agent-specific step-by-step instructions                        |
| 8   | `## <Output-File> Contents`     | Most agents       | Defines the structure of the agent's output artifact            |
| 9   | `## Completion Contract`        | Yes               | Exact `DONE:`/`ERROR:`/`NEEDS_REVISION:` format                 |
| 10  | `## Anti-Drift Anchor`          | Yes               | Final reinforcement block (see Cross-Cutting Patterns)          |

**Rules:**

- Sections MUST appear in this order
- An agent MAY omit a section only if it is marked non-universal AND has no content for that agent
- No agent may add arbitrary top-level sections outside this list; sub-sections within a section are fine
- **Orchestrator extensions:** The orchestrator has unique sections (`## Documentation Structure`, `## Workflow Steps` with numbered sub-steps, `## Parallel Execution Summary`) that extend the canonical structure. These are formally valid extensions because the orchestrator's coordination role requires structured dispatch logic that doesn't map to a single `## Workflow` section. Other agents MAY add agent-specific sections between `## Workflow` and `## Completion Contract` (e.g., `## Risk Categories` for critical-thinker, `## TDD Fallback` for implementer) as long as they don't duplicate or contradict canonical sections

### Role Preamble Pattern

Every agent begins (after frontmatter and title) with a role preamble following this pattern:

```
You are the **<Role> Agent**.

<What you do — 1-2 sentences>.
<What you NEVER do — 1 sentence>.
```

Example for the researcher:

```
You are the **Research Agent**.

You investigate the existing codebase to produce factual findings about architecture, impact areas, and dependencies. You operate in focused research mode (one focus area) or synthesis mode (merging all partials).
You NEVER modify source code, tests, or project files. You NEVER make design decisions or propose solutions.
```

---

## Cross-Cutting Patterns

These are exact text blocks that MUST appear identically (or near-identically, adjusted only for role-specific tool lists) in every agent. They exist to prevent behavioral drift and ensure consistency.

### Pattern 1: Operating Rules Block

The following block MUST appear as a `## Operating Rules` section in **every** agent file. The tool list in rule 5 varies by agent — only list tools that agent actually uses.

```markdown
## Operating Rules

1. **Context-efficient reading:** Prefer `semantic_search` and `grep_search` for discovery. Use targeted line-range reads with `read_file` (limit ~200 lines per call). Avoid reading entire files unless necessary.
2. **Error handling:**
   - _Transient errors_ (network timeout, tool unavailable, rate limit): Retry up to 2 times with brief delay. **Do NOT retry if the failure is deterministic** (e.g., the tool itself is broken, the API returned a permanent error code, or the same input will always produce the same failure).
   - _Persistent errors_ (file not found, permission denied): Include in output and continue. Do not retry.
   - _Security issues_ (secrets in code, vulnerable dependencies): Flag immediately with `severity: critical`.
   - _Missing context_ (referenced file doesn't exist, dependency not installed): Note the gap and proceed with available information.
   - **Retry budget:** Agent-level retries (this section) are for individual tool calls within the agent. The orchestrator also retries entire agent invocations once (Global Rule 4). These compose: worst case is 3 internal attempts (1 + 2 retries) × 2 orchestrator attempts = 6 total tool calls. Agents MUST NOT retry deterministic failures, which bounds real-world retries to transient issues only.
3. **Output discipline:** Produce only the deliverables specified in the Outputs section. Do not add commentary, preamble, or explanation outside the output artifact.
4. **File boundaries:** Only write to files listed in the Outputs section. Never modify files outside your output scope.
5. **Tool preferences:** [Agent-specific — e.g., "Use `multi_replace_string_in_file` for batch edits. Use `get_errors` after every file modification."]
```

**Agent-specific tool preferences (rule 5):**

| Agent                | Rule 5 Content                                                                                                                                                                                                 |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Orchestrator         | "Use `runSubagent` for all delegation. Never invoke tools that modify code or files directly."                                                                                                                 |
| Researcher           | "Use `semantic_search` for conceptual discovery, `grep_search` for exact patterns, `file_search` for existence checks, `read_file` for targeted examination."                                                  |
| Spec                 | "Use `semantic_search` and `grep_search` for minimal additional research. Use `read_file` for targeted examination."                                                                                           |
| Designer             | "Use `semantic_search` and `grep_search` for targeted research. Use `read_file` for targeted examination."                                                                                                     |
| Critical Thinker     | "Use `semantic_search` and `grep_search` to verify design claims. Use `read_file` for targeted examination of referenced code."                                                                                |
| Planner              | "Use `semantic_search` and `grep_search` for targeted research. Use `read_file` for targeted examination."                                                                                                     |
| Implementer          | "Use `multi_replace_string_in_file` for batch edits. Use `get_errors` after **every** file modification. Use `list_code_usages` before refactoring existing code (fall back to `grep_search` if unavailable)." |
| Verifier             | "Use `run_in_terminal` for build and test commands. Use `grep_search` for targeted code verification. Never use tools that modify source code."                                                                |
| Reviewer             | "Use `grep_search` for secrets/pattern scanning. Use `read_file` for targeted code review. Never use tools that modify source code."                                                                           |
| Documentation Writer | "Use `semantic_search` and `grep_search` for code discovery. Use `read_file` for targeted examination. Never use tools that modify source code."                                                               |

### Pattern 2: Anti-Drift Anchor

Every agent file MUST end with this section as the **very last content** in the file. It exploits LLM recency bias to prevent mode-switching in long sessions.

```markdown
## Anti-Drift Anchor

**REMEMBER:** You are the **<Role>**. <Core constraint 1>. <Core constraint 2>. <Core constraint 3>. Stay as <role>.
```

Exact text per agent:

| Agent                | Anti-Drift Anchor Text                                                                                                                                                                                          |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Orchestrator         | `**REMEMBER:** You are the **Orchestrator**. You coordinate agents via runSubagent. You never write code, tests, or documentation directly. You never skip pipeline steps. Stay as orchestrator.`               |
| Researcher           | `**REMEMBER:** You are the **Researcher**. You investigate and document findings. You never modify source code, tests, or project files. You never make design decisions. Stay as researcher.`                  |
| Spec                 | `**REMEMBER:** You are the **Spec Agent**. You write formal requirements specifications. You never write code, designs, or plans. You never implement anything. Stay as spec.`                                  |
| Designer             | `**REMEMBER:** You are the **Designer**. You create technical design documents. You never write code, tests, or plans. You never implement anything. Stay as designer.`                                         |
| Critical Thinker     | `**REMEMBER:** You are the **Critical Thinker**. You perform adversarial design reviews. You never write code, designs, plans, or specifications. You identify risks and weaknesses. Stay as critical thinker.` |
| Planner              | `**REMEMBER:** You are the **Planner**. You decompose work into tasks and execution waves. You never write code, tests, or documentation. You never implement tasks. Stay as planner.`                          |
| Implementer          | `**REMEMBER:** You are the **Implementer**. You write code and tests for exactly one task. You never modify other tasks' files. You never skip TDD steps. You never modify plan.md. Stay as implementer.`       |
| Verifier             | `**REMEMBER:** You are the **Verifier**. You build, test, and verify. You never modify source code or fix bugs. You report findings only. Stay as verifier.`                                                    |
| Reviewer             | `**REMEMBER:** You are the **Reviewer**. You review code for quality, correctness, and security. You never modify source code. You write review findings only. Stay as reviewer.`                               |
| Documentation Writer | `**REMEMBER:** You are the **Documentation Writer**. You generate and maintain documentation. You never modify source code or tests. You write documentation files only. Stay as documentation writer.`         |

### Pattern 3: Completion Contract Format (Three-State)

Every agent MUST include a `## Completion Contract` section with this exact structure:

```markdown
## Completion Contract

Return exactly one line:

- DONE: <agent-specific-detail-format>
- NEEDS_REVISION: <what-needs-to-change>
- ERROR: <reason>
```

The `<agent-specific-detail-format>` varies per agent (specified in Per-Agent Design below).

**Three-state semantics:**

| State             | Meaning                                                                                               | Orchestrator response                                                                             |
| ----------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `DONE:`           | Task completed successfully.                                                                          | Proceed to next step.                                                                             |
| `NEEDS_REVISION:` | Work was completed but the output has issues that need correction by a prior agent or the same agent. | Route back to the responsible agent for lightweight revision (see per-agent routing table below). |
| `ERROR:`          | Unrecoverable failure — agent cannot complete its work.                                               | Retry once; if still ERROR, halt or escalate.                                                     |

**`NEEDS_REVISION:` routing per agent:**

| Returning Agent  | Orchestrator routes to                      | Notes                                                                                                                                                       |
| ---------------- | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Critical Thinker | Designer (with `design_critical_review.md`) | Designer revises `design.md` based on review findings. Max 1 loop.                                                                                          |
| Reviewer         | Implementer(s) for affected tasks           | Lightweight fix loop — reviewer findings are passed directly to the relevant implementer(s) without a full replan. Max 1 loop before escalating to planner. |
| Verifier         | Planner (existing replan loop)              | Retains existing heavyweight replan behavior. Up to 3 iterations.                                                                                           |
| All other agents | N/A — these agents return DONE or ERROR     | Spec, designer, planner, researcher, doc-writer use only DONE/ERROR.                                                                                        |

**Key distinction:** `NEEDS_REVISION` enables lightweight correction loops, while `ERROR` on agents like the reviewer still triggers the existing heavyweight planner→implementer→verifier replan loop. This eliminates the previous inefficiency where a one-line naming fix required a full replan cycle.

### Pattern 4: `detailed thinking on` Directive (Experimental)

Every agent file SHOULD include the following directive immediately after the role preamble paragraph, before the `## Inputs` section:

```markdown
Use detailed thinking to reason through complex decisions before acting.
```

**Rationale:** Gem Team includes this in all 8 agents. Its effect is model-dependent — it activates extended chain-of-thought reasoning where supported. Cost is 1 line per agent.

**⚠ Experimental:** This directive is unvalidated against the specific models used by VS Code Copilot. It is included because: (a) cost is negligible (1 line), (b) risk is near-zero (worst case: ignored as noise), and (c) Gem Team reports positive results. If future testing shows negative effects (e.g., unexpected behavioral changes, increased latency without quality improvement), remove it from all agents. This directive is clearly labeled in agent files with a `<!-- experimental: model-dependent -->` comment so it can be easily located and removed.

### Pattern 5: `decisions.md` Lifecycle

The cross-agent decision log (`decisions.md`) has a defined lifecycle:

**Scope:** Per-feature. Located at `docs/feature/<feature-slug>/decisions.md`. Each feature gets its own decision log. For decisions that span features (e.g., "always use repository pattern"), the reviewer should note in the decision entry that it applies project-wide.

**Creation:** The reviewer creates `decisions.md` if it does not exist when the reviewer first identifies a significant architectural decision. No other agent creates this file.

**Format:**

```markdown
# Architectural Decision Log

## [YYYY-MM-DD] <Decision Title>

- **Context:** Why this decision was needed.
- **Decision:** What was decided.
- **Rationale:** Why this option was chosen over alternatives.
- **Scope:** Feature-specific / Project-wide.
- **Affected components:** List of files, modules, or packages impacted.
```

**Writers:** Reviewer only (append-only — never modify or delete existing entries).

**Readers:** Researcher (reads `decisions.md` in the current feature directory if it exists; optionally scans `docs/feature/*/decisions.md` for project-wide entries). Critical-thinker may also reference it to check for conflicts.

**Retention:** Decision entries are never deleted. As the project matures, project-wide decisions may be manually promoted to a repo-level `docs/decisions.md` by a human maintainer — this is outside the agent workflow's scope.

---

## Per-Agent Design

### 1. `orchestrator.agent.md`

#### Current State Summary

- 235 lines, free-form structure
- Has Global Rules, Documentation Structure, Workflow Steps (0-7), Parallel Execution Summary, Completion section
- Contains hardcoded rule "Implementers never build or run tests"
- No operating rules block, no anti-drift anchor, no concurrency cap
- No per-task agent routing
- No approval gates

#### Target State — Section-by-Section Outline

```
---
name: orchestrator
description: Deterministic workflow orchestrator that coordinates all subagents, supports parallel task execution, and enforces documentation structure.
---

# Orchestrator Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs
  - User feature request (via prompt)

## Outputs
  - docs/feature/<feature-slug>/initial-request.md
  - Coordination of all subagent invocations (no direct file outputs beyond initial-request.md)

## Global Rules
  [10 rules — see content spec below]

## Documentation Structure
  [File listing — same as current, plus decisions.md]

## Operating Rules
  [Shared block — orchestrator variant]

## Workflow Steps
  ### 0. Setup
  ### 1. Research (Parallel)
    #### 1.1 Dispatch Parallel Research Agents
    #### 1.2 Synthesize Research
    #### 1.2a (Conditional) Human Approval Gate — Post-Research
  ### 2. Specification
  ### 3. Design
  ### 3b. Design Review (Critical Thinking)
  ### 4. Planning
    #### 4a (Conditional) Human Approval Gate — Post-Planning
  ### 5. Implementation Loop (Parallel Wave Execution)
    #### 5.1 Parse Execution Waves
    #### 5.2 Execute Each Wave
    #### 5.3 Handle Implementation Errors
  ### 6. Verification
  ### 7. Final Review

## Parallel Execution Summary
  [ASCII diagram — updated with sub-wave notation]

## Completion Contract

## Anti-Drift Anchor
```

#### Content Specifications for Changed/New Sections

**Global Rules (replaces current):**

```markdown
## Global Rules

1. Never modify code or documentation directly — always delegate to subagents.
2. Always pass explicit file paths to subagents.
3. Require `DONE:`, `NEEDS_REVISION:`, or `ERROR:` from every subagent before proceeding.
4. Automatically retry failed subagent invocations (`ERROR:`) once before reporting failure. Do NOT retry `NEEDS_REVISION:` — route to the appropriate agent instead (see completion contract routing table).
5. Always use custom agents (never raw LLM calls) for all work.
6. **Implementers perform unit-level TDD** (write and run tests for their task). **The verifier performs integration-level verification** across all tasks.
7. **Maximum 4 concurrent subagent invocations.** If a wave contains more than 4 tasks, partition into sub-waves of ≤4 tasks. Dispatch each sub-wave sequentially, waiting for all tasks in a sub-wave to complete before dispatching the next.
8. When dispatching each task in Step 5.2, read the `agent` field from the task file. Dispatch to the named agent only if it is in the **valid task agents list**: `implementer`, `documentation-writer`. If the field is absent, default to `implementer`. If the value is unrecognized, log a warning and default to `implementer`.
9. **⚠ Experimental (platform-dependent):** If `{{APPROVAL_MODE}}` is `true`: pause for human approval after Step 1.2 (research synthesis) and after Step 4 (planning). If `false` or unset: run fully autonomously. **Fallback:** If the runtime environment does not support interactive pausing (i.e., `{{APPROVAL_MODE}}` is not substituted or the agent protocol has no pause mechanism), log: "APPROVAL_MODE requested but interactive pause not supported — proceeding autonomously" and continue without pausing. This feature requires validation against the VS Code Copilot agent protocol before relying on it.
10. Always display which subagent you are invoking and what step you are on.
```

**Documentation Structure (updated):**

Add to the file listing:

```
- decisions.md        # (Optional) Cross-feature architectural decision log. Reviewer writes; researcher reads if exists.
- design_critical_review.md  # Critical thinker output
```

**Step 1.2a — Human Approval Gate (NEW):**

```markdown
#### 1.2a (Conditional) Human Approval Gate — Post-Research

If `{{APPROVAL_MODE}}` is `true`:

- Present the user with a summary: "Research synthesis complete. See `analysis.md`. Approve to continue to Specification, or provide feedback."
- Wait for user approval before proceeding to Step 2.
- If the user provides feedback, update `analysis.md` context and re-invoke synthesis if needed.

If `{{APPROVAL_MODE}}` is `false` or unset: skip this step entirely.
```

**Step 4a — Human Approval Gate (NEW):**

```markdown
#### 4a (Conditional) Human Approval Gate — Post-Planning

If `{{APPROVAL_MODE}}` is `true`:

- Present the user with a summary: "Planning complete. See `plan.md` with <N> tasks in <M> waves. Approve to continue to Implementation, or provide feedback."
- Wait for user approval before proceeding to Step 5.
- If the user provides feedback, re-invoke the planner with the feedback.

If `{{APPROVAL_MODE}}` is `false` or unset: skip this step entirely.
```

**Step 5.2 — Execute Each Wave (updated):**

Replace the current dispatch logic with:

```markdown
#### 5.2 Execute Each Wave

For each wave, in order:

1. **Identify ready tasks:** All tasks in the current wave whose dependencies (prior waves) have completed successfully.

2. **Apply concurrency cap:** If the wave has more than 4 tasks, partition into sub-waves of ≤4 tasks each.

3. **Dispatch agents for each (sub-)wave:**
   - For each task in the (sub-)wave:
     a. Read the task file header to check for an `agent` field.
     b. If `agent` is present and in the valid task agents list (`implementer`, `documentation-writer`), dispatch to that agent.
     c. If `agent` is absent, default to `implementer`.
     d. If `agent` has an unrecognized value, log a warning and default to `implementer`.
     e. Invoke via `runSubagent` concurrently within the (sub-)wave.
   - Each agent receives: its task file, `feature.md`, `design.md`.

4. **Wait for (sub-)wave completion:**
   - Wait for ALL agents in the current (sub-)wave to return `DONE:`, `NEEDS_REVISION:`, or `ERROR:`.
   - If there are remaining sub-waves, dispatch the next sub-wave after the current one completes.

5. **Handle results:**
   - If all tasks in the full wave return `DONE:`, proceed to the next wave.
   - If any task returns `ERROR:`, record the failure but **continue waiting** for remaining tasks. Then proceed to Step 5.3.
   - `NEEDS_REVISION:` is not expected from implementers (they return DONE or ERROR). If received, treat as ERROR.
```

**Parallel Execution Summary (updated):**

```markdown
## Parallel Execution Summary

Research: [Architecture] ──┐
[Impact] ──┤── parallel (max 4) → wait → Synthesize → analysis.md
[Dependencies] ──┘
↓ (approval gate if APPROVAL_MODE)
Spec → Design → Design Review → Planning
↓ (approval gate if APPROVAL_MODE)
Wave 1: [Task A] ──┐
[Task B] ──┤── sub-wave ≤4, parallel → wait
[Task C] ──┘
[Task D] ──┐
[Task E] ──┤── sub-wave ≤4, parallel → wait → Verify
↓ (failures?)
Re-plan only failing tasks → Re-implement → Re-verify (up to 3×)
↓ (all pass)
Wave 2: [Task F] ── run → wait → Verify
↓
Final Review
```

**Completion Contract:**

```markdown
## Completion Contract

Workflow completes only when the final review returns `DONE:`.
If the workflow cannot complete after exhausting retries, return:

- ERROR: <summary of unresolved issues>

Note: The orchestrator does not return `NEEDS_REVISION:` itself — it handles `NEEDS_REVISION:` from subagents by routing to the appropriate agent.
```

#### Key Changes from Current State

1. **Global Rule 3 updated:** Now expects three states: `DONE:`, `NEEDS_REVISION:`, `ERROR:`
2. **Global Rule 4 updated:** Retry only `ERROR:`, route `NEEDS_REVISION:` to responsible agent
3. **Global Rule 6 rewritten:** "Implementers never build or run tests" → "Implementers perform unit-level TDD; verifier does integration-level verification" (Cluster A — TDD)
4. **Global Rule 7 added:** Concurrency cap of max 4 concurrent subagents with sub-wave splitting
5. **Global Rule 8 updated:** Per-task agent routing via `agent` field with explicit valid agent list (`implementer`, `documentation-writer`) (Cluster B)
6. **Global Rule 9 updated:** Optional `APPROVAL_MODE` human gates, marked experimental with concrete fallback (Cluster C)
7. **Steps 1.2a and 4a added:** Conditional approval gate steps (experimental)
8. **Step 5.2 rewritten:** Sub-wave splitting + agent routing logic with valid agent validation
9. **Documentation Structure updated:** Added `decisions.md` and `design_critical_review.md`
10. **Operating Rules block added:** Shared cross-cutting rules with retry bounds clarification
11. **Anti-Drift Anchor added:** Final reinforcement block
12. **Role preamble added:** Standardized opening

---

### 2. `researcher.agent.md`

#### Current State Summary

- ~95 lines, two modes (focused + synthesis)
- Has clear structure: modes, inputs, outputs, rules, completion contracts
- No hybrid retrieval strategy, no confidence metadata, no operating rules, no anti-drift anchor
- Does not reference `decisions.md`

#### Target State — Section-by-Section Outline

```
---
name: researcher
description: Investigates existing codebase conventions, architecture, and impacted areas. Supports focused parallel research and synthesis modes.
---

# Research Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs
  ### Focused Mode Inputs
  ### Synthesis Mode Inputs

## Outputs
  ### Focused Mode Output
  ### Synthesis Mode Output

## Operating Rules
  [Shared block — researcher variant]

## Mode 1: Focused Research
  ### Research Focus Areas [table — same as current]
  ### Retrieval Strategy [NEW]
  ### Focused Research Rules [updated with hybrid retrieval guidance]
  ### Partial Analysis File Contents [updated with confidence metadata]

## Mode 2: Synthesis
  ### Synthesis Rules [same as current]
  ### Analysis.md Contents [same as current]

## Completion Contract
  ### Focused Mode
  ### Synthesis Mode

## Anti-Drift Anchor
```

#### Content Specifications for New/Changed Sections

**Retrieval Strategy (NEW — within Mode 1):**

```markdown
### Retrieval Strategy

Follow this methodology for all codebase investigation:

1. **Conceptual discovery:** Use `semantic_search` to find code relevant to the focus area by meaning (e.g., search for concepts, not exact identifiers).
2. **Exact pattern matching:** Use `grep_search` for specific identifiers, strings, configuration keys, and known patterns.
3. **Merge and deduplicate:** Combine results from steps 1-2. Remove duplicate file references. Prioritize files that appear in both search types.
4. **Targeted examination:** Use `read_file` with specific line ranges to examine identified files in detail. Limit to ~200 lines per call.
5. **Existence verification:** Use `file_search` to confirm expected files/patterns exist (e.g., test directories, configuration files, documentation).

Do not skip steps 1-2 and jump directly to reading files. Discovery-first ensures comprehensive coverage.
```

**Partial Analysis File Contents (updated — confidence metadata added):**

Add to the existing contents list:

```markdown
- **Research Metadata:**
  - `confidence_level`: high / medium / low — overall confidence in the completeness of findings for this focus area
  - `coverage_estimate`: qualitative description of how much of the relevant codebase was examined
  - `gaps`: areas not covered, with impact assessment (what might be missed and how it could affect downstream agents)
```

**Decision Log Read Convention (added to Focused Research Rules):**

Add this rule to Focused Research Rules:

```markdown
- If `decisions.md` exists in the documentation structure, read it and incorporate prior architectural decisions into your analysis. Note any conflicts between prior decisions and current findings.
```

**Completion Contract (Focused Mode):**

```markdown
- DONE: <focus-area> — <one-line summary>
- ERROR: <reason>
```

**Completion Contract (Synthesis Mode):**

```markdown
- DONE: synthesis — <one-line summary>
- ERROR: <reason>
```

#### Key Changes from Current State

1. **Retrieval Strategy section added:** Prescribes semantic_search → grep_search → merge → read_file → file_search methodology
2. **Confidence metadata added:** Research output includes confidence_level, coverage_estimate, and gaps
3. **Decision log read convention added:** Researcher reads `decisions.md` if it exists
4. **Operating Rules block added:** Shared cross-cutting rules with researcher-specific tool preferences
5. **Anti-Drift Anchor added**
6. **Role preamble standardized**
7. **Outputs section made explicit:** Was implicit in mode descriptions, now has its own section

---

### 3. `spec.agent.md`

#### Current State Summary

- ~45 lines, minimal structure
- Has Inputs, Outputs, Workflow, Completion Contract, feature.md Contents
- No structured edge case format, no acceptance criteria verification, no operating rules, no anti-drift anchor

#### Target State — Section-by-Section Outline

```
---
name: spec
description: Produces a clear, testable feature specification from analysis.
---

# Spec Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs
  - docs/feature/<feature-slug>/initial-request.md
  - docs/feature/<feature-slug>/analysis.md

## Outputs
  - docs/feature/<feature-slug>/feature.md

## Operating Rules
  [Shared block — spec variant]

## Workflow
  [Updated — adds structured edge cases + self-verification]

## feature.md Contents
  [Updated — adds structured edge case format]

## Completion Contract

## Anti-Drift Anchor
```

#### Content Specifications for New/Changed Sections

**Workflow (updated):**

```markdown
## Workflow

1. Read `initial-request.md` to ground the specification in the original user/developer request.
2. Review `analysis.md` thoroughly.
3. Perform minimal additional research if gaps are identified in the analysis.
4. Define:
   - Functional requirements (detailed, user-facing and system behaviors)
   - Non-functional requirements (performance, security, accessibility, offline behavior)
   - Constraints and assumptions
   - Feature-level acceptance criteria (each must be testable — see rule below)
   - Edge cases with structured format (see feature.md Contents)
5. **Self-verification step:** Before returning, verify:
   - All acceptance criteria are testable — each has a clear pass/fail definition that the verifier agent can check
   - All functional requirements have at least one corresponding acceptance criterion
   - Edge cases cover failure modes, not just happy paths
   - No requirement contradicts another
     Fix any issues found before returning.
```

**feature.md Contents (updated — structured edge cases):**

Update the Edge Cases section guidance:

```markdown
- **Edge Cases & Error Handling:** Each edge case must include:
  - **Input/Condition:** What triggers the edge case
  - **Expected Behavior:** What should happen
  - **Severity if Missed:** Impact of not handling this case (critical / high / medium / low)
```

**Completion Contract:**

```markdown
- DONE: <one-line summary>
- ERROR: <reason>
```

#### Key Changes from Current State

1. **Structured edge case format added:** Each edge case requires input/condition, expected behavior, severity
2. **Self-verification step added:** Spec agent checks its own output before returning
3. **Operating Rules block added**
4. **Anti-Drift Anchor added**
5. **Role preamble standardized**
6. **Description updated:** Added "testable" to emphasize the verification-readiness quality

---

### 4. `designer.agent.md`

#### Current State Summary

- ~45 lines, minimal structure
- Has Inputs, Outputs, Workflow, design.md Contents, Completion Contract
- No security considerations, no failure/recovery section, no self-verification, no operating rules, no anti-drift anchor

#### Target State — Section-by-Section Outline

```
---
name: designer
description: Creates a technical design document aligned with the existing architecture, including security considerations and failure analysis.
---

# Designer Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs
  - docs/feature/<feature-slug>/initial-request.md
  - docs/feature/<feature-slug>/analysis.md
  - docs/feature/<feature-slug>/feature.md
  - docs/feature/<feature-slug>/design_critical_review.md (if exists — read during revision cycle when critical-thinker returns NEEDS_REVISION and orchestrator routes back to designer)

## Outputs
  - docs/feature/<feature-slug>/design.md

## Operating Rules
  [Shared block — designer variant]

## Workflow
  [Updated — adds security, failure/recovery, self-verification]

## design.md Contents
  [Updated — adds Security Considerations + Failure & Recovery sections]

## Completion Contract

## Anti-Drift Anchor
```

#### Content Specifications for New/Changed Sections

**Workflow (updated):**

```markdown
## Workflow

1. Read `initial-request.md` to ensure the design aligns with the original request and constraints.
2. Read `analysis.md` and `feature.md` thoroughly.
3. Perform additional targeted research on architecture, patterns, and conventions relevant to the design.
4. Define architecture and component responsibilities.
5. Specify data structures and APIs.
6. Document security considerations (authentication, authorization, data protection, input validation).
7. Analyze failure modes and recovery strategies.
8. Document tradeoffs and rationale for key decisions.
9. Ensure testability and maintainability.
10. **Self-verification step:** Before returning, verify:
    - The design addresses all functional and non-functional requirements from `feature.md`
    - Every acceptance criterion from `feature.md` has a clear implementation path in the design
    - Security considerations are addressed (even if the conclusion is "no security implications" with justification)
    - Failure modes are identified and recovery strategies are defined
      Fix any gaps found before returning.
```

**design.md Contents (updated):**

```markdown
## design.md Contents

- **Title & Summary:** short feature description and design goals.
- **Context & Inputs:** references to analysis.md and feature.md used.
- **High-level Architecture:** components, responsibilities, and boundaries.
- **Data Models & DTOs:** schemas, fields, and sample payloads.
- **APIs & Interfaces:** endpoints, commands/queries, signatures, and contracts.
- **Sequence / Interaction Notes:** important call flows or sequence descriptions.
- **Security Considerations:**
  - Authentication/authorization patterns (or "N/A" with justification)
  - Data protection approach
  - Threat model (high-level)
  - Input validation strategy
- **Failure & Recovery:**
  - Expected failure modes
  - Retry/fallback strategies
  - Graceful degradation approach
- **Non-functional Requirements:** performance, offline behavior, and constraints.
- **Migration & Backwards Compatibility:** DB or API migration notes if applicable.
- **Testing Strategy:** unit/integration tests to validate design and required test cases.
- **Tradeoffs & Alternatives Considered:** short rationale for key decisions.
- **Implementation Checklist & Deliverables:** files to create/update and acceptance criteria mapping.
```

**Completion Contract:**

```markdown
- DONE: <one-line summary>
- ERROR: <reason>
```

#### Key Changes from Current State

1. **Security Considerations section added:** Auth, data protection, threat model, input validation
2. **Failure & Recovery section added:** Failure modes, retry/fallback, graceful degradation
3. **Self-verification step added:** Designer checks completeness against feature.md before returning
4. **Operating Rules block added**
5. **Anti-Drift Anchor added**
6. **Role preamble standardized**
7. **Description updated:** Added "including security considerations and failure analysis"

---

### 5. `critical-thinker.agent.md`

#### Current State Summary

- ~30 lines, free-form conversational style
- **BUG: No completion contract** — orchestrator expects `DONE:`/`ERROR:` but agent doesn't define this
- **BUG: No output file specification** — orchestrator references `design_critical_review.md` but agent doesn't mention it
- No structured risk categories, no specific output format, no operating rules, no anti-drift anchor
- Instructions are vague / conversational ("Ask 'Why?'", "Play devil's advocate")

#### Target State — Section-by-Section Outline

```
---
name: critical-thinker
description: Performs adversarial design review to identify risks, assumptions, and weaknesses in the technical design before planning begins.
---

# Critical Thinker Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs
  - docs/feature/<feature-slug>/initial-request.md
  - docs/feature/<feature-slug>/design.md
  - docs/feature/<feature-slug>/feature.md

## Outputs
  - docs/feature/<feature-slug>/design_critical_review.md

## Operating Rules
  [Shared block — critical-thinker variant]

## Workflow
  [Complete rewrite — structured risk analysis]

## Risk Categories
  [NEW — structured categories to probe]

## design_critical_review.md Contents
  [NEW — defines output structure]

## Completion Contract
  [NEW — fixes P0 bug]

## Anti-Drift Anchor
```

#### Content Specifications — FULL REWRITE

**Role Preamble:**

```markdown
You are the **Critical Thinker Agent**.

You perform adversarial design reviews to identify risks, assumptions, gaps, and weaknesses in the technical design before planning begins. You probe the design against structured risk categories and ground every concern in specific technical details.
You NEVER write code, designs, plans, or specifications. You NEVER propose solutions — you identify problems. You NEVER ask interactive questions — you produce a written review document.
```

**Workflow (complete rewrite):**

```markdown
## Workflow

1. Read `initial-request.md` to understand the original intent and constraints.
2. Read `design.md` thoroughly.
3. Read `feature.md` to understand requirements the design must satisfy.
4. Perform targeted research to verify claims, assumptions, and referenced patterns in the design.
5. Analyze the design against each Risk Category (see below). For each category:
   - Identify specific risks grounded in technical details from `design.md`
   - Assess likelihood (high / medium / low) and impact (high / medium / low)
   - Note assumptions that, if wrong, would invalidate the design decision
6. Verify that the design fully addresses all functional and non-functional requirements from `feature.md`. Flag any gaps.
7. Write `design_critical_review.md` with structured findings.
8. **Self-verification:** Before returning, verify each risk is grounded in specific technical details (file paths, component names, data flows) — not generic concerns that could apply to any project. Remove any generic risks.
```

**Risk Categories (NEW):**

```markdown
## Risk Categories

Probe the design against each of these categories. Skip a category only if it is genuinely not applicable (and state why).

1. **Security vulnerabilities:** Authentication/authorization gaps, data exposure, injection vectors, insecure defaults.
2. **Scalability bottlenecks:** Single points of failure, unbounded growth, missing pagination, lack of caching strategy.
3. **Maintainability concerns:** Tight coupling, missing abstractions, unclear boundaries, violation of project conventions.
4. **Backwards compatibility risks:** Breaking API changes, data migration gaps, configuration changes affecting existing users.
5. **Edge cases not covered:** Inputs/conditions the design doesn't address that could cause failures or data corruption.
6. **Performance implications:** Expensive operations, N+1 queries, missing indexes, synchronous blocking in async paths.

For each identified risk, state:

- **What:** The specific risk
- **Where:** File, component, or design section affected
- **Likelihood:** High / Medium / Low
- **Impact:** High / Medium / Low
- **Assumption at risk:** What underlying assumption, if wrong, would trigger this risk
```

**design_critical_review.md Contents (NEW):**

```markdown
## design_critical_review.md Contents

- **Title & Summary:** one-line verdict on the design's readiness for planning.
- **Overall Risk Level:** Low / Medium / High / Critical — with justification.
- **Risks by Category:** For each applicable risk category, list identified risks with the structured format (what, where, likelihood, impact, assumption at risk).
- **Requirement Coverage Gaps:** Any requirements from `feature.md` not addressed by the design.
- **Key Assumptions:** Assumptions the design relies on that should be validated.
- **Recommendations:** Prioritized list of issues for the designer to address (if the orchestrator loops back to Step 3).
```

**Completion Contract (NEW — fixes P0 bug, uses three-state):**

```markdown
## Completion Contract

Return exactly one line:

- DONE: <one-line summary — no blocking issues found>
- NEEDS_REVISION: <one-line summary of issues the designer must address>
- ERROR: <reason — could not complete review>

Use `NEEDS_REVISION` when the design has issues that should be corrected before planning proceeds. The orchestrator will route `design_critical_review.md` back to the designer for revision (max 1 loop). Use `DONE` when the design is acceptable for planning (may still note minor non-blocking observations). Use `ERROR` only for unrecoverable failures (e.g., design.md is empty or unreadable).
```

#### Key Changes from Current State

1. **BUG FIX: Completion contract added** — Was entirely missing; orchestrator expected it
2. **BUG FIX: Output file specified** — `design_critical_review.md` now explicitly defined
3. **Complete rewrite of workflow** — From vague conversational prompts to structured risk analysis
4. **Risk Categories section added** — 6 structured categories with likelihood/impact assessment
5. **Output format defined** — `design_critical_review.md` Contents section specifies document structure
6. **Description rewritten** — From vague "challenge assumptions" to precise adversarial design review
7. **Inputs expanded** — Added `feature.md` to verify design covers all requirements
8. **Operating Rules block added**
9. **Anti-Drift Anchor added**
10. **Role preamble standardized** — No longer conversational

---

### 6. `planner.agent.md`

#### Current State Summary

- ~120 lines, well-structured
- Has Inputs, Outputs, Workflow, Dependency-Aware Planning, Task File Requirements, Completion Contract, plan.md/task contents
- No pre-mortem analysis, no task size limits, no mode detection, no agent field, no YAGNI/KISS/DRY
- Task test requirements say "not to run" — incompatible with TDD

#### Target State — Section-by-Section Outline

```
---
name: planner
description: Creates dependency-aware implementation plans with pre-mortem analysis, task size limits, and task files for parallel execution.
---

# Planner Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs
  - docs/feature/<feature-slug>/initial-request.md
  - docs/feature/<feature-slug>/analysis.md
  - docs/feature/<feature-slug>/feature.md
  - docs/feature/<feature-slug>/design.md
  - (Replan mode) docs/feature/<feature-slug>/verifier.md
  - (Extension mode) docs/feature/<feature-slug>/plan.md (existing)

## Outputs
  - docs/feature/<feature-slug>/plan.md
  - docs/feature/<feature-slug>/tasks/*.md

## Operating Rules
  [Shared block — planner variant]

## Planning Principles [NEW]

## Mode Detection [NEW]

## Workflow
  [Updated — adds mode detection, pre-mortem, task size validation]

## Task Size Limits [NEW]

## Dependency-Aware Planning
  ### Rules for Dependencies [same as current]
  ### Dependency Graph Format [same as current]

## Task File Requirements [updated — adds agent field, updates test requirements for TDD]

## Pre-Mortem Analysis [NEW]

## plan.md Contents [updated — adds pre-mortem section, optional implementation spec]

## Task File Contents [updated — adds agent field]

## Completion Contract

## Anti-Drift Anchor
```

#### Content Specifications for New/Changed Sections

**Planning Principles (NEW):**

```markdown
## Planning Principles

- **YAGNI:** Do not plan work that isn't required by the specification. Every task must trace to a requirement.
- **KISS:** Prefer simpler decompositions. Fewer, well-scoped tasks are better than many micro-tasks.
- **Tangible value:** Each task should deliver tangible, testable value. Avoid "setup-only" tasks that produce no verifiable output.
- **Minimize dependencies:** Prefer wide, shallow dependency graphs. Deep sequential chains create bottlenecks.
```

**Mode Detection (NEW):**

```markdown
## Mode Detection

At the start of the workflow, detect the planning mode:

1. **Initial mode:** No `plan.md` exists → create a full plan from scratch.
2. **Replan mode:** `verifier.md` exists with failures → create a remediation plan addressing specific failures. Do NOT re-plan already-completed tasks. Read `verifier.md` to identify exactly which tasks failed and why.
3. **Extension mode:** Existing `plan.md` with new objectives → extend the plan. Preserve completed tasks. Add new tasks without duplicating existing work.

State the detected mode at the top of your output.
```

**Task Size Limits (NEW):**

```markdown
## Task Size Limits

Every task MUST satisfy ALL of the following limits. Any task that would exceed any limit MUST be broken into subtasks:

| Limit                     | Maximum | Rationale                                          |
| ------------------------- | ------- | -------------------------------------------------- |
| Files touched             | 3       | Keeps changes focused and reviewable               |
| Task dependencies         | 2       | Limits coupling and bottlenecks                    |
| Lines changed (estimated) | 500     | Ensures tasks complete within agent context window |
| Effort rating             | Medium  | Forces large tasks to be decomposed                |

**Clarifications:**

- The 500-line limit counts **production code only**. Test code written as part of TDD does NOT count toward this limit, since TDD inherently creates test code proportional to production code. A task with 400 lines of production code + 400 lines of tests is within limits.
- The 3-file limit counts production files only. Test files do not count toward this limit.

If a task naturally requires touching 4+ files, split it by responsibility boundary (e.g., "create data model" vs. "create API endpoint" vs. "create UI component").
```

**Plan Validation (NEW — runs before Pre-Mortem):**

```markdown
## Plan Validation

After creating the task index and execution waves, and BEFORE the pre-mortem analysis, validate the plan:

1. **Circular Dependency Check:** Walk the dependency graph and confirm no circular dependencies exist. If a cycle is detected (e.g., Task A → Task B → Task A), break it by:
   - Identifying the weaker dependency and removing it
   - Splitting one of the tasks to isolate the shared concern
   - If the cycle cannot be broken, return `ERROR: circular dependency detected between <task-ids>`
2. **Task Size Validation:** Verify every task satisfies the Task Size Limits table above.
3. **Dependency Existence Check:** Verify every `depends_on` reference points to a task that exists in the plan.
```

**Pre-Mortem Analysis (NEW):**

```markdown
## Pre-Mortem Analysis

After plan validation passes, perform a pre-mortem analysis. Append this as the final section of `plan.md`:

### Pre-Mortem Format

For each task, identify the most likely failure scenario:

| Task   | Failure Scenario      | Likelihood | Impact | Mitigation                 |
| ------ | --------------------- | ---------- | ------ | -------------------------- |
| 01-... | <what could go wrong> | H/M/L      | H/M/L  | <how to prevent or handle> |

Then add:

- **Overall Risk Level:** Low / Medium / High — with one-line justification.
- **Key Assumptions:** List assumptions that, if wrong, would invalidate the plan. For each, note which tasks depend on the assumption.
```

**Task File Requirements (updated):**

```markdown
## Task File Requirements

Each task file must include:

- **Task goal:** one-line objective
- **depends_on:** list of task IDs this task depends on, or `none`
- **agent:** (optional) which agent should execute this task. Default: `implementer`. Other valid values: `documentation-writer`. If omitted, orchestrator defaults to `implementer`.
- **In-scope / Out-of-scope:** explicit boundaries
- **Acceptance criteria:** testable conditions
- **Test requirements:** tests to write as part of TDD (the implementer both writes and runs these tests)
- **Implementation steps:** ordered checklist
- **Estimated effort:** Low / Medium (max per task size limits)
- **Completion checklist:** items to verify before marking done
```

**plan.md Contents (updated):**

Add to the existing contents:

```markdown
- **Implementation Specification** (optional): When the gap between design.md and individual tasks is large, add:
  - Code structure overview (new packages/modules/files to create)
  - Affected areas (existing code that will be modified)
  - Integration points (where new code connects to existing code)
    Reference `design.md` rather than duplicating its content.
- **Pre-Mortem Analysis:** per-task failure scenarios, overall risk level, key assumptions (see Pre-Mortem Analysis section above).
```

**Completion Contract:**

```markdown
- DONE: <number of tasks created>, <number of waves>
- ERROR: <reason>
```

#### Key Changes from Current State

1. **Mode Detection added:** Initial/replan/extension mode awareness
2. **Task Size Limits added:** Max 3 files, 2 dependencies, 500 lines, medium effort
3. **Pre-Mortem Analysis added:** Per-task failure analysis appended to plan.md
4. **Planning Principles added:** YAGNI/KISS/DRY guidance
5. **Agent field added to task files:** Optional `agent` field for per-task routing
6. **Test requirements wording updated:** "tests to write as part of TDD" (was "tests to write — not to run")
7. **Implementation Specification added:** Optional section bridging design to tasks
8. **Circular dependency check added:** Pre-mortem verifies no cycles
9. **Operating Rules block added**
10. **Anti-Drift Anchor added**

---

### 7. `implementer.agent.md`

**Most significantly changed agent.**

#### Current State Summary

- ~40 lines, minimal structure
- Explicitly forbids building and testing: "Do NOT build the project", "Do NOT run tests"
- No TDD workflow, no security rules, no code quality principles, no self-reflection
- No operating rules, no anti-drift anchor

#### Target State — Section-by-Section Outline

```
---
name: implementer
description: Implements exactly one task using TDD. Writes failing tests first, then production code, verifying with get_errors after every edit.
---

# Implementer Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs (STRICT)
  - docs/feature/<feature-slug>/tasks/<task>.md
  - docs/feature/<feature-slug>/feature.md
  - docs/feature/<feature-slug>/design.md

  You MUST NOT read:
  - plan.md

## Outputs
  - Code files as specified in the task
  - Test files as specified in the task
  - Updated task file (status, completion checklist)

## Operating Rules
  [Shared block — implementer variant]

## Code Quality Principles [NEW]

## Security Rules [NEW]

## TDD Workflow [NEW — replaces old Workflow]

## TDD Fallback [NEW — for non-testable contexts]

## Rules
  [Updated — old prohibitions removed]

## Completion Contract

## Anti-Drift Anchor
```

#### Content Specifications — MAJOR REWRITE

**Role Preamble:**

```markdown
You are the **Implementer Agent**.

You implement exactly one task using Test-Driven Development. You write failing tests first, then write minimal production code to make them pass, running `get_errors` after every file edit.
You NEVER read `plan.md`. You NEVER modify files outside your task's scope. You NEVER skip TDD steps unless the TDD Fallback applies.
```

**Code Quality Principles (NEW):**

```markdown
## Code Quality Principles

- **YAGNI:** Don't implement functionality that isn't required by the task. No speculative features.
- **KISS:** Prefer the simplest correct solution. Avoid over-engineering.
- **DRY:** Extract duplication only when there are 3+ instances. Avoid premature abstraction.
- **Impact check:** Before refactoring or modifying existing code, use `list_code_usages` to check all call sites and ensure no breakage. If `list_code_usages` is unavailable, use `grep_search` as a fallback.
```

**Security Rules (NEW):**

```markdown
## Security Rules

- **Never hardcode secrets:** No API keys, tokens, passwords, or connection strings in source code. Use environment variables or configuration files.
- **Never expose PII:** No personally identifiable information in logs, error messages, comments, or test data.
- **Fix security issues:** If you discover a security vulnerability in existing code while working on your task:
  - If the fix is **within the files you are already modifying** for the task: fix it regardless of file count.
  - If it requires modifying files beyond your task's scope: document it in the task file as a finding with `severity: critical` for the reviewer to flag. Do NOT expand your file scope beyond what the task specifies.
```

**TDD Workflow (NEW — replaces old Workflow):**

```markdown
## TDD Workflow

Execute these steps in order for every task:

### 1. Understand

- Read the task file thoroughly.
- Read relevant sections of `feature.md` and `design.md` for context.
- Identify the specific files to create or modify.

### 2. Write Failing Tests

- Write test(s) that verify the task's acceptance criteria.
- Tests MUST be meaningful — they should test behavior, not implementation details.
- Run the tests. **Confirm they fail.** If tests pass before writing production code, the tests are not testing new behavior — rewrite them.

### 3. Write Production Code

- Write the **minimal** production code needed to make the failing tests pass.
- Run `get_errors` after **every file edit** to catch compilation/lint errors immediately. Fix any errors before proceeding.
- Do not write code beyond what the tests require.

### 4. Verify

- Run the tests. **Confirm they pass.**
- If tests fail, fix the production code (not the tests, unless the test itself has a bug).
- Run `get_errors` again after any fix.

### 5. Refactor (Optional)

- If the code can be improved without changing behavior, refactor.
- Run tests after refactoring. **Confirm they still pass.**
- Run `get_errors` after any refactoring edit.

### 6. Update Task File

- Check off acceptance criteria (code-level verification).
- Mark implementation as completed.
- Note any findings (security issues, edge cases discovered, etc.).

### 7. Self-Reflection (Medium/High effort tasks only)

- **Skip this step for Low effort tasks** (simple configuration changes, version bumps, single-line fixes).
- For Medium and above effort tasks, before returning, verify:
  - All task requirements are addressed
  - Tests cover the task's acceptance criteria
  - No obvious omissions or incomplete implementations
  - Code follows project conventions
  - "Would a senior engineer approve this code?"
- Fix any issues found during self-reflection before returning.
```

**TDD Fallback (NEW):**

```markdown
## TDD Fallback

The implementer detects whether TDD applies using these heuristics:

### Detection: Is a test framework available?

Scan the project for test framework configuration files:

- **JavaScript/TypeScript:** `jest.config.*`, `vitest.config.*`, `.mocharc.*`, `karma.conf.*`, `cypress.config.*`, `playwright.config.*`
- **Python:** `pytest.ini`, `pyproject.toml` (with `[tool.pytest]`), `setup.cfg` (with `[tool:pytest]`), `tox.ini`
- **.NET:** `*.Tests.csproj`, `*.Test.csproj`, or any `*.csproj` referencing `Microsoft.NET.Test.Sdk`
- **Java:** `src/test/` directory, `pom.xml` with test dependencies, `build.gradle` with test configurations
- **Rust:** `#[cfg(test)]` modules (inherent to Rust — always available)
- **Go:** `*_test.go` files (inherent to Go — always available)

If none detected, TDD is not applicable.

### Detection: Is the task testable?

Skip TDD if the task is:

- Purely configuration/documentation (no behavioral code changes)
- Infrastructure setup (creating the test framework itself — you can't test the test framework with the test framework)
- Static asset changes (images, fonts, static HTML)

### Fallback procedure:

1. Note in the task file: "TDD skipped: <reason>" (e.g., "no test framework detected", "configuration-only task", "task creates the test framework")
2. Proceed with implementation, using `get_errors` after every edit as the primary validation mechanism.
3. If the task involves code changes that _could_ be tested but the test framework is missing, note this as a finding for the verifier: "Recommend adding test framework; untested code at <file paths>."
```

**Rules (updated):**

```markdown
## Rules

- One task only — no unrelated changes.
- No inferred scope — implement only what the task specifies.
- No modifications to `plan.md` or other tasks' files.
- Run `get_errors` after every file modification without exception.
- Follow the TDD workflow for all code tasks (see TDD Fallback for exceptions).
```

**Completion Contract:**

```markdown
- DONE: <task-id>
- ERROR: <reason>
```

#### Key Changes from Current State

1. **TDD Workflow replaces old Workflow** — Write tests → confirm fail → write code → get_errors → confirm pass → refactor → self-reflect
2. **"Do NOT build" and "Do NOT run tests" REMOVED** — Implementer now runs tests as part of TDD
3. **`get_errors` after every edit added** — Catches errors immediately
4. **TDD Fallback added** — Handles non-testable contexts gracefully
5. **Security Rules added** — No hardcoded secrets, no PII, fix in-scope vulnerabilities
6. **Code Quality Principles added** — YAGNI/KISS/DRY + `list_code_usages` before refactoring
7. **Self-Reflection step added** — Quality self-check before returning
8. **Description rewritten** — From "Does not build or run tests" to "Writes failing tests first"
9. **Outputs section added** — Was implicit, now explicit
10. **Operating Rules block added**
11. **Anti-Drift Anchor added**

---

### 8. `verifier.agent.md`

#### Current State Summary

- ~110 lines, well-structured
- Has Inputs, Role, Workflow (5 steps), Targeted Re-verification, Completion Contract, verifier.md Contents
- **Hardcoded:** `dotnet build TourneyPal.sln`, `dotnet test tests/TourneyPal.Tests`
- Role defined as "sole agent responsible for building and running tests" — incompatible with implementer TDD
- No read-only enforcement, no operating rules, no anti-drift anchor

#### Target State — Section-by-Section Outline

```
---
name: verifier
description: Integration-level build, test, and verification agent. Validates that all independently-implemented tasks work together correctly.
---

# Verifier Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs
  [Same as current]

## Outputs
  - docs/feature/<feature-slug>/verifier.md

## Role [REWRITTEN]

## Operating Rules
  [Shared block — verifier variant]

## Workflow
  ### 1. Detect Build System [NEW — replaces hardcoded commands]
  ### 2. Build [updated — technology-agnostic]
  ### 3. Test Execution [updated — technology-agnostic, integration focus]
  ### 4. Task-Level Verification [same as current]
  ### 5. Feature-Level Verification [same as current]
  ### 6. Report [same as current]

## Read-Only Enforcement [NEW]

## Targeted Re-verification [same as current]

## verifier.md Contents [same as current]

## Completion Contract

## Anti-Drift Anchor
```

#### Content Specifications for Changed/New Sections

**Role (REWRITTEN):**

```markdown
## Role

The verifier is the **integration-level verification agent**. Implementer agents have already verified their individual tasks via unit-level TDD (writing and running tests during implementation). The verifier focuses on:

- **Full project build:** Does everything compile/link together after all tasks are integrated?
- **Full test suite execution:** Run ALL tests (unit tests written by implementers + any integration/e2e tests). All should pass.
- **Cross-task interaction verification:** Do independently-implemented tasks work together correctly?
- **Acceptance criteria verification:** Does the combined implementation satisfy `feature.md` requirements?

If unit tests that should have passed during TDD are now failing, this indicates a cross-task integration issue or an implementer error — flag it as high-severity.
```

**Detect Build System (NEW — Step 1):**

```markdown
### 1. Detect Build System

Scan the project root for build configuration files in this priority order:

| File(s)                              | Build System  | Build Command                   | Test Command                   |
| ------------------------------------ | ------------- | ------------------------------- | ------------------------------ |
| `package.json`                       | npm/yarn/pnpm | `npm run build` or `yarn build` | `npm test` or `yarn test`      |
| `Makefile` or `CMakeLists.txt`       | Make/CMake    | `make` or `cmake --build .`     | `make test` or `ctest`         |
| `*.sln` or `*.csproj`                | .NET          | `dotnet build`                  | `dotnet test`                  |
| `pom.xml`                            | Maven         | `mvn compile`                   | `mvn test`                     |
| `build.gradle` or `build.gradle.kts` | Gradle        | `gradle build`                  | `gradle test`                  |
| `Cargo.toml`                         | Rust/Cargo    | `cargo build`                   | `cargo test`                   |
| `pyproject.toml` or `setup.py`       | Python        | `pip install -e .` or N/A       | `pytest` or `python -m pytest` |
| `go.mod`                             | Go            | `go build ./...`                | `go test ./...`                |

**This list is non-exhaustive.** For build systems not listed (e.g., `deno.json`/`deno.jsonc` for Deno, `bun.lockb` for Bun, `composer.json` for PHP, `mix.exs` for Elixir, `build.sbt` for Scala, `Gemfile` for Ruby, `Package.swift` for Swift), detect the appropriate build/test commands using similar heuristics: scan for the build system's configuration file, then run the conventional build and test commands for that ecosystem.

If multiple build systems are detected, prefer the one referenced in `README.md` or project documentation. If none is detected, report "No recognized build system detected" and skip build/test steps — proceed directly to acceptance criteria verification.
```

**Build (Step 2 — updated):**

```markdown
### 2. Build

- Run the build command identified in Step 1.
- Record all build errors and warnings with file paths and line numbers.
- If the build fails, skip test execution and proceed directly to reporting.
```

**Test Execution (Step 3 — updated):**

```markdown
### 3. Test Execution

- Run the test command identified in Step 1.
- Run the **full** test suite (unit + integration + e2e).
- Record all test results: passing, failing, and skipped.
- Capture stack traces and failure excerpts for failing tests.
- Note: Implementers should have already run unit tests during TDD. If unit tests are failing, this indicates a cross-task integration issue or a regression — flag as high-severity in the report.
```

**Read-Only Enforcement (NEW):**

```markdown
## Read-Only Enforcement

The verifier MUST NOT modify source code, test files, configuration files, or any project files. The verifier is strictly **read-only** with respect to the codebase. The only files the verifier writes are:

- `docs/feature/<feature-slug>/verifier.md` (its output artifact)
- Task file status updates (marking tasks as verified/partially-verified/failed)

If a fix is needed, document it in `verifier.md` for the planner to address via re-implementation.
```

**Completion Contract:**

```markdown
- DONE: verification passed (<N> tasks verified, <M> tests passing)
- NEEDS_REVISION: verification failed — <N> build errors, <M> test failures, <K> tasks need fixes
- ERROR: <unrecoverable failure — e.g., no build system detected and no tests to run>

Use `NEEDS_REVISION` when tests or builds fail but the failures are addressable through a replan cycle. The orchestrator will route to the planner for targeted task remediation (up to 3 iterations). Use `DONE` only when all builds pass and all tests pass. Use `ERROR` for situations where verification cannot be performed at all.
```

#### Key Changes from Current State

1. **Role redefined:** "Sole owner of building and testing" → "Integration-level verification agent" (Cluster A — TDD)
2. **Technology-agnostic:** All hardcoded `dotnet` commands replaced with build system detection table
3. **Read-Only Enforcement added:** Explicitly prevents verifier from modifying code
4. **Unit test failure handling clarified:** If unit tests fail that should have passed during TDD, flag as high-severity
5. **Description rewritten:** "Centralized build, test, and verification agent" → "Integration-level build, test, and verification agent"
6. **Operating Rules block added**
7. **Anti-Drift Anchor added**

---

### 9. `reviewer.agent.md`

#### Current State Summary

- ~65 lines, reasonable structure
- Has Inputs, Workflow, Completion Contract, review.md Contents
- No security review, no tiered depth, no read-only enforcement, no decision log, no quality bar, no self-reflection
- No operating rules, no anti-drift anchor

#### Target State — Section-by-Section Outline

```
---
name: reviewer
description: Performs a security-aware peer-style code review with tiered depth, producing actionable findings and maintaining the architectural decision log.
---

# Reviewer Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs
  - docs/feature/<feature-slug>/initial-request.md
  - Git diff
  - Entire codebase

## Outputs
  - docs/feature/<feature-slug>/review.md
  - .github/instructions/*.instructions.md (updates if needed)
  - docs/feature/<feature-slug>/decisions.md (append only, if significant decisions identified)

## Operating Rules
  [Shared block — reviewer variant]

## Read-Only Enforcement [NEW]

## Review Depth Tiers [NEW]

## Workflow [REWRITTEN — adds security, tiers, decision log, self-reflection]

## Security Review [NEW]

## Quality Standard [NEW]

## review.md Contents [updated — adds security section]

## Completion Contract

## Anti-Drift Anchor
```

#### Content Specifications for New/Changed Sections

**Read-Only Enforcement (NEW):**

```markdown
## Read-Only Enforcement

The reviewer MUST NOT modify source code, test files, or project files. The reviewer is strictly **read-only** with respect to the codebase. The only files the reviewer writes are:

- `docs/feature/<feature-slug>/review.md` (its output artifact)
- `.github/instructions/*.instructions.md` (convention/instruction updates)
- `docs/feature/<feature-slug>/decisions.md` (architectural decision log — append only)
```

**Review Depth Tiers (NEW):**

```markdown
## Review Depth Tiers

Determine the review tier by examining the changed files and their content:

| Tier            | Trigger                                                                                                                     | Scope                                                                                            |
| --------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Full**        | Security-sensitive changes (auth, data storage, payments, admin, networking), core architecture changes, public API changes | Review every line. Apply all criteria including OWASP security review. Check all edge cases.     |
| **Standard**    | Business logic, new features, refactoring, internal API changes                                                             | Review logic correctness, test coverage, naming, patterns, code quality. Standard security scan. |
| **Lightweight** | Documentation, configuration, dependency updates, formatting, comments                                                      | Check correctness and consistency only. Standard security scan.                                  |

State the determined tier at the top of `review.md`.
```

**Workflow (REWRITTEN):**

```markdown
## Workflow

1. Read `initial-request.md` to understand the original intent and scope.
2. Examine the git diff to identify all changed files.
3. **Determine review tier** based on changed file content (see Review Depth Tiers).
4. Review code for:
   - Maintainability and readability
   - Naming conventions and project conventions
   - Test quality and coverage
   - Architectural alignment with `design.md`
   - Logic correctness
5. **Perform security review** (see Security Review section — applies to ALL tiers).
6. Call out questionable decisions with specific rationale.
7. Suggest improvements with actionable, specific recommendations.
8. If significant architectural decisions are identified (e.g., "this pattern should be used consistently," "this dependency was chosen over X for reason Y"), append them to `decisions.md` with date, context, and rationale. **If `decisions.md` does not exist, create it** (see decisions.md Lifecycle below).
9. Update `.github/instructions` based on any new conventions or patterns observed in the changes.
10. Produce `review.md`.
11. **Self-reflection:** Before returning, verify:
    - All changed files were reviewed (none skipped)
    - Review comments are actionable and specific (not vague)
    - Security scan was completed (secrets + OWASP if Full tier)
    - If blocking concerns exist, `review.md` contains actionable items the orchestrator can convert into tasks
      Fix any gaps before returning.
```

**Security Review (NEW):**

```markdown
## Security Review

### All Reviews (Every Tier)

**Heuristic pre-scan (for large diffs with 20+ changed files):** Before scanning every file, do a quick heuristic check — run `grep_search` for high-signal patterns (`password`, `secret`, `api_key`, `token`, `Bearer`) across the changed files. If no hits, note "Security pre-scan: no indicators found" and skip the deep secrets scan. Always do the full scan for diffs under 20 files.

**Standard secrets/PII scan:**

Scan all changed files for:

- Hardcoded secrets, API keys, tokens, passwords using `grep_search` with patterns: `password`, `secret`, `api_key`, `apikey`, `token`, `Bearer`, `private_key`, `AWS_`, `AZURE_`, `connection_string`
- PII in logs, error messages, or test data: names, emails, phone numbers, SSNs, credit card numbers
- Flag any findings with `severity: critical`.

### Full Tier Only — OWASP Top 10

For security-sensitive changes (auth, data, payments, admin, networking), additionally check:

1. **Injection:** SQL injection, command injection, XSS in inputs
2. **Broken authentication:** Weak auth flows, missing token validation
3. **Sensitive data exposure:** Unencrypted sensitive data, overly verbose error messages
4. **XML external entities (XXE):** If XML parsing is involved
5. **Broken access control:** Missing authorization checks, IDOR vulnerabilities
6. **Security misconfiguration:** Debug mode in production, default credentials
7. **Cross-site scripting (XSS):** Unsanitized user input in output
8. **Insecure deserialization:** Untrusted data deserialization
9. **Known vulnerabilities:** Outdated dependencies with known CVEs
10. **Insufficient logging:** Missing audit trails for security-sensitive operations
```

**Quality Standard (NEW):**

```markdown
## Quality Standard

Apply this calibration: **Would a staff engineer approve this code?**

This means the code is:

- Correct and handles edge cases
- Well-tested with meaningful tests
- Readable and self-documenting
- Maintainable and follows project conventions
- Secure and free of obvious vulnerabilities
- Performant without premature optimization
```

**review.md Contents (updated):**

Add to existing contents:

```markdown
- **Review Tier:** Full / Standard / Lightweight — with rationale.
- **Security Findings:** Results of secrets/PII scan. For Full tier, OWASP checklist results.
- **Architectural Decisions:** Significant decisions identified and logged to `decisions.md` (if any).
```

**Completion Contract:**

```markdown
- DONE: review complete — <tier> review, <N> issues (<M> blocking)
- NEEDS_REVISION: <summary of issues implementers must fix> — <N> issues requiring revision
- ERROR: <unrecoverable failure reason>

Use `NEEDS_REVISION` when the review finds issues that specific implementers can fix without a full replan (e.g., naming fixes, missing null checks, minor logic errors, style violations). The orchestrator will route the relevant findings back to the affected implementer(s) for a single lightweight fix pass. Use `ERROR` only for systemic/architectural concerns requiring a full replan through the planner.
```

#### Key Changes from Current State

1. **Security Review section added:** Secrets/PII scan for all reviews + OWASP for Full tier
2. **Review Depth Tiers added:** Full/Standard/Lightweight based on change content
3. **Read-Only Enforcement added:** Explicitly prevents code modification
4. **Decision Log write convention added:** Reviewer appends significant decisions to `decisions.md`
5. **Quality Standard added:** "Would a staff engineer approve this?" calibration
6. **Self-Reflection step added:** Reviewer checks its own review completeness
7. **Workflow rewritten:** Structured 11-step process with security and self-reflection
8. **Completion contract enriched:** Includes tier and issue counts
9. **Operating Rules block added**
10. **Anti-Drift Anchor added**
11. **Description rewritten:** Adds "security-aware" and "tiered depth"
12. **Outputs section added:** Was implicit, now explicit (includes decisions.md)

---

### 10. `feature-workflow.prompt.md`

#### Current State Summary

- ~20 lines, minimal
- Has frontmatter (name, agent), rules, `{{USER_FEATURE}}` variable
- No concurrency cap, no APPROVAL_MODE, no per-task routing

#### Target State — Section-by-Section Outline

```
---
name: Feature Workflow
agent: orchestrator
---

You are the Orchestrator agent.

Run the entire custom agent workflow end-to-end without stopping.

## Rules
  [Updated — adds concurrency, agent routing, approval mode]

## Variables
  [NEW — documents template variables]

User Feature:
{{USER_FEATURE}}
```

#### Content Specifications

**Rules (updated):**

```markdown
## Rules

- Always display which subagent you are invoking.
- Always display what step you are on.
- Never implement code yourself.
- Always delegate research, planning, design, implementation, verification, and review to subagents.
- Always use custom agents.
- Retry failed steps automatically.
- Do not ask the user questions.
- Dispatch independent agents in parallel (research focus areas, implementation waves).
- Wait for all parallel agents to complete before proceeding to the next step.
- **Maximum 4 concurrent subagent invocations per wave.** Waves with more than 4 tasks are split into sub-waves of ≤4.
- **Tasks may specify an `agent` field.** The orchestrator dispatches to the named agent (default: `implementer`).
- **If `{{APPROVAL_MODE}}` is `true`:** pause for human approval after research synthesis and after planning. Otherwise, run fully autonomously.
```

**Variables (NEW):**

```markdown
## Variables

| Variable            | Required | Default | Description                                                                                     |
| ------------------- | -------- | ------- | ----------------------------------------------------------------------------------------------- |
| `{{USER_FEATURE}}`  | Yes      | —       | The user's feature request                                                                      |
| `{{APPROVAL_MODE}}` | No       | `false` | When `true`, orchestrator pauses for human approval after research synthesis and after planning |
```

**Completion:**

```markdown
User Feature:
{{USER_FEATURE}}

Approval Mode:
{{APPROVAL_MODE}}
```

#### Key Changes from Current State

1. **Concurrency cap rule added:** Max 4 concurrent with sub-wave splitting
2. **Per-task agent routing rule added:** Tasks can specify `agent` field
3. **APPROVAL_MODE support added:** Conditional approval gates
4. **Variables section added:** Documents template variables with defaults
5. **APPROVAL_MODE variable reference added:** At the end alongside USER_FEATURE

---

### 11. `documentation-writer.agent.md` (NEW)

#### Current State Summary

Does not exist. New file.

#### Target State — Section-by-Section Outline

```
---
name: documentation-writer
description: Generates and maintains project documentation including API docs, architectural diagrams, and README updates.
---

# Documentation Writer Agent Workflow

[Role preamble — 3 sentences]
[detailed thinking on directive]

## Inputs (STRICT)
  - docs/feature/<feature-slug>/tasks/<task>.md (the assigned documentation task)
  - docs/feature/<feature-slug>/feature.md
  - docs/feature/<feature-slug>/design.md
  - Entire codebase (read-only)

## Outputs
  - Documentation files as specified in the task (API docs, READMEs, diagrams, etc.)
  - Updated task file (status, completion checklist)

## Operating Rules
  [Shared block — documentation-writer variant]

## Read-Only Enforcement

## Capabilities

## Workflow

## Completion Contract

## Anti-Drift Anchor
```

#### Content Specifications — FULL NEW FILE

**Role Preamble:**

```markdown
You are the **Documentation Writer Agent**.

You generate and maintain project documentation based on an assigned task. You analyze source code to produce accurate, up-to-date documentation including API references, architectural diagrams, README updates, and code-documentation parity verifications.
You NEVER modify source code, tests, or configuration files. You ONLY write documentation files.
```

**Read-Only Enforcement:**

```markdown
## Read-Only Enforcement

The documentation writer MUST NOT modify source code, test files, or configuration files. The documentation writer is strictly **read-only** with respect to the codebase. The only files the documentation writer creates or modifies are:

- Documentation files specified in the task (markdown, OpenAPI/Swagger YAML, Mermaid diagrams, etc.)
- The assigned task file (status updates)
```

**Capabilities:**

```markdown
## Capabilities

- **API Documentation:** Generate OpenAPI/Swagger specifications from code analysis. Document endpoints, request/response schemas, authentication, and error codes.
- **Architectural Diagrams:** Create Mermaid diagrams showing component relationships, data flow, sequence diagrams, and deployment topology.
- **README Updates:** Update project README with new features, setup instructions, API usage examples.
- **Code-Documentation Parity:** Verify that public APIs, exported functions, and configuration options are documented. Flag undocumented public interfaces.
- **Documentation Coverage Matrix:** Produce a matrix showing which components/APIs are documented and which have gaps.
```

**Workflow:**

```markdown
## Workflow

1. Read the assigned task file to understand the documentation requirements.
2. Read `feature.md` and `design.md` for feature context.
3. Analyze relevant source code using `semantic_search`, `grep_search`, and `read_file` (read-only).
4. Generate documentation as specified in the task.
5. **Verify accuracy (delta-only):** Use `get_changed_files` (or equivalent) to identify recently changed source files. Cross-reference generated documentation against **only the changed code**, not the entire codebase. This ensures parity verification is proportional to the delta, not the full project size.
6. Update the task file:
   - Check off acceptance criteria
   - Mark implementation as completed
   - Note any documentation gaps discovered (for future tasks)
```

**Completion Contract:**

```markdown
## Completion Contract

Return exactly one line:

- DONE: <task-id>
- ERROR: <reason>
```

#### Key Design Decisions

- **Not in core pipeline:** Invoked only via per-task routing (task `agent: documentation-writer`)
- **Same completion contract format** as all other agents
- **Read-only enforcement** consistent with verifier and reviewer
- **Depends on Cluster B** (per-task agent routing) being implemented

---

## Agent Communication Contracts

### Completion Protocol

Every agent terminates with exactly one line matching one of:

```
DONE: <detail>
NEEDS_REVISION: <what needs to change>
ERROR: <reason>
```

**Rules:**

- The line MUST be the last line of output
- `DONE:` indicates success; `<detail>` is agent-specific (see per-agent specs above)
- `NEEDS_REVISION:` indicates work was completed but output needs correction; triggers lightweight revision routing (see Pattern 3 routing table)
- `ERROR:` indicates unrecoverable failure; `<reason>` must be a human-readable explanation
- The orchestrator parses this line to determine next steps
- Only agents with defined revision targets use `NEEDS_REVISION:` (critical-thinker, reviewer, verifier). All other agents use only `DONE:` / `ERROR:`

### Agent-Specific DONE Formats

| Agent                  | DONE Format                                                 | NEEDS_REVISION Format (if applicable)                            |
| ---------------------- | ----------------------------------------------------------- | ---------------------------------------------------------------- |
| Orchestrator           | (N/A — orchestrator doesn't report to anything)             | N/A                                                              |
| Researcher (focused)   | `DONE: <focus-area> — <summary>`                            | N/A                                                              |
| Researcher (synthesis) | `DONE: synthesis — <summary>`                               | N/A                                                              |
| Spec                   | `DONE: <summary>`                                           | N/A                                                              |
| Designer               | `DONE: <summary>`                                           | N/A                                                              |
| Critical Thinker       | `DONE: <summary — no blocking issues>`                      | `NEEDS_REVISION: <issues designer must address>`                 |
| Planner                | `DONE: <N> tasks created, <M> waves`                        | N/A                                                              |
| Implementer            | `DONE: <task-id>`                                           | N/A                                                              |
| Verifier               | `DONE: verification passed (<N> tasks, <M> tests)`          | `NEEDS_REVISION: <N> failures requiring replan`                  |
| Reviewer               | `DONE: review complete — <tier>, <N> issues (<M> blocking)` | `NEEDS_REVISION: <N> issues implementers can fix without replan` |
| Documentation Writer   | `DONE: <task-id>`                                           | N/A                                                              |

### Orchestrator Expectations Per Agent

| Agent                  | On DONE                                          | On NEEDS_REVISION                                                                                                                                                | On ERROR                                                    |
| ---------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| Researcher (focused)   | Collect result; wait for all 3                   | N/A                                                                                                                                                              | Retry once; if still ERROR, fail research step              |
| Researcher (synthesis) | Proceed to spec (or approval gate)               | N/A                                                                                                                                                              | Retry once; if still ERROR, halt                            |
| Spec                   | Proceed to design                                | N/A                                                                                                                                                              | Retry once; if still ERROR, halt                            |
| Designer               | Proceed to critical review                       | N/A                                                                                                                                                              | Retry once; if still ERROR, halt                            |
| Critical Thinker       | Proceed to planning                              | Route `design_critical_review.md` back to designer for revision (max 1 loop). If still NEEDS_REVISION after revision, proceed to planning anyway with a warning. | Retry once; if still ERROR, halt                            |
| Planner                | Proceed to implementation (or approval gate)     | N/A                                                                                                                                                              | Retry once; if still ERROR, halt                            |
| Implementer            | Collect result; wait for all tasks in (sub-)wave | N/A (implementers use only DONE/ERROR)                                                                                                                           | Record failure; wait for remaining; proceed to verification |
| Verifier               | Proceed to review                                | Trigger replan loop via planner (up to 3×)                                                                                                                       | Trigger replan loop                                         |
| Reviewer               | Workflow complete                                | Route findings to affected implementer(s) for lightweight fix (max 1 loop). If still NEEDS_REVISION after fix, escalate to planner.                              | Trigger replan via planner                                  |
| Documentation Writer   | Collect result (same as implementer)             | N/A                                                                                                                                                              | Record failure; continue                                    |

### Artifact Dependency Chain

```
initial-request.md
  ↓ (read by all agents)
research/*.md → analysis.md
                  ↓
              feature.md
                  ↓
              design.md → design_critical_review.md
                  ↓              ↓ (fed back to designer if issues)
              plan.md + tasks/*.md
                  ↓
              [code changes via implementers]
                  ↓
              verifier.md → (replan loop if failures)
                  ↓
              review.md → (replan loop if blocking)
                  ↓
              decisions.md (optional, reviewer writes)
```

Each agent reads only the artifacts specified in its `## Inputs` section and writes only to files specified in its `## Outputs` section. The orchestrator enforces this by controlling file paths passed to each subagent.

---

## Tradeoffs & Alternatives Considered

### T1: Markdown vs. YAML State (Decided: Markdown)

**Options:** (A) Keep markdown artifacts, (B) Adopt Gem's `plan.yaml` + YAML state
**Decision:** Keep markdown.
**Rationale:** Markdown is human-readable, git-diffable, and consistent with Forge's philosophy. YAML adds schema complexity. The fixed pipeline doesn't benefit from YAML's structured data — task files and plan.md are read sequentially, not queried by key.

### T2: Two-State vs. Three-State Completion (Decided: Three-State)

**Options:** (A) Keep `DONE:`/`ERROR:` (2 states), (B) Adopt `DONE:`/`NEEDS_REVISION:`/`ERROR:` (3 states, inspired by Gem's model but using text, not JSON)
**Decision:** Adopt three-state.
**Rationale (revised from original design):** The original design rejected three-state citing migration cost, but the design's own premise is that all agents are being rewritten from scratch — migration cost is zero. Evaluating on functional merit alone, `NEEDS_REVISION:` provides clear value in two cases:

1. **Critical-thinker → Designer:** The critical-thinker can signal "design needs work" semantically with `NEEDS_REVISION:`, letting the orchestrator route back to the designer without parsing the review document for heuristic indicators.
2. **Reviewer → Implementer:** Many review findings are minor (rename a variable, add a null check). With `NEEDS_REVISION:`, the orchestrator routes directly back to the affected implementer(s) for a lightweight fix — avoiding the heavyweight planner→implementer→verifier replan loop that `ERROR:` triggers.

The third state is text-based (not JSON), consistent with Forge's text completion philosophy. Only 3 of 11 agents actively use `NEEDS_REVISION:` (critical-thinker, reviewer, verifier); the remaining agents use only `DONE:`/`ERROR:` — keeping complexity contained. The verifier's `NEEDS_REVISION:` maps to the existing replan loop, maintaining backward compatibility with the existing orchestrator flow.

### T3: Mandatory vs. Optional Human Gates (Decided: Optional)

**Options:** (A) Gem's 3 mandatory pauses, (B) Optional via `APPROVAL_MODE`, (C) No gates
**Decision:** Optional via `APPROVAL_MODE`, default autonomous.
**Rationale:** Full autonomy is a core Forge strength (end-to-end without intervention). Mandatory gates break this. Optional gates give users the choice without forcing it.

### T4: XML Tags vs. Markdown Headings for Agent Structure (Decided: Markdown Headings)

**Options:** (A) Keep markdown headings, (B) Adopt Gem's XML-like tags (`<role>`, `<workflow>`, `<final_anchor>`)
**Decision:** Keep markdown headings.
**Rationale:** This is a pragmatic choice driven by concrete advantages, not a principled claim about LLM parsing. Markdown headings are: (1) human-readable without tooling, (2) natively rendered by GitHub/VS Code, (3) consistent with the `chatagent` format specification, and (4) editable with standard markdown tooling. XML tags may improve LLM parsing — both XML tags and anti-drift anchors are hypotheses about LLM behavior, and we adopt anchors for their low cost and recency-bias exploitation. The difference is that anchors are 2-3 lines at the end of a file (minimal cost, clear rationale), while XML tags would require restructuring every section in every agent file (high cost, uncertain benefit). If XML tags prove superior in testing, this can be changed later without architectural impact.

### T5: Shared Operating Rules Approach (Decided: Duplicate in Each File)

**Options:** (A) Duplicate rules in each agent file, (B) Use a shared include/reference file
**Decision:** Duplicate in each file.
**Rationale:** The `chatagent` format does not support includes. Each agent file must be self-contained. Duplication ensures every agent has the rules regardless of how it's invoked. The operating rules block is 10-15 lines — acceptable duplication cost.

### T6: Review Depth Source (Decided: Reviewer Self-Determines)

**Options:** (A) Reviewer determines tier from diff content, (B) Planner assigns review depth in task metadata
**Decision:** Reviewer self-determines.
**Rationale:** The reviewer has the diff context and can accurately classify changes. Using planner metadata couples two agents unnecessarily and the planner may not know the actual content of changes at planning time.

### T7: Verifier Role After TDD (Decided: Run Full Suite)

**Options:** (A) Run only integration/e2e tests (unit tests already passed), (B) Run full test suite
**Decision:** Run full test suite.
**Rationale:** Unit tests that passed individually during TDD might fail when integrated across tasks (merge conflicts, incompatible changes). Running the full suite catches these cross-task regressions. The overhead is minimal — the verifier runs tests once per wave.

### T8: Anti-Drift Anchor Inclusion (Decided: Include in All)

**Options:** (A) Include in all agents, (B) Include only in agents with long sessions (orchestrator, implementer), (C) Skip entirely
**Decision:** Include in all agents.
**Rationale:** Cost is 2-3 lines per agent. Benefit is unvalidated but risk is near-zero. Gem includes it in all 8 agents. It exploits LLM recency bias — the model is more likely to remember instructions that appear at the end.

### T9: Concurrency Cap Value (Decided: 4)

**Options:** 2, 3, 4, 5, dynamic/configurable
**Decision:** Fixed at 4.
**Rationale:** Gem uses 4 in production. It balances parallelism benefit against context fragmentation risk. Making it configurable adds complexity for marginal benefit — 4 is a reasonable default for all project sizes.

### T10: Critical Thinker Output Style (Decided: Structured Document)

**Options:** (A) Keep current conversational "ask why" style, (B) Rewrite as structured risk document
**Decision:** Structured risk document.
**Rationale:** The current style assumes interactive dialogue, but the agent runs non-interactively within the pipeline. A structured document with risk categories, likelihood, and impact is more actionable for the designer (if looped back) and more informative for the planner. The "ask why" style produces vague, unfocused output in automated contexts.

---

## Implementation Checklist & Deliverables

### Files to Create/Update

All files are created in `NewAgentsAndPrompts/`:

| #   | File                            | Action           | Key Sections                                                                                                                                                                 |
| --- | ------------------------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `orchestrator.agent.md`         | Rewrite          | Global Rules (TDD, concurrency, agent routing, approval gates), Steps 1.2a + 4a (approval gates), Step 5.2 (sub-wave splitting + agent routing), Operating Rules, Anti-Drift |
| 2   | `researcher.agent.md`           | Rewrite          | Retrieval Strategy, Confidence Metadata, Decision Log Read, Operating Rules, Anti-Drift                                                                                      |
| 3   | `spec.agent.md`                 | Rewrite          | Structured Edge Cases, Self-Verification Step, Operating Rules, Anti-Drift                                                                                                   |
| 4   | `designer.agent.md`             | Rewrite          | Security Considerations, Failure & Recovery, Self-Verification, Operating Rules, Anti-Drift                                                                                  |
| 5   | `critical-thinker.agent.md`     | **Full rewrite** | Completion Contract (bug fix), Output Spec (bug fix), Risk Categories, Structured Workflow, Operating Rules, Anti-Drift                                                      |
| 6   | `planner.agent.md`              | Rewrite          | Mode Detection, Task Size Limits, Pre-Mortem Analysis, Agent Field, Planning Principles, Operating Rules, Anti-Drift                                                         |
| 7   | `implementer.agent.md`          | **Full rewrite** | TDD Workflow (replaces old workflow), TDD Fallback, Security Rules, Code Quality Principles, Self-Reflection, Operating Rules, Anti-Drift                                    |
| 8   | `verifier.agent.md`             | Rewrite          | Role Redefinition (integration-level), Build System Detection (tech-agnostic), Read-Only Enforcement, Operating Rules, Anti-Drift                                            |
| 9   | `reviewer.agent.md`             | Rewrite          | Security Review, Review Depth Tiers, Read-Only Enforcement, Decision Log Write, Quality Standard, Self-Reflection, Operating Rules, Anti-Drift                               |
| 10  | `feature-workflow.prompt.md`    | Update           | Concurrency cap, Agent routing, APPROVAL_MODE, Variables section                                                                                                             |
| 11  | `documentation-writer.agent.md` | **Create new**   | Full agent: Role, Capabilities, Read-Only, Workflow, Completion Contract, Anti-Drift                                                                                         |

### Acceptance Criteria Mapping

| AC    | Description                                               | Verified By          |
| ----- | --------------------------------------------------------- | -------------------- |
| AC-1  | All 11 files exist and are non-empty                      | File existence check |
| AC-2  | Critical-thinker has completion contract + output spec    | TS-2                 |
| AC-3  | TDD cluster consistency (3 files agree)                   | TS-3                 |
| AC-4  | Concurrency cap in orchestrator + prompt                  | TS-4                 |
| AC-5  | Planner has pre-mortem + size limits + mode detection     | TS-5                 |
| AC-6  | Security thread across implementer + reviewer + designer  | TS-6                 |
| AC-7  | Verifier is technology-agnostic                           | TS-7                 |
| AC-8  | All 10 agents have anti-drift anchors                     | TS-8                 |
| AC-9  | All agents have DONE/ERROR contracts, no contradictions   | TS-9, TS-14          |
| AC-10 | Per-task agent routing in planner + orchestrator + prompt | TS-10                |
| AC-11 | Approval gates conditional on APPROVAL_MODE               | TS-11                |
| AC-12 | Documentation writer exists and is not in core pipeline   | TS-12                |

### Cluster Coordination Requirements

**Cluster A — TDD (MUST be implemented atomically across 3 files):**

- `orchestrator.agent.md` Global Rule 6
- `implementer.agent.md` TDD Workflow + removal of "Do NOT build/run tests"
- `verifier.agent.md` Role redefinition to integration-level

**Cluster B — Per-Task Agent Routing (MUST be implemented atomically across 3 files):**

- `orchestrator.agent.md` Global Rule 8 + Step 5.2 routing logic
- `planner.agent.md` `agent` field in task file requirements
- `feature-workflow.prompt.md` agent routing rule

**Cluster C — Approval Gates (MUST be implemented atomically across 2 files):**

- `orchestrator.agent.md` Global Rule 9 + Steps 1.2a, 4a
- `feature-workflow.prompt.md` APPROVAL_MODE variable + rule

### Implementer Notes

1. **Order of implementation:** Clusters A, B, C should each be implemented as atomic units — all files in a cluster must be written/updated together before moving to the next file. Recommended order: independent agents first (spec, designer, critical-thinker, researcher), then clustered agents (Cluster A: implementer+verifier+orchestrator, Cluster B: planner+orchestrator+prompt, Cluster C: orchestrator+prompt), then documentation-writer last.

2. **Cross-referencing:** When writing an agent, verify that all references to other agents' behaviors are consistent with the design for those agents. Key cross-references:
   - Orchestrator references to planner output format (waves, tasks)
   - Planner references to implementer's TDD capability
   - Verifier references to implementer's unit test responsibility
   - Planner task file format must match what orchestrator parses

3. **Shared Operating Rules:** Copy the Operating Rules block verbatim into each agent, changing only Rule 5 (tool preferences) per the table in Cross-Cutting Patterns.

4. **Anti-Drift Anchors:** Copy the exact anchor text from the table in Cross-Cutting Patterns. Do not paraphrase.

5. **Testing:** After implementing all files, run through the 15 test scenarios from `feature.md` (TS-1 through TS-15) as a verification checklist.

6. **Three-state completion:** When writing completion contracts, only critical-thinker, reviewer, and verifier use `NEEDS_REVISION:`. All other agents use only `DONE:`/`ERROR:`. Ensure the orchestrator's routing table matches each agent's contract.

---

## Review Response

This section documents all changes made to the design in response to the critical review (`design_critical_review.md`).

### Blocking Issues

| Issue   | Summary                                                                                                              | Resolution                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------- | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **B-1** | Three-state completion contract rejected on invalid grounds (migration cost cited despite zero-cost rewrite premise) | **Adopted three-state completion.** `NEEDS_REVISION:` added as a third state. Updated: Architecture Preserved list (item 4, 11), Pattern 3 (full three-state specification with routing table), all per-agent completion contracts for critical-thinker/reviewer/verifier, Agent Communication Contracts (3-column orchestrator expectations table), Tradeoff T2 (decision reversed with functional rationale). Only 3 of 11 agents actively use NEEDS_REVISION — complexity is contained. |
| **B-2** | APPROVAL_MODE implementation unverified against VS Code Copilot agent protocol                                       | **Marked as experimental with concrete fallback.** Global Rule 9 now includes: (a) explicit "⚠ Experimental (platform-dependent)" label, (b) concrete fallback behavior: if the runtime doesn't support interactive pausing, log a warning and proceed autonomously, (c) note that this feature requires validation before relying on it. The feature is retained because the implementation cost is low and the default behavior (autonomous) is unchanged.                               |

### Major Issues

| Issue   | Summary                                                                                                                    | Resolution                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **M-1** | `detailed thinking on` directive is unvalidated and may be harmful                                                         | **Labeled as experimental.** Pattern 4 renamed to include "(Experimental)" suffix. Added: explicit warning about model-dependency, concrete removal instructions (search for `<!-- experimental: model-dependent -->` comment), and rationale for inclusion despite uncertainty (negligible cost, near-zero risk).                                                                                                                                                                                                                    |
| **M-2** | TDD fallback detection is underspecified — no heuristics for detecting test framework availability                         | **Added concrete detection heuristics.** TDD Fallback section now includes: (a) explicit file patterns to scan per ecosystem (jest.config._, pytest.ini, _.Tests.csproj, etc.), (b) conditions that make a task non-testable (config-only, infrastructure setup, static assets), (c) procedure for tasks that CREATE the test framework. Also clarified that test lines do NOT count toward the 500-line task size limit (production code only), and test files do not count toward the 3-file limit.                                 |
| **M-3** | Retry policy interaction could cause exponential retries (agent 2× internal + orchestrator 1× = 6 total attempts)          | **Added retry bounds and scope clarification.** Operating Rules error handling (rule 2) now includes: (a) explicit "do NOT retry deterministic failures" instruction, (b) "Retry budget" paragraph explaining the composition of agent-level (3 attempts) × orchestrator-level (2 attempts) = 6 max, and noting this is bounded to transient issues only since deterministic failures aren't retried.                                                                                                                                 |
| **M-4** | Orchestrator canonical structure exception weakens the standard                                                            | **Formalized orchestrator extensions.** The canonical structure rules now explicitly state that the orchestrator's unique sections are "formally valid extensions" with justification (coordination role requires structured dispatch logic). Also added a general rule: "Other agents MAY add agent-specific sections between `## Workflow` and `## Completion Contract` as long as they don't duplicate or contradict canonical sections." This makes the standard honestly extensible rather than pretending it has no exceptions. |
| **M-5** | `decisions.md` lifecycle has unresolved gaps (creation, format, scope, retention)                                          | **Added Pattern 5: decisions.md Lifecycle.** New cross-cutting pattern specifying: scope (per-feature), creation rules (reviewer creates if not exists), format (structured entries with date/context/decision/rationale/scope/affected-components), writers (reviewer only, append-only), readers (researcher, critical-thinker), and retention policy (never deleted; project-wide decisions manually promoted by humans).                                                                                                          |
| **M-6** | Reviewer has no lightweight revision path — ERROR triggers full replan for minor fixes                                     | **Resolved by adopting three-state completion (B-1).** Reviewer now uses `NEEDS_REVISION:` for issues implementers can fix without a replan. The orchestrator routes findings directly to affected implementer(s) for a single lightweight fix pass (max 1 loop). If NEEDS_REVISION persists, escalates to planner. This eliminates the inefficiency of full replan cycles for one-line naming fixes.                                                                                                                                 |
| **M-7** | Anti-drift anchor adopted but XML tags rejected with inconsistent reasoning — both are unvalidated LLM behavior hypotheses | **Reframed T4 rationale.** Tradeoff T4 now honestly states this is a "pragmatic choice driven by concrete advantages" rather than claiming XML is "unvalidated" while anchors are validated. The real reasons are enumerated: human readability, GitHub/VS Code rendering, chatagent format consistency, standard tooling compatibility. The cost-benefit difference is also noted: anchors are 2-3 lines (minimal cost), XML tags would restructure every section (high cost).                                                       |

### Minor Issues

| Issue   | Summary                                                                          | Resolution                                                                                                                                                                                                                                                            |
| ------- | -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **m-1** | Verifier build system detection table incomplete                                 | **Added non-exhaustive note** listing additional systems (Deno, Bun, PHP, Elixir, Scala, Ruby, Swift) and instruction to "detect other build systems using similar heuristics."                                                                                       |
| **m-2** | Per-task agent routing doesn't define "recognized"                               | **Added valid_task_agents list** to orchestrator Global Rule 8: explicit list (`implementer`, `documentation-writer`), with fallback behavior for unrecognized values (log warning, default to implementer).                                                          |
| **m-3** | Implementer security rule scope conflicts with task size limits                  | **Clarified scope:** Security fixes within files already being modified are always in-scope regardless of count. Security fixes requiring files beyond the task's scope should be flagged as findings, not fixed.                                                     |
| **m-4** | Self-reflection mandatory for all tasks is disproportionate for low-effort tasks | **Made conditional on effort level.** Self-reflection step (TDD Workflow step 7) now only applies to Medium/High effort tasks. Low effort tasks (config changes, version bumps, single-line fixes) skip self-reflection.                                              |
| **m-5** | Reviewer security scan runs on ALL changed files regardless of count             | **Added heuristic pre-scan.** For large diffs (20+ changed files), the reviewer first runs a quick indicator check. If no high-signal patterns found, skip the deep scan and note "Security pre-scan: no indicators found." Full scan always runs for smaller diffs.  |
| **m-6** | Documentation writer missing delta-only parity check                             | **Added delta-only verification.** Documentation writer workflow step 5 now uses `get_changed_files` to verify parity against only changed source code, not the entire codebase.                                                                                      |
| **m-7** | Designer input list missing `design_critical_review.md` for loop-back            | **Added to designer Inputs.** `design_critical_review.md (if exists — read during revision cycle)` is now explicitly listed.                                                                                                                                          |
| **m-8** | Planner's circular dependency check is in pre-mortem but should be validation    | **Moved to new Plan Validation section.** Circular dependency check now runs BEFORE the pre-mortem analysis, as part of a new "Plan Validation" step that also includes task size validation and dependency existence checks. Pre-mortem is now purely risk analysis. |

### Recommendations Addressed

| #   | Recommendation                                     | Status                                                  |
| --- | -------------------------------------------------- | ------------------------------------------------------- |
| 1   | Re-evaluate three-state completion                 | **Done** — adopted (see B-1)                            |
| 2   | Validate APPROVAL_MODE                             | **Done** — marked experimental with fallback (see B-2)  |
| 3   | Specify TDD detection heuristics                   | **Done** — explicit file patterns added (see M-2)       |
| 4   | Clarify retry policy scope                         | **Done** — retry budget paragraph added (see M-3)       |
| 5   | Define `decisions.md` fully                        | **Done** — Pattern 5 added (see M-5)                    |
| 6   | Expand verifier build system table                 | **Done** — non-exhaustive note added (see m-1)          |
| 7   | Add `valid_task_agents` list                       | **Done** — explicit list in Global Rule 8 (see m-2)     |
| 8   | Reconcile security fix scope with task size limits | **Done** — clarified (see m-3)                          |
| 9   | Add `design_critical_review.md` to designer inputs | **Done** (see m-7)                                      |
| 10  | Move circular dependency check to validation       | **Done** — new Plan Validation section (see m-8)        |
| 11  | Make self-reflection conditional on effort         | **Done** (see m-4)                                      |
| 12  | Consider delta-only security scanning              | **Done** — heuristic pre-scan for large diffs (see m-5) |
| 13  | Add delta-only parity check to doc writer          | **Done** (see m-6)                                      |
| 14  | Reframe T4 rationale                               | **Done** — honest pragmatic rationale (see M-7)         |

### Questions Answered

| #   | Question                                                   | Answer                                                                                                                         |
| --- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Has `{{APPROVAL_MODE}}` been tested?                       | No — marked as experimental with concrete fallback. Feature is retained because default behavior is unchanged and cost is low. |
| 2   | Does `chatagent` support file includes?                    | No — verified as a constraint. This is why Operating Rules are duplicated in each file (Tradeoff T5, unchanged).               |
| 3   | How does the implementer detect "no test framework"?       | Explicit file pattern scanning per ecosystem (see TDD Fallback section).                                                       |
| 4   | Do test lines count toward the 500-line limit?             | No — production code only. Clarified in Task Size Limits.                                                                      |
| 5   | Who creates `decisions.md`?                                | The reviewer, on first write (see Pattern 5).                                                                                  |
| 6   | Why wasn't `design_critical_review.md` in designer inputs? | Oversight — now added.                                                                                                         |
| 7   | What is the valid set for the `agent` field?               | `implementer`, `documentation-writer` — explicit list in Global Rule 8.                                                        |
| 8   | Functional argument against three-state?                   | None sufficient — adopted three-state (see T2 revised rationale).                                                              |
