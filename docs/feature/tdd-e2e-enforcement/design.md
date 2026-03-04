# TDD & E2E Testing Enforcement — Design Document (R2 Revision)

> **Feature:** `tdd-e2e-enforcement`
> **Run ID:** `2026-03-04T00:00:00Z`
> **Designer:** `designer` (Step 3)
> **Revision:** Round 2 (post-adversarial review)
> **Architecture:** Direction C — Hybrid Extended Verifier + Reference Document
> **Overall Risk:** 🟡
> **Decisions:** 26 (18 updated + 8 new) | **Files Modified:** 13 | **Files Created:** 1 | **Files Unchanged:** 5 | **Deviation Records:** 3

---

## 1. Title & Summary

This design transforms the agent pipeline into a system where **TDD and E2E testing are core enforcement mechanisms**, not optional practices. The implementer gains a mandatory RED-GREEN-VERIFY cycle with auditable evidence. The verifier gains a new **Tier 5 (E2E Verification)** that starts the application, runs test suites, performs step-by-step exploratory interaction, and applies adversarial variations — all driven by a formal **E2E contract** and **skills schema**. A **risk-based workflow lane system** limits E2E cost to High Risk tasks only.

The key architectural choice is **Direction C: Hybrid Extended Verifier + Reference Document**, which extends the existing verifier rather than creating a new agent, and extracts E2E coordination rules into a new `e2e-integration.md` reference document to keep the orchestrator within its 550-line budget.

**R2 Revision Summary:** This revision addresses 33 findings from 3 adversarial reviewers (security-sentinel, architecture-guardian, pragmatic-verifier). All 3 Critical and 18 Major findings are resolved. Key additions: script-generation execution model (D-20), structured command format with command allowlist (D-21, D-22), EG-2/EG-10 gate consolidation for orchestrator budget (D-16), two-phase contract validation (D-23), verifier sub-role modularity (D-24), evidence sanitization pipeline (D-25), per-variation adversarial SQL records (D-26), researcher-mediated skill creation (D-19).

---

## 2. Context & Inputs

### Inputs Consumed

| Input                        | Source     | Key Takeaways                                                                                       |
| ---------------------------- | ---------- | --------------------------------------------------------------------------------------------------- |
| `initial-request.md`         | User       | "Massive overhaul" — TDD & E2E as core enforcement. E2E = agents LITERALLY interact with live apps. |
| `spec-output.yaml`           | Spec Agent | 9 CR, 14 FR groups (55 sub-reqs), 38 AC, 16 EC, 10 constraints, 8 NFRs. Direction C recommended.    |
| `research/architecture.yaml` | Researcher | Orchestrator at 528/550 lines. 9 agents, 18 files. Existing 4-tier cascade in verifier.             |
| `research/impact.yaml`       | Researcher | 14 of 18 files require modification. Verifier most heavily impacted.                                |
| `research/dependencies.yaml` | Researcher | SQLite evidence ledger with 4 tables. No external database. Framework-agnostic.                     |
| `research/patterns.yaml`     | Researcher | Established patterns: reference documents for complex rules, typed YAML schemas, SQL evidence.      |

### R2 Review Inputs

| Input                                             | Reviewer              | Findings | Critical | Major | Minor |
| ------------------------------------------------- | --------------------- | -------- | -------- | ----- | ----- |
| `review-findings/design-security-sentinel.md`     | security-sentinel     | 11       | 2        | 5     | 4     |
| `review-findings/design-architecture-guardian.md` | architecture-guardian | 11       | 1        | 6     | 4     |
| `review-findings/design-pragmatic-verifier.md`    | pragmatic-verifier    | 14       | 0        | 9     | 5     |

### Key Constraints

- Orchestrator ≤ 550 line budget (currently 528; R2 target: 543/550)
- Existing dispatch concurrency cap: 4 agents per wave
- No container runtime — isolation via ports and processes only
- Schema changes must be additive (backward-compatible)
- Verifier interaction must be deterministic (pre-defined skills, no improvisation)
- Windows environment with VS Code GitHub Copilot agent framework

### Pushback Concerns Addressed

| #   | Concern                            | Severity | Mitigation in Design                                                                |
| --- | ---------------------------------- | -------- | ----------------------------------------------------------------------------------- |
| 1   | Orchestrator line budget risk      | Major    | D-2: E2E coordination extracted; D-16 R2: EG-2/EG-10 consolidation (543/550 target) |
| 2   | E2E flakiness amplification        | Major    | D-12: Max 1 retry per task; D-9: hard timeouts; D-18: deterministic skills          |
| 3   | Pipeline wall-clock time 3-8x      | Major    | D-8: Risk-based lanes limit E2E to High Risk only; D-9: 600s cap per task           |
| 4   | Blast radius 14/18 files           | Major    | Phased implementation recommended; additive schema changes (D-15)                   |
| 5   | Sophisticated verifier tool access | Critical | D-20: script-generation model; D-22: command allowlist; D-11: scoped to Tier 5 only |

---

## 3. High-Level Architecture

### 3.1 Architecture Selection (D-1)

**Selected: Direction C — Hybrid Extended Verifier + Reference Document**

The verifier gains Tier 5 (E2E Verification) with a 5-phase lifecycle and script-generation execution model (D-20). E2E coordination rules are extracted into `e2e-integration.md` (~500 lines with per-agent reading guides). Agent count stays at 9. The orchestrator gains parameterized lane-aware evidence gates (D-16 R2: EG-10 consolidation) and E2E concurrency management through minimal additions and reference to `e2e-integration.md`.

**R2 enhancements:**

- Verifier organized into sub-roles (D-24) with conditional Tier 5 loading — non-E2E tasks pay no context cost
- Script-generation execution model (D-20) — verifier generates test scripts from skill steps, never constructs shell commands
- Command allowlist (D-22) — regex patterns in tool-access-matrix.md restrict Tier 5 command execution
- Trust level system (D-3) — contract content classified as trusted/semi-trusted/untrusted

**Rejected alternatives:**

- **Direction A (Verifier-Only, No Extraction):** Orchestrator exceeds 550-line cap. No rule separation.
- **Direction B (Dedicated E2E Agent):** Adds 10th agent type. Overlapping scope with verifier Tiers 3-4. Orchestrator dispatch complexity increases significantly.

### 3.2 System Overview

