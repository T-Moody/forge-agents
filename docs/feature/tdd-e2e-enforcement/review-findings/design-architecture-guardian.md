# Adversarial Review: design — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-04T00:00:00Z
- **Scope:** design
- **Task ID:** tdd-e2e-enforcement-design-review
- **Artifacts Reviewed:** `design-output.yaml`, `design.md`, `spec-output.yaml`, `feature.md`, plus 7 agent/reference files under `NewAgents/.github/agents/`

---

## Security Analysis

**Category Verdict:** approve

### Finding S-1: Verifier trust boundary expansion lacks architectural enforcement

- **Severity:** Major
- **Description:** The verifier's `run_in_terminal` scope expands from "build/test/lint/git commands" to "app lifecycle management, browser automation, HTTP client, CLI interaction" (D-11). Currently, `tool-access-matrix.md` §8 scopes the verifier's `run_in_terminal` to specific operations. The design proposes a dramatic expansion — the verifier can now launch arbitrary processes (start_command), open browsers, make HTTP requests, and interact with CLIs. However, per §11 of `tool-access-matrix.md`, "compliance depends on LLM instruction-following" — there is no runtime enforcement. The design states the expansion is "limited to Tier 5 E2E phases only" but provides no architectural mechanism to enforce this boundary. A verifier could inadvertently (or through prompt injection via a malicious E2E contract) execute Tier-5-scoped commands during Tiers 1–4.
- **Affected artifacts:** `tool-access-matrix.md` §8, `verifier.agent.md`, D-11
- **Recommendation:** (1) Define an explicit regex pattern for Tier 5 allowed `run_in_terminal` commands in `tool-access-matrix.md` (similar to the Knowledge Agent's pattern in §10). (2) Add a self-verification check: "Tier 5 commands executed ONLY when `e2e_required=true` for the current task." (3) The adversarial reviewer's code review should include a check that verifier tool invocations during Tiers 1–4 do not include E2E-scope commands.
- **Evidence:** Current verifier tool-access in `tool-access-matrix.md` §8 lists 9 tools with `create_file` scoped to `verification-reports/*.yaml$`. The Knowledge Agent (§10) has an explicit regex pattern `^(echo\s+".*(SELECT|INSERT INTO instruction_updates).*"\s*\|\s*sqlite3|sqlite3)\s+.*verification-ledger\.db`. No equivalent Tier-5-scoped pattern is proposed for the verifier.

### Finding S-2: E2E contract commands executed without command-level validation

- **Severity:** Minor
- **Description:** The E2E contract's `start_command`, `shutdown_command`, and `e2e_command` fields are arbitrary strings executed via `run_in_terminal`. Contract validation (FR-2.5, D-3) checks for field presence and type correctness but does not validate the safety of the commands themselves. Since contracts are version-controlled and project-specific (CR-8), the risk is low — but a compromised or incorrectly authored contract could specify commands that damage the workspace (e.g., `rm -rf /`). The design relies on the contract being trusted project configuration.
- **Affected artifacts:** D-3, FR-2.5, `schemas.md` (E2E contract schema)
- **Recommendation:** Add an advisory note in `e2e-integration.md` that E2E contract commands should be reviewed as part of code review. Consider adding a command allowlist pattern (e.g., commands must start with a known runner like `dotnet`, `npm`, `python`, `node`, etc.) as a SHOULD-level guideline.
- **Evidence:** FR-2.5 validation checks: "all required fields present, start_command is a non-empty string, ready_check is a valid URL or command." No command content validation specified.

---

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: Verifier responsibility overload — Single Responsibility concern

- **Severity:** Major
- **Description:** The verifier expands from a 343-line 4-tier verification cascade to a ~523-line 5-tier agent with fundamentally new responsibilities. The original verifier role is "verify correctness of implementation" (read-only code analysis + build/test execution). Tier 5 adds: (a) app lifecycle management (start, readiness polling, shutdown, PID tracking), (b) browser automation (launch, navigate, click, fill, screenshot), (c) HTTP client operations (requests, response parsing), (d) CLI interaction (command execution, stdin/stdout management), (e) process management (PID tracking, force-kill, orphan cleanup), (f) evidence file management (screenshots, HAR, response logs, evidence-manifest.yaml). This transforms the verifier from a "verification agent" into a "test execution + app operations agent." The reference extraction to `e2e-integration.md` moves coordination RULES out but doesn't reduce the verifier's RUNTIME RESPONSIBILITIES — it still performs all the above operations.
- **Affected artifacts:** D-1, D-5, `verifier.agent.md`, `e2e-integration.md`
- **Recommendation:** Consider splitting Tier 5 into sub-sections with clear responsibility boundaries within the verifier definition: (a) "E2E Lifecycle Manager" section handling app start/stop/health, (b) "E2E Executor" section handling skill step execution, (c) "E2E Evidence Collector" section handling artifact capture. This doesn't require a new agent (Direction B was rightly rejected) but creates internal modularity. Also, define a clear context-loading strategy: the verifier should only load Tier 5 instructions + `e2e-integration.md` when `e2e_required=true`, not for every dispatch.
- **Evidence:** Current verifier is 343 lines. Design adds: 180 lines new + 30 lines modified = ~523 lines (exceeding the 350-line NFR-1 standard target, though FR-9 grants the verifier an exception is not documented — only the orchestrator has an explicit line budget exception at 550). `verifier.agent.md` §4.5 already handles runtime smoke for bugfix tasks (~30 lines) — Tier 5 is a 6x expansion of this concept.

### Finding A-2: Orchestrator line budget at critical threshold (≤2 lines margin)

- **Severity:** Major
- **Description:** The orchestrator is at 528/550 lines. The design adds: E2E contract discovery at Step 0, contract path propagation to dispatch context, workflow lane enforcement logic after Step 6, E2E concurrency management (tracking active E2E count, sub-wave queuing), 3 new evidence gates (EG-8, EG-9, EG-10), EG-7 promotion to blocking, and reference to `e2e-integration.md`. The design estimates +20 lines added / 15 modified (net ~20 new lines → ~548/550). This estimate appears optimistic. E2E concurrency management alone requires: (a) reading `max_concurrent_instances` from contract, (b) tracking active E2E verifiers in pipeline state, (c) sub-wave partitioning logic for E2E overflow, (d) port ordinal assignment per verifier. The current sub-wave partitioning in `dispatch-patterns.md` is ~20 lines. Adding an E2E-specific sub-limit on top would require conditional logic within the existing sub-wave algorithm. Combined with 3 new evidence gate evaluations (each needing a SQL query + result interpretation), the 20-line estimate is likely 30–40 lines, potentially exceeding the cap.
- **Affected artifacts:** D-2, D-16, D-17, `orchestrator.agent.md`
- **Recommendation:** (1) Move E2E concurrency management entirely into `e2e-integration.md` as a reference the orchestrator consults rather than inlining decision logic. (2) Move the new evidence gate SQL queries to `sql-templates.md` and reference them like existing EG-1 through EG-6. (3) Perform a line-count budget analysis during planning to confirm feasibility — the planner should produce a line-delta estimate for the orchestrator task. (4) Consider consolidating EG-8/EG-9/EG-10 into a single lane-aware composite gate (EG-10 already partially does this) to reduce the number of gate checks.
- **Evidence:** `orchestrator.agent.md` is 528 lines. D-2 rationale says "orchestrator is at 528/550 lines" and uses this to justify extracting to `e2e-integration.md`. The design-output.yaml file_inventory estimates 20 lines added + 15 modified for the orchestrator (risk 🔴). Current evidence gates (EG-1 through EG-6) occupy ~30 lines in the orchestrator. Adding EG-7 (promoted), EG-8, EG-9, EG-10 would roughly double this section.

### Finding A-3: Step-by-step browser automation mechanism architecturally undefined

- **Severity:** Critical
- **Description:** The design's core differentiator (AC-38) is that "agents actually interacted with the app — screenshots of rendered pages, real HTTP responses." For browser interaction, the verifier must perform step-by-step actions: navigate to URLs, click CSS selectors, fill form fields, take screenshots, and assert on DOM state. The design says the verifier uses Playwright CLI via `run_in_terminal` (D-11), citing `npx playwright test` and `npx playwright screenshot`. However: (a) `npx playwright test` runs pre-written test files — it does NOT support interactive step-by-step browser commands, (b) `npx playwright screenshot <url>` takes a static screenshot — it cannot click, fill, or interact with elements, (c) the Playwright CLI has no interactive mode for the actions specified in skill steps (navigate, click, fill, submit, assert_visible, etc.). The design conflates "running Playwright tests" with "interactive browser automation via CLI." To achieve step-by-step interaction, the verifier would need to: generate a temporary Playwright Node.js script encoding the skill's steps as API calls, execute it via `run_in_terminal` (`node temp-script.js`), and capture output. This mechanism is feasible (the verifier already generates throwaway scripts for Tier 3 smoke tests per §4b) but is not described anywhere in the design. Without this specification, implementers would not know how to build the browser interaction system.
- **Affected artifacts:** D-5 (Tier 5 lifecycle), D-11 (tool access), D-18 (determinism), FR-4.2, FR-4.3, `e2e-integration.md` (proposed)
- **Recommendation:** The design MUST specify the browser interaction execution model explicitly. Proposed approach: (1) The verifier generates a temporary Playwright script from skill steps (action → Playwright API mapping: navigate → `page.goto()`, click → `page.click()`, fill → `page.fill()`, assert_visible → `page.locator().isVisible()`, screenshot → `page.screenshot()`), (2) the script is written to `evidence_output_dir/<task-id>/temp-e2e-<skill-id>.js`, (3) executed via `run_in_terminal` (`npx playwright test temp-e2e-<skill-id>.js` using Playwright's test runner in a generated test file, or `node temp-e2e-<skill-id>.js` using Playwright's library API), (4) output parsed for per-step pass/fail results, (5) script deleted after execution (following existing Tier 3 temp-file cleanup pattern). A similar approach is needed for API interaction (generate a script using `fetch`/`curl` for each step) and CLI interaction (sequential `run_in_terminal` calls for each step). This execution model should be documented in `e2e-integration.md` §Interaction Execution Protocol.
- **Evidence:** D-11 rationale: "Browser interaction uses Playwright CLI (npx playwright test, npx playwright screenshot)." The skill schema (FR-3.3) defines step actions: navigate, click, fill, submit, assert_visible, assert_text, screenshot. No Playwright CLI command supports these individual actions from the command line. The current verifier Tier 3 (§4b in `verifier.agent.md`) already generates throwaway scripts, establishing precedent for the proposed approach.

### Finding A-4: Context window pressure from compounding instruction documents

- **Severity:** Major
- **Description:** The verifier's context requirements after this change: `verifier.agent.md` (~523 lines), `e2e-integration.md` (~400 lines), `schemas.md` relevant sections (~200 lines for E2E contract + skills schemas), E2E contract file (~50-100 lines), skill files (variable, potentially 200+ lines for 5 skills), plus existing reference docs (`sql-templates.md`, `tool-access-matrix.md`, `global-operating-rules.md`, `severity-taxonomy.md`, `evaluation-schema.md`). Total instruction context could exceed 1500 lines before the verifier reads any implementation code, task definitions, or codebase files. The design's context governance (D-14: mandatory 200-line `read_file` limit, FR-7.3: evidence summarization) addresses runtime reading but not the instruction-loading problem. In Copilot's agent framework, the agent definition file and referenced documents are loaded into context at dispatch time — this is not governed by `read_file` limits.
- **Affected artifacts:** D-2, D-14, FR-7, `verifier.agent.md`, `e2e-integration.md`
- **Recommendation:** (1) Structure `e2e-integration.md` with clear section headers so agents can be instructed to read only relevant sections (e.g., "Verifier: read §2 Execution Protocol and §5 Safety Rules; skip §3 Contract Creation and §4 Skill Examples"). (2) Add a design note about instruction context budget: the planner should consider the total instruction load when sizing tasks and assigning agents. (3) Consider making the verifier's Tier 5 instructions conditionally loaded — only dispatched when `e2e_required=true`, to avoid loading E2E instructions for unit-only tasks. The orchestrator already passes parameters to the verifier at dispatch — adding a `tiers_required: [1,2,3,4]` vs `[1,2,3,4,5]` parameter would allow conditional instruction loading.
- **Evidence:** `verifier.agent.md` currently references 6 documents in its header. Adding `e2e-integration.md` makes 7. The orchestrator's NFR-1 exception note says line budget is justified by "coordination complexity" — no equivalent exception or budget analysis exists for the verifier's expanded role.

### Finding A-5: e2e-integration.md scope spans multiple concerns

- **Severity:** Minor
- **Description:** The proposed `e2e-integration.md` (~400 lines) contains: E2E coordination rules, contract schema reference, skills schema reference, interaction protocols, safety rules, and example skills. This single document serves the orchestrator (concurrency rules), verifier (execution protocol, safety), planner (lane derivation context), and adversarial reviewer (E2E quality checks). While the design pattern of reference documents is established (cf. `schemas.md` at 1220 lines), `e2e-integration.md` mixes operational concerns (execution protocol), data definitions (contract/skills schema references), and examples (skill templates). Agents loading this document would need to parse through irrelevant sections.
- **Affected artifacts:** D-2, file inventory (files_created)
- **Recommendation:** Organize `e2e-integration.md` with numbered sections and a per-agent reading guide at the top (e.g., "Orchestrator: §1, §6. Verifier: §2, §3, §5. Planner: §1 only."). Consider whether example skills could be a separate appendix or inline comments to keep the core protocol sections focused.
- **Evidence:** Existing reference docs: `schemas.md` (1220 lines, single-concern: schema definitions), `sql-templates.md` (SQL patterns only), `dispatch-patterns.md` (274 lines, dispatch logic only). `e2e-integration.md` would be the first multi-concern reference document.

### Finding A-6: SQL evidence growth strategy absent

- **Severity:** Minor
- **Description:** For a single full-tdd-e2e task, the design adds ~10 new `check_name` values (tdd-compliance, e2e-contract-found, e2e-contract-validation, e2e-instance-start, e2e-readiness, e2e-suite-execution, e2e-exploratory, e2e-adversarial, e2e-instance-shutdown, e2e-test-execution). A feature with 5 high-risk tasks, 1 E2E retry each, across 2 verification iterations would generate ~100 new records per pipeline run (in addition to existing checks). The design mentions no indexing strategy for `anvil_checks` beyond the existing schema (only `id` is indexed as PRIMARY KEY). Evidence gate queries (EG-8 through EG-10) filter on `run_id`, `task_id`, and `check_name` — these would benefit from a composite index. No cleanup or archival strategy is discussed for long-lived projects.
- **Affected artifacts:** D-7, `sql-templates.md`
- **Recommendation:** (1) Add a composite index recommendation: `CREATE INDEX IF NOT EXISTS idx_anvil_checks_lookup ON anvil_checks(run_id, task_id, check_name, passed)`. (2) Note this as a planner consideration: if the planner creates tasks to modify `sql-templates.md`, include index creation in the DDL section. This is advisory, not blocking.
- **Evidence:** Current `anvil_checks` DDL (from `verification-ledger.db`): only `id INTEGER PRIMARY KEY AUTOINCREMENT` — no secondary indexes. The design adds 3 new evidence gate queries (EG-8, EG-9, EG-10) that filter on `(run_id, task_id, check_name, passed)`.

---

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Interactive skill creation workflow (FR-8.3) not architecturally designed

- **Severity:** Major
- **Description:** FR-8.3 requires collaborative skill creation in interactive mode: "discover existing test suites, analyze app structure, propose exploratory skills with navigation paths, propose adversarial skills, present to user, iterate (max 2 rounds)." The design's decision D-4 addresses skill storage, and D-18 addresses determinism, but no decision covers the skill CREATION workflow. Who creates skills? The spec says "agents" but doesn't specify which agent. The design maps FR-8 to "Orchestrator ask_questions delegation. E2E contract creation guidance" (§13 Traceability), but the orchestrator cannot create skills (it writes NO files). The planner doesn't create skills (it creates task definitions). The implementer writes code, not E2E skills. The verifier is read-only. The most natural fit is a new Step 0.5 (post-contract-discovery, pre-research) or extending the spec/designer agent's scope — but no design decision addresses this.
- **Affected artifacts:** FR-8.3, D-4, §13 traceability table (FR-8 row: "Orchestrator ask_questions delegation. E2E contract creation guidance")
- **Recommendation:** Add a design decision (D-19) specifying the skill creation workflow: (1) Which agent creates skills (likely the orchestrator delegates to a researcher instance in a new Step 0.5, or extends the spec agent's interactive mode), (2) How route/endpoint discovery works (the researcher could discover routes from project files), (3) How the user review prompt is structured (via `ask_questions`), (4) Where skills are stored during the pipeline run (contract's skill_discovery_path). The FR-8 traceability entry should reference this decision.
- **Evidence:** `design.md` §13 maps FR-8 to "Orchestrator ask_questions delegation. E2E contract creation guidance" — this is vague and doesn't specify the agent or mechanism. No decision in `design-output.yaml` addresses skill creation. `feature.md` FR-8.3 says "Discover existing test suites and create 'test-suite' type skills wrapping them... Propose 'exploratory' skills with step-by-step interaction procedures... Iterate skill refinement (max 2 iterations)."

### Finding C-2: Evidence gate expansion pressures orchestrator gate logic without budget allocation

- **Severity:** Major
- **Description:** The design adds 3 new evidence gates (EG-8: TDD compliance, EG-9: E2E completion, EG-10: lane-aware verification) and promotes EG-7 from INFORMATIONAL to BLOCKING for code tasks. The current orchestrator evaluates 2 verification gates (EG-1, EG-2) and 4 review gates (EG-3 through EG-6). After this change, the orchestrator must evaluate up to 6 verification gates (EG-1, EG-2, EG-7, EG-8, EG-9, EG-10) for full-tdd-e2e tasks — a 3x increase from the current 2. Each gate requires a SQL query via `run_in_terminal` + result interpretation. The design's routing logic in §6.3 shows the orchestrator must check different gate sets per workflow_lane (unit-only: EG-2 + EG-8; unit-integration: EG-2 + EG-7 + EG-8; full-tdd-e2e: EG-2 + EG-7 + EG-8 + EG-9). This conditional gate evaluation adds branching complexity to the orchestrator, exacerbating the line budget concern (A-2). Note: EG-10 appears to be a composite of EG-2 + lane-specific requirements, potentially obviating separate EG-2 checks — but this consolidation opportunity is not explored in the design.
- **Affected artifacts:** D-16, `orchestrator.agent.md` §Evidence Gate Logic, `sql-templates.md`
- **Recommendation:** (1) Consolidate EG-2 and EG-10 into a single lane-aware gate that subsumes the current EG-2 behavior (EG-10's query already includes the total_passed count that EG-2 checks). (2) Define a single "verification gate evaluation" routine in `sql-templates.md` that accepts `workflow_lane` as a parameter and returns a composite pass/fail, reducing the orchestrator's gate logic to a single query + result check. (3) Explicitly count the lines needed for gate evaluation in the orchestrator's line budget.
- **Evidence:** `orchestrator.agent.md` current Evidence Gate Logic section (~30 lines for EG-1/EG-2 + ~40 lines for EG-3 through EG-6). `design.md` §6.3 shows 3 different gate check paths depending on `workflow_lane`. `sql-templates.md` currently defines EG-1 through EG-7 (EG-7 informational).

### Finding C-3: Dual verification paths (Step 6a + Tier 5) lack transition mechanism

- **Severity:** Minor
- **Description:** DR-1 preserves Step 6a (Runtime Smoke Test for bugfix features) alongside the new Tier 5 (E2E Verification for full-tdd-e2e lane tasks). This creates two parallel verification paths with overlapping scope: both start the application, verify runtime behavior, and shut down. A bugfix feature for a project WITH an E2E contract and 🔴 risk tasks could trigger BOTH Step 6a (because it's a bugfix) AND Tier 5 (because `e2e_required=true`). The design doesn't clarify whether Step 6a is subsumed by Tier 5 when an E2E contract exists for a bugfix task, or whether both run. The orchestrator routing in `orchestrator.agent.md` Step 6a says "after Step 6 verification passes and before Step 7 code review" — Tier 5 runs as part of Step 6. So the sequence could be: Step 6 (Tiers 1-5 including E2E) → Step 6a (runtime smoke again) → Step 7.
- **Affected artifacts:** DR-1, `orchestrator.agent.md` §Step 6a, D-5
- **Recommendation:** Add explicit routing logic: "If Tier 5 E2E verification passed (`e2e-test-execution` passed=1), Step 6a is skipped for that task (Tier 5 subsumes the runtime smoke test). Step 6a runs only when no E2E contract exists or the task is not in the full-tdd-e2e lane." This aligns with FR-4.6's intent while preserving backward compatibility per DR-1.
- **Evidence:** `orchestrator.agent.md` lines 251-263 (Step 6a): "When the user prompt describes a runtime bug... the orchestrator MUST run a live runtime verification after Step 6 verification passes and before Step 7 code review." D-5 Phase 1-5 runs within Step 6 (Tier 5). DR-1 says "Step 6a remains as-is for bugfix features that lack an E2E contract" — but doesn't address bugfix features that HAVE an E2E contract.

---

## Summary

The design represents a well-reasoned Direction C approach with 18 decisions, thoughtful tradeoff analysis, and comprehensive spec traceability. However, it has one critical architectural gap: the **browser automation execution model is undefined** — the design conflates Playwright CLI test execution (`npx playwright test`) with step-by-step interactive browser control, which Playwright CLI does not support. The verifier would need to generate temporary scripts encoding skill steps as programmatic API calls, but this mechanism is absent from the design. Additionally, the **orchestrator line budget is at critical threshold** (~548/550), the **verifier's responsibility scope has expanded significantly** beyond its original SRP, and the **interactive skill creation workflow** (FR-8.3) lacks a design decision specifying which agent creates skills and how. The evidence gate expansion (3 new gates + 1 promotion) adds complexity to the orchestrator without a clear budget allocation.

These issues are remediable through design revision — no architectural direction change is needed. Direction C remains the correct choice.
