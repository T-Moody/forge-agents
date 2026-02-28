# Research: Patterns

## Summary

Investigation of coding patterns across Anvil (1 agent, 420 lines), Forge (24 agents, 129–544 lines), and NewAgents (9 agents, 251–800 lines) reveals 12 key findings. The most critical insights: (1) Agent instruction files are bloating — NewAgents agents doubled in size from Forge equivalents due to inlined schemas, self-verification, and error handling that should be in shared reference docs. Target ~250–350 lines per agent. (2) Anti-drift anchors are the most consistently applied pattern and should be retained. (3) SQL evidence gates (`anvil_checks`) are strictly better than prose for verification decisions. (4) The two-tiered retry pattern (agent-internal 2× + orchestrator 1×) is well-designed but identical boilerplate across 9 agents. (5) `pipeline_telemetry.db` is dead code — created but never populated. (6) The no-file-redirect rule is agent-specific when it should be global. (7) The artifact evaluation + PostMortem feedback loop from Forge was dropped in NewAgents — a significant capability regression. (8) Context7 was deliberately removed but remains useful if MCP is configured. The overarching theme: NewAgents gained structure (typed YAML, SQL gates, self-verification) but lost feedback mechanisms (evaluations, PostMortem, memory context) and bloated individual files. The overhaul should extract shared patterns into reference docs, restore the feedback loop, and keep files lean.

## Findings

### F-1: Instruction file bloat — agent files range from 129–800 lines, with diminishing returns above ~350

**Category:** pattern

Anvil is 420 lines (single monolithic agent). Forge agent files range from 129 lines (evaluation-schema.md) to 544 lines (orchestrator.agent.md), with most specialized agents at 145–250 lines (v-build 145, v-tests 192, v-tasks 182, v-feature 195, implementer 236, r-knowledge 340, post-mortem 250). NewAgents/Current system agents are significantly larger: orchestrator 800 lines, verifier 521, knowledge-agent 509, implementer 519, spec 337, planner 376, researcher 251, designer 275, adversarial-reviewer 372. The orchestrator nearly doubled from Forge (544) to NewAgents (800). The Forge implementer was 236 lines; the NewAgents implementer is 519 lines — more than doubled. This growth came from adding self-verification checklists, typed YAML schemas, SQL ledger instructions, and detailed error handling sections. The user explicitly states "Instructions are useless when too big." Forge's lighter agents (~200 lines) with external reference docs (schemas.md, evaluation-schema.md, dispatch-patterns.md) appears a better pattern than inlining everything.

**Evidence:**

- `Anvil/anvil.agent.md` — 420 lines total
- `Forge/.github/agents/orchestrator.agent.md` — 544 lines
- `Forge/.github/agents/implementer.agent.md` — 236 lines
- `NewAgents/.github/agents/orchestrator.agent.md` — 800 lines (largest agent file)
- `NewAgents/.github/agents/implementer.agent.md` — 519 lines
- `NewAgents/.github/agents/verifier.agent.md` — 521 lines

**Relevance:** The overhaul should target ~250–350 lines per agent as a sweet spot, extracting shared patterns (schemas, error handling, self-verification checklists) into reference docs rather than duplicating across agents.

---

### F-2: Anti-drift anchors are universal and consistently structured — proven effective pattern

**Category:** pattern

Every agent across all three systems includes an Anti-Drift Anchor section at the end of the file, formatted as: `## Anti-Drift Anchor` followed by `**REMEMBER:** You are the **<Name>**. <role summary>. <key constraints>. Stay as <name>.` The pattern exploits LLM recency bias — placing the identity reinforcement at the end of the instruction file means it's most likely to be in the model's recent context. All NewAgents agents include this. All Forge agents include this. Anvil does not have it explicitly labeled but has a strong role definition at the top. The anchors are concise (1–3 sentences) and include the most critical behavioral constraints. This is the most consistently applied pattern across all systems.

**Evidence:**

