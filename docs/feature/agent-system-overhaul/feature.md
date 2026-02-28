# Agent System Overhaul — Feature Specification

**Feature Slug:** `agent-system-overhaul`
**Status:** Specification Complete
**Date:** 2026-02-27
**Spec Agent Instance:** spec-agent-system-overhaul

---

## 1. Title & Short Summary

**Agent System Overhaul** — A comprehensive refactoring of the 9-agent NewAgents pipeline to fix broken features (parallel execution, multi-model review, telemetry population), restore missing Forge capabilities (artifact evaluations, post-mortem analysis, instruction files), enforce global operating rules (terminal-only testing, no-file-redirect), and reduce agent instruction file bloat through shared reference document extraction.

This is an architecture-level overhaul, not a small fix. The user has authorized full refactoring of agent definitions.

---

## 2. Background & Context

### 2.1 System Lineage

Three agent systems exist in the workspace:

| System        | Agents                   | Communication           | Verification                                   |
| ------------- | ------------------------ | ----------------------- | ---------------------------------------------- |
| **Anvil**     | 1 monolithic (420 lines) | SQL-only (anvil_checks) | SQL evidence gates                             |
| **Forge**     | 24 specialized           | Markdown memory files   | Prose-based routing                            |
| **NewAgents** | 9 parameterized          | Typed YAML + SQL        | SQL evidence gates + YAML completion contracts |

NewAgents (the current system) was built by consolidating Forge's 24 agents into 9 through parameterization (e.g., 4 CT agents → 3 adversarial-reviewer instances). It adopted Anvil's SQL evidence gating and added typed YAML schemas for deterministic routing. The root `.github/agents/` directory mirrors NewAgents.

### 2.2 Research Sources

This specification is grounded in 52 findings across 4 research outputs:

- **Architecture** (14 findings): [research/architecture.yaml](research/architecture.yaml) — Agent inventory, communication patterns, parallel execution, pipeline step evolution
- **Impact** (11 findings): [research/impact.yaml](research/impact.yaml) — Missing features, capability gaps, architectural regressions
- **Dependencies** (15 findings): [research/dependencies.yaml](research/dependencies.yaml) — Inter-agent data flow, tool contracts, file path coupling, SQLite dependencies
- **Patterns** (12 findings): [research/patterns.yaml](research/patterns.yaml) — Agent file sizes, anti-drift anchors, self-verification, retry patterns, evaluation system

### 2.3 Real Pipeline Output Evidence

Observations from inspecting `outputFromNewAgentsAfterRun/ui-bug-fixes/` (a real pipeline run):

1. **Multi-model confirmed broken**: All 6 review verdict files show `reviewer_model: "claude-opus-4.6"`. The model parameter has no effect.
2. **Pipeline telemetry empty**: `pipeline-telemetry.db` has zero rows despite being created at Step 0.
3. **Verification uses VS Code tools**: `verification-ledger.db` entries show `tool: "get_errors"` and `tool: "runTests"`.
4. **Design review loop incomplete**: Round 1 had 1 approve + 2 needs_revision, but no round 2 exists in SQL.
5. **Implementation reports are clean**: Proper YAML schemas and completion contracts work well.
6. **Evidence bundle format is good**: Well-structured with task summaries, review verdicts, known issues.

---

## 3. Pushback Analysis

Before producing this specification, 3 concerns were identified:

### Concern 1 — Critical: Multi-Model Review is Platform-Infeasible

**Issue:** REQ-2 asks for multi-model adversarial review ("different model" per reviewer). VS Code's `runSubagent` API does not respect the `model` parameter for custom `.agent.md` agents. Real output confirms all reviewers used `claude-opus-4.6`.

**Recommendation:** Reframe as "multi-perspective review" using prompt diversity (distinct system prompt personas). Model diversity becomes a Phase 2 enhancement if VS Code adds model routing support. Specification reflects this in FR-2 and DP-1.

### Concern 2 — Major: Parallel Execution is Platform-Controlled

**Issue:** REQ-1 requests fixing sequential agent execution. Research confirms the orchestrator already dispatches with parallel intent — actual concurrency depends on VS Code's runtime, not agent definitions.

**Recommendation:** Scope to "investigate and maximize parallelism within platform constraints." Include VS Code API research as an explicit task. Document platform limitations if true parallelism is unachievable.

### Concern 3 — Major: Evaluation System is High-Effort

