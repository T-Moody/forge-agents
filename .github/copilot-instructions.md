# Global Operating Rules

These rules apply to ALL agents in this pipeline. They are automatically loaded by VS Code Copilot.

## Terminal-Only Testing

All test execution MUST use `run_in_terminal` with standard CLI commands (`dotnet test`, `npm test`, `pytest`, `cargo test`, `go test`, etc.). Do NOT use VS Code's built-in test runner tools (Testing API, `runTests` task) for test execution. IDE diagnostics via `get_errors` are permitted for compile-time error detection only.

## No File-Redirect

NEVER redirect terminal command output to a file (e.g., `command > output.txt`, `command | tee output.txt`, `command 2>&1 > log.txt`). Always read output directly from the terminal via `get_terminal_output` or `run_in_terminal` return value.

## Retry Policy

- **Agent-internal**: Retry transient errors (network timeout, tool unavailable, database locked) up to 2 times. NEVER retry deterministic failures (file not found, permission denied).
- **When stuck**: After 2 failed attempts at the same operation, report the failure clearly — do not spin.

## Schema Version

All agent outputs MUST include `schema_version: "1.0"` in the common header (`agent_output`).

## Anti-Hallucination

NEVER claim verification passed without executing actual checks. INSERT evidence into SQLite BEFORE reporting results. If a tool is unavailable, report it — do not fabricate output.

## No Runtime Agent Modification

No agent may modify another agent's `.agent.md` definition file at runtime. Agent files are static instruction files.

## Bounded Feedback Loops

All feedback loops MUST have hard iteration limits:

- Design revision: max 1 iteration
- Implementation-verification: max 3 iterations
- Code review cycling: max 2 rounds
  Never create unbounded retry or revision loops.

## Output Paths

All output files MUST be written within the designated feature directory (`docs/feature/<feature-slug>/`) or agent output directories. Never write files outside the workspace root.

## SQL Safety

When writing SQL via `run_in_terminal`, use stdin piping to avoid shell metacharacter injection:

```
echo "INSERT INTO table ..." | sqlite3 database.db
```

All free-text values in SQL statements MUST have single quotes escaped (replace `'` with `''`).
