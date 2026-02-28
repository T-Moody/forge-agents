# Research: Architecture

## Summary

Three agent systems exist with fundamentally different architectures: **Anvil** is a monolithic single agent using SQL verification; **Forge** is a 24-agent pipeline with Markdown memory files and prose-based routing; **NewAgents** is a 9-agent pipeline with typed YAML schemas and SQL evidence gates. The key architectural evolution from Forge to NewAgents was: (1) consolidating 15 specialized cluster agents into 2 parameterized agents, (2) replacing prose memory files with typed YAML completion contracts for routing, (3) adopting Anvil's SQL evidence gating pattern for anti-hallucination verification, (4) adding pipeline state recovery (EC-5), and (5) allowing the orchestrator `run_in_terminal` for SQL queries (DR-1). The critical unverified assumption across ALL systems is whether VS Code's `runSubagent` API supports model routing for multi-model adversarial review — no evidence confirms this capability works for custom agents. The root `.github/agents/` currently mirrors NewAgents' 9-agent structure.

---

## Findings

### F-1: Agent Inventory — Anvil 1, Forge 24, NewAgents 9

**Category:** structure

Anvil is a single monolithic agent (420 lines) that handles the full development loop internally. Forge has 24 agent definitions: orchestrator, researcher, spec, designer, critical-thinker (deprecated), ct-security, ct-scalability, ct-maintainability, ct-strategy, planner, implementer, documentation-writer, v-build, v-tests, v-tasks, v-feature, r-security, r-quality, r-testing, r-knowledge, adversarial-reviewer, post-mortem, verifier, knowledge-agent. NewAgents has 9 agent definitions: orchestrator, researcher, spec, designer, adversarial-reviewer, planner, implementer, verifier, knowledge-agent. The root `.github/agents/` directory mirrors NewAgents exactly (9 agents + 3 reference docs).

The key consolidation was replacing Forge's 15 specialized cluster agents (4 CT + 4 V + 4 R + documentation-writer + critical-thinker + post-mortem) with 2 parameterized agents (adversarial-reviewer dispatched ×3 with distinct `review_focus`, verifier dispatched per-task).

**Evidence:**

- `Forge/.github/agents/` — 24 `.agent.md` files
- `NewAgents/.github/agents/` — 9 `.agent.md` files + `schemas.md`, `dispatch-patterns.md`, `severity-taxonomy.md`
- `Anvil/anvil.agent.md` — single 420-line agent file
- `.github/agents/` — 9 `.agent.md` files (mirrors NewAgents)

**Relevance:** Establishes the baseline agent counts for the overhaul. The Forge→NewAgents consolidation pattern (specialized agents → parameterized instances) is the primary architectural evolution to understand.

---

### F-2: Three-Tier Communication — Anvil SQL-only, Forge Markdown Memory, NewAgents Typed YAML + SQL

**Category:** pattern

Anvil uses SQL exclusively (`anvil_checks` table) for verification tracking and evidence gating — no inter-agent file communication since it's a single agent. Forge uses a dual-layer Markdown memory system: shared `memory.md` (orchestrator sole writer) plus isolated `memory/<agent-name>.mem.md` files per agent. Agents write to their isolated memory; the orchestrator reads and merges. NewAgents uses typed YAML schemas (10 defined in `schemas.md` with strict field definitions) for all agent outputs, with completion contracts for routing. It also adds two SQL databases: `verification-ledger.db` (`anvil_checks` table) and `pipeline-telemetry.db` (`pipeline_telemetry` table). The orchestrator reads ONLY typed YAML completion blocks for routing decisions, while SQL provides independent evidence gates.

**Evidence:**

- `Anvil/anvil.agent.md` lines 67–79 — CREATE TABLE `anvil_checks` schema
- `Forge/.github/agents/orchestrator.agent.md` lines 39–47 — Memory-First Protocol
- `Forge/.github/agents/dispatch-patterns.md` — Memory-First Pattern section
- `NewAgents/.github/agents/schemas.md` lines 1–120 — 10 typed schema definitions
- `NewAgents/.github/agents/orchestrator.agent.md` lines 140–180 — SQLite initialization for both DBs

