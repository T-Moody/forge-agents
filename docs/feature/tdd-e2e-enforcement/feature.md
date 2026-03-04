# Feature Specification: TDD & E2E Testing Enforcement

**Feature Slug:** `tdd-e2e-enforcement`
**Run ID:** `2026-03-04T00:00:00Z`
**Schema Version:** 1.0
**Status:** DONE
**Revision:** 2
**Revision Reason:** User clarification — E2E means agents LITERALLY run and interact with the application (browser navigation, API calls, CLI interaction), not just execute pre-written test suites.

---

## Title & Short Summary

Make TDD and E2E testing core enforcement mechanisms of the agent pipeline. E2E testing means agents **actually launch, navigate, and interact with the running application** — like a real QA tester using a browser, a developer probing an API with Postman, or a user running a CLI tool. This is fundamentally different from simply running `npx playwright test`. Agents follow pre-defined interaction skills that specify exactly which pages to visit, which forms to fill, which buttons to click, which API endpoints to probe, and which adversarial inputs to try.

---

## Background & Context

### Research Inputs

| Focus        | File                                                     | Findings                                                                          |
| ------------ | -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Architecture | [research/architecture.yaml](research/architecture.yaml) | 15 findings — pipeline structure, TDD gaps, E2E absence, modification surface     |
| Impact       | [research/impact.yaml](research/impact.yaml)             | 15 findings — blast radius (14/18 files), performance, risk matrix, compatibility |
| Dependencies | [research/dependencies.yaml](research/dependencies.yaml) | 18 findings — schemas, SQL, external tools, E2E contract design                   |
| Patterns     | [research/patterns.yaml](research/patterns.yaml)         | 15 findings — existing TDD/verification, Playwright CLI, academic research        |

### Current State

The agent system at `NewAgents/.github/agents/` is a 10-step deterministic pipeline with 9 agent types, 12 typed YAML schemas, SQL-based evidence gating, and parallel dispatch (max 4 concurrent). It already has:

- **Partial TDD** in the implementer (Steps 4-6: write failing tests → minimal code → self-fix). However, TDD is self-attested and the TDD Fallback allows skipping entirely.
- **4-tier verification cascade** in the verifier (diagnostics → build/type/lint/tests → import/smoke → operational). No E2E tier exists.
- **Runtime smoke test** for bugfix tasks only (Step 6a). This is the closest analog to E2E but is narrow and ad-hoc.
- **Risk classification** (🟢/🟡/🔴) that drives verification depth but not test-type selection.

### Critical Gaps

1. **TDD is advisory, not enforced** — compliance is self-attested; no independent verification
2. **E2E infrastructure is completely absent** — no contract schema, app lifecycle management, browser automation, or port isolation
3. **No interactive app testing** — no mechanism for agents to actually navigate pages, fill forms, click buttons, make API calls, or interact with the app as a user would
4. **Risk-based workflow lanes do not exist** — risk classification drives verification depth but not test type
5. **Parallel E2E safety is absent** — only SQLite WAL provides concurrency protection
6. **Context governance is advisory** — no enforcement of read limits or context compaction

### User Clarification — The Core Change (Revision 2)

The user clarified that **"E2E testing" and "break the app like a developer"** means agents must **literally run the application, navigate it, and actually test it** — NOT just run a pre-written test suite:

- **Web apps:** Agents launch the app, open a browser (via Playwright or similar), navigate pages, click buttons, fill forms, submit data, check results, try edge cases, try to break things — like a real QA tester sitting at a browser.
- **API apps:** Agents launch the app, make HTTP requests to endpoints, test various payloads, check responses, try malformed inputs, test auth flows — like a developer using Postman/curl to probe the API.
- **CLI apps:** Agents run the app, provide various inputs, test error handling, try unexpected inputs.
- This is **in addition to** running any pre-written E2E test suites. The agent does BOTH: runs the test suite AND performs exploratory/adversarial interactive testing.
- Skills define **both** automated test procedures AND step-by-step interaction procedures (navigation paths, forms to test, endpoints to probe, adversarial inputs to try).

### Ambiguity Resolutions

1. **E2E Skills Schema:** Define a formal skills schema NOW with reusable test AND interaction procedures.
2. **Automatic Mode E2E:** Auto mode skips E2E **setup/creation** but still **runs** existing E2E tests and interaction skills if a contract exists.
3. **Adversarial Scope:** Pre-defined adversarial scenarios in skills (no novel generation at runtime) — these include **exploratory interaction procedures with adversarial inputs**, not just test commands.

---

## Pushback Log

5 concerns were identified (4 Major, 1 Critical). Pipeline proceeds with concerns noted as known risks.

| #   | Severity | Concern                                                                                    | Recommendation                                                                                                                 |
| --- | -------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Major    | **Orchestrator line budget** — at 528/550 lines, E2E routing may exceed cap                | Extract E2E coordination rules into reference document (e2e-integration.md)                                                    |
| 2   | Major    | **E2E flakiness amplification** — exploratory interaction is inherently flakier            | Max 1 E2E re-run per task; configurable step timeouts; retry-at-step capability                                                |
| 3   | Major    | **Pipeline wall-clock time** — interaction adds 3-8x overhead to verification              | Hard 10-minute E2E timeout per task; risk-based lanes ensure only High Risk tasks pay E2E cost                                 |
| 4   | Major    | **Blast radius** — 14 of 18 files require modification                                     | Phased implementation: TDD → E2E infra → E2E interaction → polish                                                              |
| 5   | Critical | **Verifier needs sophisticated tool access** — browser automation, HTTP clients, CLI tools | Define interaction tool abstraction in contract; skills have explicit assertions so pass/fail is deterministic, not subjective |

---

## Architectural Directions Evaluated

### Direction A: Extend Existing Verifier

Add E2E as a new Tier 5 in the verifier's cascade. Verifier manages full app lifecycle, test suite execution, AND interactive exploration (browser/API/CLI). **Pros:** Minimal agent count change. **Cons:** Verifier complexity grows significantly — now needs browser automation, HTTP client, and CLI interaction tools.

### Direction B: Dedicated E2E Agent

Create `e2e-tester.agent.md` — a new 10th agent type specialized for E2E with full interaction capabilities. **Pros:** Clean separation of concerns, focused tool access. **Cons:** New agent, new schema, increased dispatch complexity.

### Direction C: Hybrid — Extended Verifier + Reference Document (Recommended)

