# E2E Coordination Reference — v1.0

> **Type:** Shared reference document (not an agent definition)
> **Purpose:** Central reference for E2E testing coordination across the agent pipeline. Defines the E2E contract specification, skills schema, Playwright CLI interaction protocol, command sanitization, command allowlist, evidence sanitization, and timeout budgets.
> **Source:** Derived from design decisions D-2, D-3, D-4, D-6, D-9, D-17, D-18, D-20, D-21, D-22, D-25.
> **Spec Requirements:** FR-2, FR-3, FR-4, FR-5, FR-13.

---

## §0 Per-Agent Reading Guide

Agents MUST read only the sections relevant to their role. This enables conditional loading and reduces context pressure.

| Agent                    | Required Sections | Purpose                                         |
| ------------------------ | ----------------- | ----------------------------------------------- |
| **Orchestrator**         | §1, §6            | Contract discovery, timeout enforcement         |
| **Verifier**             | §2, §3, §4, §5    | Interaction protocol, sanitization, allowlist   |
| **Planner**              | §1 only           | Contract fields for lane derivation             |
| **Adversarial Reviewer** | §4, §5            | Sanitization audit, allowlist compliance review |

---

## §1 E2E Contract Specification

The E2E contract is a project-level YAML file defining how to start the application, verify readiness, configure interaction tools, and discover skills. Per D-3, the contract resides at a discoverable location:

- **Primary:** `e2e-contract.yaml` at project root
- **Alternative:** `.e2e/contract.yaml`

### Trust Levels (D-3)

| Content Category    | Trust Level  | Validation                                             |
| ------------------- | ------------ | ------------------------------------------------------ |
| Contract structure  | Trusted      | Schema validation at Step 0 (D-23)                     |
| Command executables | Semi-Trusted | Must match `tier5_command_allowlist` regex (D-22)      |
| Skill content       | Untrusted    | Selectors/values sanitized; confined to scripts (D-20) |
| Evidence artifacts  | Untrusted    | Sanitization pipeline (D-25) before storage            |

Project-root contracts are **project-trusted** (same trust as source code). Contracts generated in interactive mode are **pipeline-generated** and require user approval via `ask_questions` before commands execute.

### Contract Validation (D-23)

Validation is split into two phases:

- **Phase 1 — Structural (Step 0, orchestrator):** Schema conformance, required fields present, port range valid `[1024, 65535]`, `max_concurrent_instances ≤ 4`, referenced skill files exist on disk, command executable in allowlist (D-22).
- **Phase 2 — Runtime (Tier 5, verifier):** `start_command` actually launches the app, `ready_check` returns expected status, app responds within `ready_timeout_ms`, interaction tool installed and functional.

### Structured Command Format (D-21)

All lifecycle commands use a structured format — **NOT raw shell strings**:

```yaml
start_command:
  executable: "npm" # Must match tier5_command_allowlist (D-22)
  args: ["run", "dev", "--", "--port", "{port}"]
  env:
    NODE_ENV: "test" # Values validated against pattern allowlist
```

**Executable validation rules:**

- No path separators (`/`, `\`)
- No spaces
- No shell metacharacters (`;`, `|`, `&`, `` ` ``, `$`)
- Must match one of the `tier5_command_allowlist` patterns (§5)

