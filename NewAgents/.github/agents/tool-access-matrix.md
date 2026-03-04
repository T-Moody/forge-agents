# Tool Access Matrix

Per-agent tool access rules for all 9 pipeline agents. Agents reference this document instead of maintaining inline tool access tables.

## §1 Master Matrix

| Tool                         | Orch | Res | Spec | Des | Plan | Impl | Verif | AdvRev | Know |
| ---------------------------- | ---- | --- | ---- | --- | ---- | ---- | ----- | ------ | ---- |
| read_file                    | ✅   | ✅  | ✅   | ✅  | ✅   | ✅   | ✅    | ✅     | ✅   |
| list_dir                     | ✅   | ✅  | ✅   | ✅  | ✅   | ✅   | ✅    | ✅     | ✅   |
| grep_search                  | ❌   | ✅  | ✅   | ✅  | ✅   | ✅   | ✅    | ✅     | ✅   |
| semantic_search              | ❌   | ✅  | ✅   | ✅  | ✅   | ✅   | ❌    | ✅     | ✅   |
| file_search                  | ❌   | ✅  | ✅   | ✅  | ✅   | ✅   | ✅    | ✅     | ✅   |
| create_file                  | ❌   | 🔒  | ✅   | ✅  | ✅   | ✅   | 🔒    | ✅     | ✅   |
| replace_string_in_file       | ❌   | ❌  | ✅   | ✅  | ✅   | ✅   | ❌    | ❌     | ✅   |
| multi_replace_string_in_file | ❌   | ❌  | ❌   | ❌  | ❌   | ✅   | ❌    | ❌     | ❌   |
| run_in_terminal              | 🔒   | ❌  | ❌   | ❌  | ❌   | ✅   | ✅    | 🔒     | 🔒   |
| get_terminal_output          | ✅   | ❌  | ❌   | ❌  | ❌   | ✅   | ✅    | ❌     | ❌   |
| get_errors                   | ❌   | ❌  | ❌   | ❌  | ❌   | ✅   | ✅    | ❌     | ❌   |
| memory                       | ✅   | ❌  | ❌   | ❌  | ❌   | ❌   | ❌    | ❌     | ✅   |
| ask_questions                | ✅   | ❌  | 🔒   | ❌  | ❌   | ❌   | ❌    | ❌     | ❌   |
| list_code_usages             | ❌   | ❌  | ❌   | ❌  | ❌   | ✅   | ❌    | ❌     | ❌   |

**Legend:** ✅ Allowed — ❌ Restricted — 🔒 Allowed with scope restriction (see §2–§10)

## §2 Orchestrator

- **7 allowed:** read_file, list_dir, get_terminal_output, memory, ask_questions, run_in_terminal 🔒, agent/runSubagent
- **run_in_terminal 🔒** — SQLite queries (SELECT, DDL) + git operations only. No builds, tests, or code execution.

## §3 Researcher

- **6 allowed:** read_file, list_dir, grep_search, semantic_search, file_search, create_file 🔒
- **create_file 🔒** — `research/*.yaml` only. Path must match: `research/.*\.yaml$`

## §4 Spec

- **8 allowed:** read_file, list_dir, grep_search, semantic_search, file_search, create_file, replace_string_in_file, ask_questions 🔒
- **ask_questions 🔒** — Interactive mode only (user-facing clarification questions).

## §5 Designer

- **7 allowed:** read_file, list_dir, grep_search, semantic_search, file_search, create_file, replace_string_in_file
- No run_in_terminal access.

## §6 Planner

- **7 allowed:** read_file, list_dir, grep_search, semantic_search, file_search, create_file, replace_string_in_file
- No run_in_terminal access.

## §7 Implementer

- **12 allowed (maximum):** read_file, list_dir, grep_search, semantic_search, file_search, create_file, replace_string_in_file, multi_replace_string_in_file, run_in_terminal, get_terminal_output, get_errors, list_code_usages
- Full tool access. No scope restrictions.

## §8 Verifier

- **9 allowed:** read_file, list_dir, grep_search, file_search, run_in_terminal, get_terminal_output, get_errors, create_file 🔒
- **create_file 🔒** — `verification-reports/*.yaml` only. Path must match: `verification-reports/.*\.yaml$`

### §8.1 Tier 5 run_in_terminal Expansion (D-11)

When `e2e_required=true` in the task definition, the verifier's `run_in_terminal` access is expanded to Tier 5 scope. This enables live application testing via E2E verification phases (Setup → Test Suite → Exploratory → Adversarial → Teardown).

**Phase gating:** Tier 5 commands are permitted **only** during E2E verification phases (Phases 1–5 per FR-4.2). During Tiers 1–4 (static checks, build verification, unit/integration tests, documentation review), `run_in_terminal` remains restricted to standard verifier commands (test runners, build commands). The verifier MUST NOT execute Tier 5 commands outside an active E2E verification sequence.

