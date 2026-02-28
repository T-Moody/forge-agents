# Research: Dependencies

## Summary

The NewAgents pipeline has a well-defined dependency graph with 10 typed YAML schemas, 2 SQLite databases, and 20+ deterministic file paths creating the inter-agent contract surface. The primary coupling point is `verification-ledger.db` (`anvil_checks` table), shared by 4 writers and 3 readers. Key risks identified: (1) Verifier cannot create its required output file due to tool restriction contradiction, (2) `pipeline_telemetry.db` is created but never populated, (3) review verdict file paths are inconsistent between producer and consumer agent definitions, (4) schema version checking is absent despite a formal evolution policy. External hard dependencies are `sqlite3` CLI and `git`. Context7 MCP tools were deliberately removed from the Anvil baseline. The pipeline has no circular dependencies and all feedback loops have hard iteration limits.

## Findings

### F-1: Inter-agent data flow follows strict 10-schema Producer/Consumer dependency graph

**Category:** dependency

The NewAgents pipeline defines 10 typed YAML schemas in `schemas.md` with a formal Producer/Consumer Dependency Table. The data flow is strictly sequential with well-defined feedback loops:

| Schema | Name                  | Producer             | Consumers                                          |
| ------ | --------------------- | -------------------- | -------------------------------------------------- |
| 1      | completion-contract   | ALL agents           | Orchestrator                                       |
| 2      | research-output       | Researcher ×4        | Spec, Designer                                     |
| 3      | spec-output           | Spec                 | Designer, Planner, Implementer, Verifier           |
| 4      | design-output         | Designer             | Planner, Implementer, Adversarial Reviewer         |
| 5      | plan-output           | Planner              | Orchestrator (risk_summary), Implementer, Verifier |
| 6      | task-schema           | Planner              | Implementer, Verifier (sub-output of plan-output)  |
| 7      | implementation-report | Implementer          | Verifier                                           |
| 8      | verification-report   | Verifier             | Orchestrator (gate_status), Adversarial Reviewer   |
| 9      | review-findings       | Adversarial Reviewer | Orchestrator (verdict routing), Knowledge Agent    |
| 10     | knowledge-output      | Knowledge Agent      | (terminal — no downstream consumers)               |

The Orchestrator reads ONLY completion contracts and SQL evidence for routing — it never reads full agent payloads. The `relevant_context` mechanism in task schemas limits downstream read amplification by providing exact section pointers into upstream outputs.

**Evidence:**

- `NewAgents/.github/agents/schemas.md` — Producer/Consumer Dependency Table (lines 41–55)
- `NewAgents/.github/agents/orchestrator.agent.md` — Pipeline Steps 0–9 (lines 200–570)
- `NewAgents/.github/agents/planner.agent.md` — relevant_context mechanism (lines 145–190)

**Relevance:** Defines the complete inter-agent contract surface. Any schema change requires updating all listed consumers. Breaking changes require coordinated updates across multiple agent definitions.

---

### F-2: Tool dependency catalog reveals 12 distinct tools with per-agent access control

**Category:** dependency

Each agent definition includes an explicit Tool Access table that lists allowed and restricted tools. The complete tool dependency matrix:

| Agent                    | # Tools | Key Tools                                                                                                 | Key Restrictions                                                |
| ------------------------ | ------- | --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| **Orchestrator**         | 7       | `runSubagent`, `memory`, `run_in_terminal`, `ask_questions`                                               | No `create_file`, `grep_search`, `semantic_search`              |
| **Researcher**           | 5       | `read_file`, `grep_search`, `semantic_search`                                                             | No `create_file`\* (exception for output), no `run_in_terminal` |
| **Spec**                 | 8       | All discovery + `create_file`, `ask_questions`                                                            | —                                                               |
| **Designer**             | 7       | All discovery + `create_file`                                                                             | No `run_in_terminal`, `get_errors`                              |
| **Planner**              | 7       | All discovery + `create_file`                                                                             | No `run_in_terminal`, `get_errors`                              |
| **Implementer**          | 12      | Full access including `multi_replace_string_in_file`, `run_in_terminal`, `get_errors`, `list_code_usages` | —                                                               |
| **Verifier**             | 8       | `run_in_terminal`, `get_errors`, `ide-get_diagnostics`                                                    | No `create_file`\*\*, no `semantic_search`                      |
| **Adversarial Reviewer** | 7       | `create_file`, `run_in_terminal` (git diff + SQL only)                                                    | No `replace_string_in_file`, `get_errors`                       |
| **Knowledge Agent**      | 8       | `create_file`, `memory`/`store_memory`                                                                    | No `run_in_terminal`                                            |