**Issue:** REQ-5 asks to restore the artifact evaluation/rating system. This requires modifying 7+ agents and adding a consumer. LLM-generated evaluations may be uninformative.

**Recommendation:** Phase the rollout — initially require evaluations from 3 key agents (implementer, verifier, adversarial reviewer). Monitor quality before expanding.

**Resolution:** Proceeding with specification incorporating all three recommendations.

---

## 4. Scope

### 4.1 In Scope

- Parallel execution investigation and optimization within VS Code runSubagent constraints
- Multi-perspective adversarial review with prompt diversity (replacing broken multi-model)
- Review restructure: all reviewer instances review all categories (security, architecture, correctness)
- YAML+MD dual output audit and rationalization
- Artifact evaluation system using SQLite storage
- Pipeline telemetry population and post-mortem analysis
- Terminal-only testing enforcement (global rule)
- Global no-file-redirect rule
- Conditional Context7 MCP integration
- Agent file size reduction (target 250-350 lines) via shared reference docs
- Instruction file infrastructure (`.github/copilot-instructions.md` + `.github/instructions/`)
- Expanded SQLite usage for evaluations, metrics, instruction tracking
- Verifier tool contradiction resolution
- Review verdict file path consistency fix
- VS Code runSubagent API capability research
- Instruction file management via Knowledge Agent governed updates
- Frontmatter standardization across all agent files

### 4.2 Out of Scope

- Changing the VS Code agent runtime or runSubagent API implementation
- Adding new VS Code extensions or modifying VS Code source code
- GitHub MCP integration (Anvil feature — not requested)
- Session store / cross-session recall database (replaced by VS Code memory tool)
- Changing the 10-step pipeline structure (Steps 0-9 retained)
- Restoring Forge's markdown memory system (replaced by typed YAML + SQL)
- Restoring Forge's 24-agent count (parameterized approach retained)
- Introducing circular dependencies between agents
- Breaking the bounded feedback loop invariant

---

## 5. Functional Requirements

### FR-1: Parallel Execution Investigation & Optimization

**Priority:** Must-Have | **Refs:** REQ-1, REQ-8

Investigate why agents execute sequentially despite the orchestrator dispatching via `runSubagent` with parallel intent. Research VS Code's `runSubagent` API to determine: (a) whether multiple concurrent `runSubagent` calls are supported, (b) if there's a specific invocation pattern enabling true concurrency, (c) platform limitations on parallel subagent execution. Implement whatever dispatch pattern maximizes actual parallelism.

**Acceptance Criteria:**

- **AC-1.1:** Research document exists documenting VS Code runSubagent concurrency behavior with evidence from API investigation or runtime testing.
- **AC-1.2:** Orchestrator dispatch pattern is updated to use whatever mechanism maximizes parallelism.
- **AC-1.3:** If platform limitation prevents true parallelism, a documented finding explains the constraint with evidence, and `dispatch-patterns.md` reflects reality.

---

### FR-2: Multi-Perspective Adversarial Review

**Priority:** Must-Have | **Refs:** REQ-2

Replace the broken multi-model approach with multi-perspective review. Each reviewer instance receives a distinct system prompt persona shaping its review lens (e.g., security-focused pessimist, architecture purist, pragmatic correctness checker). Diversity comes from prompt engineering, not model differences.

**Acceptance Criteria:**

- **AC-2.1:** Each adversarial reviewer instance receives a distinct `review_perspective` parameter that meaningfully changes review behavior.
- **AC-2.2:** Real review output from different perspectives produces measurably different findings.
- **AC-2.3:** The adversarial-reviewer.agent.md includes ≥3 distinct perspective definitions with different review priorities, heuristics, and severity thresholds.
- **AC-2.4:** If a working multi-model mechanism is discovered, a documented enhancement path exists.

---

### FR-3: All-Category Review Coverage

**Priority:** Must-Have | **Refs:** REQ-3

Restructure adversarial review so every reviewer instance reviews ALL categories (security, architecture, correctness) rather than each having a single `review_focus`. The `review_perspective` parameter controls the reviewer's lens and priorities, but each reviewer covers all three domains.

**Acceptance Criteria:**

- **AC-3.1:** Each reviewer instance produces findings across all three categories (verified in review-findings output).
- **AC-3.2:** Review verdict SQL INSERT includes category-specific sub-verdicts or review-findings has explicit sections per category.
- **AC-3.3:** Orchestrator evidence gate validates each reviewer submitted findings for all three categories.