**Command constraint:** Every `run_in_terminal` invocation during Tier 5 MUST match at least one pattern in the `tier5_command_allowlist` below. Commands not matching any pattern are **REJECTED** and logged in the `interaction_log` with `matched_pattern: 'REJECTED'`.

### §8.2 tier5_command_allowlist (D-22)

The following regex patterns define the complete set of allowed Tier 5 commands. This follows the §10 Knowledge Agent precedent for command-restricted tool access. Canonical source: [e2e-integration.md §5](e2e-integration.md).

| # | Pattern | Purpose |
|---|---------|---------|
| 1 | `^npx playwright test .+` | Execute generated E2E test scripts |
| 2 | `^playwright-cli .+` | Playwright CLI browser interaction (open, goto, click, fill, screenshot, snapshot, close, etc.) |
| 3 | `^npm (run\|start\|test)` | Application lifecycle management |
| 4 | `^node \.[\\/].+\.m?[jt]s$` | Local Node.js script execution (file path only, no `-e` flag) |
| 5 | `^dotnet (run\|test)` | .NET application lifecycle |
| 6 | `^python -m (pytest\|uvicorn\|flask)` | Python application lifecycle |
| 7 | `^curl -sf?o? https?://localhost[:/]` | Health check HTTP requests (localhost only) |
| 8 | `^(kill (-[0-9]+ )?[0-9]+\|taskkill /F /PID [0-9]+)$` | Process cleanup during teardown (numeric PID only — must match tracked PID from `e2e-instance-start`) |

**Evaluation:** The verifier checks its command against patterns **before** execution. First matching pattern wins. If no pattern matches, execution is blocked.

### §8.3 Tier 5 Audit Trail

Every Tier 5 `run_in_terminal` execution MUST be logged in the `interaction_log` with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `command_text` | string | The full command string as submitted |
| `matched_pattern` | string | The regex pattern that matched, or `'REJECTED'` if none matched |
| `timestamp` | ISO 8601 | When the command was submitted |
| `exit_code` | integer \| null | Process exit code (`null` if rejected before execution) |

This provides a complete audit trail for adversarial review per D-11. Rejected commands are logged but never executed.

### §8.4 Playwright CLI Session Management

The verifier uses `playwright-cli` with named sessions for browser interaction during E2E verification. All session commands match the `^playwright-cli .+` allowlist pattern (#2).

**Session lifecycle:**

| Phase | Command | Purpose |
|-------|---------|---------|
| Creation | `playwright-cli -s=verify-{task-id} open {base_url}` | Launch browser with named session |
| Interaction | `playwright-cli -s=verify-{task-id} <command>` | Execute browser commands (goto, click, fill, screenshot, snapshot, etc.) |
| Cleanup | `playwright-cli -s=verify-{task-id} close` | Close specific session |
| Emergency cleanup | `playwright-cli close-all` | Close all active sessions |
| Emergency cleanup | `playwright-cli kill-all` | Force-kill all sessions |
| Listing | `playwright-cli list` | List active sessions (for audit/debugging) |

**Session isolation:** The `-s=verify-{task-id}` naming convention ensures parallel verifier instances do not share browser state. Each verifier instance uses its own task ID as the session identifier.

## §9 Adversarial Reviewer

- **7 allowed:** read_file, list_dir, grep_search, semantic_search, file_search, create_file, run_in_terminal 🔒
- **run_in_terminal 🔒** — `git diff` commands + SQL INSERT only. No builds, tests, or arbitrary shell commands.

## §10 Knowledge Agent

- **9 allowed:** read_file, list_dir, grep_search, semantic_search, file_search, create_file, replace_string_in_file, memory, run_in_terminal 🔒
- **run_in_terminal 🔒** — SELECT on all tables + INSERT on `instruction_updates` only.
- **Allowed SQL operations:**
  - `SELECT` — any table in verification-ledger.db
  - `INSERT INTO instruction_updates` — governed update tracking
  - `PRAGMA busy_timeout` — connection configuration only
- **Prohibited DML/DDL:** UPDATE, DELETE, DROP, ALTER, CREATE, ATTACH, PRAGMA (except busy_timeout)
- Regex pattern: `^(echo\s+".*(SELECT|INSERT INTO instruction_updates).*"\s*\|\s*sqlite3|sqlite3)\s+.*verification-ledger\.db`

## §11 Enforcement Note

Scope restrictions are enforced via agent instructions. VS Code's agent runtime does not support parameterized tool access policies. Compliance depends on LLM instruction-following and is verified through the following layers:

1. **Agent prompt instructions** — each agent's `.agent.md` specifies allowed tool scopes
2. **Self-verification checklists** — agents verify commands match allowed patterns before each scoped tool call
3. **Regex validation patterns** — per-agent patterns (documented in §2–§10) define allowed operations
4. **Verifier as secondary enforcement** — the verifier reviews other agents' outputs and flags file path or command violations

This is an **accepted risk** (residual threat level: Medium). See SEC-4 and ARCH-7 review findings.