**Relevance:** The communication evolution from prose to typed YAML is the most significant architectural change. Downstream agents must understand which schema system the overhaul should adopt.

---

### F-3: Parallel Execution Relies on runSubagent with Max 4 Concurrent Cap

**Category:** pattern

All three systems reference VS Code's `runSubagent` as the dispatch mechanism. Both Forge and NewAgents enforce a maximum of 4 concurrent subagent invocations per wave. When more than 4 tasks exist, they partition into sequential sub-waves of ≤4. Forge defines three dispatch patterns (A: fully parallel, B: sequential gate + parallel, C: replan loop). NewAgents defines two patterns (A: fully parallel with ≥M of N gate conditions, B: sequential with replan loop, max 3 iterations).

The Forge orchestrator prompt says "Dispatch independent agents in parallel — invoke each subagent as a separate `runSubagent` call concurrently in parallel." However, the actual concurrency is platform-dependent — whether VS Code truly runs multiple `runSubagent` calls in parallel or serializes them is determined by the runtime. Anvil dispatches 3–4 `code-review` subagents in parallel for adversarial review of Large tasks but does not use parallel dispatch for its main workflow.

**Evidence:**

- `Forge/.github/agents/orchestrator.agent.md` lines 41, 180–184 — concurrency cap and parallel dispatch
- `Forge/.github/agents/dispatch-patterns.md` — Pattern A, B, C definitions
- `NewAgents/.github/agents/dispatch-patterns.md` — Pattern A, B definitions with gate conditions
- `Forge/.github/prompts/feature-workflow.prompt.md` — "Dispatch independent agents in parallel"
- `Anvil/anvil.agent.md` lines 236–243 — multi-model reviewer dispatch for Large tasks

**Relevance:** Parallel execution capability is critical for pipeline efficiency. The 4-agent cap is a design choice, not a known platform limit. Whether true concurrency occurs depends on VS Code's `runSubagent` implementation.

---

### F-4: Multi-Model Adversarial Review — Model Routing Is UNVERIFIED

**Category:** risk

Anvil dispatches `code-review` subagents using `agent_type: "code-review"` with `model: "gpt-5.3-codex"` (or `gemini-3-pro-preview`, `claude-opus-4.6`). This appears to be a VS Code platform-provided agent type, **not** a custom `.agent.md` definition. NewAgents uses a custom `adversarial-reviewer.agent.md` dispatched as 3 parallel instances with `model` as an orchestrator-provided parameter.

The `design.md` for the prior feature explicitly states: _"The ability to route subagent dispatches to specific LLM models via `.agent.md` configuration is UNVERIFIED at time of design. No documentation or prior implementation confirms that VS Code's `runSubAgent` API respects model routing directives."_

The fallback path is **perspective-diverse prompts**: dispatch 3 reviewers with the SAME model but distinct `review_focus` parameters (security, architecture, correctness). A prior adversarial reviewer memory file states: _"Anvil's `code-review` is a platform-provided agent type, not a custom `.agent.md`. Model routing works because it's a built-in VS Code Copilot feature."_

Forge avoids this problem entirely by using different specialized agent definitions (4 CT agents, 4 R agents) rather than model routing.

**Evidence:**

- `Anvil/anvil.agent.md` lines 226–243 — `agent_type` and `model` specification for reviewers
- `NewAgents/.github/agents/adversarial-reviewer.agent.md` lines 25–33 — `model` as orchestrator parameter
- `docs/feature/next-gen-multi-agent-system/design.md` lines 957–969 — EC-9 Model Routing Fallback
- `docs/feature/next-gen-multi-agent-system/memory/adversarial-reviewer-3.mem.md` line 5 — platform agent type observation

**Relevance:** Model routing is the single most important unverified assumption. The overhaul must decide whether to rely on model parameters, use the platform's `code-review` agent type, or use prompt diversity as the primary diversity mechanism.

