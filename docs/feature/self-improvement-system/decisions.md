# Architectural Decision Log

## [2025-02-20] Centralized Schema Reference for Artifact Evaluations

- **Context:** 14 agents needed to produce artifact evaluations in a consistent format. Options were: (A) inline the schema in each agent definition, (B) create a shared reference document, (C) add it to the orchestrator prompt.
- **Decision:** Create `evaluation-schema.md` as a standalone shared reference document in `.github/agents/`, referenced by all evaluating agents.
- **Rationale:** Option A would create 14 copies of the schema that drift over time. Option C overloads the orchestrator prompt. Option B provides single-source-of-truth semantics — a schema change propagates automatically to all agents that reference it. This approach also established the shared reference document pattern for future cross-cutting concerns.
- **Scope:** Project-wide (establishes the cross-cutting concern pattern).
- **Affected components:** `.github/agents/evaluation-schema.md` (new), 14 agent files (added reference), `.github/prompts/feature-workflow.prompt.md` (added Key Artifacts entry).

## [2025-02-20] Telemetry Accumulation in Orchestrator Context

- **Context:** The orchestrator needs to collect per-agent telemetry (timing, token counts, status) and pass it to the PostMortem agent. Options were: (A) write telemetry to `memory.md` incrementally, (B) accumulate in orchestrator working context, (C) write to a dedicated telemetry file after each step.
- **Decision:** Accumulate telemetry in the orchestrator's working context (Global Rule 13) and pass as a structured block to PostMortem in Step 8.
- **Rationale:** Option A bloats `memory.md` with transient data. Option C would require file I/O after each step and need cleanup. Option B leverages the orchestrator's existing context window — telemetry is small (one line per agent), only consumed once (by PostMortem), and doesn't persist after the pipeline completes. This resolves CT-Scalability Finding 2 (memory.md bloat) and CT-Strategy Finding 7 (data collection mechanism).
- **Scope:** Feature-specific (telemetry data flow).
- **Affected components:** `.github/agents/orchestrator.agent.md` (Global Rule 13, Step 8), `.github/agents/post-mortem.agent.md` (Input section).

## [2025-02-20] PostMortem Quantitative-Only Boundary

- **Context:** The PostMortem agent could either generate qualitative improvement recommendations or focus on quantitative metrics aggregation. The initial request included "pipeline improvement suggestions."
- **Decision:** PostMortem is quantitative-only: aggregate evaluation scores, compute metrics, identify statistical outliers. No qualitative reasoning about why metrics are low or what to change.
- **Rationale:** Qualitative improvement suggestions are R-Knowledge's responsibility (already exists). Overlap creates conflicting recommendations and unclear ownership. A quantitative-only PostMortem has a well-defined scope, deterministic outputs, and avoids the LLM self-evaluation quality concern (CT-Strategy Finding 1). The agent acts as a metrics aggregator, not a reviewer.
- **Scope:** Feature-specific (PostMortem agent boundary).
- **Affected components:** `.github/agents/post-mortem.agent.md` (Operating Rules, Workflow), `.github/agents/orchestrator.agent.md` (Step 8 dispatch).

## [2025-02-20] Tool Restriction: Keep Read Tools for Orchestrator

- **Context:** AC-5 specified restricting orchestrator tool use to `[agent, agent/runSubagent, memory]`. CT-Security Critical Finding 1 identified that removing read tools (`file_search`, `grep_search`, `read_file`) would break orchestrator orientation (reading `memory.md` at start-of-step) and verification (checking file existence before dispatch).
- **Decision:** Keep read tools in the orchestrator's allowed set while restricting write tools (`create_file`, `replace_string_in_file`, `run_in_terminal`). The spec's literal tool list was a minimum set, not an exhaustive restriction.
- **Rationale:** Read tools are essential for orchestrator function — it cannot navigate the pipeline without reading `memory.md`, checking artifact existence, or orienting on workspace structure. Write tools are the actual risk surface (orchestrator modifying source code or agent definitions). The design revision reconciled this with the spec by interpreting AC-5 as "restrict tools that enable direct modification."
- **Scope:** Project-wide (orchestrator capability boundary).
- **Affected components:** `.github/agents/orchestrator.agent.md` (Operating Rule 5), `docs/feature/self-improvement-system/design.md` (CT revision section).

## [2025-02-20] Non-Blocking Evaluation Pattern

- **Context:** Agents produce artifact evaluations as a secondary output. If evaluation generation fails (schema parsing error, context-window limitations), the agent's primary work is still valid.
- **Decision:** Evaluations are non-blocking: evaluation failure should not cause an agent ERROR. The evaluation-schema.md includes an `evaluation_error` fallback format for agents that cannot complete evaluation.
- **Rationale:** Making evaluations blocking would mean that a schema formatting mistake could cause pipeline regression in any of the 14 evaluating agents. The fallback format preserves the signal ("this agent tried to evaluate but couldn't") while letting the pipeline proceed. This is consistent with the non-blocking pattern used by R-Knowledge and PostMortem.
- **Scope:** Project-wide (evaluation system design principle).
- **Affected components:** `.github/agents/evaluation-schema.md` (Error Handling section), all 14 evaluating agent files (workflow step wording: "secondary, non-blocking").

## [2025-02-20] Dual-Track Implementation for Orchestrator Modifications

- **Context:** Two independent features (self-improvement evaluation system and tool restriction) both modify `orchestrator.agent.md`. Implementing them in parallel risked merge conflicts and made review harder.
- **Decision:** Sequence implementation as Track A (self-improvement: Step 8, Global Rule 13, docs structure) followed by Track B (tool restriction: Operating Rule 5). Each track has its own task.
- **Rationale:** The orchestrator is the pipeline's coordination hub — concurrent modifications are high-risk. Sequential implementation means each task sees the complete prior state. The dependency graph (Task 10 depends on Task 03) ensures correct ordering. This adds minimal pipeline latency since both tasks target the same file.
- **Scope:** Feature-specific (implementation strategy for this feature).
- **Affected components:** `tasks/03-orchestrator-tool-restriction.md`, `tasks/10-orchestrator-step8-telemetry.md`, `.github/agents/orchestrator.agent.md`.