```
Pipeline Steps (unchanged, with R2 enhancements):

Step 0: Init — E2E contract discovery + structural validation (D-23 Phase 1)
Step 0.5: Skill Creation — Researcher generates skills if contract discovered (D-19)
Step 1: Research (×4 parallel) — unchanged
Step 2: Spec — unchanged
Step 3: Design — unchanged
Step 3b: Design Review — unchanged
Step 4: Plan — workflow_lane, e2e_required fields added
Step 4a: Approval Gate — unchanged
Step 5: Implement — mandatory RED-GREEN-VERIFY TDD
Step 6: Verify — Tiers 1-4 + NEW Tier 5 (E2E) for full-tdd-e2e lane
         Tier 5 uses script-generation model (D-20) with command allowlist (D-22)
Step 6a: Runtime Smoke — preserved as fallback when no E2E contract exists (DR-1)
         Skipped when Tier 5 passes (D-8 explicit skip semantics)
Step 7: Code Review — TDD compliance added to review dimensions
Step 8: Knowledge Capture — E2E metrics in post-mortem
Step 9: Auto-Commit — unchanged
```

### 3.3 File Inventory

#### Files Created (1)

| File                 | Purpose                                                                                                                                                                                                 | Est. Lines | Risk |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ---- |
| `e2e-integration.md` | E2E coordination rules, contract schema, skills schema, interaction protocol, per-agent reading guides, trust levels, command allowlist reference, evidence sanitization rules, DB isolation strategies | ~500       | 🔴   |

#### Files Modified (13)

| File                            | Key Changes                                                                                                                                                                                                 | Risk |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| `verifier.agent.md`             | Tier 5 E2E (5-phase lifecycle, sub-role organization D-24), script-generation (D-20), command allowlist (D-22), TDD compliance with secondary heuristics (D-13), evidence sanitization (D-25), PID tracking | 🔴   |
| `implementer.agent.md`          | Mandatory RED-GREEN-VERIFY, tightened TDD Fallback, verify_phase fields, E2E prohibition                                                                                                                    | 🔴   |
| `orchestrator.agent.md`         | Step 0 structural validation (D-23), parameterized EG-10 (D-16 R2), EG-8/EG-9, E2E concurrency + DB isolation (D-17), net +15 lines (543/550)                                                               | 🔴   |
| `planner.agent.md`              | workflow_lane, e2e_required, e2e_contract_path fields                                                                                                                                                       | 🟡   |
| `researcher.agent.md`           | Step 0.5 skill creation workflow (D-19)                                                                                                                                                                     | 🟡   |
| `schemas.md`                    | E2E contract schema (structured command format D-21), skills schema, task-schema extension, impl-report extension                                                                                           | 🟡   |
| `sql-templates.md`              | E2E INSERT templates, EG-8/EG-9/EG-10 queries, per-variation adversarial records (D-26), composite index                                                                                                    | 🟡   |
| `tool-access-matrix.md`         | Verifier tier5_command_allowlist regex patterns (D-22)                                                                                                                                                      | 🟡   |
| `global-operating-rules.md`     | Mandatory 200-line rule, E2E safety rules, TDD immutable rule                                                                                                                                               | 🟡   |
| `dispatch-patterns.md`          | E2E parallel safety, port allocation, DB isolation strategies (D-17), concurrency cap                                                                                                                       | 🟡   |
| `adversarial-reviewer.agent.md` | TDD compliance review, test writing guidelines, E2E quality checks, evidence hash verification (D-25), command audit trail review (D-22)                                                                    | 🟡   |
| `evaluation-schema.md`          | E2E evaluation criteria                                                                                                                                                                                     | 🟢   |
| `review-perspectives.md`        | E2E review guidance per perspective                                                                                                                                                                         | 🟢   |
| `severity-taxonomy.md`          | E2E failure severity examples                                                                                                                                                                               | 🟢   |

#### Files Unchanged (5)

| File                         | Reason                                                    |
| ---------------------------- | --------------------------------------------------------- |
| `knowledge-agent.agent.md`   | Consumes new E2E evidence via existing SQL query patterns |
| `spec.agent.md`              | Produces requirements; does not interact with E2E/TDD     |
| `designer.agent.md`          | Produces designs; no E2E/TDD interaction                  |
| `context7-integration.md`    | Independent of TDD/E2E enforcement                        |
| `feature-workflow.prompt.md` | Parameterized template; no E2E content needed             |

---

## 4. Data Models & DTOs

### 4.1 E2E Contract Schema (New — in schemas.md)

The E2E contract is a project-level YAML file (`e2e-contract.yaml` at project root or `.e2e/contract.yaml`) defining how to start the application, verify readiness, configure interaction tools, and discover skills. Per D-3, contract content is classified by trust level: contract structure is **trusted** (validated at Step 0), command executables are **semi-trusted** (allowlisted D-22), and skill content (selectors, form values) is **untrusted** (sanitized before use D-25).

**Key field groups:**

| Group              | Fields                                                                                                                                                                   | Purpose                                          |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------ |
| App Lifecycle (R2) | `app_type`, `start_command: {executable, args[], env{}}`, `ready_check`, `ready_timeout_ms`, `base_url`, `shutdown_command: {executable, args[]}`, `shutdown_timeout_ms` | How to start/stop — **structured format (D-21)** |
| Port Management    | `default_port`, `port_env_var`, `port_range_start`, ~~`port_range_end`~~ (port validated [1024, 65535] per D-6)                                                          | Isolated port allocation per D-6                 |
| Runner             | `e2e_runner`, `e2e_command`, `e2e_config_path`, `e2e_env`                                                                                                                | Pre-written test suite execution                 |
| Parallelism (R2)   | `supports_parallel`, `requires_isolated_instance`, `max_concurrent_instances`, `db_isolation_strategy` (D-17)                                                            | Concurrency + DB isolation per D-17              |
| Interaction        | `interaction_type`, `interaction_tools`, `readiness_checks`, `evidence_capture`                                                                                          | Exploratory/adversarial interaction config       |
| Skills             | `skills`, `skill_discovery_path`                                                                                                                                         | Inline skills or directory reference per D-4     |
| Evidence           | `screenshot_on_failure`, `trace_on_failure`, `video_recording`, `har_capture`, `evidence_output_dir`                                                                     | Proof capture configuration                      |

**R2: Structured command format (D-21)**

```yaml
# BEFORE (R1 — rejected as injection risk):
start_command: "npm run dev -- --port 9001"

# AFTER (R2 — structured format):
start_command:
  executable: "npm"   # Must match tier5_command_allowlist (D-22)
  args: ["run", "dev", "--", "--port", "{port}"]
  env:
    NODE_ENV: "test"  # Values validated against pattern allowlist
```

**R2: DB isolation strategy (D-17)**

```yaml
db_isolation_strategy: "per-instance-db" # | "transaction-rollback" | "shared-with-locking"
```

### 4.2 E2E Skills Schema (New — in schemas.md)

Skills define reusable interaction procedures. Per D-18, all steps are pre-defined — no agent improvisation.

**Skill structure:**