**Env value validation:** Alphanumeric + common path characters only. Shell metacharacters `;|&`$` are rejected.

**`{port}` placeholder:** Validated as integer in `[1024, 65535]` before substitution.

### Port Allocation (D-6)

```
port = port_range_start + task_ordinal_index
```

- `port_range_start` MUST be in `[1024, 65535 - max_concurrent_instances]`
- `default_port` MUST be in `[1024, 65535]`
- Privileged ports (< 1024) are rejected at Step 0
- Port exhaustion check at contract validation

### DB Isolation Strategies (D-17)

```yaml
db_isolation_strategy: "per-instance-db"
# Options: "per-instance-db" | "transaction-rollback" | "shared-with-locking"
```

| Strategy               | Mechanism                                        | When to Use                 |
| ---------------------- | ------------------------------------------------ | --------------------------- |
| `per-instance-db`      | Append `_task_ordinal` to DB name/path           | Default, safest             |
| `transaction-rollback` | Wrap each skill in a transaction, rollback after | Shared DB, stateless skills |
| `shared-with-locking`  | Advisory lock before each skill, release after   | Complex shared-state apps   |

Warning is logged if `supports_parallel=true` with no `db_isolation_strategy`.

### Playwright CLI Configuration

**Installation:**

```bash
npm install -g @playwright/cli@latest
```

**Skills installation:**

```bash
playwright-cli install --skills
```

This installs agent-discoverable skill guides into `skills/playwright-cli/` and `.claude/skills/dev/` (also read by GitHub Copilot).

**Per-project config:** `.playwright/cli.config.json`

**Session isolation:** `-s=verify-{task-id}` — named sessions per verifier instance.

**Token efficiency:** CLI commands are purpose-built and avoid dumping page/accessibility data into LLM context. Explicitly recommended by Microsoft for coding agents (over MCP).

### Complete Contract Field Reference

| Group         | Field                        | Type         | Default           | Purpose                             |
| ------------- | ---------------------------- | ------------ | ----------------- | ----------------------------------- |
| App Lifecycle | `app_type`                   | enum         | —                 | `web\|api\|cli\|desktop\|library`   |
| App Lifecycle | `start_command`              | object       | —                 | `{executable, args[], env{}}`       |
| App Lifecycle | `ready_check`                | string       | —                 | URL or command for readiness        |
| App Lifecycle | `ready_timeout_ms`           | integer      | `30000`           | Max wait for readiness              |
| App Lifecycle | `base_url`                   | string       | —                 | Application base URL                |
| App Lifecycle | `shutdown_command`           | object\|null | —                 | `{executable, args[]}`              |
| App Lifecycle | `shutdown_timeout_ms`        | integer      | `10000`           | Max wait for shutdown               |
| Port          | `default_port`               | integer      | —                 | Non-parallel port                   |
| Port          | `port_env_var`               | string       | —                 | Env var name for port injection     |
| Port          | `port_range_start`           | integer      | —                 | Base port for parallel allocation   |
| Runner        | `e2e_runner`                 | string       | —                 | `playwright\|cypress\|custom\|none` |
| Runner        | `e2e_command`                | string\|null | `null`            | Pre-written suite command           |
| Runner        | `e2e_config_path`            | string\|null | `null`            | Runner config path                  |
| Runner        | `e2e_env`                    | map          | `{}`              | Runner environment vars             |
| Parallelism   | `supports_parallel`          | boolean      | `false`           | Can run concurrent instances        |
| Parallelism   | `requires_isolated_instance` | boolean      | `true`            | Each instance needs own port/DB     |
| Parallelism   | `max_concurrent_instances`   | integer      | `2`               | Hard cap: 4                         |
| Parallelism   | `db_isolation_strategy`      | enum         | `per-instance-db` | See DB Isolation table above        |
| Interaction   | `interaction_type`           | enum         | —                 | `browser\|api\|cli\|none`           |
| Interaction   | `interaction_tools`          | map          | —                 | Tool config per interaction type    |
| Interaction   | `readiness_checks`           | list         | `[]`              | Pre-interaction verification checks |
| Interaction   | `evidence_capture`           | map          | —                 | Capture methods config              |
| Skills        | `skills`                     | list         | `[]`              | Inline skill objects                |
| Skills        | `skill_discovery_path`       | string\|null | `null`            | Directory of skill YAML files       |
| Evidence      | `screenshot_on_failure`      | boolean      | `true`            | Capture screenshot on step failure  |
| Evidence      | `trace_on_failure`           | boolean      | `true`            | Capture trace on failure            |
| Evidence      | `video_recording`            | boolean      | `false`           | Record video of browser sessions    |
| Evidence      | `har_capture`                | boolean      | `false`           | Capture HAR network logs            |
| Evidence      | `evidence_output_dir`        | string       | `.e2e-evidence/`  | Output directory for evidence files |

---

## §2 Skills Schema & Storage

Skills define reusable interaction procedures. Per D-18, all steps are **pre-defined** — no agent improvisation. The verifier follows steps exactly as written.

### Skill Structure

| Field                    | Type    | Required | Purpose                                             |
| ------------------------ | ------- | -------- | --------------------------------------------------- |
| `id`                     | string  | Yes      | Unique identifier (e.g., `SKILL-001`)               |
| `name`                   | string  | Yes      | Human-readable name                                 |
| `type`                   | enum    | Yes      | `test-suite` \| `exploratory` \| `adversarial`      |
| `interaction`            | enum    | Yes      | `browser` \| `api` \| `cli` \| `test-command`       |
| `app_type_filter`        | list    | No       | App types this skill applies to, or `['*']` for all |
| `preconditions`          | list    | No       | Conditions required before skill runs               |
| `steps`                  | list    | Yes      | Ordered interaction steps                           |
| `expected_outcomes`      | list    | No       | Overall skill outcome assertions                    |
| `adversarial_variations` | list    | No       | Override objects for adversarial testing            |
| `tags`                   | list    | No       | Categorization tags                                 |
| `timeout_ms`             | integer | No       | Max time for entire skill (default `60000`)         |

### Skill Types

| Type          | Interaction Mode    | Description                                                                                 |
| ------------- | ------------------- | ------------------------------------------------------------------------------------------- |
| `test-suite`  | `test-command`      | Runs a pre-written test command. Steps: `run_command`, `assert_exit_code`, `capture_output` |
| `exploratory` | `browser\|api\|cli` | Drives agent to navigate/use the app. Full step action set.                                 |
| `adversarial` | `browser\|api\|cli` | Designed to break things: malformed input, injection, boundary values, auth bypass          |

### Step Schema

| Field        | Type         | Required | Purpose                                                     |
| ------------ | ------------ | -------- | ----------------------------------------------------------- |
| `order`      | integer      | Yes      | Execution sequence                                          |
| `action`     | string       | Yes      | Action to perform (see Action Reference below)              |
| `target`     | string       | Yes      | CSS selector, URL, endpoint, or command                     |
| `value`      | string\|null | No       | Input value for fill/submit/request body                    |
| `method`     | string\|null | No       | HTTP method (`GET\|POST\|PUT\|DELETE\|PATCH`)               |
| `headers`    | map\|null    | No       | HTTP headers for API interactions                           |
| `expect`     | string\|null | No       | Human-readable expected result (logged, not judged)         |
| `assert`     | object\|null | No       | Machine-checkable assertion (determines pass/fail)          |
| `capture`    | string\|null | No       | Evidence capture: `screenshot\|response\|har\|dom\|console` |
| `timeout_ms` | integer      | No       | Max wait for this step (default `5000`)                     |
| `on_failure` | enum         | No       | `fail` \| `continue` \| `skip_remaining` (default `fail`)   |

### Step Action Reference

- **Browser:** `navigate`, `click`, `fill`, `submit`, `scroll`, `hover`, `wait_for`, `assert_visible`, `assert_text`, `screenshot`
- **API:** `http_request`, `assert_status`, `assert_body`, `assert_header`
- **CLI:** `run_command`, `send_input`, `assert_output`, `assert_exit_code`

### Assertion Types

`status_code` (int), `text_contains` (string), `element_visible` (string), `element_not_visible` (string), `url_matches` (string), `response_body_contains` (string), `exit_code_equals` (int).

### Adversarial Variations

Each variation: `id` (string, required), `name` (string, required), `description` (string), `severity_if_missed` (enum: `critical|high|medium|low`, required), `overrides` (list of `{step_order, field, override_value}`, required), `expected_behavior` (string, required — what SHOULD happen when attack is blocked), `assert` (object — machine-checkable assertion).

### Storage (D-4)

Skills can be stored in two ways:

1. **Inline** — Embedded in the contract's `skills` list (simple projects)
2. **Directory** — Separate YAML files in `skill_discovery_path`, organized by type:
   ```
   .e2e/skills/
   ├── test-suite/
   │   └── run-playwright-suite.yaml
   ├── exploratory/
   │   ├── login-flow.yaml
   │   └── users-crud.yaml
   └── adversarial/
       └── injection-tests.yaml
   ```

---

## §3 Playwright CLI Interaction Protocol

The **primary browser interaction tool** is `@playwright/cli` (`playwright-cli` binary). This section defines the complete interaction protocol (D-20).

### Installation

```bash
npm install -g @playwright/cli@latest
playwright-cli install --skills
```

Skills installation places skill guides into `skills/playwright-cli/` and `.claude/skills/dev/`.

**Per-project config:** `.playwright/cli.config.json`

### Session Isolation

Every verifier instance uses a **named session**:

```bash
playwright-cli -s=verify-{task-id} <command>
```

Named sessions ensure parallel verifier instances do not share browser state.

### CLI Command Reference

#### Core Interaction

| Command  | Syntax                                             | Purpose                            |
| -------- | -------------------------------------------------- | ---------------------------------- |
| Open     | `playwright-cli -s=<session> open [url]`           | Launch browser                     |
| Navigate | `playwright-cli -s=<session> goto <url>`           | Navigate to URL                    |
| Click    | `playwright-cli -s=<session> click <ref>`          | Click element by accessibility ref |
| Fill     | `playwright-cli -s=<session> fill <ref> <text>`    | Fill form field                    |
| Type     | `playwright-cli -s=<session> type <text>`          | Type text                          |
| Press    | `playwright-cli -s=<session> press <key>`          | Press keyboard key                 |
| Check    | `playwright-cli -s=<session> check <ref>`          | Check checkbox                     |
| Select   | `playwright-cli -s=<session> select <ref> <value>` | Select dropdown option             |
| Hover    | `playwright-cli -s=<session> hover <ref>`          | Hover over element                 |

#### Evidence Capture

| Command    | Syntax                                                       | Purpose                        |
| ---------- | ------------------------------------------------------------ | ------------------------------ |
| Screenshot | `playwright-cli -s=<session> screenshot [--filename <path>]` | Capture screenshot             |
| Snapshot   | `playwright-cli -s=<session> snapshot [--filename <path>]`   | Capture accessibility snapshot |
| Console    | `playwright-cli -s=<session> console [min-level]`            | Get console messages           |
| Network    | `playwright-cli -s=<session> network`                        | Get network requests           |
| Tracing    | `playwright-cli -s=<session> tracing-start` / `tracing-stop` | Record traces                  |
| Video      | `playwright-cli -s=<session> video-start` / `video-stop`     | Record video                   |

#### Navigation & Session Management

- **Navigation:** `go-back`, `go-forward`, `reload`
- **Sessions:** `list` (show active), `close` (specific), `close-all`, `kill-all`, `show` (dashboard)
- **Storage:** `state-save`, `state-load`, `cookie-list`, `cookie-get <name>`, `cookie-set <name> <value>`, `cookie-delete <name>`, `cookie-clear`

All session commands use `-s=<session>` prefix. Management commands (`list`, `close-all`, `kill-all`) are global.

### Skill Step → CLI Command Mapping

`navigate`→`goto`, `click`→`click {ref}`, `fill`→`fill {ref} {text}`, `submit`→`click {ref}`, `assert`→`snapshot` + verify, `screenshot`→`screenshot`, `hover`→`hover {ref}`.

### Fallback: Script Generation for Complex Suites (D-20)

For `test-suite` type skills requiring the full Playwright Test runner:

1. Verifier reads skill YAML and contract metadata
2. Generates test script: `.e2e/generated/<skill-name>.spec.ts`
3. Translates each skill step into Playwright API calls deterministically (`navigate`→`page.goto`, `click`→`page.click`, `fill`→`page.fill`, `assert`→`expect`)
4. Executes: `npx playwright test <script-path> --reporter=json`
5. Parses JSON reporter output for per-step pass/fail
6. Captures evidence: screenshots per step, HAR log, console output

The generated script file serves as an audit artifact.

### Determinism (D-18)

- Follow steps exactly as defined in the skill — **no agent improvisation**
- Machine-checkable assertions (`assert` field) determine pass/fail deterministically
- Human-readable expectations (`expect` field) are logged but do NOT determine pass/fail
- All commands executed via `run_in_terminal`, validated against command allowlist (§5)

---

## §4 Command Sanitization Rules (D-21)

All commands from the E2E contract use the structured format. The verifier MUST sanitize all command components before execution.

### Executable Validation

The `executable` field MUST satisfy ALL of:

- No path separators (`/`, `\`)
- No spaces
- No shell metacharacters: `;`, `|`, `&`, `` ` ``, `$`, `(`, `)`, `{`, `}`
- Matches at least one pattern in `tier5_command_allowlist` (§5)

