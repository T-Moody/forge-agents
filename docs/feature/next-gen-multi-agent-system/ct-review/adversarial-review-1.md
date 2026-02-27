# Adversarial Design Review — Reviewer 1 (gpt-5.3-codex perspective)

## Overall Verdict: CONCERNS

12 findings identified across 8 focus areas. 2 Critical, 5 High, 4 Medium, 1 Low. No blocking issues that invalidate the architecture, but several bugs in the SQLite schema and evidence gating logic that will cause runtime failures if shipped as-is.

---

## Findings

### [Finding 1]: `anvil_checks` schema missing `verdict` / `severity` column — security blocker policy unenforceable via SQL

- **Severity:** Critical
- **Area:** §Decision 6 (Verification Architecture — SQL Schema), §Decision 7 (Security Blocker Policy), §Data Storage Strategy
- **Issue:** The `anvil_checks` table stores review verdicts with `phase='review'` and `passed INTEGER NOT NULL CHECK(passed IN (0, 1))`. But the security blocker policy requires detecting `severity: Blocker` on any review finding to halt the pipeline. The `passed` column is binary (0/1) — it cannot distinguish between a reviewer who found a minor concern (`passed=0`) and one who found a security Blocker (`passed=0`). The orchestrator's SQL COUNT queries (`SELECT COUNT(*) ... WHERE phase='review'`) can only verify that records EXIST, not whether any verdict is a Blocker. To enforce the security blocker policy, the orchestrator must fall back to reading Markdown findings files or the in-context completion contract — defeating the purpose of SQL-primary evidence gating. The Disagreement Resolution Protocol at §Decision 7 specifies "Any model finds security Blocker → Pipeline ERROR" but provides no SQL mechanism to detect this.
- **Impact:** The pipeline's most critical safety invariant (security Blocker = immediate halt) has no SQL-level enforcement path. If the orchestrator relies on SQL for gating (as designed), it CANNOT detect security blockers. If it reads Markdown files, it's back to prompt-level parsing — the exact problem SQLite was supposed to solve. On pipeline resume, the in-context completion contract is lost and only SQL + Markdown persist; the security blocker signal is unrecoverable from SQL alone.
- **Suggested fix:** Add `verdict TEXT CHECK(verdict IN ('approve', 'needs_revision', 'blocker'))` and `severity TEXT CHECK(severity IN ('Blocker', 'Critical', 'Major', 'Minor'))` columns to `anvil_checks`. Update review evidence gate to: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND verdict='blocker';` — if >0, pipeline ERROR. Update minimum gate to: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND verdict IS NOT NULL;` ≥3.

---

### [Finding 2]: Design review evidence gate query accumulates stale records across revision rounds