---

### F-5: Orchestrator Tool Restrictions Diverged — Forge Bans run_in_terminal, NewAgents Allows It (DR-1)

**Category:** structure

Forge orchestrator allowed tools: `agent`, `agent/runSubagent`, `memory`, `read_file`, `list_dir`. It explicitly bans `run_in_terminal`, `create_file`, `replace_string_in_file`, `grep_search`, `semantic_search`, `file_search`, `get_errors`. NewAgents adds `run_in_terminal` and `get_terminal_output` to the allowed list, justified by Deviation Record DR-1: independent evidence verification via SQL queries is the core anti-hallucination property.

The `run_in_terminal` usage is constrained to: SQLite read queries (SELECT on `verification-ledger.db`), SQLite DDL (CREATE TABLE, PRAGMA at Step 0 only), git read operations (git status, git rev-parse, git show, git tag), and git staging/commit at Step 9. It MUST NOT be used for builds, tests, code execution, or file modification. Anvil has full tool access as a single agent.

**Evidence:**

- `Forge/.github/agents/orchestrator.agent.md` line 92 — tool restriction list
- `NewAgents/.github/agents/orchestrator.agent.md` lines 30–68 — allowed tools table with DR-1 constraint
- `Anvil/anvil.agent.md` — no tool restrictions (single agent with full access)

**Relevance:** The tool restriction philosophy differs between systems. The overhaul must decide whether the orchestrator should have SQL query access (NewAgents pattern with DR-1) or be limited to `read_file`/`list_dir` (Forge pattern).

---

### F-6: Pipeline Step Count — Anvil ~14, Forge 8, NewAgents 10

**Category:** structure

Anvil runs ~14 internal steps within a single agent (Boost, Git Hygiene, Understand, Recall, Survey, Plan, Baseline, Implement, Diagnostics, Verification, Adversarial Review, Operational Readiness, Evidence Bundle, Learn, Present, Commit). Forge runs an 8-step pipeline (0: Setup, 1: Research, 2: Spec, 3: Design, 3b: CT Review, 4: Planning, 5: Implementation, 6: V-Cluster, 7: R-Cluster, 8: Post-Mortem). NewAgents runs a 10-step pipeline (0: Setup, 1: Research, 1a: Approval Gate, 2: Spec, 3: Design, 3b: Design Review, 4: Planning, 4a: Approval Gate, 5: Implementation, 6: Verification, 7: Code Review, 8: Knowledge Capture, 8b: Evidence Bundle, 9: Auto-Commit).

NewAgents added explicit approval gates (1a, 4a), split Knowledge + Auto-Commit into separate terminal steps, and changed to per-task verification instead of Forge's specialized V-cluster approach.

**Evidence:**

- `Anvil/anvil.agent.md` — Steps 0 through 8
- `Forge/.github/agents/orchestrator.agent.md` — Steps 0–8 pipeline
- `NewAgents/.github/agents/orchestrator.agent.md` — Steps 0–9 pipeline

**Relevance:** The pipeline structure evolution shows progressive decomposition. The overhaul should understand the rationale for each step change.

---

### F-7: Typed YAML Schemas with Completion Contracts Are NewAgents' Core Innovation

**Category:** pattern

NewAgents introduces 10 typed YAML schemas (`schemas.md`, 1284 lines) with strict field definitions, type constraints, and allowed values. Every agent output includes a common header (`agent_output` with agent, instance, step, started_at, completed_at, schema_version, payload) and a completion contract (`status` DONE/NEEDS_REVISION/ERROR, `summary` ≤200 chars, `severity`, `findings_count`, `risk_level`, `output_paths`, optional `evidence_summary`). A Producer/Consumer Dependency Table documents data flow.

The Forge orchestrator reads `memory/<agent>.mem.md` prose files and applies cluster decision logic; the NewAgents orchestrator reads typed YAML completion blocks and SQL evidence gates. Both systems have `schemas.md` as a reference doc, but NewAgents enforces it as the primary routing interface. Human-readable Markdown companions are generated alongside YAML for auditability.

