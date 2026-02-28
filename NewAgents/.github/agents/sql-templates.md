# SQL Templates Reference

> **Tier:** 2 (Shared Reference Document)
> **Consumers:** Orchestrator, Implementer, Verifier, Adversarial Reviewer, Knowledge Agent
> **Database:** `verification-ledger.db` (single consolidated database — Decision D-1)

All SQL DDL and DML templates for the pipeline. Agents reference specific sections instead of inlining SQL. This document is the **single source of truth** for all database operations.

---

## §0 SQL Sanitization Requirements

> **MANDATORY for all agents that write SQL.** Addresses SEC-1 (SQL injection) and SEC-2 (command injection).

### sql_escape() — String Sanitization

All agents that write SQL MUST apply the following escaping before interpolating any text value into a SQL statement:

```
sql_escape(value):
  1. Replace all single quotes with doubled quotes:  '  →  ''
  2. Strip null bytes:  \0  →  (removed)
  3. Truncate to field-specific max length (see §1 CHECK constraints)
  Result: safe for SQL string interpolation within single quotes
```

**Every free-text field** — `output_snippet`, `notes`, `missing_information`, `inaccuracies`, `impact_on_work`, `change_summary`, `command`, `tool` — MUST be processed through `sql_escape()` before insertion.

### Shell Injection Prevention (SEC-2)

SQL commands executed via `run_in_terminal` pass through the system shell. To prevent shell metacharacter interpretation of agent-generated values, **all SQL MUST be piped via stdin to sqlite3**. NEVER pass SQL inline as a command argument.

**Preferred pattern — printf pipe:**

```bash
printf '%s' "INSERT INTO table VALUES ('escaped_value');" | sqlite3 verification-ledger.db
```

**Alternative — heredoc pattern:**

```bash
sqlite3 verification-ledger.db <<'EOSQL'
INSERT INTO table VALUES ('escaped_value');
EOSQL
```

**PowerShell pattern — echo pipe:**

```powershell
echo "INSERT INTO table VALUES ('escaped_value');" | sqlite3 verification-ledger.db
```

> **Why stdin piping?** When SQL is passed as a shell argument (e.g., `sqlite3 db "INSERT ..."`), shell metacharacters in values — `$()`, backticks, pipes (`|`), semicolons (`;`) — are interpreted by the shell before reaching sqlite3. Stdin piping bypasses shell interpretation of the SQL content.

### Quoting Convention (CORR-3)

| Value Type       | Convention                        | Example                 |
| ---------------- | --------------------------------- | ----------------------- |
| String (present) | Single-quoted with `sql_escape()` | `'{sql_escape(value)}'` |
| String (absent)  | Literal `NULL` — **unquoted**     | `NULL`                  |
| Integer          | Unquoted numeric value            | `1`, `0`, `42`          |
| Boolean (0/1)    | Unquoted integer                  | `1` (true), `0` (false) |

**Rule:** Nullable string fields use `NULL` (unquoted, no surrounding quotes) when the value is absent, or `'{sql_escape(value)}'` when present. Never write `'NULL'` (quoted) for absent values.

### Self-Verification Checklist

Before executing any SQL INSERT via `run_in_terminal`, every SQL-writing agent MUST verify:

- [ ] All free-text SQL values have single quotes escaped (doubled): `'` → `''`
- [ ] Null bytes stripped from all text values
- [ ] SQL is piped via stdin (`printf '%s' "..." | sqlite3` or `echo "..." | sqlite3`)
- [ ] Text values do not exceed field-specific LENGTH limits (see §1 CHECK constraints)
- [ ] Nullable fields use unquoted `NULL` when absent (not `'NULL'`)

---

## §1 Database Initialization (Step 0 DDL)

The Orchestrator executes this DDL at Step 0 via `run_in_terminal`. Single database: `verification-ledger.db`.