Extend verifier with E2E execution AND interactive exploration, but extract rules/schemas/interaction protocols into `e2e-integration.md`. **Pros:** Keeps 9-agent count, manages complexity via reference extraction, keeps orchestrator within line budget. **Cons:** Verifier still grows, but reference doc absorbs coordination complexity.

---

## Functional Requirements

### FR-1: Mandatory TDD — Red-Green-Verify Cycle

| Sub-Req | Description                                                                                                                                                                   | Priority    |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| FR-1.1  | 3-phase TDD cycle: RED (write failing tests, confirm fail) → GREEN (minimal code to pass) → VERIFY (get_errors + typecheck + tests + failure mitigations)                     | must-have   |
| FR-1.2  | Structured TDD evidence: `tdd_red_green.tests_written_first`, `initial_run_failures > 0`, `initial_run_exit_code ≠ 0`, and new `verify_phase` object                          | must-have   |
| FR-1.3  | Tightened TDD Fallback: skip TDD ONLY when task_type ≠ 'code' AND no production source files modified AND only docs/config change. Must record `tdd_fallback_reason`          | must-have   |
| FR-1.4  | Independent TDD verification by verifier: check test files exist + import production modules, cross-check `initial_run_failures > 0`, verify behavioral_coverage completeness | must-have   |
| FR-1.5  | TDD compliance as blocking SQL evidence (`check_name='tdd-compliance'`, passed=1 required). EG-7 (behavioral coverage) promoted from INFORMATIONAL to blocking for code tasks | must-have   |
| FR-1.6  | Test writing guidelines enforced: behavior-based testing, no type-system testing, public interfaces only, no test-only hooks. Adversarial reviewer checks for violations      | should-have |
| FR-1.7  | Code quality rules (YAGNI, KISS, DRY, lint compatibility, no TODO/TBD) verified by verifier (lint) and adversarial reviewer (code quality dimension)                          | should-have |

### FR-2: Dynamic E2E Contract Schema

| Sub-Req | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Priority  |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| FR-2.1  | New typed YAML schema in schemas.md with fields: app_type, start_command, start_env, ready_check, ready_timeout_ms, base_url, shutdown_command, shutdown_timeout_ms, default_port, port_env_var, port_range_start, e2e_runner, e2e_command, e2e_config_path, e2e_env, supports_parallel, requires_isolated_instance, max_concurrent_instances, screenshot_on_failure, trace_on_failure, video_recording, har_capture, evidence_output_dir, skills, skill_discovery_path | must-have |
| FR-2.2  | **NEW:** Contract MUST specify `interaction_type` (browser\|api\|cli\|none), `interaction_tools` (tool requirements per interaction type), `readiness_checks` (pre-interaction verification), and `evidence_capture` methods (screenshots, HAR files, response logs, console logs, video, DOM snapshots)                                                                                                                                                                | must-have |
| FR-2.3  | Contract lives at discoverable project location (`e2e-contract.yaml` at root or `.e2e/contract.yaml`). Discovered at Step 0, path propagated to downstream agents                                                                                                                                                                                                                                                                                                       | must-have |
| FR-2.4  | Missing contract handling: interactive → prompt user to create (including interaction type/tools configuration); autonomous → skip E2E with documented reason                                                                                                                                                                                                                                                                                                           | must-have |
| FR-2.5  | Contract validation at discovery: required fields present, interaction_type matches app_type, interaction_tools match interaction_type, valid types. Failures recorded as `check_name='e2e-contract-validation'`                                                                                                                                                                                                                                                        | must-have |

### FR-3: E2E Skills Schema — Reusable Interaction & Test Procedures

This is the **core differentiator**. Skills define step-by-step interaction procedures that agents follow to actually use the application.

| Sub-Req | Description                                                                                                                                                                                                                                                                                                     | Priority    |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| FR-3.1  | Formal skills schema with **type field**: `test-suite` (run test command), `exploratory` (interactive navigation/use), `adversarial` (try to break things). Plus: interaction mode (browser\|api\|cli\|test-command), steps, expected_outcomes, adversarial_variations, timeout_ms                              | must-have   |
| FR-3.2  | **Skill types define interaction mode:** `test-suite` runs commands; `exploratory` drives agent to navigate pages, click buttons, fill forms, make API calls, send CLI inputs; `adversarial` applies malformed input, injection attacks, boundary values                                                        | must-have   |
| FR-3.3  | **Interaction step schema:** order, action (navigate/click/fill/submit/http_request/run_command/etc.), target (CSS selector/URL/endpoint/command), value, method, headers, expect (human-readable), assert (machine-checkable: status_code/text_contains/element_visible/etc.), capture, timeout_ms, on_failure | must-have   |
| FR-3.4  | **Adversarial variations with step overrides:** id, name, severity_if_missed, overrides (list of {step_order, field, override_value}), expected_behavior, assert. Pre-defined only — NOT generated at runtime                                                                                                   | must-have   |
| FR-3.5  | Skills stored inline in contract or as separate files organized by type (test-suite/, exploratory/, adversarial/)                                                                                                                                                                                               | must-have   |
| FR-3.6  | Interactive mode: collaborative skill creation covering test-suites AND exploratory/adversarial interaction — discover routes/endpoints, propose navigation paths, propose adversarial inputs, iterate with user (max 2 rounds)                                                                                 | should-have |
| FR-3.7  | Autonomous mode: no new skill creation, but ALL existing skills execute including exploratory and adversarial (agents still interact with the app)                                                                                                                                                              | must-have   |
| FR-3.8  | Skills persist and are reusable across pipeline runs                                                                                                                                                                                                                                                            | must-have   |
| FR-3.9  | Reference examples documented in e2e-integration.md: browser login flow, API CRUD probe, CLI input testing, adversarial SQL injection variation                                                                                                                                                                 | should-have |

#### Example: Browser Exploratory Skill