### Args Array

Arguments are passed as array elements — **never through shell interpolation**. The verifier constructs the command as a single quoted string with arguments.

### Env Value Validation

Environment variable values are validated against a pattern allowlist:

- **Allowed:** Alphanumeric characters, common path characters (`/`, `\`, `.`, `-`, `_`, `:`)
- **Rejected:** Shell metacharacters: `;`, `|`, `&`, `` ` ``, `$`

### Port Placeholder

The `{port}` placeholder in `args`:

- Validated as integer
- Range: `[1024, 65535]`
- Substituted **after** validation, **before** command execution

### Secret Pattern Scrubbing

Environment variable names matching these patterns have their values replaced with `[REDACTED]` in all evidence:

- `API_KEY`
- `SECRET`
- `TOKEN`
- `PASSWORD`
- `CREDENTIAL`

---

## §5 Command Allowlist (D-22)

**Canonical copy.** The verifier's `run_in_terminal` during Tier 5 is restricted to commands matching these regex patterns:

| #   | Pattern                                               | Purpose                                                                                               |
| --- | ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 1   | `^playwright-cli .+`                                  | Playwright CLI browser interaction                                                                    |
| 2   | `^npx playwright test .+`                             | Execute generated E2E test scripts                                                                    |
| 3   | `^npm (run\|start\|test)`                             | Application lifecycle management                                                                      |
| 4   | `^node \.[\\/].+\.m?[jt]s$`                           | Local Node.js script execution (file path only, no `-e` flag)                                         |
| 5   | `^dotnet (run\|test)`                                 | .NET application lifecycle                                                                            |
| 6   | `^python -m (pytest\|uvicorn\|flask)`                 | Python application lifecycle                                                                          |
| 7   | `^curl -sf?o? https?://localhost[:/]`                 | Health check HTTP requests (localhost only)                                                           |
| 8   | `^(kill (-[0-9]+ )?[0-9]+\|taskkill /F /PID [0-9]+)$` | Process cleanup during teardown (numeric PID only — must match tracked PID from `e2e-instance-start`) |

### Enforcement

- Commands are checked against patterns **before** execution
- Commands not matching any pattern → **REJECTED** with error logged to `interaction_log`
- Pattern `^playwright-cli .+` matches all Playwright CLI commands: `open`, `goto`, `click`, `fill`, `screenshot`, `snapshot`, `console`, `network`, `tracing-start`, `tracing-stop`, `video-start`, `video-stop`, `list`, `close-all`, `kill-all`, etc.

### Audit Trail

Every Tier 5 command execution is recorded:

| Field             | Description                                     |
| ----------------- | ----------------------------------------------- |
| `command_text`    | The full command string                         |
| `matched_pattern` | Which allowlist pattern matched (or `REJECTED`) |
| `timestamp`       | ISO 8601 execution timestamp                    |
| `exit_code`       | Command exit code                               |

---

## §6 Evidence Sanitization Pipeline (D-25)

All E2E evidence MUST pass through this pipeline before SQL insertion or file storage.

### Step 1: SQL Escaping (Mandatory)

All free-text string values use `sql_escape()` (replace `'` with `''`) before INSERT into `anvil_checks` or `pipeline_telemetry`. This applies to:

- `interaction_log` entries
- Error messages
- Assertion results
- `output_snippet` content

### Step 2: HAR Log Stripping

| Header                | Action                 |
| --------------------- | ---------------------- |
| `Authorization`       | → `[REDACTED]`         |
| `Cookie`              | → `[REDACTED]`         |
| `Set-Cookie`          | → `[REDACTED]`         |
| Request/response body | > 10KB → `[TRUNCATED]` |

### Step 3: Response Masking

- **Screenshot paths:** Validated to be within `.e2e/evidence/` directory — no path traversal (`..`)
- **Console output:** Capped at **5KB per step**
- **File naming:** `<skill-id>_step-<order>_<timestamp>.png`

### Step 4: start_env Scrubbing

Environment variable values matching secret patterns are replaced with `[REDACTED]`:

- Pattern list: `API_KEY`, `SECRET`, `TOKEN`, `PASSWORD`, `CREDENTIAL`
- Applied in ALL evidence artifacts (SQL records, log files, manifests)

### Step 5: Evidence Manifest Integrity

- The verifier generates a **SHA-256 hash** of the evidence manifest (list of artifact paths + sizes)
- Hash recorded in the verification report
- Adversarial reviewer can verify manifest integrity by recomputing the hash

### Evidence Storage Structure

```
.e2e-evidence/<task-id>/<skill-id>/
├── SKILL-001_step-01_2026-03-04T10-15-30.png
├── SKILL-001_step-04_2026-03-04T10-15-45.png
├── network.har
├── console.log
└── trace.zip
```

### SQL Evidence Records

E2E phases produce these `check_name` values in `anvil_checks`:

| check_name                | Phase       | Description                               |
| ------------------------- | ----------- | ----------------------------------------- |
| `e2e-contract-found`      | Setup       | Contract exists and is valid              |
| `e2e-contract-validation` | Setup       | Contract validation details               |
| `e2e-instance-start`      | Setup       | App started on assigned port (PID stored) |
| `e2e-readiness`           | Setup       | App passed `ready_check`                  |
| `e2e-suite-execution`     | Suite       | Test suite execution pass/fail            |
| `e2e-exploratory`         | Exploratory | Exploratory interaction composite result  |
| `e2e-adversarial`         | Adversarial | Per-variation adversarial result          |
| `e2e-adversarial-summary` | Adversarial | Composite adversarial summary             |
| `e2e-instance-shutdown`   | Teardown    | App + tools shut down cleanly             |
| `e2e-test-execution`      | Composite   | ALL E2E phases passed                     |

Per-variation adversarial records use `check_name='e2e-adversarial-<variation-name>'`. Variation names: alphanumeric + hyphens only, max 50 chars, `sql_escape()` applied (D-26).

---

## §7 Reference Examples (FR-3.9)

### Example 1: Browser Login Flow (Exploratory)

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
      assert: { type: url_matches, value: "/dashboard" }
      capture: screenshot
    - order: 5
      action: navigate
      target: "/profile"
      assert: { type: text_contains, value: "test@example.com" }
      capture: screenshot
  adversarial_variations:
    - id: "ADV-001"
      name: "sql-injection-email"
      severity_if_missed: critical
      overrides:
        - step_order: 2
          field: value
          override_value: "' OR 1=1; --"
      expected_behavior: "Error message displayed, no redirect to dashboard"
      assert: { type: element_visible, value: ".error-message" }
    - id: "ADV-002"
      name: "empty-password"
      severity_if_missed: high
      overrides:
        - step_order: 3
          field: value
          override_value: ""
      expected_behavior: "Validation error shown, form not submitted"
      assert: { type: element_visible, value: ".validation-error" }
    - id: "ADV-003"
      name: "xss-in-email"
      severity_if_missed: critical
      overrides:
        - step_order: 2
          field: value
          override_value: "<script>alert('xss')</script>"
      expected_behavior: "Input sanitized or rejected, no script execution"
      assert: { type: element_not_visible, value: "script" }
```

**Playwright CLI session lifecycle for this skill:**

```bash
playwright-cli -s=verify-task-03 open http://localhost:9001
playwright-cli -s=verify-task-03 goto http://localhost:9001/login
playwright-cli -s=verify-task-03 screenshot --filename .e2e-evidence/task-03/SKILL-001/step-01.png
playwright-cli -s=verify-task-03 fill "#email" "test@example.com"
playwright-cli -s=verify-task-03 fill "#password" "password123"
playwright-cli -s=verify-task-03 click "#submit"
playwright-cli -s=verify-task-03 screenshot --filename .e2e-evidence/task-03/SKILL-001/step-04.png
playwright-cli -s=verify-task-03 goto http://localhost:9001/profile
playwright-cli -s=verify-task-03 snapshot
playwright-cli -s=verify-task-03 screenshot --filename .e2e-evidence/task-03/SKILL-001/step-05.png
playwright-cli -s=verify-task-03 close
```

### Example 2: API CRUD Probe (Exploratory)

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
      method: PUT
      target: "/api/users/1"
      value: '{"name": "Updated User"}'
      headers: { "Content-Type": "application/json" }
      assert: { type: status_code, value: 200 }
    - order: 4
      action: http_request
      method: DELETE
      target: "/api/users/1"
      assert: { type: status_code, value: 204 }
    - order: 5
      action: http_request
      method: GET
      target: "/api/users/1"
      assert: { type: status_code, value: 404 }
      capture: response
```

### Example 3: CLI Input Testing

```yaml
skill:
  id: "SKILL-003"
  name: "test-cli-input-handling"
  type: exploratory
  interaction: cli
  steps:
    - order: 1
      action: run_command
      target: "myapp --help"
      assert: { type: exit_code_equals, value: 0 }
      capture: console
    - order: 2
      action: run_command
      target: "myapp process valid-input.json"
      assert: { type: exit_code_equals, value: 0 }
      capture: console
    - order: 3
      action: run_command
      target: "myapp process invalid-input.json"
      assert: { type: exit_code_equals, value: 1 }
      capture: console
    - order: 4
      action: run_command
      target: "myapp process nonexistent.json"
      assert: { type: exit_code_equals, value: 1 }
      capture: console
```

### Example 4: Adversarial SQL Injection Variation

```yaml
# Applied as a variation of SKILL-002 (API CRUD)
adversarial_variation:
  id: "ADV-011"
  name: "sql-injection-in-query"
  severity_if_missed: critical
  overrides:
    - step_order: 2
      field: target
      override_value: "/api/users/1 OR 1=1"
  expected_behavior: "Returns 400 or 404, not all users"
  assert: { type: status_code, value: 400 }
```

---

## §8 E2E Timeout Budget (D-9)

### Per-Phase Limits

| Phase                   | Timeout             | On Expiry                                                    |
| ----------------------- | ------------------- | ------------------------------------------------------------ |
| App startup             | 60s (max)           | Kill PID, record `e2e-readiness` passed=0                    |
| Suite execution         | 300s per task       | Kill test runner, record `e2e-suite-execution` passed=0      |
| Exploratory interaction | 180s per skill      | Kill browser/tool, skip remaining steps                      |
| Adversarial interaction | 120s per skill      | Kill browser/tool, skip remaining variations                 |
| App shutdown            | 30s (max)           | Force-kill PID                                               |
| **Total E2E per task**  | **600s (hard cap)** | **Kill all processes, record `e2e-test-execution` passed=0** |

### Realistic Budget Example

```
Startup:           30s
Suite execution:  180s
Exploratory ×2:   90s × 2 = 180s
Adversarial ×1:   60s
Shutdown:          10s
─────────────────────
Total:            460s (within 600s cap)
```

Per-phase maximums sum to 690s > 600s. For tasks with skills in all phases, effective time per phase is reduced by the 600s hard cap.

### Timeout Enforcement

- Timeout triggers **force-kill** of app process + all `playwright-cli` sessions
- Kill uses tracked PIDs and `playwright-cli kill-all`
- Timeout failures are recorded in `anvil_checks` with `passed=0`
- After timeout, the verifier proceeds to teardown (Phase 5) immediately

### Verifier E2E Lifecycle (5-Phase Summary)

| Phase | Name        | Key Actions                                                                |
| ----- | ----------- | -------------------------------------------------------------------------- |
| 1     | Setup       | Read contract, validate (runtime), start app, poll readiness, record PIDs  |
| 2     | Suite       | Execute `test-suite` skills via script generation or CLI                   |
| 3     | Exploratory | Execute `exploratory` skills step-by-step via Playwright CLI               |
| 4     | Adversarial | Execute `adversarial` skills with step overrides, record per-variation SQL |
| 5     | Teardown    | Shutdown app, close browser sessions, write evidence manifest + SHA-256    |

**Ordering guarantee (D-5):** Tier 5 MUST execute AFTER Tiers 1–4 complete and SQL committed — never interleaved. Within Tier 5, phases execute sequentially: 1 → 2 → 3 → 4 → 5.

### Parallel Instance Isolation (D-17)

- **Session naming:** `verify-{task-id}` — each verifier instance uses its own named session
- **Port allocation:** `port_range_start + task_ordinal_index` — deterministic, no collisions
- **Max concurrent browser instances:** `max_concurrent_instances` from contract (default 2, hard cap 4) — this controls how many browser contexts a **single** verifier task may spawn, NOT cross-task dispatch concurrency. Cross-task E2E dispatch is capped at 1 per [dispatch-patterns.md](dispatch-patterns.md) §E2E Concurrency.
- **Evidence isolation:** `evidence_output_dir/<task-id>/` — task-scoped directories
- **Browser isolation:** Separate browser context per verifier — no shared state
- **DB isolation:** Applied per `db_isolation_strategy` from contract
- **SQLite:** WAL mode + `busy_timeout` for concurrent access

### API/CLI App Testing

For non-browser apps, skills use API and CLI interaction modes:

- **API apps:** Skills use `http_request` action with `curl` commands (validated against allowlist pattern `^curl -sf?o? https?://localhost[:/]` — localhost only). Assertions on `status_code`, `response_body_contains`, `assert_header`.
- **CLI apps:** Skills use `run_command` and `send_input` actions executed via `run_in_terminal`. Assertions on `exit_code_equals`, `assert_output`.
- **Session management:** Not applicable for API/CLI — each step is an independent command execution.
- **Evidence:** API responses and CLI output captured in evidence files, subject to the same sanitization pipeline (§6).