- **Severity:** Critical
- **Area:** §Decision 3 (Step 3b), §Decision 6 (Evidence Gating Mechanism)
- **Issue:** The design review evidence gate runs: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review';` — must be ≥3. But if design review round 1 produces 3 records (e.g., all NEEDS_REVISION), the designer revises, and round 2 is dispatched, the 3 round-1 records still exist. If round 2 has only 2 of 3 reviewers complete (1 errors out), the total COUNT is 5, which passes the ≥3 gate — even though the current round is incomplete. Similarly, after code review round 1 (Step 7) and round 2, the Step 7 gate at `check_name LIKE 'review-code-%'` accumulates records from both rounds. The schema has no `round` or `iteration` column to distinguish review cycles. This means gates can pass on stale data from prior rounds, masking incomplete or failing current rounds.
- **Impact:** A design revision loop or code review fix-and-re-review cycle can pass the evidence gate with stale verdicts from a prior round. This silently degrades quality assurance — the gate certifies "3 reviewers checked this" when only 2 checked the current version.
- **Suggested fix:** Add `round INTEGER NOT NULL DEFAULT 1` column to `anvil_checks`. Each review cycle increments the round. Evidence gate queries filter on the current round: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND round={current_round};` ≥3. Alternatively, DELETE stale records at the start of each new round (simpler but loses audit trail).

---

### [Finding 3]: Concurrent SQLite writes will cause SQLITE_BUSY failures in default journal mode

- **Severity:** High
- **Area:** §Data Storage Strategy (SQLite Operational Notes), §Decision 3 (Steps 3b, 5, 6, 7)
- **Issue:** The design states: "SQLite handles this with WAL mode if needed, but typical pipeline concurrency is low enough for default locking." This is incorrect. In default journal mode (journal_mode=DELETE), SQLite allows only ONE writer at a time. Writers that collide get `SQLITE_BUSY` immediately (default `busy_timeout` is 0). The design has these concurrent write scenarios:
  - Step 3b: 3 adversarial reviewers in parallel, each INSERTing into `anvil_checks`
  - Step 5: ≤4 implementers in parallel, each INSERTing baseline records into `anvil_checks`
  - Step 6: ≤4 verifiers in parallel, each INSERTing cascade results into `anvil_checks`
  - Step 7: 3 adversarial reviewers in parallel, each INSERTing into `anvil_checks`
  - All parallel agents also INSERT into `pipeline_telemetry`

  With 4 concurrent Verifiers each performing ~8 INSERTs, SQLITE_BUSY is near-certain. The agents run SQL via `run_in_terminal` (shell commands), which have no built-in retry on SQLITE_BUSY. No agent definition specifies SQLITE_BUSY handling.

- **Impact:** Parallel agents will intermittently fail to write verification evidence. Lost INSERTs mean the evidence gate fails (insufficient count), triggering unnecessary replan loops. Or worse: the agent retries the terminal command, gets a different error, and reports ERROR — cascading to pipeline failure on what should be a transient concurrency issue.
- **Suggested fix:** Mandate WAL mode in the SQLite initialization: `PRAGMA journal_mode=WAL;` immediately after `CREATE TABLE IF NOT EXISTS`. WAL allows concurrent reads with one writer and dramatically reduces SQLITE_BUSY frequency. Additionally, set `PRAGMA busy_timeout=5000;` (5 seconds) so that concurrent writers queue rather than fail immediately. Add these PRAGMAs to the Verifier's "SQL auto-detection" initialization step and to every agent that writes to SQLite. Document in `schemas.md` that all SQLite access MUST begin with these PRAGMAs.

---

### [Finding 4]: No `pipeline_run_id` — verification records from resumed/re-run pipelines are indistinguishable

- **Severity:** High
- **Area:** §Decision 6 (SQL Schema), §Failure & Recovery (Partial Pipeline Recovery), §Pipeline State Model
- **Issue:** The `anvil_checks` and `pipeline_telemetry` tables have no `pipeline_run_id` or `session_id` column. The `verification-ledger.db` persists across pipeline runs (stated as "Pipeline lifetime + potential cross-session" in §Decision 5). On pipeline resume (EC-5), the orchestrator re-dispatches failed agents, which INSERT new records into the same tables. Without a run identifier, there is no way to distinguish: (a) records from the original run vs. the resumed run, (b) records from a completely fresh re-run if the user starts over without deleting the DB, (c) duplicate check records if a Verifier is re-dispatched. The COUNT-based gates will over-count when stale + fresh records coexist. `pipeline_telemetry` averages and totals are corrupted by mixing runs.
- **Impact:** Pipeline resume produces incorrect evidence gate results (inflated counts). Telemetry is unreliable for post-mortem analysis. There's no clean way to audit which records belong to which execution attempt.
- **Suggested fix:** Add `run_id TEXT NOT NULL` to both tables. Generate `run_id` at Step 0 (e.g., ISO8601 timestamp or UUID). Pass `run_id` to every agent via dispatch parameters. All INSERT statements include `run_id`. Evidence gate queries filter: `WHERE run_id='{current_run_id}'`. On resume, the orchestrator generates a new `run_id` for the resumed segment, or reuses the original `run_id` and deletes records from the failed step before re-dispatch.

---

### [Finding 5]: Design-review `task_id` semantics undefined — ambiguous for feature-level reviews

- **Severity:** High
- **Area:** §Decision 3 (Step 3b, Step 7), §Decision 6 (Evidence Gating)
- **Issue:** The `anvil_checks.task_id` conceptually comes from Anvil, where it's a per-task slug (e.g., `fix-login-crash`). But design review (Step 3b) and code review (Step 7) operate at feature-level, not task-level. The design shows evidence gate queries using `task_id='{task_id}'` for both per-task verification (Step 6) and feature-level review (Steps 3b/7). What value does `task_id` take for design reviews? Options: the feature slug, a synthetic `design-review` string, or something else — the design doesn't say. If design review and verification share the same `task_id` as the feature slug, their `phase='review'` records collide with per-task verification records in COUNT queries. If they use different `task_id` values, the orchestrator must know the mapping.
- **Impact:** Without defined `task_id` semantics for feature-level reviews, the evidence gate queries will either return wrong counts (if task_id collides with per-task records) or fail (if the orchestrator queries with the wrong task_id). Implementers will have to guess what task_id to use for review INSERTs.
- **Suggested fix:** Define explicit `task_id` conventions: per-task operations use the Planner-assigned task ID (e.g., `task-03`). Feature-level operations (design review, code review) use `{feature_slug}-design-review` or `{feature_slug}-code-review`. Document this convention in `schemas.md` and in the Adversarial Reviewer agent definition.

---

### [Finding 6]: `anvil_checks` table missing index on `task_id` — O(n) full table scans on every gate check

- **Severity:** High
- **Area:** §Data Storage Strategy (SQLite Schema Details)
- **Issue:** Every evidence gate query filters on `task_id` (`WHERE task_id='{task_id}'`). The `anvil_checks` table has no index on `task_id`. For a 20-task feature with ~8 checks per task across 3 phases (baseline, after, review), the table will have ~480+ rows. Each gate check performs a full table scan. The orchestrator runs gate checks per-task (up to 20 times for verification alone), plus 2+ times for review gates. This is ~22+ full scans of a growing table. While SQLite is fast for small datasets, this is avoidable waste that compounds with feature complexity and replan loops (worst case: 3 iterations × 20 tasks × ~8 checks = ~1440 rows).
- **Impact:** Performance degradation on large features. Not catastrophic but easily preventable. The `pipeline_telemetry` table has the same issue but is less frequently queried.
- **Suggested fix:** Add after table creation: `CREATE INDEX IF NOT EXISTS idx_anvil_task_phase ON anvil_checks(task_id, phase);` and `CREATE INDEX IF NOT EXISTS idx_telemetry_step ON pipeline_telemetry(step);`. Document in the initialization SQL block alongside `CREATE TABLE`.

---

### [Finding 7]: Adversarial Reviewer has no YAML output — breaks "typed YAML at every boundary" contract

- **Severity:** High
- **Area:** §Decision 2 (Agent Detail #8 — Adversarial Reviewer), §Decision 4 (Communication & State), §Schemas Reference
- **Issue:** Every other agent produces a typed YAML output file used for routing (e.g., `spec-output.yaml`, `verification-reports/<task-id>.yaml`). The Adversarial Reviewer produces only: (a) Markdown findings at `review-findings/<scope>-<model>.md`, and (b) SQL INSERT of verdict records into `anvil_checks`. Neither is a typed YAML output. The orchestrator's routing logic for review results depends on: reading 3 separate Markdown files to determine verdicts, or reading SQL `passed` column (which can't distinguish verdicts — see Finding 1), or reading the in-context completion contract (lost on resume). This breaks the stated design principle: "Typed YAML schemas at every boundary" (§Decision 1). The Schemas Reference (§10, `review-findings`) describes the schema as "Per-model findings in Markdown + SQL INSERT" — confirming no YAML. On pipeline resume, review routing state is unrecoverable without parsing Markdown.
- **Impact:** The orchestrator cannot determine review verdicts through the same typed YAML mechanism used for all other agents. It must fall back to Markdown parsing (unreliable, the exact problem YAML was supposed to solve) or SQL queries (insufficient — see Finding 1). Pipeline resume cannot reconstruct review routing decisions.
- **Suggested fix:** Add a per-scope YAML verdict summary file, e.g., `review-verdicts/<scope>.yaml`, produced by the orchestrator after collecting all 3 reviewer results (or by each reviewer individually alongside its Markdown). Schema: `{scope, round, verdicts: [{model, verdict, severity, findings_count}], aggregate_verdict, security_blockers}`. This preserves the typed-YAML-at-every-boundary contract and enables deterministic resume.

---

### [Finding 8]: `CREATE TABLE IF NOT EXISTS` race condition during parallel initialization

- **Severity:** Medium
- **Area:** §Decision 6 (Verifier — SQL auto-detection), §Data Storage Strategy (SQLite Operational Notes)
- **Issue:** The design states: "Verifier initializes SQLite verification ledger using `CREATE TABLE IF NOT EXISTS` on first dispatch." But in Step 5, up to 4 Implementers run in parallel, each needing to INSERT baseline records into `anvil_checks`. If no prior agent has created the table, multiple Implementers may simultaneously attempt `CREATE TABLE IF NOT EXISTS`. In default journal mode, one succeeds and the others get SQLITE_BUSY (the CREATE statement takes an exclusive lock). The failing Implementers have no retry logic specified for this case. Additionally, the `pipeline_telemetry` table initialization has the same race — every agent writes to it, but no single agent is responsible for creating it.
- **Impact:** First parallel dispatch wave (Researchers at Step 1, if they write telemetry) or Implementers (Step 5) could fail on table initialization. The failure is transient and would be fixed by retry, but no retry logic for SQLITE_BUSY is specified in agent definitions.
- **Suggested fix:** Move SQLite initialization (both tables + PRAGMAs + indexes) to Step 0 (Setup), executed by the orchestrator via `run_in_terminal` before any agent dispatch. This guarantees the schema exists before any parallel agent runs. The orchestrator already has `run_in_terminal` in its tool list (v3). Single-point initialization eliminates all CREATE TABLE races.

---

### [Finding 9]: Evidence gate queries for review don't filter on `passed` — counts all records regardless of verdict

- **Severity:** Medium
- **Area:** §Decision 6 (Evidence Gating Mechanism — Key Queries), §Decision 3 (Step 3b, Step 7 — Gate Points)
- **Issue:** The verification evidence gate correctly filters `AND passed=1`: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='after' AND passed=1;`. But the review evidence gate does NOT filter on `passed`: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review';`. This means if all 3 reviewers find issues (NEEDS_REVISION) and INSERT records with `passed=0`, the COUNT is still 3, and the ≥3 gate passes. The gate verifies "3 reviewers submitted" but not "3 reviewers approved." The design says the gate condition is "≥3 + no NEEDS_REVISION" (Gate Points table, line 367), but the "no NEEDS_REVISION" check comes from completion contracts (in-context), not from SQL. The SQL query alone is insufficient for the stated gate condition.
- **Impact:** The SQL-based evidence gate is weaker than documented. The "independent verification via SQL" claim is only half-true — SQL verifies record count, but verdict assessment still requires reading completion contracts or Markdown files. On resume, the SQL-only gate would pass even when all reviewers found issues.
- **Suggested fix:** With the `verdict` column from Finding 1's fix: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND round={current_round} AND verdict='approve';` ≥ majority threshold. Or at minimum: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND passed=1;` to ensure the gate only counts approvals.