\* Researcher's restriction says "except to write your own output files" — ambiguous.
\*\* Verifier contradiction detailed in F-3.

**Evidence:**

- Tool Access tables in all 9 agent definition files under `NewAgents/.github/agents/`

**Relevance:** Tool restrictions enforce separation of concerns but create coupling to the VS Code agent framework. Any overhaul must preserve or refine these access control boundaries.

---

### F-3: Verifier tool contradiction — must produce file but create_file is explicitly prohibited

**Category:** risk

The Verifier agent definition contains a direct contradiction:

1. **Output requirement:** `verification-reports/<task-id>.yaml` is a required output (verifier.agent.md line 6, line 50, line 339)
2. **Operating rule:** "You may only write to your designated output path (verification-reports/<task-id>.yaml)" (line 370)
3. **Tool restriction:** "You MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`" (lines 457–459)

The Verifier needs to create a new YAML file but has no file-creation tool available. This is an unresolvable contradiction in the current agent definition.

Similarly, the Researcher's restrictions say "You MUST NOT use `create_file`, `replace_string_in_file`, `run_in_terminal`, or any other file-modification or execution tools **except to write your own output files**." This exception-based phrasing is ambiguous — it simultaneously prohibits and permits `create_file`.

**Evidence:**

- `NewAgents/.github/agents/verifier.agent.md` line 6: Outputs include `verification-reports/<task-id>.yaml`
- `NewAgents/.github/agents/verifier.agent.md` line 370: "You may only write to your designated output path"
- `NewAgents/.github/agents/verifier.agent.md` lines 457–459: "You MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`"
- `NewAgents/.github/agents/researcher.agent.md` lines 234–235: "MUST NOT use `create_file`... except to write your own output files"

**Relevance:** Tool access contradictions will cause agent failures at runtime. Must be resolved — either grant `create_file` with scope restrictions or provide an alternative output mechanism.

---

### F-4: SQLite verification-ledger.db is the primary cross-agent coupling point — 4 writers, 3 readers

**Category:** dependency

The `anvil_checks` table in `verification-ledger.db` is the most connected shared resource in the pipeline. It functions as the source of truth for verification evidence.

**Writers** (via `run_in_terminal` with `sqlite3` CLI):

- **Orchestrator:** `CREATE TABLE` at Step 0 (DDL only)
- **Implementer:** `INSERT phase='baseline'` records (baseline capture, Step 5)
- **Verifier:** `INSERT phase='after'` records (verification cascade, Step 6)
- **Adversarial Reviewer:** `INSERT phase='review'` records (verdict SQL, Steps 3b/7)

**Readers** (via `run_in_terminal` with `sqlite3` CLI):

- **Orchestrator:** `SELECT` for 5+ evidence gate queries (Steps 3b, 6, 7)
- **Adversarial Reviewer:** `SELECT` for code review context (Step 7)
- **Knowledge Agent:** Reads verification data from `verification-reports/*.yaml` (CANNOT use `run_in_terminal`)

**Schema:** 14 columns with CHECK constraints on `phase`, `passed`, `verdict`, `severity`. 2 indexes. WAL journal mode + `busy_timeout=5000` for concurrent access.

All queries use `run_id` filtering to prevent cross-run contamination.

**Evidence:**