```sql
-- ═══════════════════════════════════════════════════════════════
-- verification-ledger.db — Step 0 Initialization
-- Orchestrator-only. No other agent may execute DDL statements.
-- ═══════════════════════════════════════════════════════════════

PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;

-- Validate existing DB integrity at pipeline start
-- (Skip output check — informational only; logs warning if corruption detected)
PRAGMA integrity_check;

-- ───────────────────────────────────────────────────────────────
-- Table 1: anvil_checks (verification evidence)
-- Writers: Orchestrator, Implementer, Verifier, Adversarial Reviewer
-- ───────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS anvil_checks (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id         TEXT    NOT NULL,
  task_id        TEXT,
  phase          TEXT    NOT NULL CHECK (phase IN ('baseline','after','review')),
  check_name     TEXT    NOT NULL,
  tool           TEXT,
  command        TEXT,
  exit_code      INTEGER,
  output_snippet TEXT    CHECK (LENGTH(output_snippet) <= 500),
  passed         INTEGER NOT NULL CHECK (passed IN (0, 1)),
  verdict        TEXT    CHECK (verdict IN ('approve','needs_revision','blocker')),
  severity       TEXT    CHECK (severity IN ('Blocker','Critical','Major','Minor')),
  round          INTEGER DEFAULT 1,
  instance       TEXT,
  ts             TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_anvil_run        ON anvil_checks(run_id);
CREATE INDEX IF NOT EXISTS idx_anvil_task_phase ON anvil_checks(task_id, phase);
CREATE INDEX IF NOT EXISTS idx_anvil_run_round  ON anvil_checks(run_id, round);

-- ───────────────────────────────────────────────────────────────
-- Table 2: pipeline_telemetry (execution tracking)
-- Writer: Orchestrator only (after each agent dispatch)
-- ───────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS pipeline_telemetry (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id         TEXT    NOT NULL,
  step           TEXT    NOT NULL,
  agent          TEXT    NOT NULL,
  instance       TEXT,
  started_at     TEXT    NOT NULL,
  completed_at   TEXT,
  status         TEXT    CHECK (status IN ('DONE','NEEDS_REVISION','ERROR','TIMEOUT')),
  dispatch_count INTEGER DEFAULT 1,
  retry_count    INTEGER DEFAULT 0,
  notes          TEXT    CHECK (LENGTH(notes) <= 1000),
  ts             TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_telemetry_run  ON pipeline_telemetry(run_id);
CREATE INDEX IF NOT EXISTS idx_telemetry_step ON pipeline_telemetry(run_id, step);

-- ───────────────────────────────────────────────────────────────
-- Table 3: artifact_evaluations (quality feedback)
-- Writers: Phase 1 agents (Implementer, Verifier, Adversarial Reviewer)
-- ───────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS artifact_evaluations (
  id                  INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id              TEXT    NOT NULL,
  evaluator_agent     TEXT    NOT NULL,
  evaluator_instance  TEXT,
  artifact_path       TEXT    NOT NULL CHECK (artifact_path NOT LIKE '../%' AND artifact_path NOT LIKE '/%'),
  usefulness_score    INTEGER NOT NULL CHECK (usefulness_score BETWEEN 1 AND 10),
  clarity_score       INTEGER NOT NULL CHECK (clarity_score BETWEEN 1 AND 10),
  missing_information TEXT    CHECK (LENGTH(missing_information) <= 2000),
  inaccuracies        TEXT    CHECK (LENGTH(inaccuracies) <= 2000),
  impact_on_work      TEXT    CHECK (LENGTH(impact_on_work) <= 2000),
  ts                  TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_eval_run      ON artifact_evaluations(run_id);
CREATE INDEX IF NOT EXISTS idx_eval_agent    ON artifact_evaluations(evaluator_agent);
CREATE INDEX IF NOT EXISTS idx_eval_artifact ON artifact_evaluations(artifact_path);

-- ───────────────────────────────────────────────────────────────
-- Table 4: instruction_updates (governed update audit trail)
-- Writer: Knowledge Agent only
-- ───────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS instruction_updates (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id         TEXT    NOT NULL,
  agent          TEXT    NOT NULL,
  file_path      TEXT    NOT NULL CHECK (
                   file_path LIKE '.github/instructions/%'
                   OR file_path = '.github/copilot-instructions.md'
                 ),
  change_type    TEXT    NOT NULL CHECK (change_type IN ('create','append','modify','delete')),
  change_summary TEXT    NOT NULL CHECK (LENGTH(change_summary) <= 1000),
  applied        INTEGER NOT NULL DEFAULT 0 CHECK (applied IN (0, 1)),
  ts             TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_instr_run ON instruction_updates(run_id);
```