| Field                    | Type    | Purpose                                                            |
| ------------------------ | ------- | ------------------------------------------------------------------ |
| `id`                     | string  | Unique identifier (e.g., `SKILL-001`)                              |
| `name`                   | string  | Human-readable name                                                |
| `type`                   | enum    | `test-suite` \| `exploratory` \| `adversarial`                     |
| `interaction`            | enum    | `browser` \| `api` \| `cli` \| `test-command`                      |
| `steps`                  | list    | Ordered interaction steps (action, target, value, assert, capture) |
| `adversarial_variations` | list    | Override objects for adversarial testing                           |
| `timeout_ms`             | integer | Max time for entire skill (default 60000)                          |

**Step structure:**

| Field        | Type         | Purpose                                                                                      |
| ------------ | ------------ | -------------------------------------------------------------------------------------------- |
| `order`      | integer      | Execution sequence                                                                           |
| `action`     | string       | `navigate`, `click`, `fill`, `submit`, `http_request`, `run_command`, `assert_visible`, etc. |
| `target`     | string       | CSS selector, URL, API endpoint, or CLI command                                              |
| `value`      | string\|null | Input value for fill/submit/request body                                                     |
| `assert`     | object\|null | Machine-checkable assertion (`type` + `value`)                                               |
| `capture`    | string\|null | Evidence capture (`screenshot`, `response`, `har`, etc.)                                     |
| `on_failure` | enum         | `fail` \| `continue` \| `skip_remaining`                                                     |

### 4.3 Task Schema Extension (Schema 6 — additive per D-15)

Three new fields:

| Field               | Type         | Default           | Purpose                                             |
| ------------------- | ------------ | ----------------- | --------------------------------------------------- |
| `workflow_lane`     | enum         | Derived from risk | `unit-only` \| `unit-integration` \| `full-tdd-e2e` |
| `e2e_required`      | boolean      | `false`           | Whether E2E verification is required                |
| `e2e_contract_path` | string\|null | `null`            | Path to discovered E2E contract                     |

### 4.4 Implementation Report Extension (Schema 7 — additive)

New field under `tdd_red_green`:

| Field                           | Type         | Purpose                           |
| ------------------------------- | ------------ | --------------------------------- |
| `verify_phase.get_errors_clean` | boolean      | get_errors produced no new errors |
| `verify_phase.typecheck_clean`  | boolean      | Type check passed                 |
| `verify_phase.tests_passing`    | boolean      | All tests pass after GREEN phase  |
| `tdd_fallback_reason`           | string\|null | Required when TDD is skipped      |

### 4.5 Verification Report Extension (Schema 8 — additive)

New fields:

| Field                                | Type          | Purpose                            |
| ------------------------------------ | ------------- | ---------------------------------- |
| `e2e_results`                        | object        | Aggregate E2E verification results |
| `e2e_results.suite_passed`           | boolean\|null | Test suite execution result        |
| `e2e_results.exploratory_passed`     | boolean\|null | Exploratory interaction result     |
| `e2e_results.adversarial_passed`     | boolean\|null | Adversarial interaction result     |
| `e2e_results.interaction_logs`       | list          | Per-skill interaction logs         |
| `e2e_results.artifact_paths`         | list          | Paths to evidence files            |
| `e2e_results.evidence_manifest_path` | string\|null  | Path to evidence-manifest.yaml     |

### 4.6 New check_name Patterns (SQL)

| Pattern                   | Phase   | Writer   | Purpose                              |
| ------------------------- | ------- | -------- | ------------------------------------ |
| `tdd-compliance`          | `after` | Verifier | TDD cycle verified                   |
| `tdd-fallback`            | `after` | Verifier | TDD was skipped with recorded reason |
| `e2e-contract-found`      | `after` | Verifier | Contract exists and is valid         |
| `e2e-contract-validation` | `after` | Verifier | Contract validation details          |
| `e2e-instance-start`      | `after` | Verifier | App started on assigned port         |
| `e2e-readiness`           | `after` | Verifier | App passed ready_check               |
| `e2e-suite-execution`     | `after` | Verifier | Test suite execution result          |
| `e2e-exploratory`         | `after` | Verifier | Exploratory interaction result       |
| `e2e-adversarial`         | `after` | Verifier | Adversarial interaction result       |
| `e2e-instance-shutdown`   | `after` | Verifier | App + tools shut down cleanly        |
| `e2e-test-execution`      | `after` | Verifier | Composite: all E2E phases passed     |

---

## 5. APIs & Interfaces

### 5.1 Orchestrator → Verifier Dispatch Interface

The orchestrator dispatch to verifier is enhanced with E2E context:

```yaml
# Additional dispatch context for E2E-enabled tasks
e2e_context:
  e2e_required: true
  e2e_contract_path: "e2e-contract.yaml"
  workflow_lane: "full-tdd-e2e"
  port_assignment: 9001 # port_range_start + task_ordinal, validated [1024, 65535]
  task_ordinal: 1
  db_isolation_strategy: "per-instance-db" # R2: from contract (D-17)
```

### 5.2 Orchestrator → Planner Dispatch Interface

The orchestrator provides E2E contract discovery result to the planner:

```yaml
# Additional dispatch context for planner
e2e_contract_discovered: true
e2e_contract_path: "e2e-contract.yaml"
```

### 5.3 Evidence Gate Interfaces (R2: Parameterized EG-10)

**EG-10: Lane-Aware Verification (parameterized, replaces old EG-2 — D-16 R2)**

```sql
-- EG-10(unit-only): baseline + TDD
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND check_name IN ('baseline-verified', 'tdd-compliance') AND passed = 1;
-- Expected: 2

-- EG-10(unit-integration): baseline + TDD + integration
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND check_name IN ('baseline-verified', 'tdd-compliance', 'integration-tests-passed') AND passed = 1;
-- Expected: 3

-- EG-10(full-tdd-e2e): baseline + TDD + integration + E2E
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND check_name IN ('baseline-verified', 'tdd-compliance', 'integration-tests-passed', 'e2e-test-execution') AND passed = 1;
-- Expected: 4
```

**EG-8: TDD Compliance (per task, blocking for code tasks)**

```sql
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND check_name = 'tdd-compliance' AND passed = 1;
-- Expected: 1 for code tasks
```

**EG-9: E2E Completion with Sub-Phase Cross-Checking (R2 — D-16)**

```sql
-- Primary gate
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND check_name = 'e2e-test-execution' AND passed = 1;
-- Expected: 1 when e2e_required=true

-- Sub-phase verification (cross-check)
SELECT check_name, passed FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND check_name IN ('e2e-suite-execution', 'e2e-exploratory', 'e2e-adversarial-summary');
-- All must be passed=1 for composite to be valid
```