- `NewAgents/.github/agents/schemas.md` — SQLite Schemas section (lines 950–1070)
- `NewAgents/.github/agents/orchestrator.agent.md` — Step 0 initialization (lines 148–195)
- `NewAgents/.github/agents/orchestrator.agent.md` — Evidence Gate SQL queries (lines 594–660)
- `NewAgents/.github/agents/implementer.agent.md` — SQL INSERT for Baseline Records (lines 415–460)
- `NewAgents/.github/agents/verifier.agent.md` — SQL Evidence Recording (lines 100–160)
- `NewAgents/.github/agents/adversarial-reviewer.agent.md` — SQL INSERT Format (lines 130–150)

**Relevance:** `verification-ledger.db` is the single most critical dependency in the pipeline. Its availability determines whether evidence gating functions. Any schema change requires coordinated updates across 4 agent definitions plus `schemas.md`.

---

### F-5: pipeline_telemetry.db is created but never populated — dead dependency

**Category:** risk

The `pipeline_telemetry` table has a complete lifecycle gap:

1. **CREATED:** Orchestrator at Step 0 (`orchestrator.agent.md` lines 176–195)
2. **SCHEMA:** Defined in `schemas.md` with 11 columns, 1 index, key queries
3. **EXPECTED READER:** Knowledge Agent reads it for `pipeline_telemetry_summary` (`knowledge-agent.agent.md` line 41)
4. **NO WRITER IDENTIFIED:**
   - Orchestrator says: "You NEVER accumulate telemetry (Knowledge Agent handles this at Step 8)" (line 24)
   - Knowledge Agent is RESTRICTED from `run_in_terminal`, so it CANNOT INSERT
   - No other agent has instructions to INSERT into `pipeline_telemetry`

**Result:** The table is created empty, never populated, and the Knowledge Agent's `pipeline_telemetry_summary` in the knowledge-output will always be zero/empty.

**Evidence:**

- `NewAgents/.github/agents/orchestrator.agent.md` line 24: "You NEVER accumulate telemetry"
- `NewAgents/.github/agents/orchestrator.agent.md` lines 176–195: Step 0 creates `pipeline_telemetry` table
- `NewAgents/.github/agents/knowledge-agent.agent.md` line 41: Reads from `pipeline-telemetry.db`
- `NewAgents/.github/agents/knowledge-agent.agent.md` — Tool restrictions: "You MUST NOT use `run_in_terminal`"
- `NewAgents/.github/agents/schemas.md` lines 1064–1095: `pipeline_telemetry` table definition

**Relevance:** Pipeline telemetry is a dead feature — designed but not wired. The overhaul must either add an explicit writer or remove the table and related Knowledge Agent expectations.

---

### F-6: Review verdicts file path inconsistency between Schema 9, Adversarial Reviewer, and Designer

**Category:** risk

Three documents define conflicting output paths for review verdict files:

1. **schemas.md** Schema 9 header: `review-verdicts/<scope>.yaml` — ONE file per scope
2. **Adversarial Reviewer** output table: `review-verdicts/<scope>-<model>.yaml` — THREE files per scope (one per model)
3. **Designer** input table (revision mode): `review-verdicts/design.yaml` — expects a single aggregated file

No agent is responsible for aggregating per-model verdict files into the single `review-verdicts/<scope>.yaml` that the Designer expects. The Orchestrator does not produce files. This is a **missing aggregation step**.

**Evidence:**

- `NewAgents/.github/agents/schemas.md` line 761: "YAML verdict summary at `review-verdicts/<scope>.yaml`"
- `NewAgents/.github/agents/adversarial-reviewer.agent.md` — Output Files table: `review-verdicts/<scope>-<model>.yaml`
- `NewAgents/.github/agents/designer.agent.md` — Revision Mode Inputs: `review-verdicts/design.yaml`

**Relevance:** This file path mismatch will cause Designer revision mode to fail — it will look for `review-verdicts/design.yaml` which no agent produces. Must be resolved by clarifying the output convention or adding an aggregation step.