### Column Legend

| Column                | Type | Nullable | Notes                                                                                |
| --------------------- | ---- | -------- | ------------------------------------------------------------------------------------ |
| `output_snippet`      | TEXT | Yes      | Max 500 chars. Truncate before INSERT.                                               |
| `notes`               | TEXT | Yes      | Max 1000 chars.                                                                      |
| `missing_information` | TEXT | Yes      | Max 2000 chars.                                                                      |
| `inaccuracies`        | TEXT | Yes      | Max 2000 chars.                                                                      |
| `impact_on_work`      | TEXT | Yes      | Max 2000 chars.                                                                      |
| `change_summary`      | TEXT | No       | Max 1000 chars.                                                                      |
| `artifact_path`       | TEXT | No       | Path traversal CHECK: no `../` prefix, no `/` prefix (SEC-6).                        |
| `file_path`           | TEXT | No       | Restricted to `.github/instructions/*` or `.github/copilot-instructions.md` (SEC-6). |

---

## §2 anvil_checks INSERT Template

Used by: Orchestrator, Implementer, Verifier, Adversarial Reviewer.

```sql
-- ┌─────────────────────────────────────────────────────────────┐
-- │ anvil_checks INSERT — Verification Evidence                 │
-- │ Quoting: strings = '{sql_escape(val)}', absent = NULL,     │
-- │          integers = unquoted numeric                        │
-- └─────────────────────────────────────────────────────────────┘
INSERT INTO anvil_checks (
  run_id, task_id, phase, check_name, tool, command,
  exit_code, output_snippet, passed, verdict, severity, round, instance
) VALUES (
  '{run_id}',                     -- TEXT NOT NULL
  '{task_id}',                    -- TEXT (nullable: use NULL if absent)
  '{phase}',                      -- TEXT NOT NULL: 'baseline' | 'after' | 'review'
  '{check_name}',                 -- TEXT NOT NULL
  '{sql_escape(tool)}',           -- TEXT (nullable: NULL if absent)
  '{sql_escape(command)}',        -- TEXT (nullable: NULL if absent)
  {exit_code},                    -- INTEGER (nullable: NULL if absent)
  '{sql_escape(output_snippet)}', -- TEXT (nullable: NULL if absent, max 500 chars)
  {passed},                       -- INTEGER NOT NULL: 0 or 1
  '{verdict}',                    -- TEXT (nullable: NULL if absent): 'approve'|'needs_revision'|'blocker'
  '{severity}',                   -- TEXT (nullable: NULL if absent): 'Blocker'|'Critical'|'Major'|'Minor'
  {round},                        -- INTEGER DEFAULT 1
  '{instance}'                    -- TEXT (nullable: NULL if absent)
);
```

**Shell execution pattern (printf pipe):**

```bash
printf '%s' "INSERT INTO anvil_checks (run_id, task_id, phase, check_name, tool, command, exit_code, output_snippet, passed, verdict, severity, round, instance) VALUES ('2026-02-27T14:30:00Z', 'TASK-003', 'baseline', 'baseline-ide-diagnostics', 'get_errors', NULL, NULL, 'No errors found', 1, NULL, NULL, 0, NULL);" | sqlite3 verification-ledger.db
```

**Example with nullable fields as NULL:**

```sql
INSERT INTO anvil_checks (
  run_id, task_id, phase, check_name, tool, command,
  exit_code, output_snippet, passed, verdict, severity, round, instance
) VALUES (
  '2026-02-27T14:30:00Z', 'TASK-003', 'baseline', 'baseline-build',
  'dotnet build', 'dotnet build --no-restore', 0,
  'Build succeeded. 0 Warning(s). 0 Error(s).', 1,
  NULL, NULL, 0, NULL
);
```

---

## §3 pipeline_telemetry INSERT Template

Used by: **Orchestrator only** — inserted after each agent dispatch completes (Decision D-4).

