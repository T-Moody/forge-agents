# Research: Patterns

## Focus Area

**patterns** — Testing strategy, code conventions, developer experience patterns, error handling patterns, operational concerns, existing reusable patterns.

## Summary

The Forge codebase follows a highly consistent agent definition pattern with YAML frontmatter, standardized sections (Inputs, Outputs, Operating Rules, Workflow, Completion Contract, Anti-Drift Anchor), isolated memory files, and uniform error handling. There are no YAML/JSON structured output patterns currently in use by agents — all output is Markdown. The self-improvement system must follow these conventions to remain additive.

---

## Findings

### 1. Agent File Format Pattern

Every agent file follows the `chatagent` markdown format with consistent structure:

**YAML Frontmatter** (lines 1–4 of every `.agent.md` file):

```yaml
---
name: <agent-name>
description: "<one-line description>"
---
```

- `name` is always lowercase, hyphen-separated (e.g., `r-knowledge`, `ct-security`, `v-build`)
- `description` is always a quoted string

**Standard Sections** (in order):

1. **Title & Role Description** — `# <Agent Name> Agent Workflow` / `# <Agent Name> Agent`, followed by "You are the **<Agent Name>**." role statement and behavioral constraints (what the agent NEVER does)
2. **`<!-- experimental: model-dependent -->`** — HTML comment present in most agents
3. **Inputs** — Explicit list of files the agent reads, categorized as "primary", "selective", "conditional", or "STRICT"
4. **Outputs** — Explicit list of files the agent writes, including isolated memory file
5. **Operating Rules** — Numbered rules (always 6 rules: context-efficient reading, error handling, output discipline, file boundaries, tool preferences, memory-first reading)
6. **Workflow** — Numbered steps describing the agent's procedure
7. **Completion Contract** — Exact return format (`DONE:`, `NEEDS_REVISION:`, `ERROR:`)
8. **Anti-Drift Anchor** — Final section reinforcing role constraints

**File references:**

- All 19 active agents follow this pattern: [orchestrator.agent.md](.github/agents/orchestrator.agent.md), [researcher.agent.md](.github/agents/researcher.agent.md), [spec.agent.md](.github/agents/spec.agent.md), [designer.agent.md](.github/agents/designer.agent.md), [planner.agent.md](.github/agents/planner.agent.md), [implementer.agent.md](.github/agents/implementer.agent.md), [documentation-writer.agent.md](.github/agents/documentation-writer.agent.md), all CT agents, all V agents, all R agents
- The deprecated [critical-thinker.agent.md](.github/agents/critical-thinker.agent.md) has a deprecation notice at the top

### 2. Isolated Memory File Format Pattern

Every agent writes an isolated memory file to `memory/<agent-name>.mem.md` with this structure:

```markdown
# Memory: <agent-name>

## Status

<DONE|ERROR>: <one-line summary>

## Key Findings

- <finding 1>
- <finding 2>
- ... (≤5 bullets)

## Highest Severity

<severity-value or N/A>

## Decisions Made

- <decision 1> (≤2 sentences)
<!-- Omit section if no decisions -->

## Artifact Index

- <file-path> — §<Section> (brief relevance note)
```

**Variations by agent category:**

- **Core pipeline agents** (spec, designer, planner, researcher): `Highest Severity: N/A`
- **CT sub-agents**: `Highest Severity: Critical/High/Medium/Low`
- **V sub-agents**: Status can be `DONE/NEEDS_REVISION/ERROR`; severity as PASS/FAIL
- **R sub-agents**: `Highest Severity: Blocker/Major/Minor` (canonical R taxonomy)
- **Implementer**: `Highest Severity: N/A`; filename includes task ID: `implementer-<task-id>.mem.md`
- **Documentation-writer**: filename includes task ID: `documentation-writer-<task-id>.mem.md`

**File references:**

- Explicit template in [implementer.agent.md](.github/agents/implementer.agent.md) lines 122–140
- Explicit template in [r-knowledge.agent.md](.github/agents/r-knowledge.agent.md) lines 289–310

### 3. Self-Verification Patterns Across Agents

Multiple agents implement self-verification before returning, but with different approaches:

| Agent             | Self-Verification Type                            | What It Checks                                                                                                                                                  |
| ----------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **spec**          | "Self-verification step" (step 6)                 | Acceptance criteria testable, functional requirements have criteria, edge cases cover failure modes, no contradictions                                          |
| **designer**      | "Self-verification step" (step 12)                | All requirements addressed, every AC has implementation path, security considered, failure modes identified                                                     |
| **implementer**   | "Self-Reflection" (step 7)                        | All task requirements addressed, tests cover ACs, no omissions, code follows conventions, "Would a senior engineer approve?" — **skipped for Low effort tasks** |
| **CT sub-agents** | "Self-verification" (final workflow step)         | Each finding grounded in specific technical details, vague findings strengthened with concrete references                                                       |
| **r-quality**     | "Self-Reflection" (step 9)                        | Review findings grounded, not stated but implied quality gate                                                                                                   |
| **r-security**    | "Self-Reflection" (step 9)                        | Review findings grounded                                                                                                                                        |
| **orchestrator**  | "Self-verification" (after each cluster decision) | Logs decision values and results to `memory.md` Recent Updates                                                                                                  |

**Pattern:** Self-verification is always the penultimate step, before writing isolated memory. The check is always a list of verifiable conditions. Issues found are fixed in-place before returning.

### 4. Knowledge-Suggestions.md Pattern (R-Knowledge)

The `knowledge-suggestions.md` file follows a strict structured format defined in [r-knowledge.agent.md](.github/agents/r-knowledge.agent.md) lines 202–250:

````markdown
# Knowledge Evolution Suggestions

> **WARNING:** These are proposals only. Do NOT apply without human review.
> Generated by R-Knowledge agent during pipeline execution.

## Suggestions

### 1. [Category] Title

- **Target file:** Path to file that would be modified
- **Change type:** Add / Update / Remove
- **Rationale:** Why
- **Diff:**
  ```diff
  - old content
  + new content
  ```
````

- **Risk assessment:** Low / Medium / High

## Rejected Suggestions

<!-- Suggestions filtered by KE-SAFE-6 -->

````

**Key constraints:**
- Each suggestion MUST include: What, Why, File, Diff (KE-SAFE-2)
- Each suggestion MUST be categorized as: `instruction-update`, `skill-update`, `pattern-capture`, `workflow-improvement` (KE-SAFE-3)
- Numbered sequentially
- WARNING banner at top is mandatory (KE-SAFE-5)
- Rejected suggestions section for safety-filtered items

### 5. Decisions.md Log Pattern

The `decisions.md` file is an **append-only** architectural decision log:

```markdown
# Architectural Decision Log

## [YYYY-MM-DD] <Decision Title>

- **Context:** Why this decision was needed.
- **Decision:** What was decided.
- **Rationale:** Why this option was chosen over alternatives.
- **Scope:** Feature-specific / Project-wide.
- **Affected components:** List of files, modules, or packages impacted.
````

**Key constraints (KE-SAFE-7):**

- Only R-Knowledge writes to this file
- Existing entries MUST NEVER be modified or deleted — append only
- After writing, R-Knowledge verifies all pre-existing entries remain unchanged
- If no decisions identified, the file is not created (no empty file)
- File may or may not exist; create if needed

**File reference:** [r-knowledge.agent.md](.github/agents/r-knowledge.agent.md) lines 248–270

### 6. Error Handling Patterns

**Identical across all agents** (Operating Rule 2). Every agent defines:

```markdown
2. **Error handling:**
   - _Transient errors_ (network timeout, tool unavailable, rate limit): Retry up to 2 times with brief delay. **Do NOT retry if the failure is deterministic.**
   - _Persistent errors_ (file not found, permission denied): Include in output and continue. Do not retry.
   - _Security issues_ (secrets in code, vulnerable dependencies): Flag immediately with `severity: critical`.
   - _Missing context_ (referenced file doesn't exist, dependency not installed): Note the gap and proceed with available information.
   - **Retry budget:** 3 internal attempts (1 + 2 retries) × 2 orchestrator attempts = 6 total tool calls. Agents MUST NOT retry deterministic failures.
```

**Exception:** R-Security flags security issues with `severity: blocker` instead of `severity: critical`.

**Orchestrator-level retry pattern:**

- Global Rule 4: Automatically retry failed subagent invocations (`ERROR:`) once before reporting failure
- Do NOT retry `NEEDS_REVISION:` — route to the appropriate agent instead
- R-Knowledge is non-blocking: its ERROR does not block the pipeline
- R-Security is blocking: its Blocker finding halts the pipeline

### 7. Safety Constraint Filter Pattern (KE-SAFE-\*)

R-Knowledge defines a numbered safety constraint system (KE-SAFE-1 through KE-SAFE-7):

| ID        | Name                      | Description                                            |
| --------- | ------------------------- | ------------------------------------------------------ |
| KE-SAFE-1 | File Boundaries           | R-Knowledge writes ONLY to 4 listed files              |
| KE-SAFE-2 | Suggestion Completeness   | Every suggestion must include What/Why/File/Diff       |
| KE-SAFE-3 | Suggestion Categorization | Must use one of 4 categories                           |
| KE-SAFE-5 | Human Review Warning      | WARNING banner at top of suggestions file              |
| KE-SAFE-6 | Safety Constraint Filter  | MUST NOT suggest removing/weakening safety constraints |
| KE-SAFE-7 | Append-Only Decision Log  | Never modify existing decision entries                 |

**KE-SAFE-6 rejection criteria** (from [r-knowledge.agent.md](.github/agents/r-knowledge.agent.md) lines 61–75):

- Removing error handling or try/catch blocks
- Weakening input validation
- Removing or reducing security scanning steps
- Removing anti-drift anchors or file boundary rules
- Weakening completion contract requirements
- Removing read-only enforcement rules

Rejected suggestions are moved to the "Rejected Suggestions" section with reason "safety constraint — cannot weaken."

### 8. File Boundary Enforcement Patterns

Every agent has Operating Rule 4:

> "Only write to files listed in the Outputs section. Never modify files outside your output scope."

**Additionally, some agents have explicit file boundary sections:**

- **R-Knowledge** has a dedicated `## File Boundaries (KE-SAFE-1)` section listing exactly 4 files it may write
- **R-Knowledge** additionally states: "NEVER modify any file in `NewAgentsAndPrompts/` or `.github/agents/`"
- **Orchestrator** Global Rule 12: "The orchestrator is the sole writer to shared `memory.md`"

### 9. Read-Only Enforcement Patterns

Agents that must not modify source code have explicit `## Read-Only Enforcement` sections:

| Agent                    | Read-Only Section Present | What It Says                                                                         |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------ |
| **documentation-writer** | Yes                       | "MUST NOT modify source code, test files, or configuration files"                    |
| **r-quality**            | Yes                       | "strictly read-only with respect to the codebase"                                    |
| **r-security**           | Yes                       | "strictly read-only with respect to the codebase"                                    |
| **r-testing**            | Yes                       | "MUST NOT modify source code, test files, configuration files, or any project files" |
| **v-build**              | In intro                  | "NEVER modify source code, test files, configuration files, or any project files"    |
| **v-tests**              | In intro                  | "NEVER modify source code, test files, configuration files, or any project files"    |
| **v-tasks**              | Yes                       | "strictly read-only with respect to the codebase"                                    |
| **v-feature**            | Yes                       | "strictly read-only with respect to the codebase"                                    |

**Pattern:** Read-only agents state the constraint in both:

1. The introductory role description ("You NEVER modify...")
2. A dedicated `## Read-Only Enforcement` section listing exactly which files may be written

### 10. Naming Conventions

**Agent files:**

- Pattern: `<name>.agent.md`
- Cluster agents prefixed: `ct-`, `v-`, `r-`
- Names are lowercase, hyphen-separated
- Located in `.github/agents/`

**Memory files:**

- Pattern: `memory/<agent-name>.mem.md`
- Researcher: `memory/researcher-<focus-area>.mem.md`
- Implementer: `memory/implementer-<task-id>.mem.md`
- Documentation-writer: `memory/documentation-writer-<task-id>.mem.md`

**Artifact output files:**

- Cluster-specific directories: `ct-review/`, `verification/`, `review/`
- Research outputs: `research/<focus-area>.md`
- Task files: `tasks/NN-description.md` (numerically prefixed)

**Prompt files:**

- Pattern: `<name>.prompt.md`
- Located in `.github/prompts/`
- YAML frontmatter with `name`, `agent` fields

**Documentation directory structure:**

```
docs/feature/<feature-slug>/
├── initial-request.md
├── memory.md
├── memory/<agent-name>.mem.md
├── research/*.md
├── feature.md
├── design.md
├── ct-review/ct-*.md
├── plan.md
├── tasks/*.md
├── verification/v-*.md
├── review/r-*.md, knowledge-suggestions.md
├── decisions.md
```

### 11. Structured Data Usage (Markdown Tables and Code Blocks)

**Markdown tables** are used extensively throughout agent definitions for:

- Build system detection (v-build: file → build system → commands)
- Review depth tiers (r-quality, r-security: trigger → scope)
- Task size limits (planner: limit → maximum → rationale)
- Pre-mortem analysis (planner: task → scenario → likelihood → impact → mitigation)
- Orchestrator expectations per agent (agent → DONE/NEEDS_REVISION/ERROR behavior)
- V cluster decision table (v-build status × v-tests × v-tasks × v-feature → result)
- Memory lifecycle actions (action → when → what)
- NEEDS_REVISION routing table

**Markdown code blocks** are used for:

- Template content (memory.md template, decisions.md header, suggestions format)
- Report formats (v-build.md, v-tests.md output formats)
- Execution wave format in plan.md

**No YAML or JSON structured outputs** exist in any current agent output. All agent communication is via Markdown files. The initial-request.md for this feature is the first place that proposes YAML structured output (for `artifact_evaluation` and `post_mortem_report`).

### 12. YAML/Structured Output Patterns in the Codebase

**Currently: None.** No agent produces YAML or JSON output. All inter-agent communication happens via:

- Markdown files with consistent heading structure
- Markdown tables for tabular data
- Markdown code blocks for templates and formats
- Isolated memory `.mem.md` files with standard Markdown sections

**Relevant observation:** The initial-request.md proposes YAML-structured evaluation blocks. This would be the **first** instance of structured YAML output within agent artifacts. The pattern would need to be compatible with the existing Markdown-first ecosystem.

### 13. Completion Contract Patterns

Three distinct completion contract patterns exist across agents:

| Pattern         | Used By                                                                                     | Format                                                                                                      |
| --------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Two-state**   | researcher, spec, designer, planner, implementer, documentation-writer                      | `DONE: <summary>` or `ERROR: <reason>`                                                                      |
| **Three-state** | V sub-agents (v-tests, v-tasks, v-feature), R sub-agents (r-quality, r-security, r-testing) | `DONE:`, `NEEDS_REVISION:`, or `ERROR:`                                                                     |
| **Custom**      | r-knowledge                                                                                 | `DONE: knowledge analysis complete — <N> suggestions, <M> decisions logged` or `ERROR:` (no NEEDS_REVISION) |
| **Custom**      | orchestrator                                                                                | Workflow completes when R cluster DONE; otherwise `ERROR: <summary>`                                        |
| **Custom**      | planner                                                                                     | `DONE: <N> tasks created, <M> waves`                                                                        |

### 14. Orchestrator Tool Restriction Pattern

The feature request includes restricting the orchestrator to only `[agent, agent/runSubagent, memory]` tools. Currently, the orchestrator agent's Operating Rules say:

> "Use `runSubagent` for all delegation. Never invoke tools that modify code or files directly."

But it does currently reference direct file operations for `memory.md` management (initialize, merge, prune). The tool restriction would require delegating those operations.

### 15. Non-Blocking Agent Pattern

R-Knowledge establishes a pattern for non-blocking agents:

- "R-Knowledge is non-blocking: your ERROR does not block the pipeline"
- The orchestrator proceeds without knowledge analysis if R-Knowledge fails
- R-Knowledge does NOT use `NEEDS_REVISION`
- This pattern could be reused for the PostMortem agent

### 16. Cluster Dispatch Patterns (Reusable)

Three dispatch patterns are documented in [dispatch-patterns.md](.github/agents/dispatch-patterns.md):

| Pattern                            | Description                                                  | Used By                         |
| ---------------------------------- | ------------------------------------------------------------ | ------------------------------- |
| **A — Fully Parallel**             | Dispatch ≤4 sub-agents concurrently; wait; retry errors once | CT cluster, R cluster, Research |
| **B — Sequential Gate + Parallel** | Gate agent first; on DONE dispatch remaining in parallel     | V cluster                       |
| **C — Replan Loop**                | Pattern B wrapped in iteration loop (max 3)                  | V-Replan cycle                  |

All patterns follow the memory-first architecture: each sub-agent writes isolated memory, orchestrator reads for routing.

---

## File References

| File/Folder                                                      | Relevance                                                                            |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `.github/agents/*.agent.md` (all 19 active files + 1 deprecated) | Agent file format pattern, operating rules, completion contracts, anti-drift anchors |
| `.github/agents/dispatch-patterns.md`                            | Reusable cluster dispatch patterns A, B, C                                           |
| `.github/agents/r-knowledge.agent.md`                            | KE-SAFE-\* safety constraints, knowledge-suggestions pattern, decisions.md pattern   |
| `.github/agents/implementer.agent.md`                            | Self-reflection pattern, TDD workflow, memory template                               |
| `.github/agents/spec.agent.md`                                   | Self-verification pattern                                                            |
| `.github/agents/designer.agent.md`                               | Self-verification pattern                                                            |
| `.github/agents/orchestrator.agent.md`                           | Cluster decision flows, memory lifecycle, tool restriction context                   |
| `.github/agents/v-build.agent.md`                                | Read-only enforcement, sequential gate pattern                                       |
| `.github/agents/r-security.agent.md`                             | Pipeline blocker override pattern, severity vocabulary                               |
| `.github/prompts/feature-workflow.prompt.md`                     | Prompt file format pattern, variable declaration                                     |
| `README.md`                                                      | Architecture overview, design decisions, project layout                              |

---

## Assumptions & Limitations

- **Assumption:** The self-improvement system's new artifacts (evaluation YAML blocks, telemetry logs, post-mortem reports) should follow existing Markdown-first conventions, possibly embedding YAML within Markdown code blocks rather than creating standalone YAML files.
- **Assumption:** The PostMortem agent should follow the same agent file format pattern as all existing agents.
- **Assumption:** No test framework or test infrastructure exists in this repository (it is a documentation/prompt-only repository with no source code to test).
- **Limitation:** The repository contains no `.github/instructions/` directory yet, so instruction-level patterns could not be examined.
- **Limitation:** No existing `decisions.md` was found in the docs structure, so the append-only pattern was inferred from the R-Knowledge agent definition only.

---

## Open Questions

1. **YAML within Markdown vs. standalone YAML:** The initial-request proposes YAML-structured evaluations. Should these be embedded in Markdown (code blocks) consistent with existing patterns, or standalone `.yaml` files (a new pattern)?
2. **Evaluation storage location:** Should evaluations live alongside the evaluated artifact, in a dedicated `/artifact-evaluations/` directory, or within the existing `review/` directory?
3. **PostMortem agent blocker status:** Should the PostMortem agent be blocking (like R-Security) or non-blocking (like R-Knowledge)? The initial-request doesn't specify.
4. **Orchestrator tool restriction scope:** The orchestrator currently directly manages `memory.md` (init, merge, prune). If restricted to only `[agent, agent/runSubagent, memory]`, how are these operations delegated?

---

## Research Metadata

- **confidence_level:** high — All 19 active agent files and supporting documentation were examined in detail
- **coverage_estimate:** ~95% of relevant codebase examined — all agent definitions, dispatch patterns, prompt files, and README reviewed
- **gaps:** No `.github/instructions/` directory found to examine instruction patterns. No existing `decisions.md` instance found. The ct-aggregator, v-aggregator, and r-aggregator agent files referenced in README were not found in the `.github/agents/` directory (may be deprecated/removed or referenced for v1 compatibility).