---

## 6. Sequence / Interaction Notes

### 6.1 E2E Verification Sequence (Verifier Tier 5 — R2: Script-Generation Model)

```
Phase 1 — Setup:
  1. Read E2E contract from e2e_contract_path
  2. Validate contract fields (runtime validation — D-23 Phase 2):
     - start_command.executable matches tier5_command_allowlist (D-22)
     - start_env values pass pattern allowlist (D-21)
     - Port assignment in [1024, 65535] (D-6)
  3. Apply db_isolation_strategy (D-17): create per-instance DB if needed
  4. Start app: run_in_terminal("{executable} {args...}", env={port_env_var: port})
     → command checked against allowlist before execution
  5. Record PID of app process → record e2e-instance-start
  6. Poll ready_check with exponential backoff (max ready_timeout_ms)
  7. Record readiness → record e2e-readiness
  8. Setup browser context (launch via Playwright CLI if browser interaction)
     → ONE browser context per Phase, clean state between phases

Phase 2 — Suite Execution (script-generation model — D-20):
  9. For each test-suite skill:
     a. Generate test script: .e2e/generated/<skill-name>.spec.ts
        (translate skill steps → Playwright calls deterministically)
     b. Execute: run_in_terminal("npx playwright test <script> --reporter=json")
        → must match allowlist pattern "^npx playwright test .+"
     c. Parse JSON reporter output for per-step pass/fail
     d. Capture evidence: screenshots, HAR log, console output
  10. Record per-skill pass/fail → record e2e-suite-execution

Phase 3 — Exploratory Interaction (script-generation model — D-20):
  11. For each exploratory skill:
      a. Generate test script (same as Phase 2)
      b. Execute via runner CLI
      c. For each step result:
         - Evaluate assert condition (machine-checkable)
         - Record step result in interaction_log
         - Sanitize evidence per D-25 (sql_escape, HAR strip, env scrub)
      d. Record overall skill result
  12. Record composite → record e2e-exploratory

Phase 4 — Adversarial Interaction:
  13. For each adversarial skill (and adversarial_variations within exploratory skills):
      a. Generate variation-specific script with step overrides applied
      b. Execute via runner CLI
      c. Verify expected adversarial behavior (attack BLOCKED)
      d. Capture evidence + sanitize per D-25
      e. Record INDIVIDUAL SQL row: check_name='e2e-adversarial-<variation-name>'
         → variation-name validated (alphanumeric + hyphens, max 50 chars)
         → sql_escape() applied before INSERT (D-26)
  14. Record composite summary → record e2e-adversarial-summary

Phase 5 — Teardown:
  15. Shut down app: run_in_terminal(shutdown_command) or kill PID
  16. Close browser contexts (force-kill if unresponsive)
  17. Write evidence manifest (evidence-manifest.yaml)
      → Generate SHA-256 hash of manifest (D-25 authenticity)
  18. Record → record e2e-instance-shutdown
  19. Compute composite: all phases passed → record e2e-test-execution
```

### 6.2 TDD Enforcement Sequence (Implementer)

```
RED Phase:
  1. Read task definition → identify acceptance criteria with test_method='test'
  2. Write test files FIRST (before any production code)
  3. Run tests → confirm exit code ≠ 0 (tests FAIL)
  4. Record: tdd_red_green.tests_written_first = true
  5. Record: tdd_red_green.initial_run_failures = <count>
  6. Record: tdd_red_green.initial_run_exit_code = <exit code>

GREEN Phase:
  7. Write MINIMAL production code to make tests pass (YAGNI)
  8. Run tests → confirm exit code = 0 (tests PASS)

VERIFY Phase:
  9. Run get_errors → record verify_phase.get_errors_clean
  10. Run typecheck → record verify_phase.typecheck_clean
  11. Run full test suite → record verify_phase.tests_passing
  12. If any verify check fails → self-fix loop (max 2 attempts)
  13. Record behavioral_coverage mapping each AC to its test
```

### 6.3 Workflow Lane Routing (Orchestrator — R2: Parameterized EG-10)

```
Step 0: Discover E2E contract → structural validation (D-23 Phase 1):
  - Schema conformance check
  - Required fields present
  - Port range valid [1024, 65535]
  - max_concurrent_instances ≤ 4
  - Referenced skill files exist on disk
  - Command executable in tier5_command_allowlist (D-22)
Step 0.5: If contract discovered → dispatch researcher for skill creation (D-19)
Step 4: Planner assigns workflow_lane per task:
  🟢 files only → unit-only
  At least one 🟡 file → unit-integration
  At least one 🔴 file → full-tdd-e2e (+ e2e_required if contract exists)
Step 6: After standard verification (Tiers 1-4):
  Read task.workflow_lane
  Evaluate parameterized EG-10 (D-16 R2):
    If workflow_lane = 'unit-only': EG-10(unit-only) + EG-8 (TDD)
    If workflow_lane = 'unit-integration': EG-10(unit-integration) + EG-7 + EG-8
    If workflow_lane = 'full-tdd-e2e': EG-10(full-tdd-e2e) + EG-7 + EG-8 + EG-9
  If any gate fails → NEEDS_REVISION → replan (Pattern B, max 3 iterations)
  If Tier 5 passes → skip Step 6a (D-8 explicit skip semantics)
```

### 6.4 Skip Remaining Semantics (R2 — New)

When a skill step specifies `on_failure: skip_remaining`:

- All subsequent steps in the CURRENT skill are skipped
- The skill is recorded as FAILED (not skipped)
- Subsequent skills in the same phase continue execution (independent skills)
- Phase-level pass/fail is computed from all skills in that phase

When Tier 5 passes completely:

- Step 6a (Runtime Smoke Test) is SKIPPED (not executed)
- The skip is recorded in pipeline_telemetry: `check_name='step-6a-skipped'`, `reason='tier-5-passed'`

### 6.5 Per-Interaction-Type Retry Semantics (R2 — New)

| Interaction Type | Retry on Transient Failure? | Retry on Assertion Failure? | Notes                                  |
| ---------------- | --------------------------- | --------------------------- | -------------------------------------- |
| `navigate`       | Yes (network timeout)       | No                          | Retried at task level (D-12), not step |
| `click`          | Yes (element not found)     | No                          | May indicate stale selector            |
| `fill`           | Yes (element not found)     | No                          | May indicate stale selector            |
| `assert`         | No                          | No (deterministic)          | Assertion failure = test failure       |
| `http_request`   | Yes (timeout, 5xx)          | No (4xx is valid result)    | Health check uses this                 |

---

## 7. Security Considerations

### 7.1 Secrets in E2E Artifacts

Per CR-7 and NFR-2, secrets MUST NOT appear in:

- E2E contracts (environment variables referenced by name, not value)
- Skills definitions (no hardcoded credentials)
- Interaction logs (request bodies sanitized)
- Screenshots (sensitive data masked if possible; flagged as risk if not)
- Evidence manifests, SQL output_snippets

The adversarial reviewer's security-sentinel perspective MUST check for secrets exposure in E2E artifacts.

### 7.2 Adversarial Variation Safety

Adversarial skills test for vulnerabilities (XSS, SQL injection, auth bypass) but MUST:

- Only target the local app instance (never external services)
- Use the app's own input mechanisms (no direct database manipulation)
- Assert that attacks are BLOCKED (expected_behavior describes the rejection response)
- Not persist attack payloads beyond the E2E session

### 7.3 Process Isolation

Per FR-5.3 and D-17:

- Each E2E verifier uses an isolated port (deterministic allocation per D-6, validated [1024, 65535])
- Browser instances are separate per verifier (clean context per Phase — D-20 R2)
- Evidence directories are task-scoped (no cross-task file access)
- SQLite continues with WAL mode + busy_timeout for concurrent access
- DB isolation strategy per contract (D-17 R2): per-instance-db | transaction-rollback | shared-with-locking

### 7.4 Verifier Tool Access & Command Allowlist (R2 — D-11, D-22)

The verifier gains expanded `run_in_terminal` usage for E2E, controlled by regex allowlist:

- Expansion is limited to Tier 5 E2E phases only (not Tiers 1-4)
- Every command must match a `tier5_command_allowlist` pattern before execution (D-22)
- The verifier uses the **script-generation model** (D-20) — it generates test scripts from skill steps and executes them via the test runner CLI, never constructing arbitrary shell commands
- The verifier remains read-only for source code (FR-4.4)
- PID tracking ensures process cleanup (FR-5.6)

**Command allowlist patterns (canonical copy in tool-access-matrix.md §5, referenced by e2e-integration.md):**

| Pattern                               | Purpose                            |
| ------------------------------------- | ---------------------------------- |
| `^npx playwright test .+`             | Execute generated E2E test scripts |
| `^npm (run\|start\|test)`             | Application lifecycle management   |
| `^node .+`                            | Direct Node.js script execution    |
| `^dotnet (run\|test)`                 | .NET application lifecycle         |
| `^python -m (pytest\|uvicorn\|flask)` | Python application lifecycle       |
| `^curl -s .+`                         | Health check HTTP requests         |
| `^(kill\|taskkill)`                   | Process cleanup during teardown    |

### 7.5 E2E Contract Command Sanitization (R2 — D-21)

The structured command format eliminates shell injection:

- `executable` is matched against the command allowlist (D-22)
- `args` are passed as array elements (no shell parsing/interpolation)
- `env` values are validated against a pattern allowlist (alphanumeric + common path chars)
- `{port}` placeholder is validated as integer in [1024, 65535] before substitution
- The verifier MUST reject any contract where `executable` contains path separators, spaces, or shell metacharacters

### 7.6 Evidence Sanitization Pipeline (R2 — D-25)

All E2E evidence passes through sanitization before SQL insertion or file storage:

1. **SQL escaping:** All free-text values use `sql_escape()` (replace `'` with `''`) for pipeline_telemetry INSERTs
2. **HAR log stripping:** Authorization, Cookie, Set-Cookie headers → `[REDACTED]`. Bodies > 10KB → `[TRUNCATED]`
3. **Response masking:** Screenshot paths validated (must be within `.e2e/evidence/`, no path traversal). Console output capped at 5KB per step
4. **start_env scrubbing:** Values matching secret patterns (API_KEY, SECRET, TOKEN, PASSWORD, CREDENTIAL) → `[REDACTED]`
5. **Evidence authenticity:** SHA-256 hash of evidence manifest recorded in verification report (adversarial reviewer can verify)

### 7.7 Trust Levels (R2 — D-3)

| Content Category    | Trust Level  | Validation                                                       |
| ------------------- | ------------ | ---------------------------------------------------------------- |
| Contract structure  | Trusted      | Schema validation at Step 0 (D-23)                               |
| Command executables | Semi-Trusted | Must match tier5_command_allowlist regex (D-22)                  |
| Skill content       | Untrusted    | Selectors/values sanitized; confined to generated scripts (D-20) |
| Evidence artifacts  | Untrusted    | Sanitization pipeline (D-25) before storage                      |

---

## 8. Failure & Recovery

### 8.1 E2E Failure Modes

| Failure Mode                         | Detection                                       | Recovery                                                  | Edge Case |
| ------------------------------------ | ----------------------------------------------- | --------------------------------------------------------- | --------- |
| App won't start                      | `e2e-instance-start` passed=0                   | Skip remaining E2E → NEEDS_REVISION                       | EC-1      |
| Readiness timeout                    | `e2e-readiness` passed=0 after ready_timeout_ms | Force-kill PID → NEEDS_REVISION                           | EC-2      |
| Port collision                       | Port already in use                             | Increment port +1 (D-6 fallback)                          | EC-3      |
| Flaky interaction step               | Step fails within timeout_ms                    | Respect on_failure field; retry entire Tier 5 once (D-12) | EC-4      |
| Missing interaction tool             | `e2e-contract-validation` passed=0              | Fall back to test-suite-only or skip E2E                  | EC-5      |
| TDD compliance failure               | `tdd-compliance` passed=0                       | NEEDS_REVISION to implementer                             | EC-6      |
| User declines contract creation      | Pipeline telemetry records decision             | Proceed unit-only for all tasks                           | EC-7      |
| Adversarial variation finds real bug | `e2e-adversarial` passed=0                      | NEEDS_REVISION with specific failure evidence             | EC-8      |
| Orphaned processes                   | PID tracking at teardown                        | Force-kill all tracked PIDs                               | EC-11     |
| Stale skill (selector changed)       | Step fails with element not found               | Log for knowledge agent; on_failure behavior applies      | EC-13     |

### 8.2 Timeout Hierarchy (D-9)

| Phase                   | Timeout                                      | On Expiry                                                    |
| ----------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| App startup             | `ready_timeout_ms` (default 30s, max 60s)    | Kill PID, record `e2e-readiness` passed=0                    |
| Suite execution         | 300s per task                                | Kill test runner, record `e2e-suite-execution` passed=0      |
| Exploratory interaction | 180s per skill                               | Kill browser/tool, skip remaining steps                      |
| Adversarial interaction | 120s per skill                               | Kill browser/tool, skip remaining variations                 |
| App shutdown            | `shutdown_timeout_ms` (default 10s, max 30s) | Force-kill PID                                               |
| **Total E2E per task**  | **600s (hard cap)**                          | **Kill all processes, record `e2e-test-execution` passed=0** |