```yaml
skill:
  id: "SKILL-001"
  name: "test-login-flow"
  type: exploratory
  interaction: browser
  preconditions:
    - "App is running and accessible at base_url"
    - "Login page exists at /login"
  steps:
    - order: 1
      action: navigate
      target: "/login"
      expect: "Login form visible with email and password fields"
      assert: { type: element_visible, value: "#login-form" }
      capture: screenshot
    - order: 2
      action: fill
      target: "#email"
      value: "test@example.com"
    - order: 3
      action: fill
      target: "#password"
      value: "password123"
    - order: 4
      action: click
      target: "#submit"
      expect: "Redirected to dashboard"
      assert: { type: url_matches, value: "/dashboard" }
      capture: screenshot
    - order: 5
      action: navigate
      target: "/profile"
      expect: "User profile page displays user data"
      assert: { type: text_contains, value: "test@example.com" }
      capture: screenshot
  adversarial_variations:
    - id: "ADV-001"
      name: "SQL injection in email"
      severity_if_missed: critical
      overrides:
        - step_order: 2
          field: value
          override_value: "' OR 1=1; --"
      expected_behavior: "Error message displayed, no redirect to dashboard"
      assert: { type: element_visible, value: ".error-message" }
    - id: "ADV-002"
      name: "Empty password"
      severity_if_missed: high
      overrides:
        - step_order: 3
          field: value
          override_value: ""
      expected_behavior: "Validation error shown, form not submitted"
      assert: { type: element_visible, value: ".validation-error" }
    - id: "ADV-003"
      name: "XSS in email field"
      severity_if_missed: critical
      overrides:
        - step_order: 2
          field: value
          override_value: "<script>alert('xss')</script>"
      expected_behavior: "Input sanitized or rejected, no script execution"
      assert: { type: element_not_visible, value: "script" }
```

#### Example: API Probe Skill

```yaml
skill:
  id: "SKILL-002"
  name: "test-users-crud"
  type: exploratory
  interaction: api
  steps:
    - order: 1
      action: http_request
      method: POST
      target: "/api/users"
      value: '{"name": "Test User", "email": "test@example.com"}'
      headers: { "Content-Type": "application/json" }
      assert: { type: status_code, value: 201 }
      capture: response
    - order: 2
      action: http_request
      method: GET
      target: "/api/users/1"
      assert: { type: response_body_contains, value: "Test User" }
      capture: response
    - order: 3
      action: http_request
      method: DELETE
      target: "/api/users/1"
      assert: { type: status_code, value: 204 }
    - order: 4
      action: http_request
      method: GET
      target: "/api/users/1"
      assert: { type: status_code, value: 404 }
      capture: response
  adversarial_variations:
    - id: "ADV-010"
      name: "Malformed JSON body"
      severity_if_missed: high
      overrides:
        - step_order: 1
          field: value
          override_value: '{"name": "Test User", "email": INVALID}'
      expected_behavior: "Returns 400 Bad Request with error message"
      assert: { type: status_code, value: 400 }
    - id: "ADV-011"
      name: "SQL injection in query"
      severity_if_missed: critical
      overrides:
        - step_order: 2
          field: target
          override_value: "/api/users/1 OR 1=1"
      expected_behavior: "Returns 400 or 404, not all users"
      assert: { type: status_code, value: 400 }
```

### FR-4: Live App Testing Architecture — Actual Interaction

| Sub-Req | Description                                                                                                                                                                                                                                                                                                                       | Priority  |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| FR-4.1  | Implementer writes unit/integration tests (TDD). MUST NOT start app, launch browser, make HTTP requests, or run E2E. MAY write E2E test files but MUST NOT execute                                                                                                                                                                | must-have |
| FR-4.2  | **Verifier 5-phase E2E sequence:** (1) Setup — start app, wait readiness, launch interaction tools. (2) Suite — run test-suite skills. (3) Exploratory — follow interaction steps: navigate, click, fill, submit, assert. (4) Adversarial — apply overrides, verify expected behavior. (5) Teardown — shutdown app, close browser | must-have |
| FR-4.3  | **Verifier tool access expanded:** browser automation (Playwright/MCP), HTTP client (fetch/curl), CLI interaction (terminal). Declared in tool-access-matrix.md                                                                                                                                                                   | must-have |
| FR-4.4  | Verifier remains read-only for source code during E2E. Only evidence artifacts written                                                                                                                                                                                                                                            | must-have |
| FR-4.5  | **Expanded SQL checks:** e2e-contract-found, e2e-instance-start, e2e-readiness, e2e-suite-execution, **e2e-exploratory** (NEW), e2e-adversarial, e2e-instance-shutdown, composite e2e-test-execution                                                                                                                              | must-have |
| FR-4.6  | Existing Step 6a (bugfix runtime smoke) generalized into contract-driven E2E verification for all High Risk tasks                                                                                                                                                                                                                 | must-have |
| FR-4.7  | **Structured evidence capture:** screenshots named skill-id_step-order_timestamp.png, API response logs with method/URL/status/body, console logs with timestamps, evidence-manifest.yaml                                                                                                                                         | must-have |
| FR-4.8  | **Step-by-step interaction log:** per-skill log with steps_executed, steps_passed, steps_failed, per-step action/target/result/actual_behavior/evidence_path/duration_ms                                                                                                                                                          | must-have |

### FR-5: Parallel Agent Safety

| Sub-Req | Description                                                                                                                      | Priority    |
| ------- | -------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| FR-5.1  | Port isolation: `port_range_start + task_ordinal_index`. Port passed via `port_env_var` env var                                  | must-have   |
| FR-5.2  | Max concurrent E2E limit enforced (default 2, hard cap 4). Excess queued into sub-waves                                          | must-have   |
| FR-5.3  | Task-scoped evidence directory. **Separate browser instances per verifier** (no shared browser state). SQLite WAL + busy_timeout | must-have   |
| FR-5.4  | Hard timeouts: startup 60s, suite 300s, **exploratory 180s/skill**, **adversarial 120s/skill**, shutdown 30s, total 600s/task    | must-have   |
| FR-5.5  | Deterministic test seeds supported via contract/skill configuration                                                              | should-have |
| FR-5.6  | PID tracking + cleanup: verifier tracks every spawned process **AND browser process** and ensures termination                    | must-have   |

### FR-6: Risk-Based Workflow Lanes

| Sub-Req | Description                                                                                                                                                 | Priority  |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| FR-6.1  | THREE lanes: LOW (unit only), MEDIUM (unit+integration), HIGH (full TDD + E2E including **test suite + exploratory interaction + adversarial interaction**) | must-have |
| FR-6.2  | Planner assigns `workflow_lane` per task: `unit-only`, `unit-integration`, `full-tdd-e2e`. Derived from risk classification (🟢→LOW, 🟡→MEDIUM, 🔴→HIGH)    | must-have |
| FR-6.3  | Planner adds `e2e_required` boolean and `e2e_contract_path` to each task                                                                                    | must-have |
| FR-6.4  | Orchestrator enforces lane-appropriate verification and E2E evidence gates                                                                                  | must-have |
| FR-6.5  | Lane-aware minimum signals: LOW ≥ 2, MEDIUM ≥ 3, HIGH ≥ 4 (including E2E)                                                                                   | must-have |