---

### FR-4: YAML+MD Dual Output Rationalization

**Priority:** Should-Have | **Refs:** REQ-4

Audit and rationalize the dual output pattern. Machine-consumed artifacts (research outputs, implementation reports, verification reports, review verdicts) become YAML-only. Human-facing specification documents (feature.md, design.md, plan.md, evidence-bundle.md) retain MD companions.

**Acceptance Criteria:**

- **AC-4.1:** Each output type is classified as YAML-only or YAML+MD in schemas.md with rationale.
- **AC-4.2:** Agents producing YAML-only artifacts no longer generate MD companion files.
- **AC-4.3:** Human-facing documents are still produced.
- **AC-4.4:** No downstream agent references a removed MD file.

---

### FR-5: Artifact Evaluation System with SQLite Storage

**Priority:** Must-Have | **Refs:** REQ-5, REQ-11

Restore artifact evaluation using SQLite. Define an `artifact_evaluations` table with columns for evaluator_agent, artifact_path, usefulness_score (1-10), clarity_score (1-10), missing_information, inaccuracies, run_id, timestamp. Phase 1 agents: implementer, verifier, adversarial reviewer.

**Acceptance Criteria:**

- **AC-5.1:** `artifact_evaluations` table schema defined in schemas.md with all specified columns.
- **AC-5.2:** Phase 1 agents include SQL INSERT instructions for artifact evaluations.
- **AC-5.3:** Knowledge Agent reads evaluations via SQL and includes quantitative summary in output (mean scores, worst-rated artifacts, common missing_information).
- **AC-5.4:** Evaluation data persists in SQLite, queryable by run_id, evaluator_agent, and artifact_path.

---

### FR-6: Pipeline Telemetry Population & Post-Mortem Consumer

**Priority:** Must-Have | **Refs:** REQ-5, REQ-11

Fix the dead `pipeline_telemetry.db` by having the orchestrator INSERT a record after each agent dispatch completes. The Knowledge Agent produces a post-mortem section with agent reliability metrics, bottleneck identification, recurring issues, and per-step timing.

**Acceptance Criteria:**

- **AC-6.1:** Orchestrator inserts a row into `pipeline_telemetry` after each dispatch with timing and status.
- **AC-6.2:** After a full pipeline run, `pipeline_telemetry` contains ≥1 row per dispatched agent instance.
- **AC-6.3:** Knowledge Agent output includes `pipeline_telemetry_summary` with total_dispatches, errors, per-step timing, slowest step, retry summary.
- **AC-6.4:** Post-mortem identifies top 3 bottleneck steps and any agents requiring retries.

---

### FR-7: Terminal-Only Testing Enforcement

**Priority:** Must-Have | **Refs:** REQ-6

Enforce all test execution via `run_in_terminal` with CLI commands (`dotnet test`, `npm test`, `pytest`, etc.) as a GLOBAL rule. No VS Code Testing API usage. IDE diagnostics via `get_errors` are permitted for compile-time checks only.

**Acceptance Criteria:**

- **AC-7.1:** Global operating rule in shared reference doc mandates terminal-only test execution.
- **AC-7.2:** Verifier Tier 2 explicitly uses `run_in_terminal` for test execution.
- **AC-7.3:** Implementer self-fix loop uses `run_in_terminal` for test verification.
- **AC-7.4:** After pipeline run, `verification-ledger.db` shows `tool='run_in_terminal'` for all test check_names.

---

### FR-8: Global No-File-Redirect Rule

**Priority:** Must-Have | **Refs:** REQ-6

Promote the no-file-redirect prohibition to a global rule for ALL agents with `run_in_terminal` access. No `command > output.txt`, `command | tee file`, or similar patterns. All output read directly from terminal.

**Acceptance Criteria:**

- **AC-8.1:** Global rule exists in shared reference document.
- **AC-8.2:** Every agent with `run_in_terminal` references the global rules document.
- **AC-8.3:** No agent contains inline no-file-redirect rules (deduplicated to shared doc).

---

### FR-9: Conditional Context7 MCP Integration

**Priority:** Nice-to-Have | **Refs:** REQ-7

Add Context7 as a conditional capability. When available, agents use `context7-resolve-library-id` and `context7-query-docs` for documentation lookup. When unavailable, agents skip with no errors. Instructions in shared reference doc only.

**Acceptance Criteria:**

