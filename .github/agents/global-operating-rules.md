# Global Operating Rules

Shared reference document for all pipeline agents. Agents reference specific sections (e.g., "See global-operating-rules.md §3") instead of inlining these rules.

---

## §1 Two-Tiered Retry Policy

Retries operate at two independent levels:

1. **Agent-internal (Level 1):** Each agent retries transient errors up to **2 times** with exponential backoff (1s, 2s). Deterministic failures are NEVER retried.
2. **Orchestrator-level (Level 2):** The orchestrator retries a failed agent invocation **1 time** per dispatch. If the agent returns ERROR after the orchestrator retry, the pipeline step fails.

**Transient errors** (retry-eligible): Network timeouts, tool temporarily unavailable, `SQLITE_BUSY`, database locked, HTTP 429/502/503.

**Deterministic errors** (never retry): Schema validation failure, missing required input file, permission denied, malformed YAML, unknown field values.

> **Classification rule:** If the same input would produce the same failure on re-execution, the error is deterministic. Do not retry.

## §2 Error Categories

| Category      | Examples                                           | Action                              |
| ------------- | -------------------------------------------------- | ----------------------------------- |
| Transient     | Network timeout, tool unavailable, DB locked       | Retry up to 2x with backoff         |
| Deterministic | Schema violation, missing input, invalid enum      | Report immediately, do not retry    |
| Security      | Secrets in code, vulnerable dependency, PII leak   | Flag severity: critical in findings |
| Scope         | File outside task boundary, unauthorized tool call | Abort operation, report in output   |

When logging errors: include the error category, the tool/command that failed, and the first 500 characters of output. Never log secrets, credentials, or PII.

## §3 SQLITE_BUSY Handling

All agents writing to `verification-ledger.db` MUST handle `SQLITE_BUSY`:

1. **Database-level prevention:** WAL mode (`PRAGMA journal_mode=WAL`) and `PRAGMA busy_timeout=5000` are set at Step 0 initialization.
2. **Application-level retry:** If `SQLITE_BUSY` occurs despite WAL mode, retry **3 times** with exponential backoff:
   - Attempt 1: wait 1 second
   - Attempt 2: wait 2 seconds
   - Attempt 3: wait 4 seconds
3. **After 3 failures:** Report as a transient error in the agent's completion block. Do not escalate to deterministic.

> **Note:** With WAL mode + busy_timeout, `SQLITE_BUSY` should be rare. If it occurs frequently, it indicates a concurrency issue — log for Knowledge Agent post-mortem analysis.

## §4 run_id Protocol

The `run_id` is the universal namespace filter for all pipeline data.

- **Generated:** By the Orchestrator at Step 0 (ISO 8601 timestamp format).
- **Propagated:** Passed to every agent as a parameter. Every agent includes `run_id` in all SQL INSERTs.
- **Filtered:** All SQL SELECT queries MUST include `WHERE run_id = '{run_id}'` to scope results to the current pipeline run.
- **Immutable:** Once generated, the `run_id` never changes during a pipeline run.

## §5 Completion Contract Routing Matrix

> **Canonical location:** `schemas.md §Routing Matrix`

All agents return a `completion` block with `status`, `summary`, `severity`, `findings_count`, `risk_level`, and `output_paths`. Valid statuses are:

- **DONE** — Agent completed successfully. Proceed to next step.
- **NEEDS_REVISION** — Agent completed but found issues requiring upstream revision (only Planner and Verifier support this status).
- **ERROR** — Agent encountered an unrecoverable failure. Orchestrator applies retry logic.

See **schemas.md §Routing Matrix** for the full agent × status table defining which agents support which statuses and the orchestrator's routing action for each combination. Do NOT duplicate that table here.

## §6 Self-Verification Common Checklist

Every agent MUST run these checks before returning its output:

- [ ] Output YAML contains `agent_output` with all required common header fields
- [ ] `schema_version` is `"1.0"`
- [ ] `agent_output.agent` matches the agent's own name
- [ ] `agent_output.step` matches the expected pipeline step
- [ ] `completion` block is present with `status`, `summary`, `severity`, `findings_count`, `risk_level`, `output_paths`
- [ ] `output_paths` lists all files created by this agent
- [ ] All output files actually exist at the declared paths
- [ ] No files outside the agent's declared scope were created or modified