---

### [Finding 10]: Design review gate query at Step 3b doesn't filter by `check_name` — will count code review records after Step 7

- **Severity:** Medium
- **Area:** §Decision 3 (Step 3b vs Step 7 evidence gates)
- **Issue:** The Step 3b evidence gate query is: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review';` — no `check_name` filter. The Step 7 evidence gate query is: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND check_name LIKE 'review-code-%';` — filters by `check_name`. This asymmetry means: if a pipeline has a design revision loop (Step 3b → Step 3 → Step 3b), the second run of Step 3b's gate query will count BOTH design and code review records (if they share `task_id`). More practically: the design review gate is under-specified compared to the code review gate. Step 3b should filter `check_name LIKE 'review-design-%'` for consistency and correctness.
- **Impact:** Gate query inconsistency between design and code review. Not a current runtime bug (design review runs before code review in the happy path), but becomes a bug if the pipeline structure ever changes or if `task_id` values overlap between design and per-task contexts.
- **Suggested fix:** Update Step 3b gate to: `SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND check_name LIKE 'review-design-%';` — consistent with Step 7's pattern.

---

### [Finding 11]: Contradiction with prior orchestrator-tool-restriction feature on `run_in_terminal` policy

- **Severity:** Medium
- **Area:** §Decision 2 (Orchestrator tools), §v3 Revision Log (Correction #1)
- **Issue:** The `orchestrator-tool-restriction` feature (docs/feature/orchestrator-tool-restriction/) was a deliberate, reviewed decision to PROHIBIT `run_in_terminal` for the orchestrator. That feature's design explicitly lists `run_in_terminal` as one of 8 prohibited tools, with rationale: "all file writes are delegated to subagents." The v3 design reverses this decision (adding `run_in_terminal` and `get_terminal_output` to the orchestrator's tool list) without acknowledging the contradiction or providing justification for why the prior analysis was wrong. The v3 Revision Log says "v2 incorrectly assumed `run_in_terminal` was forbidden" — but it wasn't an assumption; it was a deliberate design decision in a prior feature. This design creates a fork: the existing Forge orchestrator prohibits `run_in_terminal`, while the next-gen orchestrator allows it.
- **Impact:** Future maintainers may be confused by contradictory tool policies across the Forge and next-gen system. If the `orchestrator-tool-restriction` feature's rationale (reducing orchestrator blast radius, preventing file writes) was sound, then adding `run_in_terminal` to the next-gen orchestrator reintroduces the risks that feature was designed to mitigate. If the prior feature was wrong, that should be explicitly stated.
- **Suggested fix:** Add a decision record in §Decision 2 or §v3 Revision Log explicitly acknowledging the divergence from `orchestrator-tool-restriction` and stating why the tradeoff is different for the next-gen system (SQL evidence gating provides sufficient value to justify the increased orchestrator blast radius). Alternatively: delegate SQL queries to the Verifier or a lightweight "gate-checker" agent, preserving the orchestrator's read-only posture.