- **AC-9.1:** Context7 instructions exist in a shared reference document.
- **AC-9.2:** Implementer and researcher reference Context7 with conditional check.
- **AC-9.3:** Unavailable Context7 does not cause agent error.
- **AC-9.4:** Instructions not inlined in individual agent files.

---

### FR-10: Agent File Size Reduction via Shared Reference Docs

**Priority:** Should-Have | **Refs:** REQ-9, REQ-10

Reduce agent files to 250-350 lines (orchestrator ≤500) by extracting shared patterns into reference docs. Create `.github/copilot-instructions.md` for global rules. Extract: operating rules, tool access matrix, self-verification common items, SQL templates, Context7 instructions.

**Acceptance Criteria:**

- **AC-10.1:** No agent file (except orchestrator) exceeds 350 lines. Orchestrator ≤500 lines.
- **AC-10.2:** ≥3 new shared reference documents exist.
- **AC-10.3:** Each agent references shared docs via section pointers.
- **AC-10.4:** `.github/copilot-instructions.md` exists with global rules.
- **AC-10.5:** No shared pattern duplicated across multiple agent files.

---

### FR-11: Verifier Tool Contradiction Resolution

**Priority:** Must-Have | **Refs:** REQ-9

Resolve the contradiction where verifier must produce `verification-reports/<task-id>.yaml` but `create_file` is restricted. Grant `create_file` with scope restriction. Similarly clarify researcher's ambiguous exception.

**Acceptance Criteria:**

- **AC-11.1:** Verifier's tool access allows `create_file` with scope restriction to `verification-reports/*.yaml`.
- **AC-11.2:** Researcher's tool access explicitly allows `create_file` for `research/*.yaml` and `research/*.md`.
- **AC-11.3:** No agent has unresolvable output/tool contradictions.

---

### FR-12: Review Verdict File Path Consistency

**Priority:** Must-Have | **Refs:** REQ-9

Fix the path mismatch between adversarial reviewer output (`review-verdicts/<scope>-<model>.yaml`) and designer input (`review-verdicts/design.yaml`). Standardize on per-reviewer files with glob-based consumption.

**Acceptance Criteria:**

- **AC-12.1:** Output path in adversarial-reviewer.agent.md matches input path in designer.agent.md.
- **AC-12.2:** schemas.md Schema 9 path is consistent with actual producer output.
- **AC-12.3:** Pipeline produces verdict files at paths consumers expect.

---

### FR-13: Deterministic Schema & Contract Enforcement

**Priority:** Must-Have | **Refs:** REQ-10

Strengthen deterministic properties: explicit field types in all schemas, routing matrix documenting which agents return which statuses, templated evidence gate queries, basic schema_version presence checking.

**Acceptance Criteria:**

- **AC-13.1:** Routing matrix table in schemas.md shows agent × completion status.
- **AC-13.2:** Every schema field has explicit type and required/optional annotation.
- **AC-13.3:** Evidence gate SQL queries use shared templates (not ad-hoc inline).
- **AC-13.4:** Every agent output includes `schema_version: "1.0"` (checked in self-verification).

---

### FR-14: Expanded SQLite Usage

**Priority:** Should-Have | **Refs:** REQ-11

Expand SQLite to cover: artifact evaluations (FR-5), pipeline telemetry (FR-6), instruction update tracking. All schemas documented in schemas.md.

**Acceptance Criteria:**

- **AC-14.1:** SQLite covers ≥3 data domains: verification, evaluations, telemetry.
- **AC-14.2:** All table schemas in schemas.md with types, constraints, indexes, key queries.
- **AC-14.3:** `instruction_updates` table tracks instruction file modifications.
- **AC-14.4:** All databases use WAL mode + busy_timeout for concurrency safety.

---

### FR-15: Instruction File Infrastructure

**Priority:** Should-Have | **Refs:** REQ-5, REQ-8

Create `.github/copilot-instructions.md` and `.github/instructions/` directory. Knowledge Agent's governed update mechanism targets these files. Interactive mode requires approval; autonomous mode logs but doesn't auto-apply.

**Acceptance Criteria:**

- **AC-15.1:** `.github/copilot-instructions.md` exists with global operating rules.
- **AC-15.2:** `.github/instructions/` directory exists with ≥1 domain instruction file.
- **AC-15.3:** Knowledge Agent file paths match actual file locations.
- **AC-15.4:** Instruction updates tracked in SQLite (FR-14).

---

### FR-16: Frontmatter Standardization

