# Dependencies Research — TDD & E2E Enforcement

**Researcher Instance:** `researcher-dependencies`  
**Focus Area:** Dependencies  
**Run ID:** `2026-03-04T00:00:00Z`  
**Feature:** `tdd-e2e-enforcement`

---

## Executive Summary

The TDD & E2E enforcement feature introduces cross-cutting dependency changes across the entire 9-agent pipeline. This research maps 18 findings across 5 dependency categories: internal agent dependencies, schema dependencies, SQL/DB dependencies, external tool dependencies, and E2E contract schema design.

**Key takeaway:** The existing pipeline architecture is well-suited for E2E enforcement. The Verifier agent is the natural E2E executor (already has `run_in_terminal` access and per-task dispatch model). No new agent types are needed. The main work is: (1) a new E2E contract schema, (2) extensions to 3 existing schemas, (3) new SQL evidence gate queries, and (4) a framework-agnostic runner abstraction.

---

## 1. Internal Agent Dependencies

### 1.1 Producer/Consumer Dependency Chain (F-1)

The current pipeline has 12 typed YAML schemas flowing between 9 agents. The dependency chain for TDD/E2E enforcement:

```
Planner → task-schema (risk + AC) → Implementer → implementation-report (TDD evidence)
                                                         ↓
                                                    Verifier → verification-report (E2E evidence)
                                                         ↓
                                               anvil_checks SQL → Orchestrator (gate queries)
```

**New data flows needed for E2E:**
| Flow | From | To | Data |
|------|------|----|------|
| E2E contract | Project file | Verifier | App lifecycle config |
| E2E evidence | Verifier | anvil_checks (SQL) | Test results, lifecycle events |
| E2E gate queries | Orchestrator | verification-ledger.db | Risk-based E2E enforcement |
| Workflow lane | Planner | Orchestrator + Verifier | Risk classification → test requirements |

**Source:** `schemas.md` Producer/Consumer Dependency Table (12 entries)

### 1.2 Completion Contract Routing (F-2)

The completion contract (Schema 1) drives all orchestrator routing. Key status transitions for E2E:

- **Verifier** → `NEEDS_REVISION` when E2E fails → triggers replan loop (Pattern B, max 3 iterations)
- **evidence_summary** must include E2E check counts alongside unit/integration checks
- No new completion statuses needed — E2E failures use existing `NEEDS_REVISION` flow

### 1.3 Dispatch Pattern Constraints (F-3)

- **Max 4 concurrent agents per wave** (dispatch-patterns.md)
- Each verifier running E2E needs an isolated app instance (unique port)
- Sub-wave partitioning already handles >4 tasks
- E2E adds resource contention for: ports, databases, file system, environment variables

---

## 2. Schema Dependencies

### 2.1 Task Schema Extensions (F-4)

**Current Schema 6 fields:** `id, title, description, agent, size, risk, depends_on, acceptance_criteria, relevant_context`

**Risk → Workflow Lane mapping needed:**

| Risk | Workflow Lane      | Testing Required            |
| ---- | ------------------ | --------------------------- |
| 🟢   | `unit-only`        | Unit tests                  |
| 🟡   | `unit-integration` | Unit + integration tests    |
| 🔴   | `full-tdd-e2e`     | Full TDD + E2E verification |

**Design decision needed:** Add explicit `workflow_lane` field vs. derive from `risk` + `size`.

**test_method enum:** Currently `inspection | demonstration | test | analysis`. May need `e2e` value for acceptance criteria requiring live app testing.

### 2.2 Implementation Report Extensions (F-5)

The implementer does NOT run E2E tests (per initial request §3). However, the implementer may need to:

1. Validate that an E2E contract exists for the project
2. Write E2E test files (without executing them)
3. Report `behavioral_coverage` with a new status like `pending_e2e` for ACs requiring E2E

**Current `behavioral_coverage.status` values:** `covered | not_applicable`  
**Proposed addition:** `pending_e2e` — AC will be verified by verifier's E2E execution

### 2.3 Verification Report Extensions (F-6)

The verifier's 4-tier cascade needs a new E2E tier:

| Tier  | Current                                             | E2E Extension                                       |
| ----- | --------------------------------------------------- | --------------------------------------------------- |
| 1     | IDE diagnostics, syntax                             | (unchanged)                                         |
| 2     | Build, type-check, lint, tests, behavioral-coverage | (unchanged)                                         |
| 3     | Import/load, smoke execution                        | (unchanged)                                         |
| 4     | Operational readiness (Large only)                  | (unchanged)                                         |
| **5** | **(new)**                                           | **E2E lifecycle + test execution (High Risk only)** |

