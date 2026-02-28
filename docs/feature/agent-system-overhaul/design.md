# Agent System Overhaul — Technical Design

**Feature Slug:** `agent-system-overhaul`
**Design Agent Instance:** `designer-agent-system-overhaul`
**Date:** 2026-02-27
**Status:** Design Complete (Revision Round 2)

---

## 1. Title & Summary

**Agent System Overhaul** — A comprehensive refactoring of the 9-agent NewAgents pipeline addressing 16 functional requirements across 6 domains: broken feature fixes (parallel execution, multi-model review, telemetry), feature restoration (artifact evaluations, post-mortem analysis, instruction files), global rule enforcement (terminal-only testing, no-file-redirect), agent file size reduction via shared reference document extraction, review architecture restructure (3 perspectives × 3 categories), and expanded SQLite usage (4 tables consolidated in 1 DB).

### Design Goals

1. **Reduce agent file sizes** from 250–800 lines to 240–480 lines via 3-tier content extraction
2. **Fix broken features**: populate pipeline telemetry, replace broken multi-model review, resolve tool contradictions
3. **Restore missing capabilities**: artifact evaluation system, post-mortem analysis, instruction file infrastructure
4. **Restructure adversarial review**: 3 prompt-diverse perspectives × 3 categories = 9 independent review dimensions
5. **Consolidate SQL infrastructure**: 4 tables in single verification-ledger.db with full DDL
6. **Rationalize outputs**: YAML-only for machine artifacts, YAML+MD only for human-facing docs

---

## 2. Context & Inputs

### 2.1 Specification

- **spec-output.yaml**: 16 FRs, 7 NFRs, 6 decision points (DP-1 through DP-6), 10 edge cases, 7 risks
- **feature.md**: Human-readable specification with acceptance criteria and test scenarios

### 2.2 Research (52 findings)

| Research Output   | Findings | Key Insights                                                                     |
| ----------------- | -------- | -------------------------------------------------------------------------------- |
| architecture.yaml | 14       | 3 systems (Anvil/Forge/NewAgents), typed YAML evolution, SQL evidence gates      |
| impact.yaml       | 11       | Evaluation system absent, telemetry dead, memory removed, 16 agent files dropped |
| dependencies.yaml | 15       | 10-schema dependency graph, tool contradictions, file path inconsistencies       |
| patterns.yaml     | 12       | Agent bloat (251–800 lines), anti-drift anchors, dead telemetry DB               |

### 2.3 Decision Points Adopted from Spec

| DP   | Decision                                               | Rationale                                                          |
| ---- | ------------------------------------------------------ | ------------------------------------------------------------------ |
| DP-1 | **Option A**: Prompt perspective diversity             | VS Code ignores model params for custom agents                     |
| DP-2 | **Option B**: YAML-only for machine, YAML+MD for human | No downstream agent reads MD companions                            |
| DP-3 | **Option B**: SQLite-based evaluations                 | Queryable, aligned with REQ-11, no file proliferation              |
| DP-4 | **Option A**: No memory restoration                    | Typed YAML + SQL sufficient; evaluations fill feedback gap         |
| DP-5 | **Option C**: Tiered extraction                        | Global→copilot-instructions.md, shared→reference docs, role→inline |
| DP-6 | **Option C**: Per-reviewer verdict files with glob     | Preserves provenance, no aggregation step needed                   |

---

## 3. High-Level Architecture

### 3.1 Three-Tier Content Architecture

The design introduces a 3-tier content strategy based on DP-5:

```
Tier 1: .github/copilot-instructions.md  (auto-loaded by VS Code for ALL agents)
  └─ Global rules: terminal-only testing, no-file-redirect, retry policy, schema version,
     evidence-first, agent boundaries, bounded loops, Context7 conditional

Tier 2: .github/agents/*.md  (reference docs, read by agents that need them)
  ├─ global-operating-rules.md    — Detailed retry, error handling, routing matrix
  ├─ sql-templates.md             — All DDL + DML templates
  ├─ evaluation-schema.md         — Artifact evaluation system
  ├─ context7-integration.md      — Context7 MCP conditional usage
  ├─ review-perspectives.md       — 3 reviewer perspective definitions
  ├─ tool-access-matrix.md        — Per-agent tool access tables
  ├─ schemas.md                   — YAML schemas (existing, updated)
  ├─ dispatch-patterns.md         — Dispatch patterns (existing, updated)
  └─ severity-taxonomy.md         — Severity levels (existing, minor update)

Tier 3: .github/agents/*.agent.md  (inline in each agent file)
  └─ Role-specific workflow, I/O schema, self-verification, anti-drift anchor
```

**Why 3 tiers?** Tier 1 is auto-loaded by VS Code — every agent gets these rules without explicit reads. Tier 2 is opt-in — agents reference only the docs they need, keeping context budget focused. Tier 3 is what makes each agent unique — role-specific workflow that cannot be shared.