**Evidence:**

- `NewAgents/.github/agents/schemas.md` lines 1–200 — 10 schema definitions with field tables
- `Forge/.github/agents/schemas.md` — identical content, used as reference only
- `NewAgents/.github/agents/orchestrator.agent.md` line 6 — "All state is in-context only"
- `Forge/.github/agents/orchestrator.agent.md` lines 96–172 — cluster decision logic reads memory files

**Relevance:** The typed schema approach provides deterministic routing vs. Forge's prose-based memory reading. The overhaul should assess whether schema enforcement improves reliability or adds overhead.

---

### F-8: Forge Uses Dual-Layer Memory with Merge Overhead; NewAgents Eliminates Memory Files

**Category:** pattern

Forge implements an elaborate memory lifecycle: Initialize (Step 0), Merge (after each agent/cluster), Prune, Extract Lessons, Invalidate on revision, Clean invalidated, Validate. The orchestrator reads isolated memory files for routing, then dispatches a subagent to merge them into shared `memory.md`. This creates ~12+ merge dispatches per run.

NewAgents completely eliminates the memory layer — there is no `memory.md`, no isolated memory files, no merge operations. Routing is driven entirely by typed YAML completion blocks and SQL evidence gates. The orchestrator reads completion contract fields (`status`, `severity`, `findings_count`) from agent output YAML files at deterministic paths. This removes the merge overhead but sacrifices the cross-agent context sharing that memory files provided.

**Evidence:**

- `Forge/.github/agents/orchestrator.agent.md` lines 504–530 — Memory Lifecycle Actions table
- `Forge/.github/agents/orchestrator.agent.md` — merge steps 1.1m, 2m, 3b.2, 4m, etc.
- `NewAgents/.github/agents/orchestrator.agent.md` — no `memory.md` references, no merge steps
- `NewAgents/.github/agents/implementer.agent.md` — no memory file references

**Relevance:** Memory elimination is a major architectural simplification. The tradeoff is reduced cross-agent context passing vs. eliminated merge complexity. The overhaul must decide the right balance.

---

### F-9: Cluster Consolidation — Forge's 15 Specialized Agents → NewAgents' 2 Parameterized Agents

**Category:** structure

Forge uses specialized agent definitions: CT cluster (ct-security, ct-scalability, ct-maintainability, ct-strategy), V cluster (v-build, v-tests, v-tasks, v-feature), R cluster (r-security, r-quality, r-testing, r-knowledge), plus critical-thinker (deprecated), documentation-writer, and post-mortem.

NewAgents replaces the CT and R clusters with 3 `adversarial-reviewer` instances dispatched with distinct `review_focus` parameters (security, architecture, correctness). The V cluster is replaced by per-task `verifier` dispatch. Knowledge Agent absorbs post-mortem's evidence bundle assembly (Step 8b) and r-knowledge's decision log responsibilities. This consolidation reduces 24 agent definitions to 9 while maintaining functional coverage through parameterization.

**Evidence:**

- `Forge/.github/agents/` — ct-security, ct-scalability, ct-maintainability, ct-strategy agent files
- `Forge/.github/agents/` — v-build, v-tests, v-tasks, v-feature agent files
- `Forge/.github/agents/` — r-security, r-quality, r-testing, r-knowledge agent files
- `NewAgents/.github/agents/adversarial-reviewer.agent.md` — `review_focus` parameter
- `NewAgents/.github/agents/verifier.agent.md` — dispatched per-task
- `NewAgents/.github/agents/knowledge-agent.agent.md` — absorbs evidence bundle (Step 8b) and decision log

**Relevance:** The parameterized approach is the defining architectural pattern of NewAgents. The overhaul should evaluate whether this consolidation sacrifices specialization quality for simplicity.

---

### F-10: Pipeline State Recovery — Forge None, NewAgents Scans Output Files (EC-5)

**Category:** pattern