**New check_names for E2E:**

- `e2e-app-start` — Application started successfully
- `e2e-readiness` — Application ready check passed
- `e2e-test-execution` — E2E test suite completed
- `e2e-test-results` — Individual E2E test results
- `e2e-app-shutdown` — Application shut down cleanly
- `e2e-evidence-captured` — Screenshots/traces/logs saved

---

## 3. SQL/DB Dependencies

### 3.1 anvil_checks Table (F-7)

**Current schema:**

```sql
phase CHECK (phase IN ('baseline','after','review'))
verdict CHECK (verdict IN ('approve','needs_revision','blocker'))
severity CHECK (severity IN ('Blocker','Critical','Major','Minor'))
```

**Recommendation:** Use existing `phase='after'` with new `check_name` patterns (e2e-\*). This avoids DDL changes (table recreation) and fits existing evidence gate queries.

**Alternative considered:** Adding `phase='e2e'` requires table recreation per sql-templates.md §9. More disruptive but cleaner separation.

### 3.2 New E2E Lifecycle Table (F-8)

**Proposed new table: `e2e_runs`**

```sql
CREATE TABLE IF NOT EXISTS e2e_runs (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id          TEXT    NOT NULL,
  task_id         TEXT    NOT NULL,
  app_type        TEXT    NOT NULL CHECK (app_type IN ('web','api','cli','desktop','library')),
  port            INTEGER,
  pid             INTEGER,
  started_at      TEXT,
  ready_at        TEXT,
  tests_started_at TEXT,
  tests_completed_at TEXT,
  shutdown_at     TEXT,
  status          TEXT    CHECK (status IN ('started','ready','testing','completed','failed','timeout')),
  e2e_runner      TEXT,
  e2e_command     TEXT,
  tests_total     INTEGER,
  tests_passed    INTEGER,
  tests_failed    INTEGER,
  evidence_dir    TEXT,
  ts              TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_e2e_run ON e2e_runs(run_id);
CREATE INDEX IF NOT EXISTS idx_e2e_task ON e2e_runs(run_id, task_id);
```

**Rationale:** Lifecycle events are structured differently from verification checks. A separate table enables richer E2E-specific queries without overloading anvil_checks.

### 3.3 New Evidence Gate Queries (F-9)

**Proposed new gates:**

```sql
-- EG-8: E2E evidence exists for High Risk tasks
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND check_name LIKE 'e2e-%' AND phase = 'after';
-- Expected: ≥1 for High Risk tasks

-- EG-9: E2E lifecycle complete
SELECT COUNT(*) FROM e2e_runs
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND status = 'completed';
-- Expected: 1

-- EG-10: E2E pass rate
SELECT tests_passed, tests_failed FROM e2e_runs
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND status = 'completed';
-- Expected: tests_failed = 0
```

**EG-7 promotion:** The existing behavioral coverage gate (currently INFORMATIONAL) should be promoted to blocking for code tasks. Promotion criteria from sql-templates.md: "≥90% of code tasks naturally produce behavioral-coverage records with zero false-positive overrides."

---

## 4. External Tool Dependencies

### 4.1 Playwright CLI + Skills (F-10)

**Recommended default E2E tool for web applications.**

| Package           | Purpose                                            | Version | Install                                 |
| ----------------- | -------------------------------------------------- | ------- | --------------------------------------- |
| `@playwright/cli` | CLI-based browser automation with skills           | v0.1.1  | `npm install -g @playwright/cli@latest` |
| `@playwright/mcp` | MCP server for exploratory/long-running automation | v0.0.68 | `npx @playwright/mcp@latest`            |

**Why CLI over MCP for this system:**

- Token-efficient (avoids loading large tool schemas into agent context)
- Skills-based pattern matches the initial request's "reusable skill" concept
- Session management (`-s=name`) provides instance isolation
- Monitoring dashboard (`playwright-cli show`) enables observing agents
- Explicitly designed for GitHub Copilot and coding agents
- Headless by default (suitable for CI and agent-driven testing)

**Key CLI commands for E2E:**