---

### F-7: Deterministic file path dependency map — 20+ paths agents expect to exist

**Category:** dependency

Agents reference specific file paths deterministically. All paths are relative to `docs/feature/<feature-slug>/`. A failure to create any required path blocks downstream agents.

**Pipeline Inputs:**

- `initial-request.md` — Required by Researcher (×4), Spec, Designer

**Step 1 — Researcher (×4):**

- `research/architecture.yaml` + `research/architecture.md`
- `research/impact.yaml` + `research/impact.md`
- `research/dependencies.yaml` + `research/dependencies.md`
- `research/patterns.yaml` + `research/patterns.md`

**Step 2 — Spec:**

- `spec-output.yaml`, `feature.md`

**Step 3 — Designer:**

- `design-output.yaml`, `design.md`

**Step 3b — Adversarial Reviewer (×3):**

- `review-findings/design-<model>.md` (×3 files)
- `review-verdicts/design-<model>.yaml` (×3 files, see F-6 inconsistency)

**Step 4 — Planner:**

- `plan-output.yaml`, `plan.md`, `tasks/task-*.yaml` (N files)

**Step 5 — Implementer:**

- `implementation-reports/task-*.yaml` (N files), code/test files (variable)

**Step 6 — Verifier:**

- `verification-reports/task-*.yaml` (N files)

**Step 7 — Adversarial Reviewer (×3):**

- `review-findings/code-<model>.md` (×3), `review-verdicts/code-<model>.yaml` (×3)

**Step 8 — Knowledge Agent:**

- `knowledge-output.yaml`, `decisions.yaml`, `evidence-bundle.md`

**Databases (Step 0):**

- `verification-ledger.db`, `pipeline-telemetry.db`

**Git Artifacts:**

- `pipeline-baseline-{run_id}` (tag)

**Recovery:** The orchestrator reconstructs pipeline state by scanning these directories with `list_dir` (EC-5 — Pipeline State Recovery).

**Evidence:**

- `NewAgents/.github/agents/schemas.md` — Output paths per schema (lines 113, 197, 244, 336, 456, 511, 619, 760, 828)
- `NewAgents/.github/agents/orchestrator.agent.md` — Pipeline State Recovery EC-5 (lines 103–120)
- `NewAgents/.github/agents/orchestrator.agent.md` — Pre-implementation git tag (lines 395–400)

**Relevance:** File path conventions are a critical coupling contract. Any path change requires coordinated updates across producer and all consumer agent definitions. The recovery mechanism (EC-5) depends on these exact paths existing at known locations.

---

### F-8: External tool dependencies — sqlite3 CLI and git are hard requirements

**Category:** dependency

The pipeline depends on external tools available via `run_in_terminal`:

**Hard Dependencies (pipeline fails without these):**

- **sqlite3 CLI:** Used by Orchestrator (DDL + SELECT), Implementer (INSERT), Verifier (INSERT + SELECT), Adversarial Reviewer (INSERT). All SQL operations use: `sqlite3 <db-path> "<SQL>"`. Unavailability = no evidence gating = pipeline cannot function.
- **git:** Used by Orchestrator (status, rev-parse, show, tag, add, commit), Implementer (add, checkout for revert), Verifier (show for baseline cross-check), Adversarial Reviewer (diff --staged for code review). Unavailability = no baseline verification, no staging, no commit.

**Soft Dependencies (gracefully degraded):**

- Language-specific build tools — detected dynamically from config files (package.json, Cargo.toml, go.mod, etc.)
- Language-specific test runners — same dynamic detection
- Language-specific linters/type-checkers — optional Tier 2 checks

**Deliberately Absent from NewAgents (present in Anvil):**

- Context7 MCP tools (`context7-resolve-library-id`, `context7-query-docs`)
- `session_store` database for cross-session recall
- `report_intent` for progress display
- GitHub MCP tools for issue/PR linking

**VS Code Framework Dependencies:**