**Priority:** Should-Have | **Refs:** REQ-9

Standardize YAML frontmatter across all agent files. All agents must have `name` and `description` frontmatter for VS Code runtime discovery.

**Acceptance Criteria:**

- **AC-16.1:** Every `.agent.md` file has YAML frontmatter with `name` and `description`.
- **AC-16.2:** Frontmatter field names are consistent across all agents.

---

## 6. Non-Functional Requirements

| ID    | Title                 | Description                                     | Metric                                                                                              |
| ----- | --------------------- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| NFR-1 | Agent File Size       | Agent files must stay within line count targets | All agents ≤350 lines (orchestrator ≤500). Measured by `wc -l`.                                     |
| NFR-2 | Pipeline Determinism  | Routing decisions based on typed data only      | Zero routing decisions from unstructured text. Gates expressible as SQL COUNT or YAML field checks. |
| NFR-3 | Termination Guarantee | Pipeline always terminates                      | Design loop: max 1. Implementation loop: max 3. Review: max 2. No unbounded loops.                  |
| NFR-4 | Graceful Degradation  | Optional deps degrade gracefully                | Pipeline completes without Context7. Fails fast at Step 0 if sqlite3/git unavailable.               |
| NFR-5 | Evidence Integrity    | All evidence recorded in SQL before claims      | Every gate returns ≥1 row after verification. Zero false positives via run_id filtering.            |
| NFR-6 | Maintainability       | Shared patterns defined once                    | Zero duplicated operating rules across agent files.                                                 |
| NFR-7 | Schema Compatibility  | Schema evolution follows documented strategy    | All changes versioned with rationale. No silent field removal.                                      |

---

## 7. Constraints & Assumptions

### Constraints

1. VS Code `runSubagent` is the only dispatch mechanism
2. `sqlite3` CLI and `git` must be in system PATH
3. Maximum 4 concurrent subagent invocations per wave
4. All outputs conform to `schema_version: "1.0"`
5. Pipeline step structure (Steps 0-9) is retained
6. Root `.github/agents/` is the deployment target
7. Context7 availability cannot be assumed (MCP-dependent)
8. Agent definitions must not be modified at runtime
9. All feedback loops have hard iteration limits

### Assumptions

1. VS Code's `runSubagent` API continues to be the supported dispatch mechanism
2. The `ask_questions` tool continues to support structured multiple-choice prompts
3. VS Code's `memory` tool provides persistent cross-session knowledge storage
4. Agent definitions in `.agent.md` format remain the VS Code convention for custom agents
5. The user's workspace has `sqlite3` and `git` available in PATH

---

## 8. Acceptance Criteria Summary

All 16 functional requirements have testable acceptance criteria (40 total ACs). Key acceptance gates:

| Gate                 | Criteria                | Method                                           |
| -------------------- | ----------------------- | ------------------------------------------------ |
| Parallel execution   | AC-1.1 through AC-1.3   | Inspection of research doc + runtime observation |
| Review diversity     | AC-2.1 through AC-2.4   | Output comparison across reviewer instances      |
| Review coverage      | AC-3.1 through AC-3.3   | SQL evidence gate + output inspection            |
| Evaluation system    | AC-5.1 through AC-5.4   | SQL query verification                           |
| Telemetry population | AC-6.1 through AC-6.4   | SQL COUNT query on pipeline_telemetry            |
| Terminal testing     | AC-7.1 through AC-7.4   | SQL query on tool column in anvil_checks         |
| Agent sizes          | AC-10.1                 | `wc -l` on all .agent.md files                   |
| Tool contradictions  | AC-11.1 through AC-11.3 | Inspection of tool access tables                 |
| Path consistency     | AC-12.1 through AC-12.3 | Cross-reference producer/consumer paths          |

---

## 9. Edge Cases & Error Handling