### 3.2 Component Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        ORCHESTRATOR                               │
│  Steps 0-9 | Dispatch | Evidence Gates | Telemetry INSERT         │
│  Reads: completion contracts (YAML) + anvil_checks (SQL)          │
│  Writes: pipeline_telemetry (SQL)                                 │
└──────┬───────────────────────────────────────────────────────────┘
       │ runSubagent dispatch
       ├──→ Researcher ×4 (Step 1)      → research/*.yaml
       ├──→ Spec (Step 2)               → spec-output.yaml + feature.md
       ├──→ Designer (Step 3)           → design-output.yaml + design.md
       ├──→ Adversarial Reviewer ×3     → review-verdicts/<scope>-<perspective>.yaml
       │    (Steps 3b, 7)                  review-findings/<scope>-<perspective>.md
       │                                   anvil_checks SQL INSERT ×3 per reviewer
       ├──→ Planner (Step 4)            → plan-output.yaml + plan.md + tasks/*.yaml
       ├──→ Implementer ×N (Step 5)     → implementation-reports/*.yaml + code changes
       │                                   anvil_checks SQL INSERT (baseline)
       │                                   artifact_evaluations SQL INSERT
       ├──→ Verifier ×N (Step 6)        → verification-reports/*.yaml
       │                                   anvil_checks SQL INSERT (after)
       │                                   artifact_evaluations SQL INSERT
       └──→ Knowledge Agent (Step 8)    → knowledge-output.yaml + evidence-bundle.md
                                           + decisions.yaml
                                           instruction_updates SQL INSERT
```

### 3.3 Data Flow

```
                    YAML Schemas (Linear)
Researcher → Spec → Designer → Planner → Implementer → Verifier
     ↓          ↓        ↓         ↓          ↓            ↓
  research/  spec-out  design-out plan-out  impl-reports  verif-reports
   *.yaml    .yaml     .yaml     .yaml      *.yaml        *.yaml

                    SQL Evidence (Shared)
              ┌─────────────────────────────────────┐
              │      verification-ledger.db          │
              │  ┌──────────────────────────────┐   │
              │  │ anvil_checks (15 cols)       │←─ Implementer, Verifier,
              │  │                              │   Adversarial Reviewer
              │  └──────────────────────────────┘   │
              │  ┌──────────────────────────────┐   │
              │  │ pipeline_telemetry (12 cols)  │←─ Orchestrator (after each dispatch)
              │  └──────────────────────────────┘   │
              │  ┌──────────────────────────────┐   │
              │  │ artifact_evaluations (11 cols)│←─ Implementer, Verifier,
              │  │                              │   Adversarial Reviewer
              │  └──────────────────────────────┘   │
              │  ┌──────────────────────────────┐   │
              │  │ instruction_updates (8 cols)  │←─ Knowledge Agent
              │  └──────────────────────────────┘   │
              └─────────────────────────────────────┘
                     ↑ READ by: Orchestrator (gates), Knowledge Agent (post-mortem)
                     ↑ WRITE by: 5 agents (orchestrator, implementer, verifier, adversarial-reviewer, knowledge-agent)
```

### 3.4 Tool Access Changes Summary

| Agent           | Change                                                                              | Reason                                                                        |
| --------------- | ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Verifier        | **GAINS** `create_file` (scoped: `verification-reports/*.yaml`)                     | Resolves FR-11 tool contradiction                                             |
| Researcher      | `create_file` **clarified** (scoped: `research/*.yaml` only, no MD)                 | Resolves ambiguous exception + DP-2                                           |
| Knowledge Agent | **GAINS** `run_in_terminal` (scoped: SELECT + INSERT on `instruction_updates` only) | Enables FR-6 telemetry + FR-5 evaluation consumption + FR-15 governed updates |
| All agents      | Tool tables **extracted** to tool-access-matrix.md                                  | FR-10 file size reduction                                                     |

---

## 4. File Inventory

### 4.1 Files Created (8)

| #   | Path                                                 | Est. Lines | Purpose                                     | FRs                             |
| --- | ---------------------------------------------------- | ---------- | ------------------------------------------- | ------------------------------- |
| 1   | `.github/copilot-instructions.md`                    | 55         | Global rules (Tier 1, auto-loaded)          | FR-7, FR-8, FR-10, FR-15        |
| 2   | `.github/instructions/pipeline-conventions.md`       | 45         | Domain instruction (Knowledge Agent target) | FR-15                           |
| 3   | `NewAgents/.github/agents/global-operating-rules.md` | 130        | Shared operating rules (Tier 2)             | FR-10, FR-13                    |
| 4   | `NewAgents/.github/agents/sql-templates.md`          | 160        | SQL DDL + DML templates (Tier 2)            | FR-5, FR-6, FR-10, FR-13, FR-14 |
| 5   | `NewAgents/.github/agents/evaluation-schema.md`      | 85         | Artifact evaluation system (Tier 2)         | FR-5                            |
| 6   | `NewAgents/.github/agents/context7-integration.md`   | 45         | Context7 conditional usage (Tier 2)         | FR-9                            |
| 7   | `NewAgents/.github/agents/review-perspectives.md`    | 130        | 3 reviewer perspectives (Tier 2)            | FR-2, FR-3                      |
| 8   | `NewAgents/.github/agents/tool-access-matrix.md`     | 110        | Per-agent tool access (Tier 2)              | FR-10, FR-11                    |

### 4.2 Files Modified (12)

| #   | Path                            | Current → Target Lines | Key Changes                                                          | FRs                                               |
| --- | ------------------------------- | ---------------------- | -------------------------------------------------------------------- | ------------------------------------------------- |
| 9   | `orchestrator.agent.md`         | 800 → 550              | Extract SQL/tools, add telemetry INSERT, perspective dispatch        | FR-1, FR-6, FR-10, FR-12, FR-13, FR-14, FR-16     |
| 10  | `implementer.agent.md`          | 519 → 340              | Extract shared content, add evaluations + Context7 ref               | FR-4, FR-5, FR-7, FR-8, FR-9, FR-10, FR-16        |
| 11  | `verifier.agent.md`             | 521 → 340              | Extract SQL/tools, resolve contradiction, add evaluations            | FR-4, FR-5, FR-7, FR-10, FR-11, FR-16             |
| 12  | `knowledge-agent.agent.md`      | 509 → 345              | Grant SQL access, add post-mortem/eval queries, instruction tracking | FR-4, FR-5, FR-6, FR-10, FR-14, FR-15, FR-16      |
| 13  | `adversarial-reviewer.agent.md` | 372 → 335              | Perspectives, all-category coverage, verdict paths                   | FR-2, FR-3, FR-4, FR-5, FR-8, FR-10, FR-12, FR-16 |
| 14  | `planner.agent.md`              | 376 → 340              | Extract shared content, add frontmatter                              | FR-4, FR-10, FR-16                                |
| 15  | `spec.agent.md`                 | 337 → 320              | Extract shared content, add frontmatter                              | FR-4, FR-10, FR-16                                |
| 16  | `designer.agent.md`             | 275 → 265              | Glob verdict reading, shared doc refs                                | FR-4, FR-10, FR-12, FR-16                         |
| 17  | `researcher.agent.md`           | 251 → 240              | Drop MD output, clarify create_file scope, Context7                  | FR-4, FR-9, FR-10, FR-11, FR-16                   |
| 18  | `schemas.md`                    | 1284 → 1450            | New tables, routing matrix, format classification                    | FR-4, FR-5, FR-12, FR-13, FR-14                   |
| 19  | `dispatch-patterns.md`          | ~100 → 120             | Perspective dispatch, FR-1 findings, evidence gates                  | FR-1, FR-2, FR-3                                  |
| 20  | `severity-taxonomy.md`          | ~50 → 55               | Add YAML frontmatter                                                 | FR-16                                             |

### 4.3 Line Count Summary

| Category                  | Before    | After     | Delta                      |
| ------------------------- | --------- | --------- | -------------------------- |
| 9 agent files total       | 3,930     | 2,775     | **−1,155** (29% reduction) |
| 3 existing reference docs | ~1,434    | ~1,625    | +191                       |
| 6 new reference docs      | 0         | 760       | +760                       |
| 2 instruction files       | 0         | 100       | +100                       |
| **Net**                   | **5,364** | **5,260** | **−104**                   |

Agent files shrink 31% while total system documentation grows by only +586 lines from new shared docs. The net is nearly neutral because the extraction avoids duplication — content that was repeated across 5+ agents now exists once.

---

## 5. Shared Reference Documents Design

### 5.1 `.github/copilot-instructions.md` (Tier 1 — Global)

This file is auto-loaded by VS Code for ALL agents, including the orchestrator. It contains rules that are universally applicable and must never be violated.

**Content Outline:**

```markdown
# Global Operating Rules

These rules apply to ALL agents in the pipeline. They are non-negotiable.

## §1 Terminal-Only Testing

All test execution MUST use `run_in_terminal` with CLI commands (dotnet test, npm test,
pytest, cargo test, go test, etc.). No VS Code Testing API, runTests, or IDE test runners.
`get_errors` is permitted for compile-time diagnostics only — not test execution.

## §2 No File-Redirect

NEVER redirect terminal command output to a file (command > output.txt, command | tee file,
command 2>&1 > log.txt). Read all output directly from `get_terminal_output` or
`run_in_terminal` return value.

## §3 Retry Policy

Transient errors (network timeout, tool unavailable, SQLITE_BUSY) → retry up to 2x with
backoff. Deterministic errors (schema violation, missing required input) → never retry.
See global-operating-rules.md §1-§2 for details.

## §4 Schema Version

All YAML outputs MUST include schema_version: "1.0" in the agent_output header.
Self-verify this before returning.

## §5 Evidence-First Principle

Record SQL evidence (INSERT into anvil_checks) BEFORE claiming verification passed.
If the INSERT didn't happen, the verification didn't happen.

## §6 Agent Boundaries

NEVER modify another agent's definition files (.agent.md) at runtime.
NEVER create circular dependencies between agents.

## §7 Bounded Loops

All feedback loops MUST have hard iteration limits. Design revision: max 1.
Implementation loop: max 3. Review cycling: max 2. No unbounded cycles.

## §8 Context7 (Conditional)

If Context7 MCP tools are available, use them for library/framework documentation
lookup BEFORE guessing at API usage. If unavailable, skip with no error.
See context7-integration.md for the two-step pattern.
```

**Estimated lines:** 55

### 5.2 `global-operating-rules.md` (Tier 2)

Detailed operating rules referenced by agents via section pointers. Contains content currently duplicated across 5+ agent files.

**Key Sections:**

| Section                        | Content                                                        | Currently In                        |
| ------------------------------ | -------------------------------------------------------------- | ----------------------------------- |
| §1 Two-Tiered Retry            | Agent-internal 2x transient + orchestrator 1x per invocation   | All 9 agents                        |
| §2 Error Categories            | Transient vs deterministic classification with examples        | 7 agents                            |
| §3 SQLITE_BUSY                 | WAL mode, busy_timeout=5000, retry 3x exponential (1s, 2s, 4s) | Orchestrator, Implementer, Verifier |
| §4 run_id Protocol             | Generation, propagation, SQL filtering                         | Orchestrator, Verifier, Reviewer    |
| §5 Routing Matrix              | Reference → schemas.md (canonical per AC-13.1)                 | schemas.md (new)                    |
| §6 Self-Verification Checklist | 8 common items across all agents                               | All 9 agents                        |
| §7 State Recovery EC-5         | Scan, read completion blocks, resume                           | Orchestrator                        |
| §8 Output Naming               | Convention: `docs/feature/<slug>/` prefix                      | Multiple agents                     |
| §9 Governed Updates            | Interactive=approval, Autonomous=log-only, Safety filter       | Knowledge Agent                     |

**§9 Safety Constraint Filter — Formal Definition:**

The Safety Constraint Filter governs all instruction file updates by the Knowledge Agent. It is defined as follows:

1. **Immutable Rules (NEVER removable or weakened):**
   - Terminal-only testing (copilot-instructions.md §1)
   - No-file-redirect (copilot-instructions.md §2)
   - Agent boundary integrity (copilot-instructions.md §6)
   - Bounded loop invariants (copilot-instructions.md §7)
   - Evidence-first principle (copilot-instructions.md §5)

2. **Protected Phrases (cannot be deleted from instruction files):**
   - "MUST use run_in_terminal", "NEVER redirect", "NEVER modify another agent", "hard iteration limits", "INSERT before reporting"

3. **Pattern-Matching Rejection Criteria:**
   - Any update that removes text matching a protected phrase → REJECTED
   - Any update that adds `ALLOWED` to a previously `RESTRICTED` tool → REJECTED
   - Any update that increases loop/retry limits beyond spec-defined bounds → REJECTED
   - Any update that removes CHECK constraints or validation rules → REJECTED

4. **Evaluator Separation:**
   - In **autonomous mode**: Knowledge Agent proposes changes, logs them to `instruction_updates` with `applied=0`. Changes are NEVER auto-applied to `copilot-instructions.md`.
   - In **interactive mode**: Knowledge Agent proposes changes, presents to user for approval. Only user-approved changes are applied (`applied=1`).
   - `.github/copilot-instructions.md` is **immutable at runtime** — only modifiable via feature implementation PRs, never by any agent.
   - `.github/instructions/pipeline-conventions.md` is **governed-mutable** — updatable by Knowledge Agent with safety filters, changes logged to `instruction_updates` table.

**§5 Completion Contract Routing Matrix:**

> **Canonical location: schemas.md** (per AC-13.1). This section references the routing matrix in schemas.md rather than duplicating it. See `schemas.md §Routing Matrix` for the full agent × completion status table.

Summary: Only Planner and Verifier support `NEEDS_REVISION`. All others either succeed or fail — revision routing is handled by the Orchestrator's decision table.

### 5.3 `sql-templates.md` (Tier 2)

All SQL DDL and DML templates consolidated. Agents reference specific sections instead of inlining SQL.

**Key Sections:**

**§0 SQL Sanitization Functions (Mandatory):**

All agents that write SQL MUST apply the following escaping before interpolating any text value into an INSERT statement:

```
sql_escape(value):
  1. Replace all single quotes: ' → ''
  2. Strip null bytes: \0 → (removed)
  3. Truncate to field-specific max length (see §1 CHECK constraints)
  Result: safe for SQL string interpolation within single quotes
```

**Shell Injection Prevention:** SQL commands executed via `run_in_terminal` pass through the system shell. To prevent shell metacharacter interpretation of agent-generated values:

```
Escaping Order (mandatory):
  1. Apply sql_escape() to all text values
  2. Pipe SQL via stdin to avoid shell interpretation of values:

     Preferred pattern (stdin piping):
       echo "INSERT INTO table ..." | sqlite3 verification-ledger.db

     This avoids passing SQL as a shell argument where $(), backticks,
     pipes, and semicolons in values would be interpreted by the shell.

  If stdin piping is not feasible, additionally shell-escape values:
     PowerShell: Use single-quoted strings and -replace for internal quotes
     bash/sh: Use printf '%s' to avoid interpretation
```

**Self-Verification for SQL-Writing Agents:** Before executing any SQL INSERT via `run_in_terminal`, verify:

- [ ] All text values have been sql_escape()'d (single quotes doubled, null bytes stripped)
- [ ] SQL is piped via stdin (`echo "..." | sqlite3`) or values are additionally shell-escaped
- [ ] Text values do not exceed field-specific LENGTH limits

**§1 Database Initialization (Step 0):**

```sql
-- Single database: verification-ledger.db
-- Orchestrator executes at Step 0 via run_in_terminal

PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;
PRAGMA integrity_check; -- Validates existing DB integrity at pipeline start

-- Table 1: anvil_checks (verification evidence)
CREATE TABLE IF NOT EXISTS anvil_checks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  task_id TEXT,
  phase TEXT NOT NULL CHECK (phase IN ('baseline','after','review')),
  check_name TEXT NOT NULL,
  tool TEXT,
  command TEXT,
  exit_code INTEGER,
  output_snippet TEXT CHECK (LENGTH(output_snippet) <= 500),
  passed INTEGER NOT NULL CHECK (passed IN (0, 1)),
  verdict TEXT CHECK (verdict IN ('approve','needs_revision','blocker')),
  severity TEXT CHECK (severity IN ('Blocker','Critical','Major','Minor')),
  round INTEGER DEFAULT 1,
  instance TEXT,
  ts TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_anvil_run ON anvil_checks(run_id);
CREATE INDEX IF NOT EXISTS idx_anvil_task_phase ON anvil_checks(task_id, phase);
CREATE INDEX IF NOT EXISTS idx_anvil_run_round ON anvil_checks(run_id, round);

-- Table 2: pipeline_telemetry (execution tracking)
CREATE TABLE IF NOT EXISTS pipeline_telemetry (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  step TEXT NOT NULL,
  agent TEXT NOT NULL,
  instance TEXT,
  started_at TEXT NOT NULL,
  completed_at TEXT,
  status TEXT CHECK (status IN ('DONE','NEEDS_REVISION','ERROR','TIMEOUT')),
  dispatch_count INTEGER DEFAULT 1,
  retry_count INTEGER DEFAULT 0,
  notes TEXT CHECK (LENGTH(notes) <= 1000),
  ts TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_telemetry_run ON pipeline_telemetry(run_id);
CREATE INDEX IF NOT EXISTS idx_telemetry_step ON pipeline_telemetry(run_id, step);

-- Table 3: artifact_evaluations (quality feedback)
CREATE TABLE IF NOT EXISTS artifact_evaluations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  evaluator_agent TEXT NOT NULL,
  evaluator_instance TEXT,
  artifact_path TEXT NOT NULL CHECK (artifact_path NOT LIKE '../%' AND artifact_path NOT LIKE '/%'),
  usefulness_score INTEGER NOT NULL CHECK (usefulness_score BETWEEN 1 AND 10),
  clarity_score INTEGER NOT NULL CHECK (clarity_score BETWEEN 1 AND 10),
  missing_information TEXT CHECK (LENGTH(missing_information) <= 2000),
  inaccuracies TEXT CHECK (LENGTH(inaccuracies) <= 2000),
  impact_on_work TEXT CHECK (LENGTH(impact_on_work) <= 2000),
  ts TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_eval_run ON artifact_evaluations(run_id);
CREATE INDEX IF NOT EXISTS idx_eval_agent ON artifact_evaluations(evaluator_agent);
CREATE INDEX IF NOT EXISTS idx_eval_artifact ON artifact_evaluations(artifact_path);

-- Table 4: instruction_updates (governed update audit trail)
CREATE TABLE IF NOT EXISTS instruction_updates (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  agent TEXT NOT NULL,
  file_path TEXT NOT NULL CHECK (
    file_path LIKE '.github/instructions/%'
    OR file_path = '.github/copilot-instructions.md'
  ),
  change_type TEXT NOT NULL CHECK (change_type IN ('create','append','modify','delete')),
  change_summary TEXT NOT NULL CHECK (LENGTH(change_summary) <= 1000),
  applied INTEGER NOT NULL DEFAULT 0 CHECK (applied IN (0, 1)),
  ts TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_instr_run ON instruction_updates(run_id);
```

**§2 anvil_checks INSERT Template:**

```sql
-- Quoting convention: String fields use '{sql_escape(value)}'. Nullable string fields
-- use NULL (unquoted) when absent, or '{sql_escape(value)}' when present.
-- Integer/boolean fields are unquoted numeric values.
INSERT INTO anvil_checks (run_id, task_id, phase, check_name, tool, command,
  exit_code, output_snippet, passed, verdict, severity, round, instance)
VALUES ('{run_id}', '{task_id}', '{phase}', '{check_name}', '{sql_escape(tool)}',
  '{sql_escape(command)}', {exit_code}, '{sql_escape(output_snippet)}', {passed},
  '{verdict}', '{severity}', {round}, '{instance}');
-- For nullable fields (verdict, severity, command, output_snippet): substitute
-- NULL (unquoted, no quotes) when the value is absent.
-- Example with NULLs: ... , NULL, NULL, {passed}, NULL, NULL, {round}, ...
```

**§3 pipeline_telemetry INSERT Template (Orchestrator-only):**

```sql
INSERT INTO pipeline_telemetry (run_id, step, agent, instance, started_at,
  completed_at, status, dispatch_count, retry_count, notes)
VALUES ('{run_id}', '{step}', '{agent}', '{instance}', '{started_at}',
  '{completed_at}', '{status}', {dispatch_count}, {retry_count},
  '{sql_escape(notes)}');
-- notes is nullable: use NULL (unquoted) when absent
```

**§4 artifact_evaluations INSERT Template (Phase 1 agents):**

```sql
INSERT INTO artifact_evaluations (run_id, evaluator_agent, evaluator_instance,
  artifact_path, usefulness_score, clarity_score, missing_information,
  inaccuracies, impact_on_work)
VALUES ('{run_id}', '{evaluator_agent}', '{evaluator_instance}',
  '{artifact_path}', {usefulness_score}, {clarity_score},
  '{sql_escape(missing_information)}', '{sql_escape(inaccuracies)}',
  '{sql_escape(impact_on_work)}');
-- Free-text fields (missing_information, inaccuracies, impact_on_work) are nullable:
-- use NULL (unquoted) when absent. Truncate to 2000 chars if exceeding limit.
```

**§5 instruction_updates INSERT Template (Knowledge Agent-only):**

```sql
INSERT INTO instruction_updates (run_id, agent, file_path, change_type,
  change_summary, applied)
VALUES ('{run_id}', 'knowledge-agent', '{file_path}', '{change_type}',
  '{sql_escape(change_summary)}', {applied});
-- file_path MUST match CHECK constraint: .github/instructions/* or .github/copilot-instructions.md
-- change_summary truncated to 1000 chars
```

**§6 Evidence Gate SELECT Queries:**

```sql
-- NOTE: All review evidence gate queries use scope-qualified check_names to
-- distinguish Step 3b (design review) from Step 7 (code review) records.
-- Replace {scope} with 'design' or 'code' as appropriate per step.

-- EG-1: Baseline exists for task
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id='{run_id}' AND task_id='{task_id}' AND phase='baseline';

-- EG-2: Verification sufficient for task
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id='{run_id}' AND task_id='{task_id}' AND phase='after' AND passed=1;

-- EG-3: All reviewers submitted (expect 3) — filtered by task_id
SELECT COUNT(DISTINCT instance) FROM anvil_checks
  WHERE run_id='{run_id}' AND task_id='{task_id}' AND phase='review' AND round={round};

-- EG-4: All-category coverage per reviewer (expect 3 rows, each count=3)
-- Uses scope-qualified check_names: review-{scope}-security, etc.
SELECT instance, COUNT(DISTINCT check_name) AS cats FROM anvil_checks
  WHERE run_id='{run_id}' AND task_id='{task_id}' AND phase='review' AND round={round}
  AND check_name IN ('review-{scope}-security','review-{scope}-architecture','review-{scope}-correctness')
  GROUP BY instance HAVING cats = 3;

-- EG-5: Zero blockers — filtered by task_id
SELECT COUNT(*) FROM anvil_checks
  WHERE run_id='{run_id}' AND task_id='{task_id}' AND phase='review' AND round={round}
  AND verdict='blocker';

-- EG-6: Majority fully-approving reviewers (expect ≥2 of 3)
-- A reviewer is "fully approving" only if ALL their category verdicts are 'approve'.
-- This prevents false positives from mixed verdicts (e.g., approve+needs_revision).
SELECT COUNT(*) FROM (
  SELECT instance FROM anvil_checks
  WHERE run_id='{run_id}' AND task_id='{task_id}' AND phase='review' AND round={round}
    AND check_name IN ('review-{scope}-security','review-{scope}-architecture','review-{scope}-correctness')
  GROUP BY instance
  HAVING COUNT(CASE WHEN verdict != 'approve' THEN 1 END) = 0
);
```

**§7 Knowledge Agent Aggregation Queries:**

```sql
-- Telemetry summary
SELECT step, agent, instance, status,
  (julianday(completed_at) - julianday(started_at)) * 86400 AS duration_s
FROM pipeline_telemetry WHERE run_id='{run_id}' ORDER BY started_at;

-- Top 3 slowest steps
SELECT step, SUM((julianday(completed_at)-julianday(started_at))*86400) AS total_s
FROM pipeline_telemetry WHERE run_id='{run_id}' GROUP BY step ORDER BY total_s DESC LIMIT 3;

-- Agents needing retries
SELECT agent, instance, retry_count FROM pipeline_telemetry
  WHERE run_id='{run_id}' AND retry_count > 0;

-- Evaluation summary by agent
SELECT evaluator_agent, COUNT(*), AVG(usefulness_score) AS avg_useful,
  AVG(clarity_score) AS avg_clarity
FROM artifact_evaluations WHERE run_id='{run_id}' GROUP BY evaluator_agent;

-- Worst-rated artifacts
SELECT artifact_path, AVG(usefulness_score) AS avg_useful,
  AVG(clarity_score) AS avg_clarity
FROM artifact_evaluations WHERE run_id='{run_id}'
  GROUP BY artifact_path ORDER BY avg_useful ASC LIMIT 5;

-- Common missing information
SELECT missing_information FROM artifact_evaluations
  WHERE run_id='{run_id}' AND missing_information IS NOT NULL
  AND missing_information != '';

-- Instruction update history
SELECT * FROM instruction_updates WHERE run_id='{run_id}' ORDER BY ts;
```

**§8 SQLITE_BUSY Retry Pattern:**

Retry 3× with exponential backoff (1s, 2s, 4s). WAL mode and `busy_timeout=5000` prevent most conflicts. See global-operating-rules.md §3 for full pattern.

**§9 SQLite Schema Evolution Strategy:**

```
Additive Changes (safe, no migration needed):
  - ALTER TABLE <table> ADD COLUMN <name> <type> [DEFAULT <value>];
  - Existing queries ignore unknown columns — backward compatible.
  - New columns MUST have DEFAULT values or be nullable.

Enum Expansion (requires table recreation):
  - CHECK constraints cannot be altered in SQLite.
  - Migration pattern:
    1. CREATE TABLE <table>_new (...new CHECK constraints...);
    2. INSERT INTO <table>_new SELECT * FROM <table>;
    3. DROP TABLE <table>;
    4. ALTER TABLE <table>_new RENAME TO <table>;
  - This MUST be done at Step 0 init, never mid-pipeline.

Version Tracking:
  - schema_meta table tracks DDL version per table:
    CREATE TABLE IF NOT EXISTS schema_meta (
      table_name TEXT PRIMARY KEY,
      version INTEGER NOT NULL DEFAULT 1,
      migrated_at TEXT NOT NULL DEFAULT (datetime('now'))
    );
  - Orchestrator Step 0 checks schema_meta versions and applies
    migrations if table version < expected version.

Migration Ownership:
  - Only the Orchestrator (Step 0) may execute DDL changes.
  - No other agent may run CREATE, ALTER, or DROP statements.
  - Migration scripts are documented in sql-templates.md §9.
```

### 5.4 `evaluation-schema.md` (Tier 2)

Artifact evaluation system instructions. Replaces Forge's evaluation-schema.md (per-file YAML) with SQLite-based approach.

**Content Outline:**

1. **Overview**: Every agent that consumes upstream artifacts evaluates them after use. Evaluations are SQL INSERT operations into `artifact_evaluations` table, queryable by the Knowledge Agent.

2. **When to Evaluate**: After reading an upstream artifact and before starting work. The evaluation reflects how useful the artifact was for completing your task.

3. **Scoring Guide**:
   - `usefulness_score` (1-10): 1=completely irrelevant, 5=partially useful with major gaps, 8=very useful with minor gaps, 10=perfect — everything needed
   - `clarity_score` (1-10): 1=incomprehensible, 5=understandable but ambiguous, 8=clear with minor formatting issues, 10=crystal clear
   - `missing_information`: Free text — what you needed but the artifact didn't provide
   - `inaccuracies`: Free text — what was wrong or misleading
   - `impact_on_work`: Free text — how the artifact quality affected your task execution

4. **Phase 1 Agents** (required to evaluate):
   - **Implementer**: Evaluates task definition (tasks/TASK-NNN.yaml) — was the task clear? Was relevant_context sufficient?
   - **Verifier**: Evaluates implementation report (implementation-reports/TASK-NNN.yaml) — was the report accurate? Were changes documented?
   - **Adversarial Reviewer**: Evaluates design-output.yaml (Step 3b) or implementation + verification data (Step 7)

5. **Phase 2 Agents** (future expansion):
   - Spec evaluates research outputs; Designer evaluates spec; Planner evaluates design

### 5.5 `context7-integration.md` (Tier 2)

Conditional MCP integration instructions.

**Content Outline:**

```markdown
# Context7 MCP Integration (Conditional)

## Availability Check

Before using Context7, attempt to call `context7-resolve-library-id` with a known
library name. If the tool is not available (MCP server not configured), skip ALL
Context7 usage and proceed normally. Log: "Context7 unavailable — proceeding without
documentation lookup."

## Two-Step Usage Pattern

1. `context7-resolve-library-id` — resolve the library/framework name to a Context7 ID
2. `context7-query-docs` — query documentation with the resolved ID

## When to Use

- Before guessing at library/framework API usage (imports, method signatures, configuration)
- When implementing against an unfamiliar API
- When research requires current library documentation

## Applicable Agents

- **Implementer** (primary): Use during Step 5 when writing code against external APIs
- **Researcher** (secondary): Use during Step 1 for library-specific research topics
```

### 5.6 `review-perspectives.md` (Tier 2)

See Section 7 (Review Restructure Design) for full perspective definitions.

### 5.7 `tool-access-matrix.md` (Tier 2)

See Section 10 (Tool Access Matrix) for full matrix.

### 5.8 `.github/instructions/pipeline-conventions.md` (Tier 1)

Domain instruction file — initial content created at implementation, updated by Knowledge Agent's governed mechanism.

```markdown
# Pipeline Conventions

## Feature Directory Structure

docs/feature/<feature-slug>/
├── research/ ← Step 1 outputs (YAML-only)
├── review-findings/ ← Steps 3b, 7 outputs (MD)
├── review-verdicts/ ← Steps 3b, 7 outputs (YAML)
├── tasks/ ← Step 4 outputs (YAML)
├── implementation-reports/ ← Step 5 outputs (YAML)
├── verification-reports/ ← Step 6 outputs (YAML)
├── spec-output.yaml ← Step 2
├── feature.md ← Step 2 (human companion)
├── design-output.yaml ← Step 3
├── design.md ← Step 3 (human companion)
├── plan-output.yaml ← Step 4
├── plan.md ← Step 4 (human companion)
├── knowledge-output.yaml ← Step 8
├── evidence-bundle.md ← Step 8b (human audit)
└── decisions.yaml ← Step 8 (append-only)

## Risk Classification

🟢 Additive: new files, tests, docs, config, comments
🟡 Business logic: modifying existing functions, DB queries, UI state
🔴 Critical: auth, crypto, data deletion, public API changes

## Pipeline Steps

0: Setup | 1: Research ×4 | 1a: Gate | 2: Spec | 3: Design
3b: Review ×3 | 4: Plan | 4a: Gate | 5: Implement ×N
6: Verify ×N | 7: Review ×3 | 8: Knowledge | 8b: Evidence | 9: Commit
```

---

## 6. Agent-by-Agent Design

### 6.1 Orchestrator (800 → 480 lines)

**Current Issues:**

- 800 lines — exceeds 500-line target by 60%
- Evidence gate SQL queries inline (~70 lines of SQL)
- SQLite DDL inline (~50 lines)
- Tool access table inline (~25 lines)
- pipeline-telemetry.db created but never populated (dead dependency)
- Review dispatch uses model parameter (broken — VS Code ignores it)

**Target State:**

1. **Extract** tool access → tool-access-matrix.md (saves ~25 lines)
2. **Extract** run_in_terminal constraint details → tool-access-matrix.md (saves ~15 lines)
3. **Extract** evidence gate SQL queries → sql-templates.md §6 (saves ~70 lines)
4. **Extract** SQLite DDL → sql-templates.md §1 (saves ~50 lines)
5. **Extract** error handling/retry details → global-operating-rules.md §1-§2 (saves ~20 lines)
6. **Extract** self-verification common items → global-operating-rules.md §6 (saves ~20 lines)
7. **Compact** approval gate YAML templates inline (saves ~20 lines)
8. **Compact** decision table formatting (saves ~20 lines)
9. **Add** pipeline_telemetry INSERT after each dispatch (~10 lines, reference sql-templates.md §3)
10. **Add** consolidated DB init: single verification-ledger.db with 4 tables (reference sql-templates.md §1)
11. **Add** review dispatch with review_perspective parameter (~10 lines)
12. **Add** evidence gates for all-category review (reference sql-templates.md §6)
13. **Add** YAML frontmatter (~4 lines)
14. **Add** shared doc references (~10 lines)

**Net:** 800 − (25+15+70+50+20+20+20+20) + (10+5+10+5+4+10) = 800 − 240 + 44 = **604** → further compaction → **~550 lines**

**Compaction Itemization (604 → ~550):**

| Compaction Target                                                           | Current Est. Lines | Compacted Lines | Savings |
| --------------------------------------------------------------------------- | ------------------ | --------------- | ------- |
| Step descriptions (Steps 0-9): reduce verbose prose to concise action lists | ~200               | ~160            | ~40     |
| Routing decision table: convert prose conditions to compact table format    | ~30                | ~20             | ~10     |
| Intent/purpose statement: shorten to 2-line summary                         | ~8                 | ~3              | ~5      |
| **Total compaction**                                                        |                    |                 | **~55** |

**Revised target: ~550 lines** (604 − 55 = 549). This exceeds AC-10.1's 500-line target by ~10%. **NFR-1 exception documented:** The orchestrator's coordination complexity (9 agents, 10 steps, 6 evidence gates, telemetry INSERT, 3 feedback loops) justifies a higher line budget. The orchestrator is architecturally unique — it is the only agent with dispatch, gating, and state recovery responsibilities. Further reduction would require splitting orchestrator responsibilities into a sub-orchestrator pattern, which is out of scope for this feature.

**Pipeline Telemetry INSERT Pattern (New):**

After each `runSubagent` dispatch completes, the orchestrator records:

```sql
-- See sql-templates.md §3 for full template
INSERT INTO pipeline_telemetry (run_id, step, agent, instance, started_at,
  completed_at, status, dispatch_count, retry_count, notes)
VALUES ('{run_id}', '{step}', '{agent}', '{instance}', '{started_at}',
  '{completed_at}', '{status}', {dispatch_count}, {retry_count}, '{notes}');
```

**Updated Step 3b/7 Dispatch:**

```
Dispatch 3 adversarial-reviewer instances in parallel (Pattern A):
  Instance 1: review_perspective = "security-sentinel"
  Instance 2: review_perspective = "architecture-guardian"
  Instance 3: review_perspective = "pragmatic-verifier"

  Parameters per instance:
    review_scope: "design" (Step 3b) or "code" (Step 7)
    review_perspective: <perspective-id>
    run_id: {run_id}
    round: {round}
    feature_slug: {feature_slug}

  See review-perspectives.md for perspective definitions.
```

### 6.2 Researcher (251 → 240 lines)

**Changes:**

- **Remove** MD companion generation (research/\*.md dropped per DP-2)
- **Clarify** create_file scope: allowed for `research/*.yaml` only
- **Add** Context7 conditional reference (one line: "See context7-integration.md")
- **Extract** tool access → tool-access-matrix.md
- **Extract** error handling → global-operating-rules.md
- **Add** YAML frontmatter

**Updated Output Files Section:**

```
| File | Format | Consumer |
|------|--------|----------|
| research/<focus>.yaml | YAML-only | Spec Agent, Designer |
```

No MD companion. YAML contains all findings, evidence, and summary.

### 6.3 Spec Agent (337 → 320 lines)

**Changes:**

- **Update** input section: reads `research/*.yaml` (no .md references)
- **Extract** tool access → tool-access-matrix.md
- **Extract** error handling → global-operating-rules.md
- **Add** YAML frontmatter
- **Retain** feature.md companion (human-facing per DP-2)

### 6.4 Designer (275 → 265 lines)

**Changes:**

- **Update** revision mode inputs: glob-based verdict reading
  - Old: `review-verdicts/design.yaml` (singular, doesn't exist)
  - New: `review-verdicts/design-*.yaml` via `list_dir` on `review-verdicts/`, filter by `design-` prefix
- **Update** review-findings input: `review-findings/design-*.md` via same glob pattern
- **Extract** tool access → tool-access-matrix.md
- **Retain** design.md companion (human-facing per DP-2)

### 6.5 Planner (376 → 340 lines)

**Changes:**

- **Extract** tool access → tool-access-matrix.md
- **Extract** error handling/retry → global-operating-rules.md
- **Extract** self-verification common items → global-operating-rules.md
- **Add** YAML frontmatter
- **Retain** plan.md companion (human-facing per DP-2)
- **Retain** tasks/\*.yaml output (per-task YAML with relevant_context)

### 6.6 Implementer (519 → 340 lines)

**Changes:**

- **Extract** tool access → tool-access-matrix.md (saves ~20 lines)
- **Extract** SQL INSERT templates → sql-templates.md §2 (saves ~30 lines)
- **Extract** error handling/retry → global-operating-rules.md (saves ~20 lines)
- **Remove** inline no-file-redirect rule (now in copilot-instructions.md §2, saves ~5 lines)
- **Extract** self-verification common items → global-operating-rules.md (saves ~15 lines)
- **Compact** YAML output structure template (saves ~20 lines)
- **Compact** mode: documentation section (saves ~15 lines)
- **Compact** mode: revert section (saves ~10 lines)
- **Add** artifact evaluation INSERT after consuming task definition:
  ```
  After reading tasks/{task_id}.yaml and before starting implementation:
  Evaluate the task definition quality → INSERT into artifact_evaluations
  (See evaluation-schema.md §3 and sql-templates.md §4)
  ```
- **Add** Context7 conditional reference
- **Add** YAML frontmatter

### 6.7 Verifier (521 → 340 lines)

**Changes:**

- **Extract** tool access → tool-access-matrix.md (saves ~20 lines)
- **Extract** SQL INSERT template + safety net init → sql-templates.md (saves ~40 lines)
- **Extract** check_name convention → sql-templates.md (saves ~20 lines)
- **Extract** error handling → global-operating-rules.md (saves ~20 lines)
- **Extract** self-verification common items → global-operating-rules.md (saves ~15 lines)
- **Compact** tier descriptions (saves ~30 lines)
- **Compact** YAML output structure (saves ~20 lines)
- **RESOLVE** tool contradiction (FR-11):
  ```
  create_file: ALLOWED — scope restriction: verification-reports/*.yaml ONLY
  ```
- **Add** artifact evaluation INSERT for implementation reports:
  ```
  After reading implementation-reports/{task_id}.yaml:
  Evaluate the implementation report quality → INSERT into artifact_evaluations
  (See evaluation-schema.md §3 and sql-templates.md §4)
  ```
- **Add** YAML frontmatter

### 6.8 Adversarial Reviewer (372 → 335 lines)

**Major Restructure:**

1. **Replace** `review_focus` + `model` parameters with single `review_perspective` parameter:
   - Old: `review_focus: security|architecture|correctness`, `model: gpt-5.3-codex|gemini-3-pro|claude-opus-4.6`
   - New: `review_perspective: security-sentinel|architecture-guardian|pragmatic-verifier`

2. **Add** all-category coverage requirement:

   ```
   Regardless of your review_perspective, you MUST produce findings for ALL THREE categories:
   1. Security findings (authentication, authorization, data exposure, injection)
   2. Architecture findings (coupling, cohesion, scalability, patterns)
   3. Correctness findings (edge cases, error handling, test coverage)

   Your review_perspective determines your LENS and PRIORITIES, not your scope.
   See review-perspectives.md for your perspective's per-category guidance.
   ```

3. **Update** output paths:
   - Verdict: `review-verdicts/<scope>-<perspective>.yaml` (e.g., `review-verdicts/design-security-sentinel.yaml`)
   - Findings: `review-findings/<scope>-<perspective>.md` (e.g., `review-findings/design-security-sentinel.md`)

4. **Update** SQL INSERT: 3 INSERTs per review (one per category, scope-qualified):

   ```sql
   -- INSERT 1: Security category (scope-qualified check_name)
   INSERT INTO anvil_checks (..., check_name, ...) VALUES (..., 'review-{scope}-security', ...);
   -- INSERT 2: Architecture category
   INSERT INTO anvil_checks (..., check_name, ...) VALUES (..., 'review-{scope}-architecture', ...);
   -- INSERT 3: Correctness category
   INSERT INTO anvil_checks (..., check_name, ...) VALUES (..., 'review-{scope}-correctness', ...);
   -- {scope} = 'design' (Step 3b) or 'code' (Step 7)
   ```

5. **Extract** review focus details → review-perspectives.md
6. **Add** artifact evaluation INSERT
7. **Add** YAML frontmatter

**Updated Verdict YAML Structure:**

```yaml
agent_output:
  agent: "adversarial-reviewer"
  instance: "adversarial-reviewer-<perspective>"
  step: "step-3b" # or "step-7"
  schema_version: "1.0"
  payload:
    review_scope: "design" # or "code"
    review_perspective: "security-sentinel"
    category_verdicts:
      security:
        verdict: "approve"
        severity: "Minor"
        findings_count: 2
      architecture:
        verdict: "needs_revision"
        severity: "Major"
        findings_count: 3
      correctness:
        verdict: "approve"
        severity: null
        findings_count: 1
    overall_verdict: "needs_revision" # worst of category verdicts
    overall_severity: "Major"
completion:
  status: "DONE"
  summary: "Design review from security-sentinel perspective: 6 findings (2 security, 3 architecture, 1 correctness)"
```

### 6.9 Knowledge Agent (509 → 345 lines)

**Responsibility Inventory (ARCH-1 response):**

All 9 responsibilities share a single cohesion axis: **pipeline learning and observability**. The Knowledge Agent is the sole agent responsible for extracting, persisting, and acting on lessons learned from each pipeline run.

| #   | Responsibility                                | Justification                                | Cohesion Group |
| --- | --------------------------------------------- | -------------------------------------------- | -------------- |
| 1   | Knowledge capture (store_memory)              | Core: persist cross-session learnings        | Learning       |
| 2   | Architectural decision log (decisions.yaml)   | Core: structured decision tracking           | Learning       |
| 3   | Evidence bundle assembly (evidence-bundle.md) | Core: human-readable pipeline audit          | Observability  |
| 4   | Cross-session knowledge persistence           | Core: ensures continuity across runs         | Learning       |
| 5   | SQL telemetry queries and summarization       | New: data-driven pipeline health analysis    | Observability  |
| 6   | Artifact evaluation consumption               | New: quality analysis for feedback loop      | Observability  |
| 7   | instruction_updates INSERT tracking           | New: governed update audit trail             | Learning       |
| 8   | Enhanced post-mortem with SQL aggregation     | New: data-driven bottleneck identification   | Observability  |
| 9   | Instruction file updates (governed)           | New: apply learnings to future pipeline runs | Learning       |

**Phase 2 split consideration:** If line budget pressure increases, responsibilities 5/6/8 (Observability group) could be extracted to a dedicated "Pipeline Analyst" agent. However, the SQL query patterns are simple SELECT aggregations sharing the same DB connection, and splitting would add orchestrator dispatch overhead for minimal complexity reduction. Retained as single agent for Phase 1.

**Major Enhancement:**

1. **Grant** `run_in_terminal` with scope restriction:

   ```
   run_in_terminal: ALLOWED — scope: SELECT on all tables + INSERT on instruction_updates only
   MUST NOT execute: UPDATE, DELETE, DROP, ALTER, CREATE, PRAGMA (except busy_timeout), ATTACH
   Allowed patterns:
     SELECT ... FROM anvil_checks|pipeline_telemetry|artifact_evaluations|instruction_updates ...
     INSERT INTO instruction_updates ...
   ```

2. **Add** post-mortem data collection via SQL:

   ```
   Pipeline Telemetry Summary:
     Run queries from sql-templates.md §7:
     - Total dispatches, per-step timing, slowest steps, retry summary

   Artifact Evaluation Summary:
     - Mean usefulness/clarity per evaluator agent
     - Top 5 worst-rated artifacts
     - Common missing_information themes

   Instruction Update History:
     - All instruction updates from current run
   ```

3. **Enhance** evidence bundle (Step 8b) with new sections:

   ```markdown
   ## Pipeline Health (from pipeline_telemetry)

   - Total dispatch count: N
   - Total wall-clock time: X seconds
   - Top 3 bottleneck steps: Step 5 (45s), Step 7 (30s), Step 6 (25s)
   - Agents requiring retries: verifier-TASK-003 (1 retry)

   ## Artifact Quality (from artifact_evaluations)

   - Mean usefulness score: 7.2/10
   - Mean clarity score: 8.1/10
   - Worst-rated artifact: tasks/TASK-005.yaml (usefulness: 4/10)
   - Common missing information: "deployment constraints", "error handling requirements"
   ```

4. **Add** instruction update tracking:

   ```sql
   INSERT INTO instruction_updates (run_id, agent, file_path, change_type,
     change_summary, applied)
   VALUES ('{run_id}', 'knowledge-agent', '{file_path}', '{change_type}',
     '{change_summary}', {applied_0_or_1});
   ```

5. **Extract** tool access, error handling, governed update details, self-verification common items

---

## 7. Review Restructure Design

### 7.1 Three Perspectives

The review system changes from 3 reviewers × 1 category each to 3 reviewers × 3 categories each. Diversity comes from perspective (DP-1, Option A) rather than model.

#### Perspective 1: Security Sentinel

**Philosophy:** Assume adversarial inputs. Every trust boundary is a potential breach point. When in doubt, classify severity higher.

**Per-Category Approach:**

| Category                         | Security Sentinel's Lens                                                                                                                                                                |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Security** (primary expertise) | Threat modeling, attack surface analysis, auth/authz correctness, data exposure, injection vectors, cryptographic usage. Lowest severity threshold — flags Medium what others flag Low. |
| **Architecture** (security lens) | Trust boundary placement, defense in depth, least privilege in component design, secrets management patterns, data flow security.                                                       |
| **Correctness** (security lens)  | Error handling that could leak data, unsafe default values, race conditions with security implications, resource exhaustion vulnerabilities.                                            |

**Severity Calibration:** Most strict. Blocker threshold: any unauthenticated access, any plaintext secret, any SQL injection path.

#### Perspective 2: Architecture Guardian

**Philosophy:** Structural quality determines long-term maintainability. Technical debt compounds. Pragmatic tradeoffs are acceptable when documented.

**Per-Category Approach:**

| Category                             | Architecture Guardian's Lens                                                                                                                 |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Security** (structural lens)       | Architecture-level security patterns, authentication middleware placement, authorization layer design, data flow isolation.                  |
| **Architecture** (primary expertise) | SOLID principles, coupling/cohesion metrics, dependency direction, API contract design, scalability bottlenecks, error handling consistency. |
| **Correctness** (structural lens)    | Contract violations, integration boundary bugs, state management correctness, component lifecycle issues.                                    |

**Severity Calibration:** Moderate. Blocker threshold: circular dependencies, SOLID violations that cascade across 3+ components, API contracts that can't be versioned.

#### Perspective 3: Pragmatic Verifier

**Philosophy:** Does it work? Edge cases matter more than elegance. Tests prove behavior; everything else is opinion.

**Per-Category Approach:**

| Category                            | Pragmatic Verifier's Lens                                                                                                                            |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Security** (behavioral lens)      | Input validation edge cases (empty, null, overflow, unicode), boundary condition exploitation, error path information leakage.                       |
| **Architecture** (practical lens)   | Practical maintainability concerns (not theoretical purity), "will the next developer understand this?", test infrastructure adequacy.               |
| **Correctness** (primary expertise) | Functional completeness vs acceptance criteria, boundary value analysis, error path handling, resource cleanup, race conditions, test coverage gaps. |

**Severity Calibration:** Most lenient on style, strictest on behavior. Blocker threshold: wrong functional behavior, data corruption path, test suite that passes when code is broken.

### 7.2 All-Category Coverage

Each reviewer MUST produce findings across all 3 categories. The review_perspective shapes priorities and severity thresholds, not scope. Evidence gate validates coverage:

```sql
-- Each reviewer must have 3 distinct scope-qualified check_names
SELECT instance, COUNT(DISTINCT check_name) AS cats
FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}' AND phase = 'review' AND round = {round}
  AND check_name IN ('review-{scope}-security','review-{scope}-architecture','review-{scope}-correctness')
GROUP BY instance HAVING cats = 3;
-- Expected: 3 rows (one per reviewer), each with cats=3
-- {scope} = 'design' for Step 3b, 'code' for Step 7
```

If a reviewer produces findings for fewer than 3 categories, the evidence gate fails and the orchestrator retries that reviewer with explicit instruction to cover all categories.

### 7.3 Updated Evidence Gate Queries

See sql-templates.md §6 for full query templates. The orchestrator runs these at Steps 3b and 7:

1. **All submitted**: 3 distinct instances (filtered by task_id to distinguish design vs code review)
2. **All-category**: Each instance has 3 distinct scope-qualified check_names (e.g., `review-design-security`)
3. **Zero blockers**: No record with verdict='blocker' (filtered by task_id)
4. **Majority fully-approving**: ≥2 instances where ALL 3 category verdicts are 'approve' (uses subquery with `HAVING COUNT(CASE WHEN verdict != 'approve' THEN 1 END) = 0`)

Security Blocker Policy (retained): If ANY single finding has verdict='blocker', the pipeline immediately returns ERROR regardless of majority.

### 7.4 Verdict File Naming Convention

**Pattern:** `review-verdicts/<scope>-<perspective>.yaml`

**Examples:**
| Step | Scope | Files |
|---|---|---|
| 3b (Design Review) | design | `review-verdicts/design-security-sentinel.yaml`, `review-verdicts/design-architecture-guardian.yaml`, `review-verdicts/design-pragmatic-verifier.yaml` |
| 7 (Code Review) | code | `review-verdicts/code-security-sentinel.yaml`, `review-verdicts/code-architecture-guardian.yaml`, `review-verdicts/code-pragmatic-verifier.yaml` |

**Consumer Discovery:**

```
1. list_dir("review-verdicts/")
2. Filter files matching pattern: <scope>-*.yaml
3. Read each matching file
```

**Review Findings (MD) follow same pattern:**
`review-findings/design-security-sentinel.md`, etc.

---

## 8. SQLite Schema Design

### 8.1 Database Consolidation

**Decision (D-1):** Consolidate all tables into single `verification-ledger.db`. Drop separate `pipeline-telemetry.db`.

**Before:**

```
verification-ledger.db  → anvil_checks
pipeline-telemetry.db   → pipeline_telemetry (empty, dead)
```

**After:**

```
verification-ledger.db  → anvil_checks
                        → pipeline_telemetry
                        → artifact_evaluations
                        → instruction_updates
```

**Rationale:**

- Single PRAGMA configuration (WAL + busy_timeout)
- Single Step 0 initialization
- Single DB file for Knowledge Agent queries
- All tables share run_id namespace
- Fewer files in workspace

**Migration:** Orchestrator Step 0 DDL updated to create all 4 tables in verification-ledger.db. Drop pipeline-telemetry.db creation. No data migration needed (telemetry table was always empty).

### 8.2 `artifact_evaluations` Table

```sql
CREATE TABLE IF NOT EXISTS artifact_evaluations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  evaluator_agent TEXT NOT NULL,
  evaluator_instance TEXT,
  artifact_path TEXT NOT NULL CHECK (artifact_path NOT LIKE '../%' AND artifact_path NOT LIKE '/%'),
  usefulness_score INTEGER NOT NULL CHECK (usefulness_score BETWEEN 1 AND 10),
  clarity_score INTEGER NOT NULL CHECK (clarity_score BETWEEN 1 AND 10),
  missing_information TEXT CHECK (LENGTH(missing_information) <= 2000),
  inaccuracies TEXT CHECK (LENGTH(inaccuracies) <= 2000),
  impact_on_work TEXT CHECK (LENGTH(impact_on_work) <= 2000),
  ts TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_eval_run ON artifact_evaluations(run_id);
CREATE INDEX IF NOT EXISTS idx_eval_agent ON artifact_evaluations(evaluator_agent);
CREATE INDEX IF NOT EXISTS idx_eval_artifact ON artifact_evaluations(artifact_path);
```

**Column Notes:**

- `evaluator_agent`: Agent type performing evaluation (e.g., "implementer", "verifier")
- `evaluator_instance`: Specific instance (e.g., "implementer-TASK-001")
- `artifact_path`: Relative path to evaluated artifact (e.g., "tasks/TASK-001.yaml"). CHECK constraint prevents path traversal (`../`) and absolute paths (`/`).
- `usefulness_score`: 1 (irrelevant) to 10 (everything needed)
- `clarity_score`: 1 (incomprehensible) to 10 (crystal clear)
- `missing_information`: What the evaluator needed but wasn't in the artifact
- `inaccuracies`: What was wrong or misleading in the artifact
- `impact_on_work`: How artifact quality affected task execution

**Phase 1 Writers:**

| Agent                | Evaluates                                      | When                                    |
| -------------------- | ---------------------------------------------- | --------------------------------------- |
| Implementer          | tasks/{task_id}.yaml                           | After reading task, before implementing |
| Verifier             | implementation-reports/{task_id}.yaml          | After reading report, before verifying  |
| Adversarial Reviewer | design-output.yaml (3b) or impl+verif data (7) | After reading, before reviewing         |

### 8.3 `instruction_updates` Table

```sql
CREATE TABLE IF NOT EXISTS instruction_updates (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  agent TEXT NOT NULL,
  file_path TEXT NOT NULL CHECK (
    file_path LIKE '.github/instructions/%'
    OR file_path = '.github/copilot-instructions.md'
  ),
  change_type TEXT NOT NULL CHECK (change_type IN ('create','append','modify','delete')),
  change_summary TEXT NOT NULL CHECK (LENGTH(change_summary) <= 1000),
  applied INTEGER NOT NULL DEFAULT 0 CHECK (applied IN (0, 1)),
  ts TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_instr_run ON instruction_updates(run_id);
```

**Column Notes:**

- `agent`: Always "knowledge-agent" (only writer)
- `file_path`: Target instruction file — CHECK constraint enforces allowed paths (`.github/instructions/*` or `.github/copilot-instructions.md` only)
- `change_type`: create (new file), append (add content), modify (edit existing), delete (remove content)
- `change_summary`: Human-readable description of the change (max 1000 chars)
- `applied`: 0 = proposed but not applied (autonomous mode), 1 = applied (interactive mode with approval)

### 8.4 `pipeline_telemetry` Population Strategy

**Writer:** Orchestrator only (after each agent dispatch completes)

**When:** Immediately after `runSubagent` returns, before processing the next dispatch or evidence gate.

**Data Source:** The orchestrator has natural access to all telemetry data:

- `started_at`: Timestamp before runSubagent call
- `completed_at`: Timestamp after runSubagent returns
- `status`: Read from agent's completion contract
- `dispatch_count`: 1 (first attempt) or 2+ (retries)
- `retry_count`: Number of retries for this instance

**Example Flow:**

```
1. Record started_at = now()
2. Call runSubagent(researcher, {focus: "architecture"})
3. Record completed_at = now()
4. Read completion status from research/architecture.yaml
5. INSERT INTO pipeline_telemetry (run_id, step, agent, instance, started_at,
   completed_at, status, dispatch_count, retry_count, notes)
   VALUES ('{run_id}', 'step-1', 'researcher', 'researcher-architecture',
   '{started_at}', '{completed_at}', 'DONE', 1, 0, null);
```

### 8.5 Key Queries for Knowledge Agent

See sql-templates.md §7 for full query list. Summary:

| Query               | Purpose                     | Output                                    |
| ------------------- | --------------------------- | ----------------------------------------- |
| Per-step timing     | Identify bottleneck steps   | step, duration_seconds                    |
| Top 3 slowest       | Bottleneck ranking          | step, total_seconds                       |
| Retry summary       | Agent reliability           | agent, instance, retry_count              |
| Eval by agent       | Evaluator consistency       | evaluator, count, avg_useful, avg_clarity |
| Worst artifacts     | Quality improvement targets | artifact_path, avg_useful, avg_clarity    |
| Missing info        | Common gaps                 | missing_information text list             |
| Instruction history | Governed update audit       | full instruction update records           |

---

## 9. YAML+MD Output Rationalization

Per DP-2, outputs are classified as YAML-only (machine-consumed) or YAML+MD / MD-only (human-facing):

| Artifact                       | Producer             | Format        | Rationale                                                |
| ------------------------------ | -------------------- | ------------- | -------------------------------------------------------- |
| research/\*.yaml               | Researcher           | **YAML-only** | No downstream agent reads .md; Spec reads .yaml directly |
| spec-output.yaml + feature.md  | Spec                 | **YAML+MD**   | Human approval gate at Step 1a reviews feature.md        |
| design-output.yaml + design.md | Designer             | **YAML+MD**   | Human approval gate, design review uses .md              |
| plan-output.yaml + plan.md     | Planner              | **YAML+MD**   | Human approval gate at Step 4a reviews plan.md           |
| tasks/\*.yaml                  | Planner              | **YAML-only** | Machine-consumed by Implementer via relevant_context     |
| implementation-reports/\*.yaml | Implementer          | **YAML-only** | Machine-consumed by Verifier                             |
| verification-reports/\*.yaml   | Verifier             | **YAML-only** | Machine-consumed by Orchestrator for gate status         |
| review-verdicts/\*.yaml        | Adversarial Reviewer | **YAML-only** | Machine-consumed by Orchestrator + Designer              |
| review-findings/\*.md          | Adversarial Reviewer | **MD-only**   | Human-readable review narrative with code snippets       |
| knowledge-output.yaml          | Knowledge Agent      | **YAML-only** | Terminal output, no downstream agent                     |
| evidence-bundle.md             | Knowledge Agent      | **MD-only**   | Human audit trail — primary end-of-pipeline document     |
| decisions.yaml                 | Knowledge Agent      | **YAML-only** | Append-only decision log                                 |

**Net Change:** Researcher drops 4 MD files per run. No other agent output format changes (implementation-reports, verification-reports, review-verdicts were already YAML-only; review-findings were already MD-only).

**schemas.md Update:** Add an "Output Format Classification" table documenting each artifact's format with rationale. Update Schema 2 (research-output) to note YAML-only.

---

## 10. Tool Access Matrix

### Updated Matrix

| Tool                   | Orch | Res | Spec | Des | Plan | Impl | Verif   | AdvRev | Know    |
| ---------------------- | ---- | --- | ---- | --- | ---- | ---- | ------- | ------ | ------- |
| agent/runSubagent      | ✅   | ❌  | ❌   | ❌  | ❌   | ❌   | ❌      | ❌     | ❌      |
| memory                 | ✅   | ❌  | ❌   | ❌  | ❌   | ❌   | ❌      | ❌     | ✅      |
| ask_questions          | ✅   | ❌  | ✅¹  | ❌  | ❌   | ❌   | ❌      | ❌     | ❌      |
| read_file              | ✅   | ✅  | ✅   | ✅  | ✅   | ✅   | ✅      | ✅     | ✅      |
| list_dir               | ✅   | ✅  | ✅   | ✅  | ✅   | ✅   | ✅      | ✅     | ✅      |
| grep_search            | ❌   | ✅  | ✅   | ✅  | ✅   | ✅   | ✅      | ✅     | ✅      |
| semantic_search        | ❌   | ✅  | ✅   | ✅  | ✅   | ✅   | ❌      | ✅     | ✅      |
| file_search            | ❌   | ✅  | ✅   | ✅  | ✅   | ✅   | ✅      | ✅     | ✅      |
| create_file            | ❌   | ✅² | ✅   | ✅  | ✅   | ✅   | **✅³** | ✅     | ✅      |
| replace_string_in_file | ❌   | ❌  | ✅   | ✅  | ✅   | ✅   | ❌      | ❌     | ✅      |
| multi_replace          | ❌   | ❌  | ❌   | ❌  | ❌   | ✅   | ❌      | ❌     | ❌      |
| run_in_terminal        | ✅⁴  | ❌  | ❌   | ❌  | ❌   | ✅   | ✅      | ✅⁵    | **✅⁶** |
| get_terminal_output    | ✅   | ❌  | ❌   | ❌  | ❌   | ✅   | ✅      | ❌     | ❌      |
| get_errors             | ❌   | ❌  | ❌   | ❌  | ❌   | ✅   | ✅      | ❌     | ❌      |
| list_code_usages       | ❌   | ❌  | ❌   | ❌  | ❌   | ✅   | ❌      | ❌     | ❌      |

**Footnotes:**

1. ¹ Interactive mode only
2. ² Scoped: `research/*.yaml` only
3. ³ **NEW** — Scoped: `verification-reports/*.yaml` only (FR-11 resolution)
4. ⁴ Scoped: SQL queries + git operations only (DR-1)
5. ⁵ Scoped: `git diff` + SQL INSERT only
6. ⁶ **NEW** — Scoped: SELECT on all tables + INSERT on `instruction_updates` only (FR-6/FR-15 enablement)

---

## 11. Security Considerations

### 11.1 Agent Boundary Integrity

- No agent can modify another agent's definition files (CR-8, preserved)
- Knowledge Agent's governed updates target only `.github/instructions/` and `.github/copilot-instructions.md` — never `.agent.md` files
- Safety constraint filter (formally defined in global-operating-rules.md §9) rejects changes that weaken security rules
- **Mutability classification (ARCH-5 response):**
  - `.github/copilot-instructions.md` is **immutable at runtime** — only modifiable via feature implementation, never by any agent
  - `.github/instructions/pipeline-conventions.md` is **governed-mutable** — updatable by Knowledge Agent with safety filters

### 11.2 SQL Injection Prevention

- All SQL queries use string formatting (not parameterized — sqlite3 CLI limitation)
- **Mandatory sanitization (SEC-1 response):** All text values MUST be processed through `sql_escape()` before interpolation:
  - Replace `'` → `''` (single-quote doubling)
  - Strip null bytes: `\0` → removed
  - Truncate to field-specific max length per CHECK constraints
- `sql_escape()` is defined in sql-templates.md §0 as the canonical function. INSERT templates in §2–§5 use `sql_escape()` wrappers on all free-text fields.
- run_id is ISO 8601 format (no special characters)
- output_snippet is LENGTH-constrained to 500 chars
- All other free-text fields have LENGTH CHECK constraints (see §1 DDL)
- **Self-verification checklist item** for all SQL-writing agents: "Verify all text values are sql_escape()'d before INSERT"

### 11.3 Shell Injection Prevention

- **Dual injection surface (SEC-2 response):** SQL commands passed to `run_in_terminal` traverse the system shell before reaching sqlite3. Shell metacharacters (`$()`, backticks, `|`, `;`) in agent-generated values would be interpreted by the shell.
- **Mandatory mitigation:** Pipe SQL via stdin to sqlite3 to avoid shell interpretation:
  ```
  echo "INSERT INTO ..." | sqlite3 verification-ledger.db
  ```
- **Escaping order** (when stdin piping is not feasible): (1) SQL-escape via `sql_escape()`, then (2) shell-escape (platform-specific)
- This is documented as a global operating rule in copilot-instructions.md and enforced via self-verification checklists.

### 11.4 Tool Scope Restrictions

- **Acknowledged limitation (SEC-4 response):** All tool scope restrictions are enforced via prompt instructions only. VS Code's agent runtime does not support parameterized tool access policies. Compliance depends on LLM instruction-following capability. This is an **accepted risk** with residual threat level: Medium.
- **Mitigation layers:**
  1. **Agent prompt instructions**: Each agent's definition specifies allowed tool scopes
  2. **Self-verification checklist**: Before each scoped tool call, agents verify command matches allowed patterns
  3. **Regex patterns per agent** (documented in tool-access-matrix.md §11):
     - Knowledge Agent `run_in_terminal`: `^(echo\s+".*(SELECT|INSERT INTO instruction_updates).*"\s*\|\s*sqlite3|sqlite3)\s+.*verification-ledger\.db`
     - Verifier `create_file`: path must match `verification-reports/.*\.yaml$`
     - Researcher `create_file`: path must match `research/.*\.yaml$`
  4. **Verifier as secondary enforcement**: The verifier agent reviews other agents' outputs and can flag file path violations
- Orchestrator's run_in_terminal limited to SQL + git (no builds, tests, code execution)
- Knowledge Agent's run_in_terminal: SELECT on all tables + INSERT on `instruction_updates` only. MUST NOT execute: UPDATE, DELETE, DROP, ALTER, CREATE, PRAGMA (except busy_timeout), ATTACH
- Verifier's create_file limited to verification-reports/\*.yaml
- Reviewer's run_in_terminal limited to git diff + SQL INSERT

### 11.5 Evidence Integrity

- SQL evidence gates are the anti-hallucination mechanism
- No agent can claim verification without corresponding SQL INSERT
- run_id filtering prevents cross-run contamination
- **task_id filtering (CORR-1 response):** Evidence gate queries use task_id to distinguish Step 3b (design review) from Step 7 (code review) records
- Evidence gates run independently by orchestrator (not self-reported by agents)
- **Source-of-truth hierarchy (ARCH-4 response):** SQL INSERTs into anvil_checks are the authoritative evidence for pipeline routing decisions. YAML category_verdicts are derived summaries for human consumption and revision context. If discrepancy exists, SQL prevails.

### 11.6 Path Validation

- **Database-level enforcement (SEC-6 response):**
  - `instruction_updates.file_path`: CHECK constraint limits to `.github/instructions/%` or `.github/copilot-instructions.md`
  - `artifact_evaluations.artifact_path`: CHECK constraint prevents path traversal (`NOT LIKE '../%' AND NOT LIKE '/%'`)
- **Knowledge Agent self-verification:** Verify `file_path` starts with `.github/instructions/` or equals `.github/copilot-instructions.md` before INSERT

### 11.7 Database Resilience

- **Data criticality tiers (ARCH-6 response):**
  1. **Mission-critical:** `anvil_checks` — pipeline cannot proceed without valid evidence records
  2. **Important:** `pipeline_telemetry` — post-mortem analysis degrades without timing data
  3. **Informational:** `artifact_evaluations`, `instruction_updates` — quality analysis degrades but pipeline function is unaffected
- **Integrity check:** `PRAGMA integrity_check` runs at Step 0 init for existing DB files
- **Corruption recovery priority:** anvil_checks reconstruction from agent output files > telemetry > evaluations
- **Run ID collision risk (SEC-7 response):** run_id uses ISO 8601 timestamp. Collision risk is low for normal operation. A `runs` metadata table or random suffix is a Phase 2 enhancement. Current mitigation: never manually reuse a run_id.

---

## 12. Failure & Recovery

### 12.1 Expected Failure Modes

| Failure                                | Detection                            | Recovery                                                                                   |
| -------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------ |
| SQLITE_BUSY under parallel writes      | Error from sqlite3 CLI               | Retry 3x with exponential backoff (1s, 2s, 4s). WAL mode prevents most conflicts.          |
| Reviewer covers <3 categories          | Evidence gate EG-4 returns <3 rows   | Orchestrator retries reviewer with explicit instruction                                    |
| Context7 MCP unavailable               | Tool not found error                 | Skip documentation lookup, log unavailability, continue normally                           |
| sqlite3 CLI not in PATH                | Step 0 `sqlite3 --version` fails     | Pipeline ERROR with clear diagnostic at Step 0                                             |
| Pipeline interrupted mid-run           | Incomplete output files              | EC-5 recovery: scan feature dir, read completion blocks, resume from first incomplete step |
| Evaluation scores all high (no signal) | Knowledge Agent detects low variance | Log concern in post-mortem; not a failure, just uninformative data                         |
| Instruction update conflicts           | Safety constraint filter             | Reject in autonomous mode; present to user in interactive mode                             |
| Agent file exceeds line target         | Implementation review                | Extract additional content to shared docs                                                  |
| Verdict path mismatch                  | Consumer reports missing file        | Designer uses glob pattern (list_dir) — resilient to naming changes                        |

### 12.2 Pipeline State Recovery (EC-5)

Preserved from current design. On resume:

1. Orchestrator scans `docs/feature/<slug>/` with `list_dir`
2. Reads each output's completion block
3. Maps outputs to pipeline steps
4. Resumes from first incomplete step
5. run_id filtering ensures SQL queries only see current-run data

---

## 13. Migration & Backward Compatibility

### 13.1 Breaking Changes

| Change                                | Impact                                 | Migration                                                    |
| ------------------------------------- | -------------------------------------- | ------------------------------------------------------------ |
| pipeline-telemetry.db eliminated      | Orchestrator Step 0 DDL changes        | Update Step 0 to create all tables in verification-ledger.db |
| Research MD companions removed        | No downstream agent affected           | Remove Researcher MD output instructions                     |
| review_focus → review_perspective     | Adversarial Reviewer dispatch changes  | Update orchestrator dispatch parameters                      |
| model parameter removed               | Adversarial Reviewer parameter changes | Remove model from dispatch, add review_perspective           |
| Verdict path convention changed       | Designer input path changes            | Update Designer revision mode inputs to glob pattern         |
| Knowledge Agent gains run_in_terminal | Tool access change                     | Update tool-access-matrix.md and Knowledge Agent definition  |

### 13.2 Non-Breaking Changes

- New shared reference documents (additive — agents reference them, no removal)
- New SQLite tables (additive — new agents write to them)
- Frontmatter additions (additive — agents that already have frontmatter retain it)
- copilot-instructions.md creation (additive — auto-loaded by VS Code)

### 13.3 Deployment Strategy

1. Create new reference docs first (can coexist with existing agents)
2. Create .github/copilot-instructions.md and .github/instructions/
3. Modify agents in dependency order: researcher → spec → designer → planner → implementer → verifier → adversarial-reviewer → knowledge-agent → orchestrator (last, depends on all others)
4. Update schemas.md with new tables and routing matrix
5. Update dispatch-patterns.md with new review dispatch
6. Mirror NewAgents/.github/agents/ to .github/agents/

---

## 14. Testing Strategy

### 14.1 Unit-Level Validation

| Test                         | Method                                                                  | Linked AC |
| ---------------------------- | ----------------------------------------------------------------------- | --------- |
| Agent file sizes             | `wc -l .github/agents/*.agent.md`                                       | AC-10.1   |
| No duplicated rules          | `grep -rl "Never redirect" .github/agents/*.agent.md` returns 0 results | AC-10.5   |
| Frontmatter present          | Each .agent.md starts with `---\nname:`                                 | AC-16.1   |
| Tool contradictions resolved | Verifier has create_file in tool table                                  | AC-11.1   |
| Verdict paths consistent     | Producer path = consumer glob pattern                                   | AC-12.1   |

### 14.2 Integration-Level Validation (Post-Deployment)

| Test                   | Method                                                                                                  | Linked AC      |
| ---------------------- | ------------------------------------------------------------------------------------------------------- | -------------- |
| Telemetry populated    | `SELECT COUNT(*) FROM pipeline_telemetry WHERE run_id=?` ≥ total dispatches                             | AC-6.1, AC-6.2 |
| Terminal-only testing  | `SELECT check_name, tool FROM anvil_checks WHERE check_name LIKE '%test%'` shows tool='run_in_terminal' | AC-7.4         |
| All-category review    | EG-4 query returns 3 rows                                                                               | AC-3.1, AC-3.3 |
| Evaluation data exists | `SELECT evaluator_agent, COUNT(*) FROM artifact_evaluations GROUP BY evaluator_agent` has ≥3 agents     | AC-5.2         |
| Perspective diversity  | ≥30% unique findings across 3 reviewers                                                                 | AC-2.2         |
| Context7 graceful skip | Researcher completes DONE without Context7                                                              | AC-9.3         |

---

## 15. Decision Records

### D-1: Database Consolidation into Single File

| Field            | Value                                                                                                                                          |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Context**      | Currently 2 DB files (verification-ledger.db, pipeline-telemetry.db). FR-5 and FR-14 add 2 more tables. Need to decide consolidation strategy. |
| **Alternatives** | (a) Single verification-ledger.db with all tables, (b) Keep separate DB files per concern, (c) New pipeline.db replacing both                  |
| **Selected**     | (a) Single verification-ledger.db                                                                                                              |
| **Rationale**    | Single WAL config, single busy_timeout, single Step 0 init. All tables share run_id namespace. Avoids breaking existing SQL paths.             |
| **Confidence**   | High — additive change to existing DB, no query path breaks                                                                                    |
| **Risk**         | 🟡 — More concurrent writers to one file, mitigated by WAL + busy_timeout                                                                      |

### D-2: Knowledge Agent SQL Access via run_in_terminal

| Field            | Value                                                                                                                                                                             |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Context**      | Knowledge Agent needs to read pipeline_telemetry and artifact_evaluations, and INSERT into instruction_updates. Currently restricted from run_in_terminal.                        |
| **Alternatives** | (a) Grant run_in_terminal (SELECT + INSERT on instruction_updates), (b) Orchestrator extracts SQL data into YAML dispatch context, (c) Knowledge Agent reads from YAML files only |
| **Selected**     | (a) Grant run_in_terminal (SELECT + INSERT on instruction_updates)                                                                                                                |
| **Rationale**    | Direct SQL access is more reliable. Knowledge Agent already reads from verification data. Scope restriction (SELECT + INSERT on instruction_updates only) maintains safety.       |
| **Confidence**   | High — scope restriction provides safety equivalent to current state                                                                                                              |
| **Risk**         | 🟢 — Tight scope restriction, SELECT + INSERT on instruction_updates only, no UPDATE/DELETE/DROP capability                                                                       |

### D-3: Review Findings Format Stays MD-Only

| Field            | Value                                                                                                                                  |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Context**      | FR-4 rationalizes output formats. Review findings are narrative documents — what format?                                               |
| **Alternatives** | (a) MD-only (current), (b) YAML-only, (c) YAML+MD                                                                                      |
| **Selected**     | (a) MD-only                                                                                                                            |
| **Rationale**    | Review findings are narrative with code snippets. Machine signal is in verdict YAML + SQL. No consumer parses findings MD for routing. |
| **Confidence**   | High — clear consumer analysis from research                                                                                           |
| **Risk**         | 🟢 — No format change, status quo preserved                                                                                            |

### D-4: Per-Dispatch Telemetry INSERT

| Field            | Value                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------ |
| **Context**      | FR-6 requires populating pipeline_telemetry. When should INSERTs happen?                   |
| **Alternatives** | (a) After each dispatch completes, (b) Batch at end of pipeline, (c) Agent self-reporting  |
| **Selected**     | (a) After each dispatch completes                                                          |
| **Rationale**    | Survives pipeline interruption. Incremental records. Orchestrator has natural timing data. |
| **Confidence**   | High — incremental INSERT is simplest and most resilient                                   |
| **Risk**         | 🟢 — Minimal complexity, additive SQL operation                                            |

### D-5: Six Focused Reference Docs

| Field            | Value                                                                                                      |
| ---------------- | ---------------------------------------------------------------------------------------------------------- |
| **Context**      | FR-10 requires ≥3 shared reference docs. How many and how granular?                                        |
| **Alternatives** | (a) 6 focused docs, (b) 2-3 large monolithic docs, (c) 8+ micro docs                                       |
| **Selected**     | (a) 6 focused docs                                                                                         |
| **Rationale**    | Agents reference only needed docs — focused is better for context budget. Not so many as to be fragmented. |
| **Confidence**   | High — clear content classification per doc                                                                |
| **Risk**         | 🟢 — Additive creation, no breaking changes                                                                |

### D-6: Researcher YAML-Only Output

| Field            | Value                                                                                                      |
| ---------------- | ---------------------------------------------------------------------------------------------------------- |
| **Context**      | DP-2 says machine-consumed artifacts should be YAML-only. Research outputs have no human reader.           |
| **Alternatives** | (a) YAML-only (drop .md), (b) Keep both                                                                    |
| **Selected**     | (a) YAML-only                                                                                              |
| **Rationale**    | No downstream agent reads research/\*.md. YAML findings contain all detail.                                |
| **Confidence**   | Medium — YAML is less human-readable, but research YAML is well-structured with detail and evidence fields |
| **Risk**         | 🟢 — Reduces output effort, no consumer impact                                                             |

### D-7: Perspective-Based Verdict Naming

| Field            | Value                                                                                                               |
| ---------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Context**      | DP-6 recommends per-reviewer files. What naming convention?                                                         |
| **Alternatives** | (a) `<scope>-<perspective>.yaml`, (b) `<scope>-<model>.yaml` (current), (c) `<scope>-<N>.yaml` (numeric)            |
| **Selected**     | (a) `<scope>-<perspective>.yaml`                                                                                    |
| **Rationale**    | Perspectives are the diversity mechanism (not models). Names have semantic meaning. Consumers glob by scope prefix. |
| **Confidence**   | High — clean pattern, preserves provenance                                                                          |
| **Risk**         | 🟢 — Breaking change from current naming, but current naming is broken (all same model)                             |

### D-8: Three Category-Specific SQL INSERTs Per Reviewer

| Field            | Value                                                                                                                         |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Context**      | All-category coverage needs evidence. How to record per-category review evidence?                                             |
| **Alternatives** | (a) 3 INSERTs per reviewer (1 per category), (b) 1 aggregate INSERT per reviewer, (c) Category sub-verdicts in YAML only      |
| **Selected**     | (a) 3 INSERTs per reviewer                                                                                                    |
| **Rationale**    | Enables deterministic evidence gate: COUNT(DISTINCT check_name) = 3 per reviewer. No YAML parsing needed for gate validation. |
| **Confidence**   | High — aligns with evidence-first principle                                                                                   |
| **Risk**         | 🟡 — 9 INSERTs per review round (was 3), slight increase in SQL operations                                                    |

---

## 16. Deviation Records

### DR-1: Knowledge Agent Gains run_in_terminal

| Field                | Value                                                                                                                                                                                                           |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Spec Requirement** | FR-6                                                                                                                                                                                                            |
| **Deviation**        | Granting run_in_terminal to Knowledge Agent (currently restricted). Spec contemplated this: "requires granting run_in_terminal for read-only SQL, or having the orchestrator extract telemetry into a summary." |
| **Justification**    | Direct SQL access is more reliable than orchestrator-mediated extraction. Scope restriction (SELECT on all tables + INSERT on instruction_updates only) maintains safety.                                       |

### DR-2: Database Consolidation into verification-ledger.db

| Field                | Value                                                                                                                                                                                  |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Spec Requirement** | FR-14                                                                                                                                                                                  |
| **Deviation**        | Consolidating pipeline-telemetry.db into verification-ledger.db instead of maintaining separate files. Spec says "consider consolidating into fewer DB files or a single pipeline.db." |
| **Justification**    | Consolidation into existing verification-ledger.db avoids breaking SQL query paths. All tables share run_id namespace.                                                                 |

### DR-3: Researcher create_file Scope Narrowing

| Field                | Value                                                                                                                                                                                                    |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Spec Requirement** | AC-11.2                                                                                                                                                                                                  |
| **Deviation**        | AC-11.2 defines the researcher's allowed output path pattern as `research/*.yaml, research/*.md`. The design narrows this to `research/*.yaml` only, removing MD output per DP-2 output rationalization. |
| **Justification**    | DP-2 mandates YAML-only for machine-consumed artifacts. No downstream agent reads `research/*.md`. Spec agent reads `research/*.yaml` directly. MD companions served no machine consumer.                |

---

## 17. Implementation Checklist

### Phase 1: Infrastructure (No agent changes required)

- [ ] Create `.github/copilot-instructions.md` with global rules
- [ ] Create `.github/instructions/pipeline-conventions.md`
- [ ] Create `NewAgents/.github/agents/global-operating-rules.md`
- [ ] Create `NewAgents/.github/agents/sql-templates.md`
- [ ] Create `NewAgents/.github/agents/evaluation-schema.md`
- [ ] Create `NewAgents/.github/agents/context7-integration.md`
- [ ] Create `NewAgents/.github/agents/review-perspectives.md`
- [ ] Create `NewAgents/.github/agents/tool-access-matrix.md`

### Phase 2: Schema & Reference Doc Updates

- [ ] Update `schemas.md`: routing matrix, format classification, new table DDL, updated Schema 9
- [ ] Update `dispatch-patterns.md`: perspective dispatch, FR-1 findings
- [ ] Update `severity-taxonomy.md`: frontmatter

### Phase 3: Agent Modifications (Dependency Order)

- [ ] Modify `researcher.agent.md` (no upstream deps)
- [ ] Modify `spec.agent.md` (depends on researcher output format)
- [ ] Modify `designer.agent.md` (depends on verdict path convention)
- [ ] Modify `adversarial-reviewer.agent.md` (depends on perspective definitions)
- [ ] Modify `planner.agent.md`
- [ ] Modify `implementer.agent.md`
- [ ] Modify `verifier.agent.md` (FR-11 tool contradiction)
- [ ] Modify `knowledge-agent.agent.md` (FR-6 SQL access)
- [ ] Modify `orchestrator.agent.md` (depends on all other agents)

### Phase 4: Deployment

- [ ] Mirror `NewAgents/.github/agents/` to `.github/agents/`
- [ ] Verify all agent files within line count targets
- [ ] Verify no duplicated operating rules across agents
- [ ] Validate all file references resolve correctly

### Acceptance Criteria Mapping

| FR    | ACs                                         | Addressed By                                              |
| ----- | ------------------------------------------- | --------------------------------------------------------- |
| FR-1  | AC-1.1, AC-1.2, AC-1.3                      | dispatch-patterns.md update + orchestrator                |
| FR-2  | AC-2.1, AC-2.2, AC-2.3, AC-2.4              | review-perspectives.md + adversarial-reviewer             |
| FR-3  | AC-3.1, AC-3.2, AC-3.3                      | adversarial-reviewer + sql-templates.md EG-4              |
| FR-4  | AC-4.1, AC-4.2, AC-4.3, AC-4.4              | schemas.md format table + researcher (drop MD)            |
| FR-5  | AC-5.1, AC-5.2, AC-5.3, AC-5.4              | evaluation-schema.md + sql-templates.md + 3 agents        |
| FR-6  | AC-6.1, AC-6.2, AC-6.3, AC-6.4              | orchestrator INSERT + knowledge-agent SQL                 |
| FR-7  | AC-7.1, AC-7.2, AC-7.3, AC-7.4              | copilot-instructions.md §1                                |
| FR-8  | AC-8.1, AC-8.2, AC-8.3                      | copilot-instructions.md §2                                |
| FR-9  | AC-9.1, AC-9.2, AC-9.3, AC-9.4              | context7-integration.md                                   |
| FR-10 | AC-10.1, AC-10.2, AC-10.3, AC-10.4, AC-10.5 | 6 new ref docs + agent extractions                        |
| FR-11 | AC-11.1, AC-11.2, AC-11.3                   | tool-access-matrix.md + verifier + researcher             |
| FR-12 | AC-12.1, AC-12.2, AC-12.3                   | verdict naming convention + designer glob                 |
| FR-13 | AC-13.1, AC-13.2, AC-13.3, AC-13.4          | schemas.md routing matrix + sql-templates.md              |
| FR-14 | AC-14.1, AC-14.2, AC-14.3, AC-14.4          | sql-templates.md DDL + WAL config                         |
| FR-15 | AC-15.1, AC-15.2, AC-15.3, AC-15.4          | copilot-instructions.md + instructions/ + knowledge-agent |
| FR-16 | AC-16.1, AC-16.2                            | All agent frontmatter additions                           |

---

## 18. Revision Notes (Round 2)

This section documents all changes made in response to adversarial review Round 1 findings from three reviewers (security, architecture, correctness).

### Critical Findings Addressed (4)

| ID         | Finding                                                           | Resolution                                                                                                                                                                                                                                                          | Affected Sections                                                             |
| ---------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **SEC-1**  | SQL injection via unparameterized templates                       | Defined mandatory `sql_escape()` function in sql-templates.md §0. Updated all INSERT templates (§2–§5) to wrap free-text fields with `sql_escape()`. Added self-verification checklist item. Added LENGTH CHECK constraints to all free-text fields in DDL.         | §5.3 (§0, §2–§5), §11.2                                                       |
| **SEC-2**  | Command injection via shell-interpreted SQL                       | Added §11.3 Shell Injection Prevention section. Mandated stdin piping (`echo "..." \| sqlite3 db`) as preferred pattern. Defined escaping order: SQL-escape then shell-escape.                                                                                      | §5.3 (§0), §11.3                                                              |
| **CORR-1** | Evidence gate queries conflate design/code review                 | Added `task_id` filter to all evidence gate queries (EG-1 through EG-6). Added scope-qualified check_names (`review-{scope}-security` etc.) to EG-4 and EG-6. Design review uses `task_id='{slug}-design-review'`, code review uses `task_id='{slug}-code-review'`. | §5.3 (§6), §7.2, §7.3, design-output.yaml evidence_gate_queries + key_queries |
| **CORR-2** | EG-6 majority approval false positives under D-8 3-INSERT pattern | Replaced EG-6 with subquery: `SELECT COUNT(*) FROM (SELECT instance ... GROUP BY instance HAVING COUNT(CASE WHEN verdict != 'approve' THEN 1 END) = 0)`. A reviewer is "fully approving" only if ALL 3 category verdicts are 'approve'.                             | §5.3 (§6 EG-6), §7.3 point 4                                                  |

### Major Findings Addressed (10)

| ID         | Finding                                                     | Resolution                                                                                                                                                                                                                                                                      | Affected Sections                                                     |
| ---------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **SEC-3**  | Knowledge Agent scope contradiction (SELECT-only vs INSERT) | Corrected scope to "SELECT on all tables + INSERT on `instruction_updates` only". Updated all occurrences in design.md and design-output.yaml. Added explicit DML prohibition list (UPDATE, DELETE, DROP, ALTER, CREATE, ATTACH).                                               | §3.4, §6.9, §10 (footnote 6), §11.4, design-output.yaml (7 locations) |
| **SEC-4**  | Tool scope restrictions instructional-only                  | Acknowledged as accepted risk (residual threat: Medium) in §11.4. Added regex patterns per agent for allowed commands. Added self-verification checks before each scoped tool call.                                                                                             | §11.4                                                                 |
| **SEC-5**  | Safety Constraint Filter underspecified                     | Formally defined in §5.2 (global-operating-rules.md §9) with: immutable rules list, protected phrases, pattern-matching rejection criteria, evaluator separation (Knowledge Agent proposes, user approves; copilot-instructions.md immutable at runtime).                       | §5.2 (§9)                                                             |
| **SEC-6**  | Unvalidated file paths in SQL tables                        | Added CHECK constraints: `instruction_updates.file_path` limited to `.github/instructions/%` or `.github/copilot-instructions.md`. `artifact_evaluations.artifact_path` prevents path traversal (`NOT LIKE '../%' AND NOT LIKE '/%'`). Added Knowledge Agent self-verification. | §5.3 (§1 DDL), §8.2, §8.3, §11.6, design-output.yaml DDLs             |
| **ARCH-1** | Knowledge Agent responsibility bloat (9 functions)          | Added responsibility inventory table mapping all 9 functions to justification and cohesion group (Learning vs Observability). Documented Phase 2 split consideration for "Pipeline Analyst" agent if needed.                                                                    | §6.9                                                                  |
| **ARCH-2** | SQLite schema evolution strategy absent                     | Added §5.3 §9 covering: ALTER TABLE ADD COLUMN (safe), enum expansion via table recreation, schema_meta version tracking table, migration ownership (Orchestrator Step 0 only).                                                                                                 | §5.3 (§9)                                                             |
| **ARCH-3** | Orchestrator line count target unsubstantiated              | Revised target from ~480 to **~550 lines**. Added compaction itemization table (step descriptions, routing table, intent statement = ~55 lines saved from 604). Documented NFR-1 exception with justification for coordination complexity.                                      | §6.1                                                                  |
| **CORR-3** | SQL template quoting inconsistency                          | Documented explicit quoting convention in §2 template: string fields use `'{sql_escape(value)}'`, nullable fields use `NULL` (unquoted) when absent. Added examples with NULLs.                                                                                                 | §5.3 (§2)                                                             |
| **CORR-4** | Routing matrix duplication                                  | Designated schemas.md as canonical location per AC-13.1. §5.2 (global-operating-rules.md §5) now references schemas.md instead of duplicating the table.                                                                                                                        | §5.2 (§5)                                                             |
| **CORR-7** | Researcher scope narrowing undocumented                     | Added DR-3 documenting AC-11.2 scope narrowing from `research/*.yaml, research/*.md` to `research/*.yaml` only, with DP-2 rationale.                                                                                                                                            | §16, design-output.yaml deviation_records                             |

### Minor Findings Addressed (8)

| ID         | Finding                              | Resolution                                                                                                                                                         |
| ---------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **SEC-7**  | No run_id uniqueness enforcement     | Documented collision risk as low; Phase 2 enhancement noted for `runs` metadata table. Added to §11.7.                                                             |
| **SEC-8**  | Unbounded free-text fields           | Added LENGTH CHECK constraints to all free-text fields in DDL (notes≤1000, missing_information≤2000, inaccuracies≤2000, impact_on_work≤2000, change_summary≤1000). |
| **ARCH-4** | Dual evidence representation drift   | Defined source-of-truth hierarchy in §11.5: SQL is authoritative for routing, YAML is derived summary.                                                             |
| **ARCH-5** | Mutable Tier 1 files coupling        | Established mutability classification in §11.1: copilot-instructions.md = immutable at runtime; pipeline-conventions.md = governed-mutable.                        |
| **ARCH-6** | Single DB as single point of failure | Documented data criticality tiers in §11.7. Added PRAGMA integrity_check at Step 0. Defined corruption recovery priority.                                          |
| **ARCH-7** | Prompt-only enforcement              | Covered by SEC-4 response in §11.4 with accepted risk acknowledgment and regex patterns.                                                                           |
| **CORR-5** | Data flow diagram: 4 writers → 5     | Updated §3.3 to "5 writers" including knowledge-agent. Updated design-output.yaml data_flow.                                                                       |
| **CORR-6** | anvil_checks column count: 14 → 15   | Updated §3.3 annotation to "15 cols".                                                                                                                              |
| **CORR-8** | AC-2.4 model diversity path missing  | Added Phase 2 Enhancement section to dispatch-patterns.md content outline in design-output.yaml.                                                                   |

### Files Modified

- **design.md**: §3.3, §3.4, §4.2, §4.3, §5.2, §5.3, §6.1, §6.8, §6.9, §7.2, §7.3, §8.2, §8.3, §10, §11 (new subsections 11.1–11.7), §16 (DR-3), §18 (this section)
- **design-output.yaml**: architecture.data_flow, architecture.tool_access_matrix, agent_designs (orchestrator target_lines, knowledge-agent scope), shared_documents (sql-templates content_outline, global-operating-rules §9), review_design.evidence_gate_queries, sqlite_schemas (all 4 table DDLs + key_queries), decisions (D-2 rationale), deviation_records (DR-3 added), completion block

---

_Generated by Designer Agent — 2026-02-27 (Revision Round 2)_