```sql
-- ┌─────────────────────────────────────────────────────────────┐
-- │ pipeline_telemetry INSERT — Execution Tracking              │
-- │ Orchestrator-only. One INSERT per agent dispatch.           │
-- └─────────────────────────────────────────────────────────────┘
INSERT INTO pipeline_telemetry (
  run_id, step, agent, instance, started_at,
  completed_at, status, dispatch_count, retry_count, notes
) VALUES (
  '{run_id}',                  -- TEXT NOT NULL
  '{step}',                    -- TEXT NOT NULL (e.g., 'step-1', 'step-5')
  '{agent}',                   -- TEXT NOT NULL (e.g., 'implementer')
  '{instance}',                -- TEXT (nullable: NULL if single instance)
  '{started_at}',              -- TEXT NOT NULL (ISO 8601)
  '{completed_at}',            -- TEXT (nullable: NULL if still running)
  '{status}',                  -- TEXT (nullable): 'DONE'|'NEEDS_REVISION'|'ERROR'|'TIMEOUT'
  {dispatch_count},            -- INTEGER DEFAULT 1
  {retry_count},               -- INTEGER DEFAULT 0
  '{sql_escape(notes)}'        -- TEXT (nullable: NULL if absent, max 1000 chars)
);
```

**Shell execution pattern:**

```bash
printf '%s' "INSERT INTO pipeline_telemetry (run_id, step, agent, instance, started_at, completed_at, status, dispatch_count, retry_count, notes) VALUES ('2026-02-27T14:30:00Z', 'step-5', 'implementer', 'implementer-TASK-003', '2026-02-27T14:30:00Z', '2026-02-27T14:35:22Z', 'DONE', 1, 0, NULL);" | sqlite3 verification-ledger.db
```

---

## §4 artifact_evaluations INSERT Template

Used by: **Phase 1 agents** — Implementer, Verifier, Adversarial Reviewer.

```sql
-- ┌─────────────────────────────────────────────────────────────┐
-- │ artifact_evaluations INSERT — Quality Feedback              │
-- │ CHECK: usefulness_score/clarity_score BETWEEN 1 AND 10     │
-- │ CHECK: artifact_path NOT LIKE '../%' AND NOT LIKE '/%'     │
-- └─────────────────────────────────────────────────────────────┘
INSERT INTO artifact_evaluations (
  run_id, evaluator_agent, evaluator_instance,
  artifact_path, usefulness_score, clarity_score,
  missing_information, inaccuracies, impact_on_work
) VALUES (
  '{run_id}',                              -- TEXT NOT NULL
  '{evaluator_agent}',                     -- TEXT NOT NULL
  '{evaluator_instance}',                  -- TEXT (nullable: NULL if absent)
  '{artifact_path}',                       -- TEXT NOT NULL (no ../ or / prefix!)
  {usefulness_score},                      -- INTEGER NOT NULL: 1-10
  {clarity_score},                         -- INTEGER NOT NULL: 1-10
  '{sql_escape(missing_information)}',     -- TEXT (nullable: NULL if absent, max 2000 chars)
  '{sql_escape(inaccuracies)}',            -- TEXT (nullable: NULL if absent, max 2000 chars)
  '{sql_escape(impact_on_work)}'           -- TEXT (nullable: NULL if absent, max 2000 chars)
);
```

**Path validation (SEC-6):** The `artifact_path` column has a CHECK constraint preventing path traversal:

- `artifact_path NOT LIKE '../%'` — blocks relative parent traversal
- `artifact_path NOT LIKE '/%'` — blocks absolute paths

Always use workspace-relative paths (e.g., `docs/feature/my-feature/design-output.yaml`).

**Shell execution pattern:**

```bash
printf '%s' "INSERT INTO artifact_evaluations (run_id, evaluator_agent, evaluator_instance, artifact_path, usefulness_score, clarity_score, missing_information, inaccuracies, impact_on_work) VALUES ('2026-02-27T14:30:00Z', 'implementer', 'implementer-TASK-003', 'tasks/TASK-003.yaml', 8, 9, NULL, NULL, 'Clear task definition, sufficient context provided');" | sqlite3 verification-ledger.db
```

---

## §5 instruction_updates INSERT Template

Used by: **Knowledge Agent only**.