| ID    | Input/Condition                                 | Expected Behavior                                   | Severity if Missed |
| ----- | ----------------------------------------------- | --------------------------------------------------- | ------------------ |
| EC-1  | Context7 MCP not configured                     | Agent logs unavailability and skips; no error       | Medium             |
| EC-2  | sqlite3 CLI not in PATH                         | Orchestrator fails fast at Step 0 with diagnostic   | Critical           |
| EC-3  | SQLITE_BUSY under parallel writes               | Retry 3x with exponential backoff (1s, 2s, 4s)      | High               |
| EC-4  | Reviewer covers only 1 of 3 categories          | Evidence gate detects; triggers retry               | High               |
| EC-5  | Evaluation scores uniformly high (no signal)    | Knowledge Agent notes low variance as concern       | Low                |
| EC-6  | Agent file exceeds 350-line target              | Designer extracts content to shared docs            | Medium             |
| EC-7  | Pipeline resume with partial SQLite data        | run_id filtering isolates in-progress records       | Medium             |
| EC-8  | Instruction update conflicts with existing rule | Safety filter rejects; logged in autonomous mode    | High               |
| EC-9  | Verdict path mismatch at runtime                | Consumer reports ERROR with missing file diagnostic | Critical           |
| EC-10 | VS Code API reveals working parallel dispatch   | Dispatch-patterns.md updated; orchestrator adopts   | Medium             |

---

## 10. User Stories / Flows

### Story 1: Standard Feature Pipeline Run

A developer invokes the orchestrator with a feature request. The orchestrator:

1. Initializes SQLite databases (verification-ledger.db, pipeline-telemetry.db) at Step 0
2. Dispatches 4 researchers in parallel (maximum parallelism within platform constraints)
3. After each dispatch, INSERTs a telemetry record into pipeline_telemetry
4. Runs approval gate (interactive) or auto-proceeds (autonomous)
5. Dispatches spec, designer, planner sequentially
6. Dispatches 3 adversarial reviewers with distinct perspectives — each reviews all 3 categories
7. Evidence gate validates all reviewers submitted all-category findings with zero blockers
8. Dispatches implementers (≤4 parallel), each evaluating their task definition in SQLite
9. Dispatches verifiers (≤4 parallel), each evaluating implementation reports and running tests via `run_in_terminal`
10. Dispatches 3 code reviewers with same perspective diversity
11. Knowledge Agent reads telemetry + evaluations from SQL, produces post-mortem analysis
12. Auto-commit at Step 9

### Story 2: Pipeline Resume After Interruption

The pipeline is interrupted mid-implementation. On restart:

1. Orchestrator scans feature directory with `list_dir`
2. Reads each output's completion block to determine status
3. Resumes from first incomplete step (e.g., Step 5 if 3 of 8 tasks completed)
4. run_id filtering ensures SQL queries only see current-run evidence
5. Partial SQLite records from interrupted agents are isolated by run_id

### Story 3: Context7 Available

A researcher is implementing an API integration. Context7 MCP server is configured:

1. Researcher checks Context7 availability (conditional tool access)
2. Calls `context7-resolve-library-id` to find the library ID
3. Calls `context7-query-docs` with resolved ID for current documentation
4. Uses documentation in research findings instead of guessing at API usage

### Story 4: Reviewer Perspective Diversity

Three adversarial reviewers are dispatched for design review:

1. **Security Pessimist**: Prioritizes threat modeling, assumes adversarial inputs, low severity threshold for security findings
2. **Architecture Purist**: Prioritizes SOLID principles, coupling analysis, scalability concerns
3. **Pragmatic Correctness Checker**: Prioritizes functional correctness, edge cases, test coverage gaps
   Each reviews all 3 categories but through their distinct lens.

---

## 11. Decision Points

### DP-1: Multi-Perspective Review Strategy

**Recommendation: Option A — Prompt perspective diversity**

Since VS Code ignores model parameters for custom agents, diversity must come from prompt engineering. Three reviewer instances with distinct system prompt personas (different priorities, heuristics, severity thresholds). This works immediately with no platform dependency. Multi-model support can be layered on later if VS Code adds the capability.

Options evaluated: (A) Prompt diversity only, (B) Separate agent files per perspective, (C) Anvil's platform agent_type, (D) Research runtime model routing first.

### DP-2: YAML+MD Dual Output Handling

**Recommendation: Option B — YAML-only for machine artifacts, YAML+MD for human-facing docs**

Research outputs, implementation reports, verification reports, and review verdicts are machine-consumed — YAML-only. Feature specs (feature.md), designs (design.md), plans (plan.md), and evidence bundles retain MD companions for human review.

### DP-3: Evaluation System Implementation

**Recommendation: Option B — SQLite-based evaluations**

An `artifact_evaluations` table in verification-ledger.db provides queryable evaluation data aligned with REQ-11. Scores are numeric and best suited for SQL aggregation. Avoids file proliferation of per-artifact YAML evaluation files.

### DP-4: Memory System

**Recommendation: Option A — No memory (maintain current approach)**