### FR-7: Subagent Context Governance

| Sub-Req | Description                                                                                                         | Priority    |
| ------- | ------------------------------------------------------------------------------------------------------------------- | ----------- |
| FR-7.1  | Max 200 lines per `read_file` call — elevated to mandatory rule                                                     | should-have |
| FR-7.2  | Prefer semantic_search/grep_search before read_file; no full-repo reads                                             | should-have |
| FR-7.3  | E2E evidence summarized (≤ 500 chars in SQL), full artifacts stored as files, referenced via evidence-manifest.yaml | must-have   |
| FR-7.4  | Implementer reads ONLY relevant_context files; violations flagged by adversarial reviewer                           | should-have |

### FR-8: Interactive Mode Requirements

| Sub-Req | Description                                                                                                                                                       | Priority    |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| FR-8.1  | Escalate ambiguity via ask_questions (orchestrator asks on behalf of subagents)                                                                                   | must-have   |
| FR-8.2  | E2E contract creation wizard **including interaction_type and interaction_tools selection** (interactive only)                                                    | should-have |
| FR-8.3  | **Collaborative skill creation covering test-suites AND exploratory/adversarial interaction** — discover routes, propose navigation paths, iterate (max 2 rounds) | should-have |
| FR-8.4  | Spec agent pushback on ambiguity; all subagents escalate rather than speculate                                                                                    | must-have   |

### FR-9: Orchestrator Rules

| Sub-Req | Description                                                                                                     | Priority    |
| ------- | --------------------------------------------------------------------------------------------------------------- | ----------- |
| FR-9.1  | Orchestrator MUST NOT write code, run builds/tests/E2E, **launch browsers**, or **make HTTP requests** directly | must-have   |
| FR-9.2  | Enforce workflow lanes via dispatch instructions (including interaction requirements) and evidence gates        | must-have   |
| FR-9.3  | Manage E2E concurrency (track active E2E count **including browser/API sessions**, enforce limit, queue excess) | must-have   |
| FR-9.4  | E2E coordination rules extracted to reference document for line budget                                          | should-have |

### FR-10: Copilot Best Practices

| Sub-Req | Description                                                                               | Priority    |
| ------- | ----------------------------------------------------------------------------------------- | ----------- |
| FR-10.1 | Implementers behave like real developers: small files, explicit interfaces, strong typing | should-have |
| FR-10.2 | Knowledge agent updates Copilot instructions when patterns drift                          | should-have |
| FR-10.3 | No speculative abstraction; YAGNI enforced by adversarial reviewer                        | should-have |

### FR-11: Test Replay Logs & Evidence

| Sub-Req | Description                                                                                                                                                                                          | Priority    |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| FR-11.1 | Capture: inputs, seeds, traces, **screenshots at each interaction step**, console logs, **HAR files**, **API response logs**, DOM snapshots in `evidence_output_dir/<task-id>/<skill-id>/`           | should-have |
| FR-11.2 | evidence-manifest.yaml indexes all artifacts with metadata (type, skill-id, step-order, timestamp, size). Verification report includes `artifact_paths`                                              | must-have   |
| FR-11.3 | Knowledge agent includes **interaction metrics** in post-mortem: suite pass rate, **exploratory step pass rate**, **adversarial detection rate**, timing per skill, flakiness rate, failure patterns | should-have |

### FR-12: Workflow Completion Gate

| Sub-Req | Description                                                                                                                                                     | Priority  |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| FR-12.1 | COMPLETE = unit tests pass + lane-appropriate verification + **E2E pass (suite + exploratory + adversarial)** (if required) + no security blockers + lint clean | must-have |
| FR-12.2 | TDD compliance + behavioral coverage elevated to blocking gates for code tasks                                                                                  | must-have |

### FR-13: Agent Performance Guardrails

| Sub-Req | Description                                                                                                                                 | Priority    |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| FR-13.1 | Hard E2E timeouts per stage: startup 60s, suite 300s, **exploratory 180s/skill**, **adversarial 120s/skill**, shutdown 30s, total 600s/task | must-have   |
| FR-13.2 | Max 1 E2E retry per task (covers transient failures including **browser crashes**)                                                          | must-have   |
| FR-13.3 | Escalation if E2E fails across 2 consecutive replan iterations                                                                              | should-have |

### FR-14: Deterministic Output Contracts

| Sub-Req | Description                                                                                                            | Priority  |
| ------- | ---------------------------------------------------------------------------------------------------------------------- | --------- |
| FR-14.1 | All outputs structured YAML. E2E evidence follows schema. **Interaction logs are structured YAML, not free-form text** | must-have |
| FR-14.2 | E2E results in BOTH anvil_checks (SQL) and verification-reports (YAML) **with per-skill interaction logs**             | must-have |

---

## Non-Functional Requirements

| ID    | Title                  | Description                                                                                                                                | Priority |
| ----- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | -------- |
| NFR-1 | E2E Performance Budget | E2E per task ≤ 600s (suite + exploratory + adversarial). Total overhead ≤ 30 min. Pipeline ≤ 60 min                                        | must     |
| NFR-2 | Security               | No secrets in contracts, skills, interaction logs, screenshots, HAR files. Env vars by name only. Screenshots must not capture PII         | must     |
| NFR-3 | Reliability            | Hard timeouts on all E2E ops including each interaction step. Transient retry once (port conflict, browser crash). Deterministic fail-fast | must     |
| NFR-4 | Maintainability        | Contracts/skills versioned YAML alongside app code. Backward-compatible. Skills evolved as app changes                                     | must     |
| NFR-5 | Compatibility          | Framework-agnostic: any app type, any interaction tool. Contract's interaction_type is the abstraction layer                               | must     |
| NFR-6 | Scalability            | Up to 4 concurrent E2E instances with port isolation and **separate browser instances**                                                    | should   |
| NFR-7 | Observability          | All lifecycle events in anvil_checks. **Every interaction step logged.** Evidence manifests. Post-mortem interaction metrics               | must     |
| NFR-8 | **Determinism (NEW)**  | **All interaction steps pre-defined in skills. Agents follow steps exactly — no improvisation.** Ensures reproducibility                   | must     |

---

## Constraints & Assumptions