```bash
playwright-cli install --skills          # Install reusable skills
playwright-cli open [url] --headless     # Open browser session
playwright-cli -s=task-01 open [url]     # Named session for isolation
playwright-cli screenshot                # Capture evidence
playwright-cli tracing-start/stop        # Capture trace
playwright-cli video-start/stop          # Capture video
playwright-cli close                     # Close browser
playwright-cli list                      # List active sessions
playwright-cli close-all                 # Clean up all sessions
```

### 4.2 Framework-Agnostic Runner Abstraction (F-11)

The system must support multiple E2E runners based on the project's E2E contract:

| Category | Runners                                       | When Used                             |
| -------- | --------------------------------------------- | ------------------------------------- |
| Web      | Playwright CLI, Playwright, Cypress, Selenium | Web applications                      |
| API      | curl, httpie, Newman, REST Client             | API-only applications                 |
| Desktop  | Appium, WinAppDriver                          | Desktop applications                  |
| CLI      | Custom scripts, expect                        | CLI tools                             |
| Library  | N/A                                           | No E2E needed (unit/integration only) |

**Abstraction layer design:** The Verifier reads the E2E contract to determine:

1. Which runner to use (`e2e_runner` field)
2. How to start/stop the app (`start_command`, `shutdown_command`)
3. How to verify readiness (`ready_check`)
4. How to run tests (`e2e_command`)
5. How to collect evidence (`evidence_output_dir`, screenshot/trace flags)

### 4.3 Port Allocation & Instance Isolation (F-12)

**Strategy for parallel E2E instances:**

```
Port = base_port + task_ordinal
Example: base_port=5000, tasks=[task-01, task-02, task-03, task-04]
  task-01 → port 5001
  task-02 → port 5002
  task-03 → port 5003
  task-04 → port 5004
```

**Isolation requirements per instance:**

- Unique port (via `port_env_var` from E2E contract)
- Unique Playwright session name (`-s=task-{id}`)
- Unique temp directory for evidence
- Process PID tracking for cleanup
- Timeout enforcement (kill process if stalled)

---

## 5. E2E Contract Schema Design (F-13)

### Proposed Schema: `e2e-contract`

```yaml
# e2e-contract.yaml — lives at project root or .e2e/contract.yaml
schema_version: "1.0"

# App Lifecycle
app_type: "web" # web | api | cli | desktop | library
start_command: "dotnet run" # Command to start the application
start_env: # Environment variables for app start
  ASPNETCORE_ENVIRONMENT: "Development"
  ASPNETCORE_URLS: "http://localhost:${PORT}"
ready_check: "http://localhost:${PORT}/health" # URL or command for readiness
ready_timeout_ms: 30000 # Max wait for readiness (default 30000)
base_url: "http://localhost:${PORT}" # Base URL for the running app
shutdown_command: null # Graceful shutdown (default: SIGTERM)
shutdown_timeout_ms: 10000 # Max wait for shutdown (default 10000)

# Port Management
default_port: 5000 # Default port
port_env_var: "PORT" # Env var name for port override
port_range_start: 5000 # Base for parallel instances

# E2E Runner
e2e_runner: "playwright-cli" # playwright-cli | playwright | cypress | custom
e2e_command: "npx playwright test" # Command to execute E2E tests
e2e_config_path: null # Path to runner config (optional)
e2e_env: # Additional env vars for test runner
  BASE_URL: "http://localhost:${PORT}"

# Parallelism
supports_parallel: false # Can multiple instances run simultaneously?
requires_isolated_instance: true # Does each test need its own app instance?
max_concurrent_instances: 4 # Max parallel instances (≤ concurrency cap)

# Evidence Collection
screenshot_on_failure: true # Capture screenshots on test failure
trace_on_failure: true # Capture Playwright traces on failure
video_recording: false # Record video of test session
evidence_output_dir: ".e2e-evidence/" # Output directory for evidence artifacts

# Skills (interactive mode)
skills: [] # List of reusable E2E skill references
skill_path: null # Path to skills directory
```

### Contract Discovery

The Verifier discovers the E2E contract via:

1. Check `e2e_contract_path` in the task definition (if specified)
2. Look for `e2e-contract.yaml` at project root
3. Look for `.e2e/contract.yaml`
4. If no contract found and task is High Risk → log warning, skip E2E (not a blocking error)
5. In interactive mode → prompt user to create contract

---

## 6. TDD Enforcement Dependencies (F-15)

### Current TDD Infrastructure (Existing)