Typed YAML completion contracts + SQL evidence gates provide sufficient routing data. The evaluation system (FR-5) and telemetry (FR-6) address the feedback loop gap. If specific context gaps emerge during implementation, a lightweight SQLite context table can be added.

### DP-5: Agent Content Extraction Strategy

**Recommendation: Option C — Tiered extraction**

Three tiers: (1) Global rules go to `.github/copilot-instructions.md` (auto-loaded by VS Code), (2) Agent-shared patterns go to reference docs in `.github/agents/`, (3) Role-specific content stays inline in agent files.

### DP-6: Review Verdict File Path Convention

**Recommendation: Option C — Per-reviewer files with glob consumption**

Each reviewer writes `review-verdicts/<scope>-<perspective>.yaml`. Consumers discover files via `list_dir` with scope prefix. Preserves per-reviewer provenance, requires no aggregation step, works with parallel dispatch.

---

## 12. Test Scenarios

### TS-1: Telemetry Population Verification

**Linked to:** AC-6.1, AC-6.2

```sql
-- After full pipeline run:
SELECT COUNT(*) FROM pipeline_telemetry WHERE run_id = '{run_id}';
-- Expected: ≥ total agent dispatches (typically 15-25)
```

**Pass:** Row count ≥ number of dispatched agent instances.
**Fail:** Zero rows or count < dispatched instances.

### TS-2: Terminal-Only Test Execution

**Linked to:** AC-7.4

```sql
-- After verification step:
SELECT check_name, tool FROM anvil_checks
WHERE run_id = '{run_id}' AND check_name LIKE '%test%';
-- Expected: tool = 'run_in_terminal' for all test checks
```

**Pass:** All test-related check_names show `tool='run_in_terminal'`.
**Fail:** Any check_name shows `tool='runTests'` or `tool='get_errors'` for test execution.

### TS-3: Multi-Perspective Review Diversity