- `NewAgents/.github/agents/researcher.agent.md` lines 249–251 — Anti-Drift Anchor section
- `NewAgents/.github/agents/implementer.agent.md` lines 519–521 — Anti-Drift Anchor section
- `Forge/.github/agents/v-build.agent.md` lines 145–147 — Anti-Drift Anchor section
- `Forge/.github/agents/v-tests.agent.md` lines 192–194 — Anti-Drift Anchor section
- `README.md` lines 195 and 411 — documents anti-drift anchors as a formal system pattern

**Relevance:** Anti-drift anchors are a proven, zero-cost pattern that should be retained in the overhaul. Keep them short (1–3 sentences) and at the end of every agent file.

---

### F-3: Self-verification evolved from absent (Anvil) to checklists (NewAgents) — adds length but enforces output quality

**Category:** pattern

Anvil has no self-verification section — it relies on external verification (anvil_checks SQL). Forge agents have no standardized self-verification section either — they rely on the V-cluster (v-build, v-tests, v-tasks, v-feature) for post-hoc verification. NewAgents introduced embedded `## Self-Verification` sections in every agent (researcher, spec, planner, designer, implementer, verifier, knowledge-agent, orchestrator, adversarial-reviewer). These are checkbox-style checklists that the agent runs before returning (e.g., schema compliance, citation integrity, companion consistency). The pattern adds 30–50 lines to each agent but catches common failures before the orchestrator needs to retry. The tradeoff: it increases file size but reduces retry waste.

**Evidence:**

- `NewAgents/.github/agents/researcher.agent.md` line 192 — Self-Verification section
- `NewAgents/.github/agents/verifier.agent.md` line 387 — Self-Verification section
- `NewAgents/.github/agents/spec.agent.md` line 291 — Self-Verification section
- `NewAgents/.github/agents/orchestrator.agent.md` line 771 — Self-Verification section
- `Forge/.github/agents/implementer.agent.md` — no Self-Verification section

**Relevance:** The overhaul should keep self-verification but consider extracting common checklist items into a shared reference doc to reduce per-agent bloat.

---

### F-4: SQL evidence gates vs markdown memory — SQL is strictly better for verification, markdown unavoidable for context

**Category:** pattern

Three verification/evidence approaches exist across systems. (1) Anvil: SQL-only via `anvil_checks` INSERT + COUNT queries. Every verification step is a SQL record. "INSERT before you report" is a core rule. (2) Forge: Markdown memory system — shared `memory.md` + isolated `memory/<agent>.mem.md` files. The orchestrator merges memories. No SQL. Evidence is prose-based, making gate verification subjective. (3) NewAgents: Hybrid — typed YAML for routing + SQL `anvil_checks` for evidence gates. The orchestrator independently verifies gates via SQL queries filtering on `run_id` and `round`. The SQL approach provides: deterministic gate checks (COUNT queries), tamper-resistance, queryable history, and cross-run analysis. Forge's markdown memory provides: rich context transfer, artifact indexing, and orientation data — but is unreliable for binary gate decisions. NewAgents correctly separates concerns: SQL for gates, YAML for routing, but the markdown memory system was dropped, losing the context-transfer benefit.

**Evidence:**

- `Anvil/anvil.agent.md` lines 67–79 — `CREATE TABLE anvil_checks`
- `Forge/.github/agents/orchestrator.agent.md` line 39 — Memory-First Protocol
- `Forge/.github/agents/v-tests.agent.md` line 41 — memory-first reading rule
- `NewAgents/.github/agents/orchestrator.agent.md` line 19 — evidence gate verification via SQL
- `NewAgents/.github/agents/verifier.agent.md` line 12 — SQL INSERT per check

**Relevance:** The overhaul should retain SQL evidence gates (strictly better than prose for verification) and consider restoring lightweight context transfer for agents that need orientation data.

---

### F-5: Error handling is two-tiered: agent-internal retries (2×) + orchestrator retry (1× per agent)

**Category:** pattern