- `agent/runSubagent` — core dispatch mechanism (Orchestrator only)
- `ask_questions` — interactive mode approval gates (Orchestrator, Spec)
- `memory`/`store_memory` — cross-session knowledge (Orchestrator, Knowledge Agent)
- `get_errors` — IDE diagnostics (Implementer, Verifier)
- `ide-get_diagnostics` — enhanced IDE diagnostics (Verifier)
- `list_code_usages` — reference finder (Implementer, with `grep_search` fallback)

**Evidence:**

- `NewAgents/.github/agents/implementer.agent.md` line 429: `sqlite3 verification-ledger.db`
- `NewAgents/.github/agents/orchestrator.agent.md` lines 60–70: `run_in_terminal` allowed operations
- `NewAgents/.github/agents/orchestrator.agent.md` lines 128–135: git hygiene checks
- `NewAgents/.github/agents/verifier.agent.md` lines 220–280: dynamic tool detection
- `Anvil/anvil.agent.md` lines 364–367: Context7 MCP reference (not in NewAgents)

**Relevance:** `sqlite3` and `git` are hard runtime requirements that cannot be gracefully degraded. If the overhaul targets environments where `sqlite3` may not be available, an alternative evidence storage mechanism is needed.

---

### F-9: run_id serves as universal cross-agent namespace filter

**Category:** dependency

The `run_id` parameter (ISO 8601 timestamp, e.g., `2026-02-26T14:30:00Z`) is generated by the Orchestrator at Step 0 and passed to every downstream agent. It serves as the universal namespace filter:

- **SQL:** Every evidence gate query filters `WHERE run_id = '{run_id}' AND ...`
- **Git:** Baseline tag includes run_id: `pipeline-baseline-{run_id}`
- **YAML:** Every agent output includes `run_id` in the payload or header context
- **Dispatch:** Passed to Implementer, Verifier, Adversarial Reviewer as an explicit parameter

If `run_id` is corrupted, mismatched, or not propagated correctly, evidence gates will return zero results and the pipeline will stall or produce false negatives.

**Evidence:**

- `NewAgents/.github/agents/orchestrator.agent.md` line 85: `run_id` in `pipeline_state`
- `NewAgents/.github/agents/orchestrator.agent.md` line 145: `run_id` generation at Step 0
- `NewAgents/.github/agents/schemas.md` — Evidence Gate Queries (lines 1000–1060): all use `run_id` filter
- `NewAgents/.github/agents/verifier.agent.md` line 37: `run_id` as orchestrator-provided parameter

**Relevance:** `run_id` is a single point of failure for cross-agent data integrity. Any overhaul must ensure reliable propagation.

---

### F-10: No circular agent dependencies — pipeline is strictly linear with bounded feedback loops

**Category:** dependency

The pipeline has NO circular dependencies between agents. The execution flow is strictly: Step 0 → 1 → 2 → 3 → 3b → 4 → 5 → 6 → 7 → 8 → 9.

**Feedback Loops (all bounded):**

1. **Design revision:** Step 3b → Step 3 → Step 3b — Max 1 design revision loop
2. **Implementation-verification:** Step 5 → 6 → Planner (replan) → 5 → 6 — Max 3 iterations
3. **Code review cycling:** Step 7 → 5 (fix) → 6 (verify) → 7 (re-review) — Max 2 review rounds

After exhausting iteration budget, the pipeline proceeds with documented findings and `Confidence: Low`.

Agent references are unidirectional — no agent reads the output of an agent that reads its output (except through bounded orchestrator-mediated loops).

**Evidence:**

- `NewAgents/.github/agents/dispatch-patterns.md` — Pattern B with max 3 iterations (lines 60–90)
- `NewAgents/.github/agents/dispatch-patterns.md` — Code Review Cycling max 2 rounds (lines 95–105)
- `NewAgents/.github/agents/orchestrator.agent.md` — Decision Table (lines 570–610)
- `NewAgents/.github/agents/orchestrator.agent.md` — Retry Budgets (lines 668–720)