1. **Windows environment** with VS Code GitHub Copilot agent framework
2. **Orchestrator ≤ 550 line budget** — E2E coordination must be extractable to reference documents
3. **Dispatch concurrency cap of 4** — E2E concurrent instances cannot exceed 4
4. **SQLite-based evidence** (verification-ledger.db, WAL mode) — no external databases
5. **No container runtime assumed** — isolation via ports and process management only
6. **Additive schema changes only** — backward compatibility with existing pipeline runs
7. **Tool-access-matrix constraints** — verifier executes E2E; orchestrator coordinates via SQL gates only
8. **E2E contracts are project-specific** — not pipeline-global
9. **Browser automation requires Playwright CLI or equivalent** — must be declared in contract, not assumed
10. **Verifier interaction is deterministic** — all steps are pre-defined in skills, not improvised at runtime

### Assumptions

- Target projects can run on localhost with configurable ports
- E2E runners and interaction tools (Playwright CLI, curl, etc.) are pre-installed or installable via the E2E contract
- Health check endpoints or equivalent readiness probes are available for web/API apps
- The existing 3-iteration Pattern B replan loop is sufficient for E2E failure recovery
- Browser automation tools can run in headless mode on the target system

---

## Acceptance Criteria

### TDD Enforcement (AC-1 through AC-4)

| ID   | Criteria                                                                                                                                                                      | Method     |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| AC-1 | Implementation report for code tasks: `tdd_red_green.tests_written_first=true` AND `initial_run_failures > 0`. If either is false, verifier records `tdd-compliance` passed=0 | test       |
| AC-2 | Verifier records `check_name='tdd-compliance'` SQL INSERT. Must be passed=1 for task to proceed past Step 6                                                                   | test       |
| AC-3 | Non-code tasks with TDD skipped: `tdd_fallback_reason` field present and non-empty in implementation report                                                                   | inspection |
| AC-4 | Verify phase produces `tdd_red_green.verify_phase` with `get_errors_clean`, `typecheck_clean`, `tests_passing` boolean fields                                                 | inspection |

### E2E Contract with Interaction Capabilities (AC-5 through AC-8)

| ID   | Criteria                                                                                                                                                 | Method        |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| AC-5 | schemas.md contains complete E2E contract schema with **interaction_type, interaction_tools, readiness_checks, evidence_capture** plus all FR-2.1 fields | inspection    |
| AC-6 | Pipeline discovers `e2e-contract.yaml` at Step 0 and records path in pipeline_telemetry                                                                  | test          |
| AC-7 | Autonomous mode without contract: no e2e-\* checks; pipeline_telemetry has 'e2e-skipped'                                                                 | test          |
| AC-8 | Interactive mode without contract + High Risk tasks: ask_questions invoked for contract creation **including interaction type/tools setup**              | demonstration |

### E2E Skills — Interaction Procedures (AC-9 through AC-12)

| ID    | Criteria                                                                                                                                                                                                                                                 | Method     |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| AC-9  | Skills schema includes **type** (test-suite\|exploratory\|adversarial), **interaction** (browser\|api\|cli\|test-command), step-by-step actions (navigate/click/fill/http_request/etc.), assert objects, adversarial_variations with step overrides      | inspection |
| AC-10 | **Exploratory skills cause actual interaction:** verifier navigates pages, clicks elements, fills forms, makes API requests — NOT just runs a command. interaction_log contains per-step results with action/target/result/actual_behavior/evidence_path | test       |
| AC-11 | **Adversarial variations apply overrides and verify behavior:** inject SQL, send malformed JSON, etc. Each variation produces e2e-adversarial SQL INSERT. interaction_log shows override applied and actual behavior observed                            | test       |
| AC-12 | Autonomous mode: no skill creation, but ALL existing skills (including exploratory/adversarial) execute with full interaction                                                                                                                            | test       |

### Live App Interaction (AC-13 through AC-18)

| ID    | Criteria                                                                                                                                                                                         | Method     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| AC-13 | Implementer prohibits app startup, browser launch, HTTP requests to running app, or any live interaction                                                                                         | inspection |
| AC-14 | High Risk task produces SQL INSERTs: e2e-contract-found, e2e-instance-start, e2e-readiness, e2e-suite-execution, **e2e-exploratory**, e2e-adversarial, e2e-instance-shutdown, e2e-test-execution | test       |
| AC-15 | **Browser interaction proof:** evidence_output_dir contains screenshots of actual rendered pages; interaction_log shows navigate/click/fill actions                                              | test       |
| AC-16 | **API interaction proof:** evidence_output_dir contains response logs with real HTTP status codes and bodies from the running app                                                                | test       |
| AC-17 | **CLI interaction proof:** interaction_log shows run_command/send_input actions with captured stdout/stderr                                                                                      | test       |
| AC-18 | Verifier read-only for source code; only evidence artifacts written                                                                                                                              | inspection |

### Parallel Safety (AC-19 through AC-22)

| ID    | Criteria                                                                                                    | Method        |
| ----- | ----------------------------------------------------------------------------------------------------------- | ------------- |
| AC-19 | Concurrent E2E verifiers use distinct ports AND **separate browser instances**                              | test          |
| AC-20 | At most max_concurrent_instances tasks with e2e_required=true in any single sub-wave                        | test          |
| AC-21 | Hung app killed after ready_timeout_ms; **browser/interaction tools also closed**; e2e-readiness passed=0   | demonstration |
| AC-22 | No orphaned processes after verification — **both app and browser PIDs** tracked and killed/shutdown logged | test          |

### Workflow Lanes (AC-23 through AC-26)

| ID    | Criteria                                                                                                   | Method     |
| ----- | ---------------------------------------------------------------------------------------------------------- | ---------- |
| AC-23 | All tasks in plan-output.yaml have workflow_lane field: 'unit-only', 'unit-integration', or 'full-tdd-e2e' | inspection |
| AC-24 | Tasks with 🔴 risk file have workflow_lane='full-tdd-e2e'                                                  | test       |
| AC-25 | 'full-tdd-e2e' task verification: ≥ 4 passed checks including e2e-test-execution                           | test       |
| AC-26 | 'unit-only' task: **no e2e-\* checks, no browser launch, no HTTP requests to app, no live interaction**    | test       |

### Context, Interactive Mode, Orchestrator, Practices (AC-27 through AC-31)