```sql
-- ┌─────────────────────────────────────────────────────────────┐
-- │ instruction_updates INSERT — Governed Update Audit Trail    │
-- │ CHECK: file_path LIKE '.github/instructions/%'             │
-- │    OR  file_path = '.github/copilot-instructions.md'       │
-- └─────────────────────────────────────────────────────────────┘
INSERT INTO instruction_updates (
  run_id, agent, file_path, change_type, change_summary, applied
) VALUES (
  '{run_id}',                        -- TEXT NOT NULL
  'knowledge-agent',                 -- TEXT NOT NULL (always 'knowledge-agent')
  '{file_path}',                     -- TEXT NOT NULL (must match CHECK constraint!)
  '{change_type}',                   -- TEXT NOT NULL: 'create'|'append'|'modify'|'delete'
  '{sql_escape(change_summary)}',    -- TEXT NOT NULL (max 1000 chars)
  {applied}                          -- INTEGER NOT NULL: 0 or 1
);
```

**file_path validation (SEC-6):** The CHECK constraint restricts `file_path` to:

- `.github/instructions/*` — agent-specific instruction files
- `.github/copilot-instructions.md` — global copilot instructions

Any other path value will cause a CHECK constraint failure. This prevents the Knowledge Agent from recording (and later applying) modifications to files outside the governed instruction scope.

**Shell execution pattern:**

```bash
printf '%s' "INSERT INTO instruction_updates (run_id, agent, file_path, change_type, change_summary, applied) VALUES ('2026-02-27T14:30:00Z', 'knowledge-agent', '.github/instructions/pipeline-conventions.md', 'append', 'Added retry pattern guidance based on pipeline run analysis', 0);" | sqlite3 verification-ledger.db
```

---

## §6 Evidence Gate Queries

> **CRITICAL:** All queries include `task_id` filter to distinguish Step 3b (design review) from Step 7 (code review) records (CORR-1). Check names are scope-qualified: `review-{scope}-security`, `review-{scope}-architecture`, `review-{scope}-correctness`. Replace `{scope}` with `design` or `code` as appropriate.

### EG-1: Baseline Exists (per task)

Verifies that baseline records were captured before implementation.

```sql
-- EG-1: Baseline exists for task
-- Expected: ≥1 (at least one baseline check recorded)
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}'
  AND task_id = '{task_id}'
  AND phase = 'baseline';
```

### EG-2: Verification Sufficient (per task)

Verifies post-implementation checks passed. Threshold: ≥2 for Standard risk, ≥3 for Large risk tasks.

```sql
-- EG-2: Verification sufficient for task
-- Expected: ≥2 (Standard) or ≥3 (Large)
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}'
  AND task_id = '{task_id}'
  AND phase = 'after'
  AND passed = 1;
```

### EG-3: Review Completion (all reviewers submitted)

Verifies all 3 reviewers have submitted evidence for the given scope and round.

```sql
-- EG-3: All reviewers submitted
-- Expected: 3 (one per perspective)
SELECT COUNT(DISTINCT instance) FROM anvil_checks
WHERE run_id = '{run_id}'
  AND task_id = '{task_id}'
  AND phase = 'review'
  AND round = {round};
```

### EG-4: Blocker Check (per scope)

Verifies zero blocker-severity findings in the review round.

```sql
-- EG-4: Zero blockers
-- Expected: 0
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}'
  AND task_id = '{task_id}'
  AND phase = 'review'
  AND round = {round}
  AND verdict = 'blocker';
```

### EG-5: Category Coverage (each reviewer × 3 categories)

Verifies each reviewer submitted findings for all 3 categories (security, architecture, correctness).

```sql
-- EG-5: All-category coverage per reviewer
-- Expected: 3 rows (each with cats = 3)
SELECT instance, COUNT(DISTINCT check_name) AS cats
FROM anvil_checks
WHERE run_id = '{run_id}'
  AND task_id = '{task_id}'
  AND phase = 'review'
  AND round = {round}
  AND check_name IN (
    'review-{scope}-security',
    'review-{scope}-architecture',
    'review-{scope}-correctness'
  )
GROUP BY instance
HAVING cats = 3;
```

### EG-6: Majority Approval (scope-qualified, handles multi-INSERT)