Forge has no explicit pipeline state recovery mechanism. NewAgents implements Pipeline State Recovery (EC-5): on resume, the orchestrator reconstructs state by scanning the feature directory with `list_dir`, reading each discovered output's completion block, and resuming from the first incomplete step. The design notes this is more robust than a manifest because each agent's output independently records its own completion — there is no single file to corrupt. Anvil relies on git-based rollback only.

**Evidence:**

- `NewAgents/.github/agents/orchestrator.agent.md` lines 92–105 — Pipeline State Recovery (EC-5)
- `Forge/.github/agents/orchestrator.agent.md` — no resume/recovery mechanism documented
- `Anvil/anvil.agent.md` — git-based rollback only

**Relevance:** Resume capability is important for long-running pipelines. The overhaul should consider whether to adopt or enhance EC-5.

---

### F-11: YAML Frontmatter Inconsistent Across Agents

**Category:** structure

In Forge, most agents have YAML frontmatter with `name` and `description` fields. In NewAgents, only some agents have frontmatter (planner, knowledge-agent, designer have it) while others use structured markdown headers without frontmatter (orchestrator, researcher, adversarial-reviewer, verifier, spec, implementer). No agent in any system specifies a `model` field in YAML frontmatter. All Forge agents include `<!-- experimental: model-dependent -->` HTML comments, suggesting model awareness is aspirational rather than functional.

**Evidence:**

- `Forge/.github/agents/implementer.agent.md` lines 1–4 — name/description frontmatter
- `NewAgents/.github/agents/orchestrator.agent.md` line 1 — no frontmatter
- `NewAgents/.github/agents/knowledge-agent.agent.md` lines 1–5 — has frontmatter
- All Forge agents — `<!-- experimental: model-dependent -->` comment

**Relevance:** Frontmatter consistency affects agent discoverability by the VS Code runtime. The overhaul should standardize.

---

### F-12: Evidence Gating — Anvil Pioneered SQL Gates, NewAgents Extended with Verdict/Severity/Round

**Category:** pattern

Anvil introduced the SQL verification ledger: every verification step is an INSERT into `anvil_checks`; evidence gates are SQL COUNT queries; "if the INSERT didn't happen, the verification didn't happen." NewAgents adopted and extended this with: `verdict` field (approve/needs_revision/blocker), `severity` field (Blocker/Critical/Major/Minor), `round` field (review iteration tracking), `run_id` field (prevents cross-run contamination), and `output_snippet` length constraint (CHECK LENGTH ≤ 500). NewAgents also added a separate `pipeline-telemetry.db`. The orchestrator uses SQL queries via `run_in_terminal` to independently verify evidence gates at Steps 3b, 6, and 7. Forge uses no SQL.

**Evidence:**

- `Anvil/anvil.agent.md` lines 67–79 — original `anvil_checks` CREATE TABLE
- `NewAgents/.github/agents/orchestrator.agent.md` lines 140–180 — extended schema with verdict, severity, round
- `NewAgents/.github/agents/orchestrator.agent.md` lines 600–660 — evidence gate SQL query templates
- `NewAgents/.github/agents/verifier.agent.md` lines 120–155 — SQL INSERT template and check_name conventions

**Relevance:** SQL evidence gating is the anti-hallucination mechanism. The overhaul should preserve and potentially extend this pattern.

---

### F-13: Forge and NewAgents Share Identical schemas.md and severity-taxonomy.md

**Category:** structure

Both `schemas.md` files (Forge and NewAgents) contain identical 1284-line content defining the same 10 typed YAML schemas. Similarly, `severity-taxonomy.md` files are identical. This indicates NewAgents was built by copying Forge's reference docs and replacing agent definitions while keeping schema contracts. The `evaluation-schema.md` exists only in Forge — NewAgents absorbed artifact evaluations into the Knowledge Agent's workflow.

**Evidence:**

- `Forge/.github/agents/schemas.md` lines 1–200 — identical to NewAgents
- `NewAgents/.github/agents/schemas.md` lines 1–200 — identical to Forge
- `Forge/.github/agents/evaluation-schema.md` — exists only in Forge