| ID    | Criteria                                                                                           | Method        |
| ----- | -------------------------------------------------------------------------------------------------- | ------------- |
| AC-27 | global-operating-rules.md contains mandatory 200-line read_file limit                              | inspection    |
| AC-28 | E2E output_snippet in anvil_checks ≤ 500 characters                                                | test          |
| AC-29 | Interactive mode: ask_questions invoked on ambiguity detection                                     | demonstration |
| AC-30 | Orchestrator run_in_terminal restricted to SQLite/git — **no browser, no HTTP, no E2E execution**  | inspection    |
| AC-31 | Adversarial reviewer checks YAGNI/KISS/DRY; review-perspectives.md includes code quality dimension | inspection    |

### Replay, Completion, Guardrails, Outputs (AC-32 through AC-38)

| ID    | Criteria                                                                                                                                                                             | Method        |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------- |
| AC-32 | Evidence stored in evidence_output_dir/<task-id>/<skill-id>/; **evidence-manifest.yaml** written; artifact_paths in verification report                                              | test          |
| AC-33 | Task with e2e_required=true not marked COMPLETE without e2e-test-execution passed=1 **(suite + exploratory + adversarial all passed)**                                               | test          |
| AC-34 | EG-7 (behavioral coverage) and tdd-compliance are blocking for code tasks                                                                                                            | inspection    |
| AC-35 | Total E2E per task ≤ 600s **(including suite + exploratory + adversarial)**; timeout → app killed + browser closed + passed=0                                                        | test          |
| AC-36 | Max 2 e2e-test-execution records per task per round (1 attempt + 1 retry)                                                                                                            | test          |
| AC-37 | E2E results in both anvil_checks SQL and verification-reports YAML **with structured interaction logs**                                                                              | test          |
| AC-38 | **CORE DIFFERENTIATOR:** Evidence proves agents actually interacted with the app — screenshots of rendered pages, real HTTP responses, CLI output — not just test command exit codes | demonstration |

---

## Edge Cases & Error Handling

| ID    | Input / Condition                                                         | Expected Behavior                                                                                                                           | Severity if Missed |
| ----- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| EC-1  | E2E contract exists but start_command fails                               | Verifier records e2e-instance-start passed=0, **no browser launch**, skips remaining E2E, returns NEEDS_REVISION                            | High               |
| EC-2  | App starts but never passes ready_check within timeout                    | Force-kill after ready_timeout_ms, **do NOT launch browser**, record e2e-readiness passed=0                                                 | Critical           |
| EC-3  | Two verifiers assigned same port                                          | Second verifier detects conflict, increments port, retries. No port → queue for next sub-wave                                               | High               |
| EC-4  | **Interaction step flaky** (element not found due to rendering delay)     | Step retries within timeout_ms. on_failure='continue' proceeds to next step. Flakiness logged in interaction_log                            | Medium             |
| EC-5  | **interaction_type='browser' but Playwright not installed**               | Record e2e-contract-validation passed=0; fall back to test-suite-only or skip E2E                                                           | High               |
| EC-6  | tdd_red_green claimed but tests don't import production modules           | Verifier detects mismatch, tdd-compliance passed=0, NEEDS_REVISION                                                                          | High               |
| EC-7  | Interactive mode — user declines E2E contract creation                    | Proceed without E2E; record user decision; classify all tasks as unit-only/unit-integration                                                 | Medium             |
| EC-8  | **Adversarial variation discovers real bug** (SQL injection bypasses)     | e2e-adversarial passed=0; interaction_log captures actual behavior; **screenshot of unauthorized access**; NEEDS_REVISION                   | Critical           |
| EC-9  | Autonomous mode + existing contract + all skill types                     | ALL skill types execute (commands, **browser interaction**, **API probing**, adversarial); no prompts; no skill creation                    | High               |
| EC-10 | App requires database; E2E contract has DATABASE_URL in start_env         | Each E2E instance uses unique URL or non-parallel mode; warn if not configured                                                              | High               |
| EC-11 | **Verifier crashes mid-interaction** (context overflow with browser open) | **Both app AND browser orphaned processes detected**; PID cleanup attempted; re-dispatch within limits                                      | Critical           |
| EC-12 | Skill file referenced but missing or malformed                            | Record e2e-contract-validation passed=0 for missing skill; execute remaining valid skills                                                   | Medium             |
| EC-13 | **Stale skill — CSS selector doesn't exist on page** (app changed)        | Step fails with 'element not found' after timeout_ms; on_failure followed; **screenshot of actual page state**; knowledge agent flags stale | Medium             |
| EC-14 | **API returns unexpected content type** (HTML instead of JSON)            | Step records actual content type and body (truncated); assertion fails; evidence captures full response; verifier does NOT crash            | Medium             |
| EC-15 | **SPA — navigation doesn't cause page reload**                            | navigate steps wait for content changes, not page load events; assert/expect verify correct content                                         | High               |
| EC-16 | **interaction_type='none' (library) but skills exist**                    | Only test-suite skills execute; exploratory/adversarial skipped with logged reason; e2e-exploratory/adversarial recorded as N/A             | Medium             |

---

## User Stories / Flows

### Story 1: Implementer Follows Mandatory TDD

1. Planner assigns task TASK-003 with `workflow_lane='full-tdd-e2e'` and `e2e_required=true`
2. Implementer reads task from `tasks/TASK-003.yaml`
3. Implementer writes failing test → confirms test FAILS (exit code ≠ 0)
4. Implementer writes minimal production code → confirms tests PASS
5. Implementer runs VERIFY phase: get_errors → typecheck → full test suite
6. Implementer records `tdd_red_green` with `tests_written_first=true`, `initial_run_failures=3`, `verify_phase={get_errors_clean: true, typecheck_clean: true, tests_passing: true}`
7. Implementer produces `implementation-reports/TASK-003.yaml` with behavioral_coverage mapping each AC to a test function

### Story 2: Verifier Runs Full E2E with App Interaction (Web App)

1. Verifier reads `implementation-reports/TASK-003.yaml` and sees `e2e_required=true`
2. Verifier reads E2E contract from `e2e-contract.yaml`, sees `interaction_type: browser`, `interaction_tools.browser.tool: playwright`
3. Verifier validates contract → records `e2e-contract-found` passed=1
4. **Phase 1 — Setup:** Verifier starts app: `dotnet run --urls=http://localhost:5002` (port 5000 + ordinal 2). Polls `http://localhost:5002/health` until 200. Records `e2e-readiness` passed=1. **Launches Playwright browser instance.**
5. **Phase 2 — Suite:** Verifier runs `npx playwright test --base-url=http://localhost:5002`. All tests pass → records `e2e-suite-execution` passed=1
6. **Phase 3 — Exploratory:** Verifier executes SKILL-001 "test-login-flow":
   - Step 1: **Navigates browser to /login** → takes screenshot → asserts #login-form visible ✓
   - Step 2: **Fills #email with "test@example.com"** ✓
   - Step 3: **Fills #password with "password123"** ✓
   - Step 4: **Clicks #submit** → takes screenshot → asserts URL matches /dashboard ✓
   - Step 5: **Navigates to /profile** → takes screenshot → asserts "test@example.com" visible ✓
   - Records `e2e-exploratory` passed=1, writes interaction_log with 5 steps all passed