**Linked to:** AC-2.2
After a design review with 3 perspective instances, compare the findings:
**Pass:** ≥30% unique findings (findings appearing in only one reviewer's output) across the three review outputs.
**Fail:** <10% unique findings (reviewers produced near-identical results).

### TS-4: All-Category Coverage

**Linked to:** AC-3.1, AC-3.3

```sql
-- Per reviewer instance, check category coverage:
SELECT DISTINCT check_name FROM anvil_checks
WHERE run_id = '{run_id}' AND phase = 'review' AND instance = '{reviewer_instance}';
-- Expected: entries for security, architecture, AND correctness
```

**Pass:** Each reviewer has check_names spanning all 3 categories.
**Fail:** Any reviewer missing a category.

### TS-5: Artifact Evaluation Population

**Linked to:** AC-5.2, AC-5.4

```sql
SELECT evaluator_agent, COUNT(*), AVG(usefulness_score), AVG(clarity_score)
FROM artifact_evaluations WHERE run_id = '{run_id}'
GROUP BY evaluator_agent;
-- Expected: rows for implementer, verifier, adversarial_reviewer
```

**Pass:** ≥3 evaluator agents with non-null scores.
**Fail:** Zero rows or missing Phase 1 agents.

### TS-6: Agent File Size Compliance

**Linked to:** AC-10.1

```bash
wc -l .github/agents/*.agent.md
```

**Pass:** All agents ≤350 lines except orchestrator ≤500 lines.
**Fail:** Any agent exceeds its limit.

### TS-7: No Duplicated Operating Rules

**Linked to:** AC-10.5

```bash
grep -rl "Never redirect.*terminal.*output.*file" .github/agents/*.agent.md
```

**Pass:** Returns only the shared reference document (0 agent files contain the rule inline).
**Fail:** Returns ≥1 agent file with inline duplication.

### TS-8: Verifier Can Create Output File

**Linked to:** AC-11.1
Dispatch verifier for a task. Verifier successfully creates `verification-reports/<task-id>.yaml`.
**Pass:** File exists with valid YAML and completion contract.
**Fail:** Verifier errors due to tool restriction or file not created.

### TS-9: Verdict Path Consistency

**Linked to:** AC-12.1, AC-12.3
After Step 3b, check that files exist at paths Designer expects:

```bash
ls review-verdicts/design-*.yaml
```

**Pass:** ≥3 files matching the pattern. Designer reads all via glob.
**Fail:** Files at unexpected paths or Designer can't find them.

### TS-10: Context7 Graceful Degradation

**Linked to:** AC-9.3
Run researcher without Context7 MCP configured.
**Pass:** Researcher completes with DONE status — no Context7 error in output.
**Fail:** Researcher errors or stalls trying to reach Context7.

### TS-11: SQLite Concurrency Under Parallel Dispatch

**Linked to:** EC-3
Dispatch 4 agents writing to verification-ledger.db simultaneously.
**Pass:** All 4 complete without SQLITE_BUSY errors (or recover via retry).
**Fail:** Any agent fails permanently due to database contention.

### TS-12: Instruction Update Tracking

**Linked to:** AC-14.3, AC-15.4
Knowledge Agent proposes an instruction update:

```sql
SELECT * FROM instruction_updates WHERE run_id = '{run_id}';
```

**Pass:** Row exists with agent, file_path, change_type, change_summary.
**Fail:** No tracking record for the update.

---

## 13. Dependencies & Risks

### External Dependencies

| Dependency                    | Type | Impact if Unavailable                               |
| ----------------------------- | ---- | --------------------------------------------------- |
| VS Code `runSubagent` API     | Hard | Pipeline cannot dispatch agents                     |
| `sqlite3` CLI                 | Hard | No evidence gating — pipeline cannot function       |
| `git` CLI                     | Hard | No baseline, no staging, no commit                  |
| Context7 MCP server           | Soft | Agents skip documentation lookup, proceed without   |
| Language-specific build tools | Soft | Tier 2 verification skipped, Tier 3 mandatory       |
| VS Code `ask_questions` tool  | Soft | Interactive mode unavailable; autonomous mode works |
| VS Code `memory` tool         | Soft | Cross-session knowledge not persisted               |

### Risk Summary

| Risk                                              | Likelihood | Impact | Mitigation                                        |
| ------------------------------------------------- | ---------- | ------ | ------------------------------------------------- |
| RISK-1: Multi-model remains infeasible            | High       | Medium | Prompt diversity as primary mechanism             |
| RISK-2: Parallel execution not achievable         | Medium     | High   | Optimize sequential dispatch; document limitation |
| RISK-3: Evaluation overhead without benefit       | Medium     | Low    | Phased rollout; monitor quality                   |
| RISK-4: File size targets conflict with additions | High       | Medium | Aggressive extraction to shared docs              |
| RISK-5: Instruction updates introduce drift       | Low        | High   | Governed updates + SQLite audit trail             |
| RISK-6: Review restructure increases token cost   | High       | Medium | Scope limits per category per reviewer            |
| RISK-7: SQLite contention under parallelism       | Medium     | Medium | WAL mode + retry-on-busy logic                    |

---

## 14. Requirement Traceability Matrix

| User Req | Functional Req             | Acceptance Criteria                | Test Scenario            |
| -------- | -------------------------- | ---------------------------------- | ------------------------ |
| REQ-1    | FR-1                       | AC-1.1, AC-1.2, AC-1.3             | — (research output)      |
| REQ-2    | FR-2                       | AC-2.1, AC-2.2, AC-2.3, AC-2.4     | TS-3                     |
| REQ-3    | FR-3                       | AC-3.1, AC-3.2, AC-3.3             | TS-4                     |
| REQ-4    | FR-4                       | AC-4.1, AC-4.2, AC-4.3, AC-4.4     | — (inspection)           |
| REQ-5    | FR-5, FR-6, FR-15          | AC-5._, AC-6._, AC-15.\*           | TS-1, TS-5, TS-12        |
| REQ-6    | FR-7, FR-8                 | AC-7._, AC-8._                     | TS-2, TS-7               |
| REQ-7    | FR-9                       | AC-9.\*                            | TS-10                    |
| REQ-8    | FR-1, FR-15                | AC-1.1, AC-15.\*                   | TS-12                    |
| REQ-9    | FR-10, FR-11, FR-12, FR-16 | AC-10._, AC-11._, AC-12._, AC-16._ | TS-6, TS-7, TS-8, TS-9   |
| REQ-10   | FR-13                      | AC-13.\*                           | — (inspection)           |
| REQ-11   | FR-5, FR-6, FR-14          | AC-5._, AC-6._, AC-14.\*           | TS-1, TS-5, TS-11, TS-12 |

All 11 user requirements have corresponding functional requirements, acceptance criteria, and (where applicable) test scenarios.

---

_Generated by Spec Agent — 2026-02-27_