**Relevance:** The absence of circular dependencies and bounded feedback loops ensure the pipeline always terminates. This property should be preserved in any overhaul.

---

### F-11: Completion contract is the universal integration point — 3-state protocol drives all routing

**Category:** dependency

Every agent produces a completion block conforming to Schema 1 (`completion-contract`). The Orchestrator reads ONLY this block for routing decisions.

**States:** `DONE` | `NEEDS_REVISION` | `ERROR`

| Agent                | Returns NEEDS_REVISION? |
| -------------------- | ----------------------- |
| Researcher           | No                      |
| Spec                 | No                      |
| Designer             | No                      |
| Planner              | **Yes**                 |
| Implementer          | No                      |
| Verifier             | **Yes**                 |
| Adversarial Reviewer | No                      |
| Knowledge Agent      | No                      |

Only Planner and Verifier can return `NEEDS_REVISION`. Revision routing for other agents is handled by the Orchestrator's decision table based on evidence gates, not agent self-reporting.

**Evidence:**

- `NewAgents/.github/agents/schemas.md` — Schema 1: completion-contract (lines 58–95)
- `NewAgents/.github/agents/orchestrator.agent.md` — Decision Table (lines 570–610)
- `NewAgents/.github/agents/verifier.agent.md` — Completion Contract (lines 340–365)
- `NewAgents/.github/agents/planner.agent.md` — Completion Contract (lines 308–320)

**Relevance:** The completion contract is the minimum viable integration surface between agents. Changing the routing protocol requires updating all agent completion contracts.

---

### F-12: relevant_context pointers create controlled coupling between Planner and Implementer

**Category:** dependency

The Planner produces `tasks/*.yaml` files with a `relevant_context` block containing exact section pointers into upstream outputs:

```yaml
relevant_context:
  design_sections:
    - "design-output.yaml#payload.decisions[id='D-8']"
  spec_requirements:
    - "spec-output.yaml#payload.functional_requirements[id='FR-4']"
  files_to_modify:
    - path: "src/auth/handler.ts"
      risk: "🔴"
```

Rules:

- Every task MUST have ≥1 `design_sections` + ≥1 `spec_requirements` pointer
- Implementer reads ONLY referenced sections (trusts Planner curation)
- If a pointer references a non-existent section, Implementer logs warning and proceeds
- Late-pipeline agents (Verifier, Adversarial Reviewer) are NOT bound by `relevant_context`

**Evidence:**

- `NewAgents/.github/agents/planner.agent.md` — Relevant Context Mechanism (lines 145–190)
- `NewAgents/.github/agents/implementer.agent.md` — Relevant Context Consumption (lines 315–340)
- `NewAgents/.github/agents/schemas.md` — Schema 6 task-schema (lines 455–510)

**Relevance:** The `relevant_context` mechanism trades completeness for efficiency. Quality of Planner output directly determines Implementer effectiveness.

---

### F-13: ask_questions creates mode-dependent interactive coupling

**Category:** dependency

Two agents depend on the `ask_questions` tool for interactive mode:

- **Orchestrator:** 3 decision points (Step 0 pushback, Step 1a post-research, Step 4a post-planning)
- **Spec Agent:** 1 decision point (Step 2 pushback)

In autonomous mode, `ask_questions` is bypassed entirely with default options auto-selected. This creates a bifurcated execution path: interactive mode has 4 human-in-the-loop gates; autonomous mode has zero.

The `approval_mode` (autonomous|interactive) is tracked in orchestrator pipeline state but is NOT explicitly passed as a parameter to Spec — it's embedded in the dispatch context.

**Evidence:**

- `NewAgents/.github/agents/orchestrator.agent.md` lines 17, 138, 257, 386: `ask_questions` usage
- `NewAgents/.github/agents/spec.agent.md` lines 158–195: Interactive mode pushback
- `NewAgents/.github/agents/orchestrator.agent.md` line 83: `approval_mode` in pipeline state