Agents with additional self-verification items define them in their own instruction files, referencing this common checklist as the baseline.

## §7 Pipeline State Recovery (EC-5)

If the orchestrator is interrupted mid-pipeline, it recovers state on restart:

1. **Scan** the feature directory (`docs/feature/<slug>/`) for existing output files.
2. **Read** the `completion` block from each found output file.
3. **Resume** from the first incomplete step — the earliest step without a `status: DONE` completion block.
4. **Skip** steps that have already completed successfully (their output files exist with valid completion blocks).

Agents MUST write their output files atomically (complete YAML, not partial) so the orchestrator can reliably parse completion blocks during recovery.

## §8 Output Naming Conventions

All pipeline outputs use deterministic paths under the feature directory:

```
docs/feature/<slug>/
├── research/              # Researcher outputs (*.yaml, *.md)
├── spec-output.yaml       # Spec Agent output
├── design-output.yaml     # Designer output
├── plan-output.yaml       # Planner output
├── tasks/                 # Planner task definitions (TASK-*.yaml)
├── implementation-reports/ # Implementer reports (TASK-*.yaml)
├── verification-reports/  # Verifier reports (TASK-*.yaml)
├── review-findings/       # Adversarial Reviewer findings (<scope>-<perspective>.md)
├── review-verdicts/       # Adversarial Reviewer verdicts (<scope>-<perspective>.yaml)
├── knowledge-output.yaml  # Knowledge Agent output
└── evidence-bundle.md     # Knowledge Agent evidence bundle
```

Agents MUST use these exact paths. The orchestrator validates output paths against this convention.

## §9 Governed Instruction Updates & Safety Constraint Filter

### Update Modes

- **Interactive mode:** Knowledge Agent proposes changes and presents them to the user. Only user-approved changes are applied (`applied=1` in `instruction_updates` table).
- **Autonomous mode:** Knowledge Agent proposes changes and logs them to `instruction_updates` with `applied=0`. Changes are NEVER auto-applied. They are available for user review in the next interactive session.

### Safety Constraint Filter — Formal Definition

All proposed instruction file updates MUST pass the Safety Constraint Filter before application. This filter enforces the following rules:

#### 1. Immutable Rules (NEVER removable or weakened)

These rules are permanently protected and cannot be modified, weakened, or removed by any agent:

- **Terminal-only testing** — All test execution via `run_in_terminal` with CLI commands
- **No-file-redirect** — Never redirect terminal output to files
- **Agent boundary integrity** — No agent may modify another agent's definition at runtime
- **Bounded loop invariants** — All feedback loops have hard iteration limits
- **Evidence-first principle** — Record SQL evidence before claiming verification

#### 2. Protected Phrase Patterns

The following phrases MUST NOT be deleted or weakened in any instruction file. Any update that removes text matching these patterns is **automatically rejected**:

- `"MUST use run_in_terminal"`
- `"NEVER redirect"`
- `"NEVER modify another agent"`
- `"hard iteration limits"`
- `"INSERT before reporting"`
- Any phrase containing: `MUST NOT`, `NEVER`, `RESTRICTED`, `PROHIBITED`

#### 3. Pattern-Matching Rejection Criteria

Updates are **automatically rejected** if they match any of these patterns:

- Removal of text matching a protected phrase → **REJECTED**
- Adding `ALLOWED` to a previously `RESTRICTED` tool → **REJECTED**
- Increasing loop/retry limits beyond spec-defined bounds → **REJECTED**
- Removing `CHECK` constraints or validation rules → **REJECTED**
- Adding `runtime`, `dynamic`, or `self-modify` in the context of agent definitions → **REJECTED**

#### 4. Evaluator Separation

- The **Knowledge Agent** proposes changes (actor). It MUST NOT approve its own changes.
- The **user** (interactive mode) or **orchestrator** (autonomous logging) evaluates changes.
- `.github/copilot-instructions.md` is **immutable at runtime** — only modifiable via feature implementation PRs, never by any agent at runtime.
- `.github/instructions/pipeline-conventions.md` is **governed-mutable** — updatable by Knowledge Agent with safety filters applied; all changes logged to the `instruction_updates` table.
