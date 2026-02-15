# Architecture Research — Forge vs. Gem Team

**Focus Area:** architecture  
**Summary:** Gem Team employs a significantly more structured agent definition architecture with XML-like semantic tags, shared cross-agent operating rules, anti-drift anchoring, three-state completion contracts, and explicit tool/context management patterns — most of which Forge lacks entirely and the existing comparison documents failed to identify.

---

## 1. Agent Definition Structure Patterns

### Gem Team Structure

Every Gem agent follows an identical structural template using XML-like semantic tags inside a markdown chatagent file:

```
---
name: gem-<role>
description: "<one-line>"
disable-model-invocation: true|false
user-invocable: true
---

<agent>
detailed thinking on

<role>      ... one-line role identity ...        </role>
<expertise> ... comma-separated skill areas ...   </expertise>
<workflow>  ... ordered steps with verbs ...      </workflow>
<operating_rules> ... shared + agent-specific ... </operating_rules>
<final_anchor> ... behavioral constraints ...     </final_anchor>
</agent>
```

Additional agent-specific tags appear as needed:

- `<valid_subagents>` — orchestrator only; explicit allowlist of delegatable agents
- `<task_size_limits>` — planner only; hard limits on task scope
- `<plan_format_guide>` — planner only; full YAML schema for plan output
- `<research_format_guide>` — researcher only; full YAML schema for findings
- `<review_criteria>` — reviewer only; tiered depth thresholds
- `<approval_gates>` — devops only; security and deployment approval conditions
- `<mission>` — chrome-tester only; concise mission statement

### Forge Structure

Forge agents use plain markdown with YAML frontmatter:

````
```chatagent
---
name: <role>
description: "<one-line>"
---

# <Agent> Workflow

## Inputs
## Outputs
## Workflow
## Rules
## Completion Contract
## <output>.md Contents
````

### Architectural Differences

| Dimension                 | Forge                                   | Gem Team                                                               |
| ------------------------- | --------------------------------------- | ---------------------------------------------------------------------- |
| **Structural formalism**  | Free-form markdown headings             | XML-like semantic tags with consistent ordering                        |
| **Role identity**         | Embedded in prose                       | Explicit `<role>` tag                                                  |
| **Skills declaration**    | Not declared                            | Explicit `<expertise>` tag                                             |
| **Frontmatter fields**    | `name`, `description`                   | `name`, `description`, `disable-model-invocation`, `user-invocable`    |
| **Output schemas**        | Prose descriptions of expected contents | Embedded YAML schema guides (plan_format_guide, research_format_guide) |
| **Anti-drift mechanism**  | None                                    | `<final_anchor>` tag at end of every agent                             |
| **Agent-specific config** | Inline markdown sections                | Dedicated XML tags per concern                                         |

### Analysis — What This Means for Forge

Forge's free-form markdown is more human-readable but less parse-friendly. The lack of a consistent template means agents vary in how they declare inputs, outputs, workflow steps, and constraints. Gem's XML tags serve as **semantic boundaries** that may help LLMs distinguish between different instruction categories (role vs. workflow vs. rules), potentially reducing prompt ambiguity.

The embedded YAML schemas in Gem (plan_format_guide, research_format_guide) are particularly notable — they provide the receiving agent with the exact output structure expected, functioning as a **contract-by-schema** pattern. Forge relies on prose descriptions of expected contents, which are less precise.

---

## 2. Prompt Engineering Patterns

### 2.1 Detailed Thinking Mode

**Every** Gem agent begins with `detailed thinking on` immediately after the `<agent>` tag. This directive activates extended chain-of-thought reasoning in the underlying model.

**Forge agents have no equivalent.** No Forge agent includes any thinking mode directive.

**Not mentioned in existing comparison documents.**

### 2.2 Communication Rules (Shared Across ALL Gem Agents)

Every Gem agent contains this identical block in `<operating_rules>`:

> "Communication: Output ONLY the requested deliverable. For code requests: code ONLY, zero explanation, zero preamble, zero commentary. For questions: direct answer in ≤3 sentences. Never explain your process unless explicitly asked 'explain how'."

**Forge agents have no communication style directives.** Agent output format is uncontrolled — agents may produce verbose explanations, preamble, or commentary at their discretion.

**Not analyzed in existing comparison documents.**

### 2.3 Context-Efficient File Reading Rules

Every Gem agent contains:

> "Context-efficient file reading: prefer semantic search, file outlines, and targeted line-range reads; limit to 200 lines per read"

This is a **context window management strategy** — preventing agents from consuming excessive context with large file reads that crowd out reasoning space.

**Forge agents have no file reading guidelines.** Agents may read entire files of arbitrary size.

**Not mentioned in existing comparison documents.**

### 2.4 Batching and Tool Preference Rules

Every Gem agent contains:

> "Built-in preferred; batch independent calls"

> "Prefer multi_replace_string_in_file for file edits (batch for efficiency)"

These rules optimize for fewer tool invocations and faster execution.

**Forge agents have no tool usage optimization rules.**

**Not mentioned in existing comparison documents.**

### 2.5 Tool Activation Patterns

Gem agents declare which tool categories to activate before use:

- Researcher: `activate_website_crawling_and_mapping_tools`, `activate_research_and_information_gathering_tools`
- Implementer/Reviewer/DevOps/Doc-Writer: `activate_vs_code_interaction`
- Chrome Tester: `activate_web_interaction`

These appear to be environment-specific (possibly MCP tool activation), but architecturally they represent **explicit tool registration** — ensuring agents only access the tools they need.

**Forge agents have no tool activation or registration pattern.**

**Mentioned briefly in existing comparison but not analyzed as an architectural pattern.**

### 2.6 Quality Bar Phrasing

Gem reviewer uses a concrete quality heuristic:

> "Quality Bar: 'Would a staff engineer approve this?'"

This provides a mental model for the agent to calibrate output quality. Forge has no equivalent calibration statement.

**Not mentioned in existing comparison documents.**

---

## 3. Agent Isolation & Boundaries

### Gem Team Isolation Mechanisms

Gem uses **five distinct isolation patterns**:

1. **`<final_anchor>` tag** — Every agent ends with a behavioral anchor that repeats core constraints and includes "stay as [role]" to prevent mode switching. Examples:
   - Researcher: `"stay as researcher"`
   - Implementer: `"stay as implementer"`
   - Orchestrator: `"ONLY coordinate via runSubagent - never execute directly"`

   This exploits the **recency bias** in LLMs — instructions at the end of the prompt carry disproportionate weight. It acts as an anti-drift mechanism.

2. **`disable-model-invocation: true`** — Applied to the orchestrator only. This is a platform-level mechanism that prevents the orchestrator from executing code, even if prompted to do so.

3. **`<valid_subagents>` tag** — Explicit allowlist of agents the orchestrator can invoke. Prevents routing errors.

4. **Explicit negative rules per agent:**
   - Researcher: "NEVER create plan.yaml or tasks", "NEVER invoke other agents", "NEVER pause for user feedback"
   - Reviewer: "read-only; never modify code"
   - Doc Writer: "docs-only: never modify source code", "Never document non-existent code"
   - Implementer: "Never bypass linting/formatting", "Adhere to tech_stack; no unapproved libraries"

5. **"Stay as [role]" in operating_rules** — Separate from the final_anchor, some agents also include "Stay as orchestrator, no mode switching" in their operating rules.

### Forge Isolation Mechanisms

Forge uses **two isolation patterns**:

1. **Explicit input restrictions** — The implementer declares `You MUST NOT read: plan.md`. This is the only agent with a negative input restriction.

2. **Role-level prohibitions** — The orchestrator says "Never modify code or docs directly". The implementer says "Do NOT build the project", "Do NOT run tests". The verifier does NOT explicitly say "don't modify code" (a gap).

### Gaps in Forge

- **No anti-drift mechanism** — No final anchor or repeated constraint block. Long agent conversations may drift from instructions.
- **No explicit role anchoring** — No "stay as [role]" statement to prevent agents from assuming other roles.
- **No valid_subagents restriction** — The orchestrator doesn't declare which agents it can call.
- **Inconsistent negative boundaries** — Only the implementer has explicit "MUST NOT read" constraints. Other agents don't declare what they should NOT access.
- **No `disable-model-invocation` equivalent** — The orchestrator relies on prose instructions rather than platform-level enforcement.

**The `<final_anchor>` pattern is not mentioned in either existing comparison document. This is a significant omission — it's a core architectural pattern in Gem Team.**

---

## 4. State Management Architecture

### Gem Team: YAML State File with In-Place Mutation

- Central state: `docs/plan/{plan_id}/plan.yaml`
- Plan ID serves as a **namespace key** for all artifacts: research findings, plans, evidence
- State transitions are tracked in-place: `pending → in_progress → completed → failed → blocked`
- The orchestrator mutates `plan.yaml` directly: "Update status to `in_progress`" before dispatching, then updates status based on result
- `manage_todos` integration — orchestrator updates a todo list alongside plan.yaml
- `plan_id` is generated with "unique identifier name and date" — acts as both namespace and audit key
- Mode detection enables three plan lifecycle states: `initial | replan | extension`

### Forge: Immutable Markdown Artifacts

- Central namespace: `docs/feature/<feature-slug>/`
- Plan is `plan.md` (markdown) — written once, modified only during re-planning
- Individual task files in `tasks/*.md` — implementers mark acceptance criteria as completed
- No explicit status tracking — status is inferred from orchestrator flow (which wave is current)
- No artifact-level versioning or history
- Re-planning creates/updates task files but doesn't track what changed from the original plan

### Architectural Differences (Deeper Than Existing Comparison)

The existing comparison notes the Markdown vs. YAML distinction but misses these deeper implications:

| Dimension            | Forge                                 | Gem Team                                                                  |
| -------------------- | ------------------------------------- | ------------------------------------------------------------------------- |
| **State location**   | Distributed across multiple files     | Centralized in single plan.yaml                                           |
| **State mutation**   | Mostly immutable; rewrite on re-plan  | In-place field updates per task                                           |
| **Status tracking**  | Implicit (orchestrator flow position) | Explicit (task status field in YAML)                                      |
| **Plan lifecycle**   | No mode concept; always re-creates    | Three modes: initial, replan, extension                                   |
| **Namespace scheme** | `<feature-slug>`                      | `{plan_id}` with date                                                     |
| **Evidence storage** | No concept                            | `docs/plan/{plan_id}/evidence/{task_id}/` with screenshots, logs, network |
| **Resumability**     | Cannot resume mid-pipeline            | Can resume from any task state (pending tasks remain)                     |

The resumability gap is significant: if Forge is interrupted mid-pipeline, the orchestrator must restart from scratch because there's no stateful record of which tasks completed. Gem's `plan.yaml` with explicit task statuses enables recovery.

**Evidence storage is not mentioned in either existing comparison document. Gem's `docs/plan/{plan_id}/evidence/{task_id}/` structure (with subfolders for screenshots, logs, network) provides structured audit artifacts beyond what Forge's markdown files capture.**

---

## 5. Error Handling & Resilience Patterns

### Gem Team: Categorized Error Handling

Every Gem agent includes error handling rules in `<operating_rules>` categorized by type:

- **Transient errors → handle** (retry, fallback)
- **Persistent errors → escalate** (return failed status)
- **Security issues → immediate action** (fix or halt)
- **Missing context → blocked** (not failed)

Additional agent-specific error patterns:

- Implementer: "Security issues → fix immediately or escalate", "Test failures → fix all or escalate", "Vulnerabilities → fix before handoff"
- DevOps: "Plaintext secrets → halt and abort"
- Planner: "Missing research → reject", "Circular deps → halt", "Security → halt"
- Reviewer: "Security issues → must fail", "Missing context → blocked", "Invalid handoff → blocked"

### Forge: Ad-Hoc Error Handling

- Orchestrator: Retry researcher once on ERROR; max 3 verification loops; loop back to planning on review ERROR
- All other agents: No explicit error handling rules — agents report DONE or ERROR with no guidance on error classification or handling strategy
- No distinction between transient and persistent errors
- No security-specific error handling

### Three-State vs. Two-State Returns

| Framework | Return States                         | Implications                                                                               |
| --------- | ------------------------------------- | ------------------------------------------------------------------------------------------ |
| Gem       | `success`, `failed`, `needs_revision` | Orchestrator can differentiate between terminal failure and recoverable issues             |
| Forge     | `DONE`, `ERROR`                       | All non-success outcomes are treated identically; orchestrator cannot distinguish severity |

Gem's `needs_revision` state allows the orchestrator to route work back to the **same agent** for correction without a full re-plan cycle. Forge's binary model requires either acceptance or a full retry/re-plan.

**The three-state return model is not analyzed as an architectural pattern in either existing comparison document. The existing comparison mentions it in passing (JSON handoff) but doesn't analyze the implications of `needs_revision` as a distinct state.**

---

## 6. Common Operating Rules (Cross-Cutting Concerns)

### Gem Team: Shared Base Rules

Gem implements what is effectively a **"base class"** for agent operating rules. The following rules appear **identically verbatim** across ALL 8 Gem agents:

1. **Context-efficient file reading** — "prefer semantic search, file outlines, and targeted line-range reads; limit to 200 lines per read"
2. **Tool preference** — "Built-in preferred; batch independent calls"
3. **File editing** — "Prefer multi_replace_string_in_file for file edits (batch for efficiency)"
4. **Communication style** — "Output ONLY the requested deliverable. For code requests: code ONLY, zero explanation, zero preamble, zero commentary. For questions: direct answer in ≤3 sentences. Never explain your process unless explicitly asked 'explain how'."
5. **Error categories** — transient→handle, persistent→escalate pattern
6. **Tool activation** — agent-specific but always declared

Additional shared patterns (present in most agents): 7. **Memory integration** — "Memory CREATE: Include citations (file:line) and follow /memories/memory-system-patterns.md format", "Memory UPDATE: Refresh timestamp when verifying existing memories" 8. **Detailed thinking on** — present in ALL agents as first directive 9. **Stay as [role]** — present in final_anchor of ALL agents 10. **JSON return format** — all agents return `{"status": "...", "task_id|plan_id": "...", "summary": "..."}`

### Forge: No Shared Rules

Forge agents are independently authored with no shared operating rules:

- No common communication style
- No common file reading strategy
- No common tool usage preferences
- No common error handling pattern
- No common return format (beyond the DONE/ERROR contract)

Each Forge agent defines its own rules (or lacks them entirely). The researcher, spec, designer, and critical-thinker have minimal rule sets. The orchestrator, planner, implementer, verifier, and reviewer have more detailed rules but they are not consistent across agents.

### Implications

Without shared operating rules, Forge agents may:

- Read files inconsistently (some reading entire files, some using targeted reads)
- Produce inconsistent output verbosity (some agents producing preamble, others not)
- Handle errors differently or not at all
- Use tools inefficiently (sequential single-edit calls vs. batched multi-edit)

**The concept of shared operating rules as a cross-cutting concern is not analyzed in either existing comparison document. The comparison mentions specific differences (e.g., hybrid retrieval) but doesn't identify the pattern of shared rules as an architectural feature.**

---

## 7. Agent Completion Contracts

### Forge

```
DONE: <summary>
ERROR: <reason>
```

- Text-based, human-readable
- Two states only
- Summary is free-form text
- No task/plan identifier in the return — the orchestrator must track which agent produced which result
- Different agents have slightly different DONE formats:
  - Researcher: `DONE: <focus-area> — <summary>` or `DONE: synthesis — <summary>`
  - Planner: `DONE: <number of tasks created>, <number of waves>`
  - Verifier: `DONE: verification passed (<N> tasks verified, <M> tests passing)`
  - Implementer: `DONE: <task-id>`
  - Reviewer: `DONE: review complete`

### Gem Team

```json
{
  "status": "success|failed|needs_revision",
  "plan_id": "[plan_id]",
  "summary": "[brief summary]"
}
```

or

```json
{
  "status": "success|failed|needs_revision",
  "task_id": "[task_id]",
  "summary": "[brief summary]"
}
```

- JSON-structured, machine-parseable
- Three states (success, failed, needs_revision)
- Always includes a namespace identifier (plan_id or task_id)
- Agent-specific fields (e.g., reviewer adds review_status and review_depth)

### Architectural Comparison

| Dimension                | Forge                                | Gem Team                              |
| ------------------------ | ------------------------------------ | ------------------------------------- |
| **Format**               | Free-form text                       | Structured JSON                       |
| **States**               | 2 (DONE, ERROR)                      | 3 (success, failed, needs_revision)   |
| **Identifier**           | Varies by agent; sometimes included  | Always included (plan_id or task_id)  |
| **Machine-parseability** | Low (requires text parsing)          | High (JSON.parse)                     |
| **Extensibility**        | Add more text                        | Add more JSON fields                  |
| **Human-readability**    | High                                 | Medium                                |
| **Consistency**          | Each agent has different DONE format | All agents follow same JSON structure |

---

## 8. Architectural Patterns Not Covered in Existing Comparison Documents

The existing comparison documents (`comparison-forge-vs-gem-team.md` and `optimization-from-gem-team.md`) miss several significant architectural patterns:

### 8.1 `<final_anchor>` Anti-Drift Pattern

Every Gem agent ends with a `<final_anchor>` tag that **repeats** the core behavioral constraints. This is a deliberate prompt engineering technique exploiting LLM recency bias — instructions appearing last in the context window carry disproportionate weight.

Example (gem-researcher):

> "Save research_findings\*{focus_area}.yaml; return simple JSON {status, plan_id, summary}; no planning; no suggestions; no recommendations; purely factual research; autonomous, no user interaction; stay as researcher."

This acts as a **behavioral guardrail** that reinforces identity and constraints at the point where the model generates its response. Forge has nothing equivalent.

### 8.2 `detailed thinking on` Directive

Present in ALL 8 Gem agents. This activates extended chain-of-thought reasoning. Forge includes no thinking mode directives.

### 8.3 Evidence Storage Architecture

Gem chrome-tester defines a structured evidence directory:

> `docs/plan/{plan_id}/evidence/{task_id}/` with subfolders `screenshots/`, `logs/`, `network/`. Files named by timestamp and scenario.

This is a **structured audit trail** beyond what either framework's main comparison covers.

### 8.4 Memory System Architecture

Gem references `/memories/memory-system-patterns.md` as a shared memory convention. Multiple agents (researcher, planner, orchestrator) both read and write to this system with citation verification:

- "Memory READ: Verify citations (file:line) before using stored memories"
- "Memory CREATE: Include citations (file:line)"
- "Memory UPDATE: Refresh timestamp when verifying existing memories"

The existing comparison mentions cross-agent memory but doesn't analyze the citation verification or the shared memory format convention.

### 8.5 `mcp_sequential-th_sequentialthinking` for Complex Reasoning

Gem planner uses:

> "Use mcp_sequential-th_sequentialthinking ONLY for multi-step reasoning (3+ steps)"

This is a separate tool for structured sequential reasoning, distinct from "detailed thinking on." Not mentioned in existing documents.

### 8.6 `reference_cache` for Standards

Gem chrome-tester uses:

> "Use reference_cache for WCAG standards"

This suggests a caching mechanism for external standards that agents reference repeatedly. Not mentioned in existing documents.

### 8.7 `manage_todos` Integration

Gem orchestrator uses `manage_todos` alongside plan.yaml updates:

> "Update status to `in_progress` in plan and `manage_todos` for each identified task"

This integrates with an external task tracking system. Not mentioned in existing documents.

### 8.8 `walkthrough_review` and `plan_review` Presentation Tools

Gem uses dedicated tools for human interaction:

- `plan_review` — for presenting findings and plans; used as a pause point
- `walkthrough_review` — for presenting final summaries ("ALWAYS when ending/response/summary")
- `ask_questions` — fallback when the above tools are unavailable

These are not generic "pause and ask" mechanisms but specialized presentation tools. Not analyzed structurally in existing documents.

### 8.9 Agent-Specific Plan Fields

Gem's plan_format_guide includes agent-specific task fields:

- `gem-implementer`: `tech_stack`, `test_coverage`
- `gem-reviewer`: `requires_review`, `review_depth`, `security_sensitive`
- `gem-chrome-tester`: `validation_matrix` with scenarios/steps/expected_results
- `gem-devops`: `environment`, `requires_approval`
- `gem-documentation-writer`: `audience`, `coverage_matrix`

This means the plan encodes not just what to do but **how** each specific agent type should approach the task. Forge's task files are agent-agnostic.

### 8.10 Idempotency Requirement

Gem devops explicitly requires all operations to be idempotent:

> "All tasks idempotent", "Ensure idempotency", "Use atomic operations"

This is an architectural safety pattern for infrastructure operations. Neither comparison document mentions it.

### 8.11 `list_code_usages` Before Refactoring

Gem implementer requires:

> "Always use list_code_usages before refactoring"

This ensures impact analysis before making changes. Forge implementer has no equivalent pre-edit analysis requirement.

### 8.12 `user-invocable` Flag

All Gem agents include `user-invocable: true` in frontmatter. This suggests the framework considers which agents can be directly invoked by users vs. only by other agents. Forge has no equivalent — all agents are implicitly invocable.

---

## File References

### Forge Agent Files (`.github/agents/`)

| File                                                                           | Structural Notes                                                                      |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| [orchestrator.agent.md](../../../.github/agents/orchestrator.agent.md)         | Largest agent; 7-step pipeline; no shared operating rules; no anti-drift              |
| [researcher.agent.md](../../../.github/agents/researcher.agent.md)             | Two modes (focused/synthesis); no retrieval strategy; no communication rules          |
| [spec.agent.md](../../../.github/agents/spec.agent.md)                         | Minimal — no rules section, no workflow detail, no error handling                     |
| [designer.agent.md](../../../.github/agents/designer.agent.md)                 | Minimal — no rules section, no error handling                                         |
| [critical-thinker.agent.md](../../../.github/agents/critical-thinker.agent.md) | Unique: question-asking agent; no structured output format; no completion contract    |
| [planner.agent.md](../../../.github/agents/planner.agent.md)                   | Detailed dependency model; no pre-mortem; no task size limits; no mode detection      |
| [implementer.agent.md](../../../.github/agents/implementer.agent.md)           | Strict input restrictions; no TDD; no error handling; no tool guidance                |
| [verifier.agent.md](../../../.github/agents/verifier.agent.md)                 | Hardcoded build commands (`dotnet build TourneyPal.sln`); comprehensive report format |
| [reviewer.agent.md](../../../.github/agents/reviewer.agent.md)                 | General code review; no security scanning; no tiered depth                            |

### Forge Prompt

| File                                                                              | Notes                                                         |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| [feature-workflow.prompt.md](../../../.github/prompts/feature-workflow.prompt.md) | Entry point; delegates to orchestrator; no approval mode flag |

### Gem Team Agent Files (Remote)

| File                              | Key Architectural Patterns                                                                                      |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| gem-orchestrator.agent.md         | `disable-model-invocation: true`; `<valid_subagents>`; max 4 concurrent; `manage_todos`; `walkthrough_review`   |
| gem-researcher.agent.md           | `<research_format_guide>` YAML schema; hybrid retrieval; confidence tracking; memory integration                |
| gem-planner.agent.md              | `<task_size_limits>`; `<plan_format_guide>` YAML schema; mode detection; pre-mortem; agent-specific task fields |
| gem-implementer.agent.md          | Full TDD cycle; `get_errors` after every edit; `list_code_usages` before refactoring; YAGNI/KISS/DRY            |
| gem-reviewer.agent.md             | `<review_criteria>` tiered depth; OWASP scanning; quality bar heuristic; grep_search for secrets/PII            |
| gem-chrome-tester.agent.md        | Evidence storage structure; `reference_cache` for standards; validation matrix; observation-first loop          |
| gem-devops.agent.md               | `<approval_gates>` for security/production; idempotency requirement; health checks                              |
| gem-documentation-writer.agent.md | Parity enforcement; coverage matrix; diagram generation; read-only source code                                  |

---

## Assumptions & Limitations

1. **Gem agent files were fetched via URL** — the analysis is based on the raw content of the remote files as retrieved. If the Gem Team repository has been updated since fetching, findings may be stale.
2. **Memory system patterns file** (`/memories/memory-system-patterns.md`) was not available for direct analysis — conclusions about Gem's memory architecture are inferred from agent references to it.
3. **Tool activation functions** (e.g., `activate_vs_code_interaction`) are assumed to be environment-specific (possibly MCP tool groups); their exact mechanism was not verified.
4. **`detailed thinking on` directive** — its exact effect depends on the underlying model; assumed to activate extended chain-of-thought reasoning.
5. **`reference_cache` and `manage_todos`** — these appear to be external tool integrations; their architecture was inferred from usage patterns, not direct inspection.
6. **Forge's `chatagent` frontmatter fields** — only `name` and `description` are used; it's unclear if the platform supports additional fields like `disable-model-invocation` or `user-invocable`.

---

## Open Questions

1. **Does the GitHub Copilot `chatagent` format support `disable-model-invocation` and `user-invocable` frontmatter fields?** If so, Forge should adopt them.
2. **Would XML-like tags (e.g., `<role>`, `<workflow>`, `<final_anchor>`) within markdown chatagent files be parsed correctly by GitHub Copilot?** If they're treated as plain text (which they likely are), their benefit is purely as LLM prompt engineering markers — which still has value.
3. **How much improvement does `detailed thinking on` actually provide?** Is it a model-specific directive or a general prompt engineering technique?
4. **Should Forge adopt the 200-line file read limit?** This may conflict with scenarios where reading entire files is necessary (e.g., verifier inspecting test results).
5. **Can Forge's DONE/ERROR text contract be extended to three states without breaking orchestrator parsing?** A minimal change to support `NEEDS_REVISION:` would provide better error routing.
6. **Would adding `<final_anchor>` to Forge agents actually reduce behavioral drift?** This is testable but hasn't been validated.
7. **Should Forge agents include a shared "Common Operating Rules" document referenced by all agents**, or should the rules be duplicated in each agent file? Gem duplicates them; a shared include might be cleaner but depends on platform support.