**Relevance:** The interactive/autonomous mode split affects pipeline execution flow. Any overhaul must maintain both modes or explicitly deprecate one.

---

### F-14: Anvil's Context7 and session_store dependencies are deliberately absent from NewAgents

**Category:** dependency

Comparing Anvil to NewAgents reveals deliberately removed external dependencies:

| Anvil Dependency         | NewAgents Equivalent  | Status               |
| ------------------------ | --------------------- | -------------------- |
| Context7 MCP tools       | Codebase search only  | Removed              |
| `session_store` database | VS Code `memory` tool | Replaced             |
| `ask_user`               | `ask_questions`       | Renamed/restructured |
| `report_intent`          | (none)                | Removed              |
| GitHub MCP tools         | (none)                | Removed              |

These removals reduce external coupling but also remove documentation lookup and session recall capabilities.

**Evidence:**

- `Anvil/anvil.agent.md` lines 364–367: Context7 MCP tool references
- `Anvil/anvil.agent.md` lines ~130–155: `session_store` query
- `NewAgents/.github/agents/` — no agent references Context7, `session_store`, `report_intent`, or GitHub MCP

**Relevance:** The deliberate removal of external MCP dependencies simplifies the pipeline. The overhaul should decide whether to maintain this simplification or selectively restore capabilities.

---

### F-15: Schema version coupling — all agents hardcode schema_version 1.0 with no version negotiation

**Category:** risk

Every agent output must include `schema_version: "1.0"` in the common header. The Schema Evolution Strategy defines additive (minor bump) and breaking (major bump) policies. However:

- **No consumer validates** `schema_version`
- The orchestrator "NEVER performs schema validation" (`orchestrator.agent.md` line 24)
- Agents self-validate via Self-Verification checklists (no cross-agent checking)
- 4 schemas were removed from v1: `pipeline-manifest`, `approval-request`, `decision-record`, `verification-record`

**Risk:** A schema mismatch between producer and consumer would cause silent failures — the consumer reads a field that doesn't exist or has changed type, with no error message.

**Evidence:**

- `NewAgents/.github/agents/schemas.md` — Schema Evolution Strategy (lines 1240–1284)
- `NewAgents/.github/agents/schemas.md` — Schema Version Requirement (lines 12–14)
- `NewAgents/.github/agents/orchestrator.agent.md` line 24: "You NEVER perform schema validation"

**Relevance:** The lack of version negotiation means schema evolution requires careful manual coordination. An overhaul could add runtime version checking or maintain the current trust-based approach.

---

## Source Files Examined

- `NewAgents/.github/agents/schemas.md`
- `NewAgents/.github/agents/orchestrator.agent.md`
- `NewAgents/.github/agents/dispatch-patterns.md`
- `NewAgents/.github/agents/researcher.agent.md`
- `NewAgents/.github/agents/spec.agent.md`
- `NewAgents/.github/agents/designer.agent.md`
- `NewAgents/.github/agents/planner.agent.md`
- `NewAgents/.github/agents/implementer.agent.md`
- `NewAgents/.github/agents/verifier.agent.md`
- `NewAgents/.github/agents/adversarial-reviewer.agent.md`
- `NewAgents/.github/agents/knowledge-agent.agent.md`
- `NewAgents/.github/agents/severity-taxonomy.md`
- `Anvil/anvil.agent.md`

## Research Metadata

- **Confidence Level:** high
- **Coverage Estimate:** Comprehensive — all 12 NewAgents system files fully read; Anvil reference partially read (lines 1–200 of 420, sufficient for dependency comparison). All inter-agent data flows, tool contracts, file paths, SQLite schemas, and external dependencies mapped.
- **Gaps:** Anvil lines 200–420 were not read (low impact — only needed for full feature comparison, not dependency mapping). Forge system (28 files) not read — out of scope for NewAgents dependency analysis but may contain additional patterns. Runtime behavior verification (actual agent execution traces) was not performed — findings are based on agent definitions only.