7. **Phase 4 — Adversarial:** Verifier executes adversarial variations of SKILL-001:
   - ADV-001: Replays login flow but **fills #email with "' OR 1=1; --"** → asserts .error-message visible ✓
   - ADV-002: Replays login flow but **fills #password with ""** → asserts .validation-error visible ✓
   - ADV-003: Replays login flow but **fills #email with `<script>alert('xss')</script>`** → asserts no script element ✓
   - Records 3× `e2e-adversarial` all passed=1
8. **Phase 5 — Teardown:** Verifier closes browser. Sends SIGTERM to app, waits 10s → records `e2e-instance-shutdown` passed=1. Records composite `e2e-test-execution` passed=1
9. Verifier writes evidence-manifest.yaml with paths to 8 screenshots, interaction_logs for SKILL-001 + adversarial variations

### Story 3: Verifier Probes API Endpoints (API App)

1. Verifier reads contract, sees `interaction_type: api`, `interaction_tools.api.tool: curl`
2. Starts app, waits for readiness
3. **Runs test-suite skills** (e.g., `npm test`)
4. **Executes API exploratory skill SKILL-002:**
   - Step 1: **POST /api/users with JSON body** → captures response → asserts 201 ✓
   - Step 2: **GET /api/users/1** → captures response → asserts body contains "Test User" ✓
   - Step 3: **DELETE /api/users/1** → asserts 204 ✓
   - Step 4: **GET /api/users/1** → asserts 404 ✓
5. **Executes adversarial variations:**
   - ADV-010: **POST /api/users with malformed JSON** → asserts 400 ✓
   - ADV-011: **GET /api/users/1 OR 1=1** → asserts 400 ✓
6. All response bodies and status codes logged in evidence_output_dir

### Story 4: Autonomous Mode with Existing Contract & Skills

1. Pipeline starts in autonomous mode; Step 0 discovers `e2e-contract.yaml` with skills of all types
2. Pipeline records contract path in `pipeline_telemetry`
3. Planner classifies TASK-007 as 🔴 → `workflow_lane='full-tdd-e2e'`, `e2e_required=true`
4. **No interactive prompts occur; no new skills are created**
5. Verifier executes ALL skill types: runs test commands, **navigates browser following exploratory skills**, **applies adversarial variations** — all automatically
6. E2E passes → task proceeds to code review

### Story 5: Interactive Mode Without Contract

1. Pipeline starts in interactive mode; Step 0 does NOT find `e2e-contract.yaml`
2. Planner classifies TASK-012 as 🔴 (security-sensitive API change)
3. Orchestrator detects High Risk task + no contract → invokes `ask_questions`:
   - "No E2E contract found. This project has High Risk tasks. What type of application is this?"
   - Options: Web app / API / CLI / Library
4. User selects "Web app" → pipeline proposes `interaction_type: browser`, `interaction_tools.browser.tool: playwright`
5. Pipeline guides through start_command, ready_check, base_url
6. **Pipeline analyzes app routes and proposes exploratory skills:**
   - "I found routes: /login, /dashboard, /profile, /settings. Shall I create interaction skills for these flows?"
7. User approves → pipeline creates skills with navigation steps, form fills, and adversarial variations
8. Contract + skills written to project; E2E verification runs

### Story 6: Adversarial Interaction Finds a Real Bug

1. Verifier executes SKILL-001 adversarial variation ADV-001 ("SQL injection in email")
2. **Fills login email with `' OR 1=1; --`** and submits
3. Expected behavior: "Error message displayed, no redirect to dashboard"
4. **Actual behavior: Browser redirected to /dashboard with session cookie** (SQL injection succeeded!)
5. Verifier **captures screenshot of dashboard** showing unauthorized access
6. Records `e2e-adversarial` passed=0 with failure details in interaction_log
7. Verifier returns NEEDS_REVISION → orchestrator routes to replan loop
8. Implementer receives failure evidence (screenshot + interaction_log) and must fix the vulnerability

---

## Test Scenarios

| #    | Scenario                                                                                                                  | Validates    | Type        |
| ---- | ------------------------------------------------------------------------------------------------------------------------- | ------------ | ----------- |
| T-1  | Dispatch implementer with code task; verify implementation report contains tdd_red_green with initial_run_failures > 0    | AC-1, AC-4   | Automated   |
| T-2  | Query anvil_checks for tdd-compliance check after verifier completes; verify passed=1 for valid TDD                       | AC-2         | Automated   |
| T-3  | Dispatch implementer with docs-only task; verify tdd_fallback_reason is present                                           | AC-3         | Automated   |
| T-4  | Place e2e-contract.yaml in project; run pipeline; verify pipeline_telemetry has contract path                             | AC-6         | Automated   |
| T-5  | Remove e2e-contract.yaml; run autonomous pipeline; verify no e2e-\* checks + 'e2e-skipped' in telemetry                   | AC-7         | Automated   |
| T-6  | **Define exploratory skill; run verifier; verify interaction_log has navigate/click/fill actions with per-step results**  | AC-10        | Automated   |
| T-7  | **Define adversarial variation; run verifier; verify override applied and e2e-adversarial SQL INSERT produced**           | AC-11        | Automated   |
| T-8  | Run 2 concurrent verifiers in same wave; verify distinct ports **and separate browser instances** in e2e-instance-start   | AC-19        | Automated   |
| T-9  | Set ready_timeout_ms=5000; start app that takes 30s to boot; verify force-kill + **no browser launched** + passed=0       | AC-21        | Automated   |
| T-10 | Classify task with 🔴 file; verify workflow_lane='full-tdd-e2e' in plan-output.yaml                                       | AC-24        | Automated   |
| T-11 | Run full-tdd-e2e task; verify ≥ 4 passed checks including e2e-test-execution                                              | AC-25        | Automated   |
| T-12 | Run unit-only task; verify zero e2e-\* checks **and no browser launch**                                                   | AC-26        | Automated   |
| T-13 | Task with e2e_required=true; make e2e-test-execution fail; verify task not marked COMPLETE                                | AC-33        | Automated   |
| T-14 | Run E2E that exceeds 600s; verify timeout + **app and browser killed** + passed=0                                         | AC-35        | Automated   |
| T-15 | Verify e2e results in both SQL (anvil_checks) and YAML (verification-reports) **with interaction_logs**                   | AC-37        | Automated   |
| T-16 | **Web app: verify evidence_output_dir contains .png screenshots of actual rendered pages**                                | AC-15, AC-38 | Automated   |
| T-17 | **API app: verify evidence_output_dir contains response logs with real HTTP status codes and bodies**                     | AC-16, AC-38 | Automated   |
| T-18 | Interactive mode, missing contract, High Risk task: verify ask_questions invoked **with interaction_type selection**      | AC-8         | Manual/Demo |
| T-19 | **SPA navigation: verify exploratory skill waits for content change, not page reload**                                    | EC-15        | Automated   |
| T-20 | **Stale skill with invalid CSS selector: verify step fails with screenshot of actual page, on_failure behavior followed** | EC-13        | Automated   |