---

### [Finding 12]: No `output_snippet` length constraint — unbounded TEXT column can bloat SQLite

- **Severity:** Low
- **Area:** §Data Storage Strategy (SQLite Schema — `anvil_checks` table)
- **Issue:** The Anvil source (anvil.agent.md) mentions "the first 500 chars of command output" for `output_snippet`, but the `anvil_checks` schema has `output_snippet TEXT` with no length constraint. The design doesn't carry forward Anvil's 500-char truncation rule. If a build or test command produces verbose output (e.g., full stack traces, large test suite results), the agent may INSERT the entire output. With ~480+ rows for a 20-task feature, the DB file can grow significantly. SQLite handles large TEXT values but query performance degrades.
- **Impact:** Minor — SQLite handles this gracefully for typical usage. Could become a problem for extremely verbose build systems or test runners. The bigger issue is that losing the 500-char truncation rule means the design is less specified than Anvil for this behavior.
- **Suggested fix:** Document in the Verifier and Implementer agent definitions that `output_snippet` MUST be truncated to the first 500 characters. Optionally add a CHECK constraint: `output_snippet TEXT CHECK(LENGTH(output_snippet) <= 500)`.

---

## Focus Area Summary

### 1. SQLite Schema Design

**Status: BUGS FOUND.** Missing `verdict`/`severity` columns (Finding 1), missing `round` column (Finding 2), missing `run_id` (Finding 4), missing index (Finding 6), unbounded `output_snippet` (Finding 12). The `anvil_checks` schema was adopted from Anvil where it supports a single-agent single-task flow, but it's insufficient for the multi-agent multi-round pipeline use case.