### 8.3 Retry Policy (D-12)

- **E2E retry:** Max 1 retry per task (total 2 attempts). Retryable: port conflict, startup timeout, browser crash. Non-retryable: contract invalid, tool missing, assertion failure.
- **Replan loop:** If E2E fails after retry, enters Pattern B replan (max 3 iterations) per FR-6.4.
- **Escalation:** If same task fails E2E across 2 consecutive replans, escalate to interactive mode or record as known limitation (FR-13.3).

---

## 9. Non-Functional Requirements Mapping

| NFR                     | Spec ID | Design Coverage                                                                                      |
| ----------------------- | ------- | ---------------------------------------------------------------------------------------------------- |
| E2E Performance Budget  | NFR-1   | D-9: 600s per task, 30min total target. D-8: lanes limit E2E to High Risk only.                      |
| No Secrets in Artifacts | NFR-2   | §7.1: E2E contract uses env var names, not values. Adversarial reviewer checks.                      |
| Reliability             | NFR-3   | D-12: 1 retry for transient failures. D-9: hard timeouts. PID tracking (FR-5.6).                     |
| Maintainability         | NFR-4   | D-4: Skills in version-controlled YAML files. D-15: Additive schema changes.                         |
| Framework-Agnostic      | NFR-5   | D-3: Contract specifies all tool config. D-11: CLI-based interaction via run_in_terminal.            |
| Parallel Isolation      | NFR-6   | D-6: Deterministic port allocation. D-17: Concurrency management. FR-5.3: Task-scoped evidence dirs. |
| Observability           | NFR-7   | D-7: Per-phase SQL evidence. FR-4.8: Interaction logs. FR-4.7: Evidence manifests.                   |
| Determinism             | NFR-8   | D-18: Pre-defined skills, no improvisation. Machine-checkable assertions.                            |

---

## 10. Migration & Backwards Compatibility

### 10.1 Schema Evolution (per D-15, CR-5)

All schema changes are **additive** (minor version bump):

- Task schema (Schema 6): 3 new optional fields with defaults
- Implementation report (Schema 7): 1 new optional section
- Verification report (Schema 8): 1 new optional section
- New check_name values: free TEXT field, no CHECK constraint changes
- New evidence gate queries: additive to sql-templates.md

**No DDL changes required.** The anvil*checks table's `check_name` column accepts any TEXT value. New check_name patterns (e2e-*, tdd-\_) are added by convention, not by schema constraint.

### 10.2 Graceful Degradation

- **No E2E contract:** Pipeline skips E2E enforcement entirely. All tasks treated as unit-only or unit-integration based on risk. Existing behavior preserved.
- **No skills defined:** E2E Tier 5 skips Phases 2-4 (Suite, Exploratory, Adversarial). Only Phases 1 and 5 (Setup/Teardown) execute to validate the contract and app lifecycle.
- **Old task schemas (no workflow_lane field):** Treated as unit-only verification. Backward-compatible.
- **Step 6a (Runtime Smoke Test):** Preserved as fallback for bugfix features without E2E contracts (DR-1).

---

## 11. Testing Strategy

### 11.1 What the Planner Should Decompose

Implementation should proceed in phases:

1. **Phase 1 — TDD Strengthening:** Modify `implementer.agent.md` (RED-GREEN-VERIFY), `verifier.agent.md` (TDD compliance check), `schemas.md` (verify_phase fields), `sql-templates.md` (EG-8).
2. **Phase 2 — E2E Contract & Infrastructure:** Create `e2e-integration.md`, modify `schemas.md` (E2E contract + skills schemas), modify `dispatch-patterns.md`, modify `global-operating-rules.md`.
3. **Phase 3 — E2E Verification:** Modify `verifier.agent.md` (Tier 5), modify `orchestrator.agent.md` (lane enforcement + gates), modify `planner.agent.md` (lane fields), modify `tool-access-matrix.md`.
4. **Phase 4 — Review & Polish:** Modify `adversarial-reviewer.agent.md`, `evaluation-schema.md`, `review-perspectives.md`, `severity-taxonomy.md`.

### 11.2 Verification Approach

Each phase is verified through the existing pipeline verification cascade:

- **Tiers 1-2:** Schema validation, syntax checking, document structure verification
- **Tier 3:** Cross-reference checks (every FR has implementation path, every AC has verification method)
- **Adversarial review:** TDD compliance review dimension, E2E quality checks

---

## 12. Tradeoffs & Alternatives Considered

### 12.1 Decision Summary Table (R2: 26 decisions)

| ID   | Title                                                | Risk | Confidence | Selected Alternative                                                  |
| ---- | ---------------------------------------------------- | ---- | ---------- | --------------------------------------------------------------------- |
| D-1  | Architecture Selection                               | 🟡   | High       | Direction C — Hybrid Extended Verifier + Reference Document           |
| D-2  | E2E Coordination Extraction (R2: reading guides)     | 🟢   | High       | New `e2e-integration.md` (~500 lines, per-agent reading guides)       |
| D-3  | E2E Contract Location (R2: trust levels)             | 🟢   | High       | Project-level discoverable file with trust level classification       |
| D-4  | E2E Skills Storage                                   | 🟢   | Medium     | Referenced skill directory alongside contract                         |
| D-5  | Verifier Tier 5 Architecture (R2: ordering)          | 🔴   | High       | 5-phase lifecycle with PID persistence, per-phase browser context     |
| D-6  | Port Allocation (R2: range validation, DB isolation) | 🟡   | High       | Deterministic offset, validated [1024, 65535], db_isolation_strategy  |
| D-7  | SQL Evidence (R2: per-variation, index)              | 🟡   | High       | Per-variation adversarial records with composite index                |
| D-8  | Workflow Lane Mapping (R2: explicit tier gating)     | 🟡   | High       | Lane-to-tier table, explicit skip semantics                           |
| D-9  | E2E Timeout Architecture (R2: arithmetic note)       | 🟡   | Medium     | 600s total cap with per-phase limits, realistic budget example (460s) |
| D-10 | TDD Fallback Tightening                              | 🟡   | High       | Strict criteria with SQL evidence                                     |
| D-11 | Verifier Tool Access (R2: command allowlist)         | 🔴   | Medium     | Script-generation model + regex command allowlist + audit trail       |
| D-12 | E2E Retry Policy                                     | 🟡   | Medium     | Max 1 retry per task                                                  |
| D-13 | TDD Compliance (R2: secondary heuristics)            | 🟡   | Medium     | Structural analysis + secondary plausibility heuristics               |
| D-14 | Context Governance Enforcement                       | 🟢   | High       | Mandatory 200-line read_file limit                                    |
| D-15 | Task Schema Extension                                | 🟢   | High       | Additive fields to existing Schema 6                                  |
| D-16 | Evidence Gates (R2: EG-2/EG-10 consolidation)        | 🟡   | High       | Parameterized EG-10 (3 lane variants), EG-9 sub-phases, net +15 lines |
| D-17 | E2E Concurrency (R2: DB isolation)                   | 🟡   | High       | Sub-wave enforcement + db_isolation_strategy propagation              |
| D-18 | Interaction Determinism                              | 🟢   | High       | Pre-defined skills, no agent improvisation                            |
| D-19 | Skill Creation Workflow **(NEW R2)**                 | 🟡   | Medium     | Researcher-mediated at Step 0.5                                       |
| D-20 | Browser Execution Model **(NEW R2, CRITICAL)**       | 🔴   | High       | Script generation from skill steps, executed via test runner CLI      |
| D-21 | Command Sanitization **(NEW R2, CRITICAL)**          | 🔴   | High       | Structured {executable, args[]} format, no shell interpolation        |
| D-22 | Command Allowlist **(NEW R2, CRITICAL)**             | 🔴   | High       | Regex patterns in tool-access-matrix.md §5                            |
| D-23 | Contract Validation **(NEW R2)**                     | 🟡   | High       | Structural at Step 0, runtime at Tier 5                               |
| D-24 | Verifier Modularity **(NEW R2)**                     | 🟡   | Medium     | Sub-roles with conditional Tier 5 loading                             |
| D-25 | Evidence Sanitization **(NEW R2)**                   | 🔴   | High       | sql_escape, HAR stripping, env scrubbing, SHA-256 manifest hash       |
| D-26 | Per-Variation SQL Records **(NEW R2)**               | 🟡   | High       | Individual + composite adversarial records with index                 |