NewAgents defines a consistent error handling pattern across all agents: (1) agent-internal retries for transient errors (network timeout, tool unavailable, database locked) — up to 2 times, never for deterministic failures; (2) orchestrator-level retries — 1 retry per agent invocation on ERROR. These compose: worst case is 3 internal attempts × 2 orchestrator attempts = 6 total tool calls. The pattern distinguishes transient vs. deterministic failures: transient are retried, deterministic are not. This is explicitly documented in `spec.agent.md` (line 287), `researcher.agent.md` (line 184), and `verifier.agent.md` (line 378). Forge has simpler retry: orchestrator retries failed agents once (Global Rule 4). No agent-internal retry guidance. Anvil: "When stuck after 2 attempts, explain what failed and ask for help. Don't spin." — pragmatic but unstructured.

**Evidence:**

- `NewAgents/.github/agents/spec.agent.md` line 287 — retry budget documentation
- `NewAgents/.github/agents/researcher.agent.md` line 184 — transient retry up to 2 times
- `NewAgents/.github/agents/verifier.agent.md` line 378 — transient retry up to 2 times
- `Forge/.github/agents/orchestrator.agent.md` line 36 — retry failed agents once
- `Anvil/anvil.agent.md` line 410 — stuck after 2 attempts, ask for help

**Relevance:** The two-tiered retry pattern from NewAgents is well-designed. The overhaul should retain it but consider extracting the error handling boilerplate into a shared reference doc since it's identical across 9 agents.

---

### F-6: Context7 was Anvil-only, deliberately removed from NewAgents — remains useful if MCP is available

**Category:** dependency

Anvil includes explicit Context7 MCP tool integration (lines 364–367): `context7-resolve-library-id` to find a library's ID, then `context7-query-docs` with the resolved ID. The instruction says "Do this BEFORE guessing at API usage." NewAgents deliberately removed this — no agent references Context7. The dependencies researcher confirmed this was intentional removal. Context7 provides live documentation lookup, reducing hallucination risk for library APIs. However, it's an MCP tool dependency — if the MCP server isn't configured, references to it are noise.

**Evidence:**

- `Anvil/anvil.agent.md` lines 364–367 — Context7 usage instructions
- `docs/feature/agent-system-overhaul/research/dependencies.yaml` lines 520–552 — Context7 deliberately absent
- `NewAgents/.github/agents/` — zero references to `context7` in any agent file

**Relevance:** The overhaul should make Context7 conditional: if MCP is configured, agents (especially implementer) should use it. Add it to a shared reference doc rather than inline in each agent.

---

### F-7: Pushback system evolved from informal (Anvil) to structured (NewAgents) — never halts autonomously

**Category:** pattern

Anvil has informal pushback: a `## Pushback` section (lines 12–30) listing implementation and requirements concerns, followed by `ask_user` with choices. It evaluates every request before executing. NewAgents formalized this into the Spec agent (`spec.agent.md` lines 127–215) with a structured pushback system: trigger criteria, severity classification, interactive vs. autonomous mode handling, a `pushback_log` YAML structure, and explicit "pushback NEVER autonomously halts" rule. The orchestrator also performs lightweight pushback at Step 0 (`orchestrator.agent.md` line 136–139). Both systems agree: in autonomous mode, concerns are logged but execution continues. Forge has no formalized pushback.

**Evidence:**

- `Anvil/anvil.agent.md` lines 12–30 — Pushback section with `ask_user`
- `NewAgents/.github/agents/spec.agent.md` lines 127–215 — structured pushback system
- `.github/agents/orchestrator.agent.md` lines 136–139 — lightweight pushback at Step 0
- `NewAgents/.github/agents/spec.agent.md` line 337 — anti-drift anchor includes pushback note

**Relevance:** Dual-layer pushback (orchestrator lightweight + spec detailed) is a good pattern. The overhaul should retain it with the "never halts autonomously" invariant.

---

### F-8: Terminal output file-redirect prohibition: inconsistently applied — only in implementer and V-cluster agents

**Category:** pattern

The rule "Never redirect command output to a file" appears in 4 agents across Forge and NewAgents: Forge implementer (line 59, line 224), Forge v-build (line 38), Forge v-tests (line 42), and NewAgents/Current implementer (line 344). The explicit prohibition covers patterns like `command > output.txt`, `command | tee output.txt`, writing `build_output.txt` or `test_results.txt`. The reason: "Writing intermediate output files triggers unnecessary permission prompts." However, this rule is NOT present in researchers, spec, designer, planner, verifier, knowledge-agent, adversarial-reviewer, or the orchestrator.