### 2. Evidence Gating

**Status: BUGS FOUND.** COUNT gates on reviews don't filter `passed` (Finding 9), stale round accumulation (Finding 2), design review gate missing `check_name` filter (Finding 10), security blocker undetectable via SQL (Finding 1). The SQL-primary gating claim is undermined by these gaps — the orchestrator still needs to read completion contracts for verdict assessment.

### 3. Orchestrator SQL via `run_in_terminal`

**Status: CONCERNS.** Functional but contradicts prior `orchestrator-tool-restriction` decision (Finding 11). SQL injection risk via string interpolation (noted by ct-security, inherited from Anvil, not addressed). No SQLITE_BUSY retry logic specified.

### 4. Per-task Verification Dispatch

**Status: SOUND.** The per-task dispatch design bounds context well. The main issue is concurrent writes to shared SQLite (covered under #5).

### 5. Concurrent Access

**Status: BUGS FOUND.** Default SQLite journal mode will cause SQLITE_BUSY on concurrent writes (Finding 3). CREATE TABLE race condition (Finding 8). No agent-level retry logic for database lock errors.

### 6. Pipeline Recovery

**Status: CONCERNS.** Recovery is directory-scan-based (YAML files), not SQLite-based. No `run_id` to distinguish runs (Finding 4). Duplicate INSERTs on re-dispatch (no upsert/cleanup). Adversarial review verdicts not recoverable from SQL or YAML on resume (Finding 7).

### 7. Adversarial Review Mechanism

**Status: CONCERNS.** Well-specified structurally, but verdict INSERT schema is too coarse for routing decisions (Finding 1), no typed YAML output for resume (Finding 7), `task_id` undefined for feature-level reviews (Finding 5). Critical (non-security) findings from 1/3 models are logged but not acted on — by design, but worth acknowledging as a quality tradeoff.

### 8. Typed YAML + SQLite Dual-Track Data Flow

**Status: CONCERNS.** Adversarial Reviewer is the one agent that breaks the typed-YAML-at-every-boundary contract (Finding 7). The dual-track design is conceptually sound but has a gap: SQL provides insufficient verdict granularity for routing, and Markdown provides no structured parsing. The orchestrator must use THREE formats (in-context completion contracts for verdicts, SQL for counts, Markdown for details) instead of the documented two.