### 12.2 Low/Medium Confidence Decisions — Revisit Guidance

| Decision                       | Confidence | What would raise confidence                                                | When to revisit                                  |
| ------------------------------ | ---------- | -------------------------------------------------------------------------- | ------------------------------------------------ |
| D-4: Skills Storage            | Medium     | Real-world usage showing whether inline or directory is preferred          | After 3 pipeline runs with E2E skills            |
| D-9: E2E Timeout Architecture  | Medium     | Actual E2E execution timing data from diverse projects                     | After first production E2E pipeline run          |
| D-11: Verifier Tool Access     | Medium     | Security review of actual commands executed during E2E                     | After adversarial review of implemented Tier 5   |
| D-12: E2E Retry Policy         | Medium     | Flakiness rate data from real E2E executions                               | After 5 pipeline runs with E2E verification      |
| D-13: TDD Compliance           | Medium     | False positive/negative rate of structural analysis + secondary heuristics | After 10 code tasks verified with TDD compliance |
| D-19: Skill Creation (R2)      | Medium     | Researcher's ability to generate valid skills from contract analysis       | After first pipeline run with skill creation     |
| D-24: Verifier Modularity (R2) | Medium     | Whether conditional loading actually saves context in practice             | After first Tier 5 implementation                |

---

## 13. Implementation Checklist & Deliverables

### Spec Requirement → Design Traceability (R2 Updated)

| Spec Section                  | Requirements    | Design Coverage                                                                                    |
| ----------------------------- | --------------- | -------------------------------------------------------------------------------------------------- |
| FR-1 (Mandatory TDD)          | FR-1.1–FR-1.7   | D-10 (fallback), D-13 (verification + R2 heuristics), D-16 (gates). Implementer TDD cycle.         |
| FR-2 (E2E Contract)           | FR-2.1–FR-2.5   | D-3 (location + trust levels), D-2 (reference doc), D-21 (structured commands), D-23 (validation). |
| FR-3 (E2E Skills)             | FR-3.1–FR-3.9   | D-4 (storage), D-18 (determinism), D-19 (skill creation workflow).                                 |
| FR-4 (Live App Testing)       | FR-4.1–FR-4.8   | D-5 (Tier 5), D-11 (tool access), D-20 (script-generation), D-22 (allowlist). 5-phase lifecycle.   |
| FR-5 (Parallel Safety)        | FR-5.1–FR-5.6   | D-6 (ports + range validation), D-17 (concurrency + DB isolation), D-9 (timeouts). PID tracking.   |
| FR-6 (Workflow Lanes)         | FR-6.1–FR-6.5   | D-8 (lane mapping + tier gating table), D-15 (task schema), D-16 (parameterized EG-10).            |
| FR-7 (Context Governance)     | FR-7.1–FR-7.4   | D-14 (mandatory 200-line rule). Evidence summarization rules. D-2 (per-agent reading guide).       |
| FR-8 (Interactive Mode)       | FR-8.1–FR-8.4   | D-19 (skill creation via researcher). E2E contract creation guidance in interactive mode.          |
| FR-9 (Orchestrator Rules)     | FR-9.1–FR-9.4   | D-2 (extraction), D-16 (consolidation, 543/550 budget), D-17 (concurrency).                        |
| FR-10 (Copilot Practices)     | FR-10.1–FR-10.3 | Adversarial reviewer YAGNI/KISS/DRY checks. Knowledge agent instruction updates.                   |
| FR-11 (Test Replay)           | FR-11.1–FR-11.3 | Evidence manifest (FR-4.7), D-25 (SHA-256 hash). Artifact paths. E2E metrics in post-mortem.       |
| FR-12 (Completion Gate)       | FR-12.1–FR-12.2 | D-16 (parameterized gates). EG-7 blocking. EG-8 blocking. EG-9 blocking with sub-phases.           |
| FR-13 (Performance)           | FR-13.1–FR-13.3 | D-9 (timeouts), D-12 (retry), FR-13.3 escalation.                                                  |
| FR-14 (Deterministic Outputs) | FR-14.1–FR-14.2 | D-7 (SQL evidence + D-26 per-variation), D-18 (determinism). Dual recording (SQL + YAML).          |

### Common Requirements Coverage

| CR                                      | Coverage                                                                                  |
| --------------------------------------- | ----------------------------------------------------------------------------------------- |
| CR-1: schema_version                    | All schemas include `schema_version: "1.0"` — unchanged                                   |
| CR-2: Testing is proof                  | D-16: EG-7/EG-8/EG-9/EG-10 parameterized gates enforce testing as completion criterion    |
| CR-3: No silent failures                | D-7: All E2E outcomes as SQL INSERTs. D-26: Per-variation records. Per-phase check_names. |
| CR-4: Framework-agnostic                | D-3/D-11: Contract specifies tools. D-22: Allowlist covers multiple frameworks.           |
| CR-5: Backward-compatible               | D-15: Additive schema changes. Graceful degradation without E2E contract.                 |
| CR-6: Bounded loops                     | D-12: Max 1 E2E retry. Existing Pattern B max 3 iterations.                               |
| CR-7: No secrets                        | §7.1 + §7.6: Evidence sanitization pipeline (D-25). Env scrubbing.                        |
| CR-8: Version-controlled YAML           | D-3/D-4: Contract and skills in project repository.                                       |
| CR-9: Suite + Exploratory + Adversarial | D-5: Tier 5 phases 2-4. D-20: Script-generation per phase. D-18: Pre-defined skills.      |