> **Addresses CORR-2:** A reviewer is "fully approving" only if ALL their category verdicts are `approve`. This subquery prevents false positives from mixed verdicts (e.g., one `approve` + one `needs_revision` would NOT count as approving under Decision D-8's 3-INSERT-per-reviewer pattern).

```sql
-- EG-6: Majority fully-approving reviewers
-- Expected: ≥2 of 3
-- A reviewer approves only if ALL 3 category verdicts are 'approve'.
SELECT COUNT(*) FROM (
  SELECT instance
  FROM anvil_checks
  WHERE run_id = '{run_id}'
    AND task_id = '{task_id}'
    AND phase = 'review'
    AND round = {round}
    AND check_name IN (
      'review-{scope}-security',
      'review-{scope}-architecture',
      'review-{scope}-correctness'
    )
  GROUP BY instance
  HAVING COUNT(CASE WHEN verdict != 'approve' THEN 1 END) = 0
);
```

**How EG-6 works with D-8 (3 INSERTs per reviewer):**

1. Each reviewer INSERTs 3 records — one per category (`review-{scope}-security`, `review-{scope}-architecture`, `review-{scope}-correctness`)
2. The subquery groups by `instance` (reviewer) and checks that NONE of the 3 verdicts are non-approve
3. `HAVING COUNT(CASE WHEN verdict != 'approve' THEN 1 END) = 0` — zero non-approve verdicts means full approval
4. Outer `COUNT(*)` counts how many reviewers fully approved — must be ≥2 for majority

---

## §7 Post-Mortem Analysis Queries

Used by: **Knowledge Agent** for pipeline analysis and post-mortem generation.

### Telemetry Summary

```sql
-- Per-step timing summary
SELECT step, agent, instance, status,
  (julianday(completed_at) - julianday(started_at)) * 86400 AS duration_s
FROM pipeline_telemetry
WHERE run_id = '{run_id}'
ORDER BY started_at;
```

### Slowest Steps

```sql
-- Top 3 slowest steps
SELECT step,
  SUM((julianday(completed_at) - julianday(started_at)) * 86400) AS total_s
FROM pipeline_telemetry
WHERE run_id = '{run_id}'
GROUP BY step
ORDER BY total_s DESC
LIMIT 3;
```

### Retry Analysis

```sql
-- Agents that required retries
SELECT agent, instance, retry_count
FROM pipeline_telemetry
WHERE run_id = '{run_id}'
  AND retry_count > 0;

-- Total retry count for the run
SELECT SUM(retry_count) AS total_retries
FROM pipeline_telemetry
WHERE run_id = '{run_id}';
```

### Evaluation Summary

```sql
-- Mean evaluation scores per artifact type (by evaluator agent)
SELECT evaluator_agent, COUNT(*) AS eval_count,
  AVG(usefulness_score) AS avg_useful,
  AVG(clarity_score) AS avg_clarity
FROM artifact_evaluations
WHERE run_id = '{run_id}'
GROUP BY evaluator_agent;
```

### Worst-Rated Artifacts

```sql
-- Bottom 5 artifacts by usefulness
SELECT artifact_path,
  AVG(usefulness_score) AS avg_useful,
  AVG(clarity_score) AS avg_clarity
FROM artifact_evaluations
WHERE run_id = '{run_id}'
GROUP BY artifact_path
ORDER BY avg_useful ASC
LIMIT 5;
```

### Missing Information Report

```sql
-- Common missing information across evaluations
SELECT artifact_path, missing_information
FROM artifact_evaluations
WHERE run_id = '{run_id}'
  AND missing_information IS NOT NULL
  AND missing_information != '';
```

### Instruction Update History

```sql
-- All instruction updates for the run
SELECT * FROM instruction_updates
WHERE run_id = '{run_id}'
ORDER BY ts;
```

---

## §8 Instruction Update Audit Queries

Used by: Orchestrator and Knowledge Agent for reviewing instruction file changes.

### Pending Updates (Not Yet Applied)

```sql
-- Updates proposed but not yet applied
SELECT id, file_path, change_type, change_summary
FROM instruction_updates
WHERE run_id = '{run_id}'
  AND applied = 0
ORDER BY ts;
```

### Update Frequency by File

```sql
-- Which instruction files are updated most frequently (cross-run)
SELECT file_path, COUNT(*) AS update_count
FROM instruction_updates
GROUP BY file_path
ORDER BY update_count DESC;
```

### Applied vs Pending Ratio

```sql
-- Applied vs pending updates for the current run
SELECT
  SUM(CASE WHEN applied = 1 THEN 1 ELSE 0 END) AS applied_count,
  SUM(CASE WHEN applied = 0 THEN 1 ELSE 0 END) AS pending_count
FROM instruction_updates
WHERE run_id = '{run_id}';
```

### Full Audit Trail for a Specific File

```sql
-- Complete update history for a specific instruction file (cross-run)
SELECT run_id, change_type, change_summary, applied, ts
FROM instruction_updates
WHERE file_path = '{file_path}'
ORDER BY ts DESC;
```

---

## §9 Schema Evolution Strategy

> **Addresses ARCH-2.** SQLite does not support ALTER TABLE MODIFY COLUMN or ALTER TABLE DROP COLUMN (pre-3.35.0). All schema changes must follow these patterns.

### Safe Additive Changes (No Migration Needed)

```sql
-- Adding a new column is always safe.
-- Existing queries ignore unknown columns — backward compatible.
-- New columns MUST have DEFAULT values or be nullable.
ALTER TABLE anvil_checks ADD COLUMN new_field TEXT DEFAULT NULL;
```

Additive changes apply immediately and do not affect existing data or queries.

### Enum Expansion (Requires Table Recreation)

CHECK constraints cannot be altered in SQLite. To expand an enum (e.g., adding a new `phase` value):

```sql
-- Step 1: Create new table with updated CHECK constraints
CREATE TABLE anvil_checks_new (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  -- ... all existing columns ...
  phase          TEXT NOT NULL CHECK (phase IN ('baseline','after','review','rollback')),
  -- ... rest of columns ...
  ts             TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Step 2: Copy existing data
INSERT INTO anvil_checks_new SELECT * FROM anvil_checks;

-- Step 3: Drop old table
DROP TABLE anvil_checks;

-- Step 4: Rename new table
ALTER TABLE anvil_checks_new RENAME TO anvil_checks;

-- Step 5: Re-create indexes
CREATE INDEX IF NOT EXISTS idx_anvil_run        ON anvil_checks(run_id);
CREATE INDEX IF NOT EXISTS idx_anvil_task_phase ON anvil_checks(task_id, phase);
CREATE INDEX IF NOT EXISTS idx_anvil_run_round  ON anvil_checks(run_id, round);
```

> **IMPORTANT:** Table recreation MUST be done at **Step 0 init only**, never mid-pipeline. Existing rows with old enum values remain valid — the new CHECK is a superset.

### Version Tracking

Track schema versions with a `schema_meta` table:

```sql
CREATE TABLE IF NOT EXISTS schema_meta (
  table_name  TEXT PRIMARY KEY,
  version     INTEGER NOT NULL DEFAULT 1,
  migrated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Initialize version tracking for all tables
INSERT OR IGNORE INTO schema_meta (table_name, version) VALUES ('anvil_checks', 1);
INSERT OR IGNORE INTO schema_meta (table_name, version) VALUES ('pipeline_telemetry', 1);
INSERT OR IGNORE INTO schema_meta (table_name, version) VALUES ('artifact_evaluations', 1);
INSERT OR IGNORE INTO schema_meta (table_name, version) VALUES ('instruction_updates', 1);
```

### Migration Procedure

At Step 0, the Orchestrator checks schema versions and applies migrations:

```
1. Read current version: SELECT version FROM schema_meta WHERE table_name = '<table>';
2. If version < expected_version:
   a. Execute migration DDL (ALTER TABLE or table recreation)
   b. UPDATE schema_meta SET version = <new_version>, migrated_at = datetime('now')
      WHERE table_name = '<table>';
3. If schema_meta table doesn't exist: create it + create all tables (fresh install)
```

### Migration Ownership

- **Only the Orchestrator (Step 0)** may execute DDL changes (`CREATE`, `ALTER`, `DROP`).
- No other agent may run DDL statements.
- All migration scripts are documented in this section (§9) before deployment.