---

## Dependencies & Risks

### File Modification Dependencies

| Impact                              | Files                                                                                                                                             |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **HIGH** (major structural)         | verifier.agent.md, orchestrator.agent.md, implementer.agent.md, schemas.md, sql-templates.md, **tool-access-matrix.md** (verifier tools expanded) |
| **MEDIUM** (schema/field additions) | planner.agent.md, dispatch-patterns.md, global-operating-rules.md                                                                                 |
| **LOW** (minor additions)           | spec.agent.md, designer.agent.md, adversarial-reviewer.agent.md, knowledge-agent.agent.md, severity-taxonomy.md, evaluation-schema.md             |
| **NEW** (new files)                 | e2e-integration.md (reference doc: E2E coordination rules, interaction protocol, contract schema, skill examples, safety rules)                   |

### External Dependencies

| Dependency                         | Required         | Notes                                                                         |
| ---------------------------------- | ---------------- | ----------------------------------------------------------------------------- |
| **Playwright CLI** (or equivalent) | Per project      | For browser interaction. Must be pre-installed or install_command in contract |
| Node.js 18+                        | For Playwright   | Only if project uses Playwright CLI for browser interaction                   |
| **curl/fetch**                     | For API apps     | For API interaction. Usually pre-installed on most systems                    |
| Target app health endpoint         | For web/API apps | Required for ready_check; app must expose readiness probe                     |

### Risk Register

| Risk                                                      | Likelihood | Impact | Mitigation                                                                       |
| --------------------------------------------------------- | ---------- | ------ | -------------------------------------------------------------------------------- |
| **Exploratory interaction flakiness** (timing, rendering) | High       | High   | Configurable step timeouts; on_failure behavior; max 1 retry; flakiness tracking |
| E2E flakiness causes pipeline thrashing                   | Medium     | High   | Max 1 retry, escalation after 2 replan iterations                                |
| Orchestrator exceeds 550-line budget                      | High       | Medium | Extract E2E coordination to reference document                                   |
| Port conflicts in parallel E2E                            | Low        | Medium | Deterministic port_range_start + ordinal; conflict detection + retry             |
| **E2E + interaction adds unacceptable latency**           | High       | High   | 10-minute hard timeout; risk-based lanes; only High Risk tasks                   |
| Context overflow from E2E evidence                        | Medium     | Medium | Summarized output_snippet (≤500 chars); artifacts referenced by path             |
| **Browser/interaction tool not installed**                | Medium     | Medium | Contract validation at Step 0; graceful fallback to test-suite only              |
| **Stale skills (app changed, selectors invalid)**         | Medium     | Medium | Step on_failure behavior; knowledge agent flags staleness; skill updates         |
| E2E contract missing for production projects              | High       | Medium | Graceful degradation — unit-only verification when no contract exists            |

---

## Schema Changes Summary

### Modified Schemas

| Schema                           | Change                                                                                                                      |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Schema 6 (task-schema)           | Add: `workflow_lane`, `e2e_required`, `e2e_contract_path` fields                                                            |
| Schema 7 (implementation-report) | Add: `tdd_red_green.verify_phase` object; make `tdd_fallback_reason` required when TDD skipped                              |
| Schema 8 (verification-report)   | Add: E2E results section with lifecycle phases, **interaction_logs per skill**, artifact_paths, evidence-manifest reference |
| test_method enum                 | Add `e2e` value alongside existing inspection/demonstration/test/analysis                                                   |
| **tool-access-matrix.md**        | **Add browser automation, HTTP client, and CLI interaction tools for verifier role**                                        |

### New Schemas

| Schema            | Description                                                                                                                                         |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| E2E Contract      | Machine-readable YAML: app lifecycle, runner, **interaction_type, interaction_tools, evidence_capture**, parallelism, skills                        |
| E2E Skills        | Reusable procedures: **type** (test-suite/exploratory/adversarial), **interaction steps**, outcomes, **adversarial_variations with step overrides** |
| Interaction Log   | **NEW:** Per-skill step-by-step log: steps_executed, steps_passed, steps_failed, per-step action/target/result/actual_behavior/evidence_path        |
| Evidence Manifest | **NEW:** Machine-readable index of all captured artifacts with metadata (type, skill-id, step-order, timestamp, size)                               |

### SQL Changes

| Change                                                                                                                                                                                                                                | Type                                                 |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| New check_names: e2e-contract-found, e2e-contract-validation, e2e-instance-start, e2e-readiness, **e2e-suite-execution**, **e2e-exploratory**, e2e-adversarial, e2e-instance-shutdown, e2e-test-execution (composite), tdd-compliance | Additive (no DDL change)                             |
| New evidence gates: EG-8 (E2E evidence for High Risk), EG-9 (E2E lifecycle complete)                                                                                                                                                  | New SQL queries                                      |
| EG-7 promotion                                                                                                                                                                                                                        | Change from INFORMATIONAL to blocking for code tasks |
| Optional: e2e_runs table for lifecycle tracking                                                                                                                                                                                       | Additive DDL (CREATE TABLE IF NOT EXISTS)            |

---

_Specification produced by the Spec Agent (Revision 2). Core change from revision 1: E2E testing means agents **actually interact with the running application** — navigate browsers, make API calls, run CLI commands — not just execute pre-written test suites. See [spec-output.yaml](spec-output.yaml) for the typed machine-readable version._