**Evidence:**

- `Forge/.github/agents/implementer.agent.md` line 59 — no file-redirect rule with rationale
- `Forge/.github/agents/v-build.agent.md` line 38 — never redirect command output
- `Forge/.github/agents/v-tests.agent.md` line 42 — never redirect command output
- `NewAgents/.github/agents/implementer.agent.md` line 344 — no file-redirect rule
- `.github/agents/implementer.agent.md` line 344 — same rule in current system

**Relevance:** The overhaul should promote the no-file-redirect rule to a global operating rule (in a shared reference doc or project instructions) rather than duplicating it in select agents.

---

### F-9: pipeline_telemetry.db is created but never populated — dead SQL pattern

**Category:** risk

NewAgents orchestrator creates `pipeline_telemetry.db` at Step 0 with a `pipeline_telemetry` table (`run_id`, `step`, `agent`, `instance`, `started_at`, `completed_at`, `status`, `dispatch_count`, `retry_count`, `notes`). The Knowledge Agent at Step 8 is listed as a consumer for `pipeline_telemetry_summary`. However, NO agent has instructions to INSERT into `pipeline_telemetry`. The orchestrator tracks telemetry in its context window instead. Forge's approach of passing telemetry via dispatch context to PostMortem actually works; the SQL table is unused.

**Evidence:**

- `NewAgents/.github/agents/orchestrator.agent.md` lines 176–195 — `pipeline_telemetry` table DDL
- `docs/feature/agent-system-overhaul/research/dependencies.yaml` lines 176–194 — dead dependency finding
- `Forge/.github/agents/orchestrator.agent.md` line 50 — telemetry in context, dispatched to PostMortem
- No agent in NewAgents has `INSERT INTO pipeline_telemetry` instructions

**Relevance:** The overhaul must either populate `pipeline_telemetry` (add INSERT instructions to the orchestrator after each dispatch) or remove the table. A populated SQL telemetry table would be strictly better than context-window telemetry.

---

### F-10: Instruction update pattern: Forge auto-applies via R-Knowledge, NewAgents governs via Knowledge Agent

**Category:** pattern

Forge's R-Knowledge agent auto-applies instruction and skill updates to files under `.github/instructions/` and `.github/skills/`. It has a safety constraint filter (KE-SAFE-6) that rejects changes weakening safety constraints. It explicitly NEVER modifies agent definitions. NewAgents' Knowledge Agent has a governed update mechanism for `.github/instructions/` with an additional constraint: in autonomous mode, instruction changes require approval. Both systems share: (1) never modify agent definitions, (2) never weaken safety constraints, (3) log all changes for transparency. The key difference: Forge auto-applies; NewAgents requires approval in autonomous mode. Neither system currently has `.github/instructions/` files to operate on.

**Evidence:**

- `Forge/.github/agents/r-knowledge.agent.md` line 50 — auto-apply instruction updates
- `Forge/.github/agents/r-knowledge.agent.md` line 61 — file boundaries for instruction updates
- `NewAgents/.github/agents/knowledge-agent.agent.md` lines 373–430 — governed instruction updates
- `NewAgents/.github/agents/knowledge-agent.agent.md` line 509 — anti-drift anchor mentions governed instructions

**Relevance:** The overhaul should decide whether to create `.github/instructions/` files (making this capability active) and whether to auto-apply (Forge) or govern (NewAgents) changes.

---

### F-11: Artifact evaluation system (Forge-only) provides quantitative feedback loop — absent from NewAgents

**Category:** pattern

Forge has a dedicated `evaluation-schema.md` (129 lines) defining a standardized YAML schema for artifact evaluations. Every agent rates upstream artifacts on `usefulness_score` (1–10) and `clarity_score` (1–10), plus structured fields for `useful_elements`, `missing_information`, `information_not_used`, `inaccuracies`, and `impact_on_work`. These feed into the PostMortem agent (250 lines) which produces quantitative cross-agent metrics, run logs, and trend analysis. NewAgents dropped this entire system — no evaluation schema, no artifact rating, no PostMortem equivalent. The Knowledge Agent produces knowledge updates and decision logs but not quantitative agent performance metrics.