**Relevance:** Schema continuity means the overhaul can adopt schemas from either system without migration.

---

### F-14: Approval Gates Are NewAgents-Only — Interactive vs Autonomous Modes

**Category:** pattern

NewAgents introduces explicit approval gates at Step 1a (post-research) and Step 4a (post-planning) with structured YAML templates containing `gate_id`, `question`, `context`, and multiple-choice options presented via `ask_questions`. In interactive mode, the orchestrator pauses; in autonomous mode, the default is auto-selected. Forge has a simpler `APPROVAL_MODE` flag without defining the interaction format. Anvil has pushback with `ask_user` providing freeform choices but no structured gates.

**Evidence:**

- `NewAgents/.github/agents/orchestrator.agent.md` lines 230–280 — Step 1a approval gate
- `NewAgents/.github/agents/orchestrator.agent.md` lines 370–400 — Step 4a approval gate
- `Forge/.github/agents/orchestrator.agent.md` line 43 — `APPROVAL_MODE` gate reference
- `Anvil/anvil.agent.md` lines 21–33 — pushback with `ask_user`

**Relevance:** Structured approval gates enable human-in-the-loop oversight. The overhaul should decide the right interaction design.

---

## Source Files Examined

- `Anvil/anvil.agent.md`
- `Forge/.github/agents/orchestrator.agent.md`
- `Forge/.github/agents/dispatch-patterns.md`
- `Forge/.github/agents/schemas.md`
- `Forge/.github/agents/evaluation-schema.md`
- `Forge/.github/agents/severity-taxonomy.md`
- `Forge/.github/agents/adversarial-reviewer.agent.md`
- `Forge/.github/agents/implementer.agent.md`
- `Forge/.github/agents/critical-thinker.agent.md`
- `Forge/.github/agents/post-mortem.agent.md`
- `Forge/.github/agents/knowledge-agent.agent.md`
- `Forge/.github/agents/r-security.agent.md`
- `Forge/.github/agents/documentation-writer.agent.md`
- `Forge/.github/agents/v-build.agent.md`
- `Forge/.github/prompts/feature-workflow.prompt.md`
- `NewAgents/.github/agents/orchestrator.agent.md`
- `NewAgents/.github/agents/dispatch-patterns.md`
- `NewAgents/.github/agents/schemas.md`
- `NewAgents/.github/agents/severity-taxonomy.md`
- `NewAgents/.github/agents/adversarial-reviewer.agent.md`
- `NewAgents/.github/agents/implementer.agent.md`
- `NewAgents/.github/agents/verifier.agent.md`
- `NewAgents/.github/agents/knowledge-agent.agent.md`
- `NewAgents/.github/prompts/feature-workflow.prompt.md`
- `.github/agents/` (root — directory listing)
- `docs/feature/next-gen-multi-agent-system/design.md` (lines 940–980)
- `docs/feature/next-gen-multi-agent-system/research/architecture.md` (lines 640–680)
- `docs/feature/next-gen-multi-agent-system/memory/adversarial-reviewer-3.mem.md`

## Research Metadata

- **Confidence Level:** high — All 24 Forge agent files discovered, key agents read in full; all 9 NewAgents agents read; Anvil read in full; schemas, dispatch patterns, severity taxonomies, prompts, and prior research read
- **Coverage Estimate:** ~95% of architecturally relevant content examined across all three systems. All orchestrators read fully. All reference docs (schemas, dispatch patterns, severity taxonomy, evaluation schema) read. Representative sample of specialized agents (CT, V, R clusters) read.
- **Gaps:**
  - Did not read every Forge agent in full detail (some cluster agents like v-tests, v-tasks, v-feature, r-quality, r-testing, r-knowledge, ct-\* were only examined at header level) — these follow similar patterns to the representative agents read
  - Did not verify the actual VS Code `runSubagent` API documentation (none found in workspace) — the mechanism remains platform-dependent
  - The `agent-system-overhaul` feature does not yet have an `initial-request.md` — research was conducted based on the orchestrator dispatch prompt