### Deliverables

1. **`e2e-integration.md`** — New reference document (~500 lines, per-agent reading guides)
2. **Modified agent definitions** — verifier, implementer, orchestrator, planner, researcher, adversarial-reviewer (6 files)
3. **Modified reference documents** — schemas.md, sql-templates.md, tool-access-matrix.md, global-operating-rules.md, dispatch-patterns.md, evaluation-schema.md, review-perspectives.md (7 files)
4. **Total: 14 files** (1 created + 13 modified)

---

## Appendix A: Deviation Records

### DR-1: Step 6a Preservation

- **Spec requirement:** FR-4.6 — "The existing Runtime Smoke Test (Step 6a) becomes E2E verification driven by the E2E contract"
- **Deviation:** Step 6a is preserved as-is for projects without E2E contracts. E2E Tier 5 replaces Step 6a only when a contract exists AND the task is in the full-tdd-e2e lane, which is a superset of the spec's behavior.
- **Rationale:** Graceful degradation (CR-5) requires existing bugfix workflows to work without an E2E contract.

### DR-2: TDD Compliance Verification Method

- **Spec requirement:** FR-1.4 — "cross-check against git diff showing test files committed before or alongside production files"
- **Deviation:** Design uses structural analysis (test imports + red phase evidence) instead of git diff temporal ordering.
- **Rationale:** The implementer commits all changes in a single `git add -A`, so commit timestamps don't distinguish test-first ordering. Structural analysis provides equivalent confidence. See D-13.

### DR-3: Contract Validation Split (R2 — New)

- **Spec requirement:** FR-4.1 — "orchestrator discovers and validates the E2E contract"
- **Deviation:** Contract validation is split into two phases: structural at Step 0 (orchestrator) and runtime at Tier 5 Phase 1 (verifier).
- **Rationale:** Some contract properties (schema conformance, file existence) are statically checkable and should fail fast. Other properties (commands work, health check responds) require runtime execution. See D-23.

---

## Appendix B: Self-Verification Checklist

- [x] Every functional requirement (FR-1 through FR-14) has a clear implementation path
- [x] Every acceptance criterion (AC-1 through AC-38) is mapped to a design element or marked N/A with justification
- [x] All 26 design decisions use the decision-record format with justification scoring
- [x] All decisions have at least 2 alternatives
- [x] Confidence levels assigned with concrete definitions (see §12.2 for Low/Medium guidance)
- [x] Risk classifications consistent (🟢/🟡/🔴 per per-file criteria)
- [x] Security considerations addressed (§7, including R2 additions §7.4-§7.7)
- [x] Failure modes identified with recovery strategies (§8)
- [x] `design-output.yaml` conforms to Schema 4
- [x] All sections of `design.md` present
- [x] No code, tests, or plan content written — design artifacts only
- [x] 3 deviation records documented with rationale (DR-1, DR-2, DR-3)
- [x] All 3 Critical findings addressed (D-20, D-21, D-22)
- [x] All 18 Major findings addressed
- [x] All 12 Minor findings addressed

---

## Appendix C: Revision History (R2)

### Findings Addressed

**Critical (3/3 resolved):**

- S-1 (security-sentinel): Command injection via start_command → D-21 structured command + D-22 allowlist
- S-5 (security-sentinel): Tool access boundary undefined → D-22 regex allowlist + D-11 updated
- A-3 (architecture-guardian): Browser execution model undefined → D-20 script-generation model

**Major (18/18 resolved):**

- S-2 (security-sentinel): SQL injection via evidence → D-25 sanitization pipeline
- S-3 (security-sentinel): Secrets in start_env → D-25 env scrubbing
- S-4 (security-sentinel): Evidence artifacts uncontrolled → D-25 size caps + path validation
- A-1 (security-sentinel): Contract trust boundary → D-3 trust levels + D-23 validation
- A-2 (security-sentinel): Verifier as process manager → D-5 PID persistence + lifecycle
- C-1 (security-sentinel): Hollow test detection → D-13 secondary heuristics
- C-2 (security-sentinel): Composite gate SPOF → D-16 parameterized EG-10
- S-1 (architecture-guardian): Trust boundary crossing → D-3 + D-23
- A-1 (architecture-guardian): Verifier SRP → D-24 sub-role modularity
- A-2 (architecture-guardian): Orchestrator budget → D-16 consolidation (543/550)
- A-4 (architecture-guardian): Context pressure → D-2 per-agent reading guides
- C-1 (architecture-guardian): Skill creation undefined → D-19 researcher workflow
- C-2 (architecture-guardian): Evidence gate budget → D-16 explicit analysis
- S-1 (pragmatic-verifier): RED phase self-attested → D-13 heuristics
- A-1 (pragmatic-verifier): Browser lifecycle → D-5 + D-20
- A-2 (pragmatic-verifier): Orchestrator budget → D-16
- C-1 (pragmatic-verifier): Contract validation timing → D-23 + DR-3
- C-2 (pragmatic-verifier): Per-variation SQL → D-26
- C-3 (pragmatic-verifier): Lane-to-tier mapping → D-8 gating table
- C-4 (pragmatic-verifier): DB isolation → D-6 + D-17

**Minor (12/12 addressed):**

- S-6 (security-sentinel): Port range → D-6 [1024, 65535]
- S-7 (security-sentinel): External URLs → D-20 script-generation restricts navigation
- S-2 (architecture-guardian): Command validation → D-22 regex allowlist
- A-5 (architecture-guardian): e2e-integration scope → D-2 reading guides
- A-6 (architecture-guardian): SQL growth → D-7 + D-26 with index
- C-3 (architecture-guardian): Dual paths → D-8 explicit table
- S-2 (pragmatic-verifier): Evidence authenticity → D-25 SHA-256 hash
- S-3 (pragmatic-verifier): Secrets in capture → D-25 env scrubbing
- C-5 (pragmatic-verifier): skip_remaining → D-5 + §6.4
- C-6 (pragmatic-verifier): Timeout arithmetic → D-9 budget example
- C-7 (pragmatic-verifier): EG-10 variants → D-16 three variants
- C-8 (pragmatic-verifier): Step retry → D-5 + §6.5
