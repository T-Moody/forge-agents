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