**Evidence:**

- `Forge/.github/agents/evaluation-schema.md` — 129 lines, standardized evaluation YAML
- `Forge/.github/agents/post-mortem.agent.md` — 250 lines, quantitative analysis
- `NewAgents/.github/agents/` — no evaluation-schema.md, no post-mortem agent
- `docs/feature/agent-system-overhaul/research/impact.yaml` line 242 — confirms evaluation system absent

**Relevance:** The overhaul should restore the artifact evaluation + PostMortem feedback loop. Without quantitative feedback, there's no data-driven basis for improving agent instructions over time.

---

### F-12: Completion contracts are consistent across NewAgents: DONE/NEEDS_REVISION/ERROR with typed YAML

**Category:** pattern

Every NewAgents agent returns one of three statuses: DONE, NEEDS_REVISION, or ERROR. The orchestrator reads the completion block to determine routing. Key nuances: (1) Researcher and Spec do NOT return NEEDS_REVISION. (2) Implementer returns only DONE or ERROR — self-fixes up to 2 times first. (3) Verifier returns all three. (4) The orchestrator uses completion status + SQL evidence gates for routing decisions — it never parses prose. Forge has the same three-state contract but relies on prose parsing from memory files for some routing. Anvil has no formal completion contract.

**Evidence:**

- `NewAgents/.github/agents/researcher.agent.md` line 171 — does not return NEEDS_REVISION
- `NewAgents/.github/agents/spec.agent.md` line 270 — does not return NEEDS_REVISION
- `NewAgents/.github/agents/implementer.agent.md` line 521 — DONE or ERROR only
- `Forge/.github/agents/orchestrator.agent.md` line 35 — three-state completion contract
- `.github/agents/orchestrator.agent.md` line 111 — reads completion blocks for routing

**Relevance:** Typed completion contracts + SQL gates is the right routing pattern. The overhaul should standardize which agents support which statuses and document this in a routing matrix.

---

## Source Files Examined

- `Anvil/anvil.agent.md`
- `Forge/.github/agents/orchestrator.agent.md`
- `Forge/.github/agents/implementer.agent.md`
- `Forge/.github/agents/r-knowledge.agent.md`
- `Forge/.github/agents/evaluation-schema.md`
- `Forge/.github/agents/post-mortem.agent.md`
- `Forge/.github/agents/v-build.agent.md`
- `Forge/.github/agents/v-tests.agent.md`
- `Forge/.github/agents/v-tasks.agent.md`
- `Forge/.github/agents/v-feature.agent.md`
- `NewAgents/.github/agents/orchestrator.agent.md`
- `NewAgents/.github/agents/implementer.agent.md`
- `NewAgents/.github/agents/researcher.agent.md`
- `NewAgents/.github/agents/spec.agent.md`
- `NewAgents/.github/agents/planner.agent.md`
- `NewAgents/.github/agents/designer.agent.md`
- `NewAgents/.github/agents/verifier.agent.md`
- `NewAgents/.github/agents/knowledge-agent.agent.md`
- `NewAgents/.github/agents/adversarial-reviewer.agent.md`
- `.github/agents/orchestrator.agent.md`
- `.github/agents/implementer.agent.md`
- `NewAgents/README.md`
- `README.md`
- `docs/feature/agent-system-overhaul/research/dependencies.yaml`
- `docs/feature/agent-system-overhaul/research/impact.yaml`
- `docs/feature/agent-system-overhaul/research/architecture.yaml`

## Research Metadata

- **Confidence Level:** high
- **Coverage Estimate:** All 10 key questions from the dispatch prompt addressed with cross-system evidence
- **Gaps:** Did not exhaustively measure every Forge agent file's exact line count (used representative samples). Did not search for `store_memory` usage patterns across systems — only covered instruction file updates.