| Component                   | Location                         | Current Status                       |
| --------------------------- | -------------------------------- | ------------------------------------ |
| `tdd_red_green` field       | Schema 7 (implementation-report) | Conditional for code tasks           |
| `behavioral_coverage` field | Schema 7 (implementation-report) | Conditional for code tasks           |
| Behavioral coverage check   | Verifier Tier 2 (§3e)            | Always-run for code tasks            |
| EG-7 gate                   | sql-templates.md §6              | INFORMATIONAL (not blocking)         |
| TDD Fallback                | implementer.agent.md             | Allows skipping TDD in several cases |

### Gaps to Close for Mandatory TDD

1. **EG-7 promotion:** Make behavioral coverage blocking (not just informational)
2. **TDD verification:** Verifier should cross-check `tdd_red_green.tests_written_first` against git history
3. **Tighter fallback rules:** Reduce cases where TDD can be skipped
4. **Red phase verification:** Verify `initial_run_failures > 0` (tests actually failed before code)

---

## 7. Cross-Cutting Dependencies Summary

### Files That Need Modification

| File                        | Changes Needed                                                     | Risk |
| --------------------------- | ------------------------------------------------------------------ | ---- |
| `schemas.md`                | New Schema 13 (e2e-contract), extend Schemas 6, 7, 8               | 🔴   |
| `sql-templates.md`          | New e2e_runs table DDL, new EG-8/9/10 queries, EG-7 promotion      | 🟡   |
| `orchestrator.agent.md`     | Risk-based E2E routing, new evidence gates, E2E approval gate      | 🟡   |
| `verifier.agent.md`         | E2E Tier 5 cascade, E2E lifecycle management, contract consumption | 🔴   |
| `implementer.agent.md`      | Mandatory TDD enforcement, E2E readiness reporting                 | 🟡   |
| `planner.agent.md`          | Workflow lane assignment in task-schema                            | 🟢   |
| `dispatch-patterns.md`      | E2E resource isolation requirements, port allocation               | 🟢   |
| `global-operating-rules.md` | TDD mandatory rules, E2E timeout policies                          | 🟢   |
| `tool-access-matrix.md`     | No changes needed (Verifier already has required access)           | 🟢   |
| `evaluation-schema.md`      | No changes needed                                                  | 🟢   |

### Dependency Graph (Simplified)

```
E2E Contract (new Schema 13)
    ├── consumed by: Verifier (E2E execution)
    ├── referenced by: task-schema (Schema 6) via e2e_contract_path
    └── discovered by: Verifier (file search) or Orchestrator (interactive setup)

Risk-Based Workflow Lane
    ├── defined by: Planner (task-schema risk field)
    ├── consumed by: Orchestrator (gate routing)
    └── consumed by: Verifier (determines if E2E required)

E2E Evidence
    ├── produced by: Verifier (anvil_checks INSERT + e2e_runs INSERT)
    ├── consumed by: Orchestrator (EG-8/9/10 queries)
    └── consumed by: Knowledge Agent (post-mortem analysis)

TDD Enforcement
    ├── enforced by: Implementer (tdd_red_green recording)
    ├── verified by: Verifier (behavioral-coverage check)
    └── gated by: Orchestrator (EG-7 promoted to blocking)
```

---

## 8. Online Research Findings

### Playwright CLI + Skills (microsoft/playwright-cli)

- **v0.1.1** (2026, Apache-2.0, 4.7k stars)
- CLI-based browser automation designed for coding agents
- Skills pattern: install reusable skills, reference from agent workflows
- Session management: named sessions for parallel isolation
- Monitoring dashboard: observe all agent browser sessions
- Recommended over MCP for high-throughput coding agents

### Playwright MCP (@playwright/mcp)

- **v0.0.68** (1.5M+ weekly downloads)
- MCP server for persistent browser context and exploratory automation
- Better for self-healing tests and long-running autonomous workflows
- More token-expensive than CLI approach
- Supports VS Code, Cursor, Claude Desktop, and other MCP clients

### Key Architectural Pattern

The Playwright team explicitly recommends CLI + Skills over MCP for coding agents:

> "CLI + SKILLs better suited for high-throughput coding agents that must balance browser automation with large codebases, tests, and reasoning within limited context windows."

This aligns with the agent system's context governance principles (initial request §6: "Subagents must operate within bounded context windows").

---

_Research completed: 2026-03-04T00:45:00Z_
