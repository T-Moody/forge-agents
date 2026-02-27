# Adversarial Design Review — Reviewer 2 (gemini-3-pro-preview perspective)

## Overall Verdict: CONCERNS

The v3 design is architecturally sound in its macro structure (lean pipeline, zero-merge memory, SQLite evidence ledger) but contains **4 critical issues**, **6 high-severity issues**, and **5 medium issues** that, if shipped as-is, would cause runtime failures, requirement violations, or silent data corruption. The most alarming problems are: (1) the design's core evidence-verification mechanism depends on orchestrator tools that feature.md explicitly forbids, (2) parallel SQLite writers will deadlock without mandatory WAL mode, (3) a stale v2 code path attempts to read a file that doesn't exist yet, and (4) the baseline cross-check mitigation is physically impossible as described.

---

## Findings

### [Finding 1]: FR-1.6 / CR-4 / FR-5.3 — Design contradicts 4+ feature.md requirements with no formal deviation record

- **Severity:** Critical
- **Area:** §Decision 2 (Orchestrator tools), §Decision 7 (always-3 reviewers), feature.md §FR-1.6, §FR-5.3, §CR-4, §US-1, §TS-2
- **Issue:** The design makes two deliberate deviations from feature.md that it justifies internally but never formally reconciles:
  1. **Orchestrator tools:** FR-1.6 says allowed tools are `agent`, `agent/runSubagent`, `memory`, `read_file`, `list_dir`. The v3 design adds `run_in_terminal` and `get_terminal_output` to the orchestrator's approved tool list. The **entire SQL-primary evidence verification architecture** depends on the orchestrator running `SELECT COUNT(*)` queries via `run_in_terminal`. If FR-1.6 is normative, the orchestrator cannot verify evidence independently — the v3 design's central anti-hallucination claim collapses.
  2. **Always-3 reviewers:** FR-5.3 says "Reviewer count MUST be determined by risk classification: 1 for standard, 3 for Large/🔴." CR-4 also specifies 1 reviewer for standard. US-1 says "1 adversarial reviewer evaluates design (standard risk)." TS-2 says "Verify exactly 1 reviewer is dispatched." The design uses always-3 with justification but creates a normative contradiction — the implementation would **fail** TS-2 as written.
- **Impact:** An implementer following feature.md would build a system that contradicts the design. An implementer following the design would produce a system that fails the feature.md test scenarios. This ambiguity will cause implementation confusion and test failures.
- **Suggested fix:** Either (a) update feature.md to formally supersede FR-1.6, FR-5.3, CR-4, US-1, and TS-2 with the v3 decisions, creating an explicit deviation log, or (b) add a "Feature Spec Deviations" section to the design that lists every requirement the design intentionally overrides, with the rationale and a pointer to which spec items need updating before implementation begins. The design should never silently contradict its normative source.

---

### [Finding 2]: SQLite concurrent write hazard — parallel agents writing to same DB

- **Severity:** Critical
- **Area:** §Decision 6 (Verification Architecture), §Data Storage Strategy, §Decision 3 (Steps 3b, 5-6, 7)
- **Issue:** The design dispatches multiple agents in parallel that all INSERT into the same `verification-ledger.db`:
  - **Step 3b:** 3 adversarial reviewers in parallel → 3 concurrent SQLite writers
  - **Step 6:** Up to 4 verifiers in parallel → 4 concurrent SQLite writers
  - **Step 7:** 3 adversarial reviewers in parallel → 3 concurrent SQLite writers

  SQLite's default journal mode uses file-level locking. Concurrent writes cause `SQLITE_BUSY` errors. The design mentions "WAL mode if needed" in a SQLite operational note, but does not mandate it. Without WAL mode enabled at table creation time, every parallel write phase will randomly fail.

- **Impact:** Non-deterministic `SQLITE_BUSY` errors during every parallel write phase. Agents will fail to INSERT verdicts. Evidence gate queries will return incorrect counts (missing INSERTs). The pipeline will incorrectly block or error out. This is a **runtime crash** in the expected happy path, not an edge case.
- **Suggested fix:** (1) Mandate `PRAGMA journal_mode=WAL;` immediately after `CREATE TABLE IF NOT EXISTS` in the Verifier and Adversarial Reviewer initialization. (2) Add a retry-on-busy strategy for SQL INSERTs (e.g., `PRAGMA busy_timeout = 5000;`). (3) Document that WAL mode is a hard requirement, not a "if needed" option. (4) Add WAL mode initialization to the Verifier's "SQL auto-detection" step.

---

### [Finding 3]: Stale v2 sequence — orchestrator reads plan-output.yaml before it exists

- **Severity:** Critical
- **Area:** §Sequence/Interaction Notes (Standard Feature Flow), approximately line 1580
- **Issue:** The Standard Feature Flow sequence contains:
  ```
  → Orchestrator: Dispatch Designer (Step 3)
      → Designer: Produce design-output.yaml + design.md
      → Orchestrator: Read completion contract
  → Orchestrator: Read plan-output.yaml overall_risk_summary    ← BUG
  → Orchestrator: Dispatch Adversarial Reviewer ×3 (Step 3b)
  ```
  The orchestrator attempts to read `plan-output.yaml` **after Step 3 (Design) and before Step 3b (Design Review)**. But `plan-output.yaml` is produced at **Step 4 (Planning)**, which hasn't happened yet. This is a stale v2 artifact: in v2, the orchestrator needed the risk summary to determine reviewer count (1 vs 3). In v3, reviewer count is always 3, making this read both unnecessary and impossible.
- **Impact:** If implemented literally, the orchestrator will attempt to `read_file` on a non-existent path, get an error, and potentially trigger error handling or crash. Even if treated as a no-op failure, it wastes a tool call and creates confusion.
- **Suggested fix:** Remove the line `→ Orchestrator: Read plan-output.yaml overall_risk_summary` from the Standard Feature Flow sequence. It's dead code from the v2 risk-driven reviewer count logic.

---

### [Finding 4]: Baseline cross-check is physically impossible as described

- **Severity:** Critical
- **Area:** §Decision 6 (Baseline Capture Integration, item 5), §Decision 2 (Verifier detail)
- **Issue:** The design states: "Verifier independently re-runs Tier 1 checks (IDE diagnostics) on the baseline files before running the full cascade, comparing its reading against the Implementer's reported baseline." However:
  1. The Verifier runs **after** the Implementer has modified the files.
  2. The original baseline files no longer exist on disk — they've been overwritten by the implementation.
  3. The Verifier is "read-only for source code" — it cannot `git stash`, `git checkout`, or otherwise restore baseline file state.
  4. IDE diagnostics (`ide-get_diagnostics`) run on the **current** file state, not a historical state.

  The Verifier literally cannot access the pre-implementation file state to independently verify the baseline. The only "baseline" available to the Verifier is what the Implementer self-reported in its implementation report — which is exactly the data this cross-check was supposed to independently verify.

- **Impact:** This mitigation (listed as one of three anti-hallucination layers) provides zero actual independent verification. It creates a false sense of security. The CT Revision Finding #8 claims "Mitigation 2 — Baseline cross-check" as a real defense, but it's an illusion.
- **Suggested fix:** Either (a) have the Implementer create a git commit or tag BEFORE making changes, so the Verifier can `git diff` against it; or (b) have the Implementer `git stash` baseline diagnostics into a file that the Verifier can read; or (c) drop the baseline cross-check claim entirely and be honest that baseline verification relies on the Implementer's self-report, with the actual independent verification being the Verifier's post-implementation cascade compared against the Implementer's baseline report (a weaker but real mitigation); or (d) most robustly, have the orchestrator or a setup step capture `git rev-parse HEAD` before implementation dispatch, and have the Verifier use `git show <commit>:<path>` to inspect baseline files.

---

### [Finding 5]: Design review `task_id` undefined — SQL evidence gate will fail

- **Severity:** High
- **Area:** §Decision 3 (Step 3b), §Decision 6 (Evidence Gating), §Decision 7
- **Issue:** All SQL evidence gate queries use `task_id` as a filter: `WHERE task_id='{task_id}'`. But at Step 3b (Adversarial Design Review), no tasks exist yet — task decomposition happens at Step 4. What `task_id` do design reviewers use when INSERTing their verdicts? What `task_id` does the orchestrator use in its COUNT query?

  The Anvil `anvil_checks` schema was designed for per-task verification in a single-agent context. The design transplants it to a pipeline where some reviews are feature-scoped (design review) rather than task-scoped, but doesn't define how feature-scoped records are identified.

- **Impact:** If each reviewer invents a different `task_id`, the orchestrator's COUNT query won't find them. If they omit `task_id`, the INSERT will fail (NOT NULL constraint). If they all use a convention like `task_id='design-review'`, that convention is never documented. This will cause evidence gate failures at Step 3b on every pipeline run.
- **Suggested fix:** Define an explicit convention for non-task-scoped `task_id` values. E.g., `task_id='{feature_slug}-design-review'` for Step 3b and `task_id='{feature_slug}-code-review'` for Step 7. Document this in the `anvil_checks` schema description and in the Adversarial Reviewer's dispatch parameters. The orchestrator must use the same convention in its queries.

---

### [Finding 6]: Code review receives entire `git diff --staged` — context bomb for large features

- **Severity:** High
- **Area:** §Decision 7 (Phase Placement), §Decision 3 (Step 7)
- **Issue:** Step 7 dispatches adversarial code reviewers with `git diff --staged` as their primary input. For a 20-task / 50+ file feature, the staged diff could be thousands of lines. Each reviewer must parse the entire diff in context. The design has per-task dispatch for Verifiers (bounded context per invocation), but code review is feature-level (unbounded context per invocation).

  The targeted context mechanism (Planner's `relevant_context`) was designed for Implementers, not reviewers. Code reviewers get the entire diff.

- **Impact:** For large features, the adversarial code review will hit context window limits. Reviewers will truncate or miss late-in-diff issues. Critical security vulnerabilities in files at the end of a large diff could be silently skipped.
- **Suggested fix:** Either (a) partition code review by wave (review each wave's diff separately, not the entire feature diff), or (b) partition by risk level (review 🔴 file changes first in a separate pass), or (c) have the Planner pre-partition the diff into review-sized chunks that respect file boundaries, with each reviewer receiving a chunk rather than the full diff. Option (a) is simplest and aligns with the per-wave implementation structure.

---

### [Finding 7]: No explicit git staging step between implementation and code review

- **Severity:** High
- **Area:** §Decision 3 (pipeline flow between Steps 6 and 7), §Decision 7
- **Issue:** Anvil explicitly requires `git add -A` before adversarial review so reviewers can see changes via `git diff --staged` (Anvil Step 5c). The design specifies `git diff --staged` as the code review input (Step 7), but never specifies WHO performs `git add` or WHEN. The Implementer? The Verifier? The Orchestrator (can't — write-restricted)? There's no pipeline step for staging.
- **Impact:** If changes aren't staged, `git diff --staged` returns nothing. All three reviewers see an empty diff. They INSERT `phase='review'` verdicts with no findings. Evidence gate passes (COUNT ≥ 3). Pipeline proceeds with zero actual code review. This is a silent total review bypass.
- **Suggested fix:** Add explicit staging responsibility. Options: (a) Make it the Implementer's final step — after self-check, run `git add -A`. (b) Add it as a sub-step of the Orchestrator at the beginning of Step 7 (orchestrator has `run_in_terminal`). (c) Add it to the Verifier's workflow as a post-verification step. Document this as a mandatory pre-condition for Step 7 in the pipeline structure.

---

### [Finding 8]: Operational Readiness checks dropped from Anvil

- **Severity:** High
- **Area:** §Decision 2 (Agents Removed table), design overall
- **Issue:** Anvil's Step 5d "Operational Readiness" performs three critical checks for Large tasks: observability (error logging with context), degradation (external dependency failure handling), and secrets (hardcoded values that should be env vars/config). These checks are INSERTed into `anvil_checks` as `check_name = 'readiness-{type}'`.

  The design doesn't include these checks in any agent's workflow. The Verifier's tiered cascade (Tier 1-3) covers build/test/diagnostics but has no operational readiness tier. The initial-request.md explicitly says "Retain and extend" Anvil features.

- **Impact:** Loss of production-readiness verification. Hardcoded secrets, swallowed exceptions, and missing degradation handling would pass through the pipeline undetected. These are exactly the class of issues that verification cascades (build/lint/test) don't catch.
- **Suggested fix:** Add Operational Readiness as a **Tier 4** in the Verifier's cascade, run for Large tasks only (matching Anvil's scope). Checks: observability, degradation, secrets. INSERTs use `phase='after'`, `check_name='readiness-{type}'`. This directly ports Anvil's Step 5d into the unified Verifier at minimal design cost.

---

### [Finding 9]: Session recall (`session_store` SQL) dropped from Anvil

- **Severity:** High
- **Area:** §Decision 5 (Memory System Redesign), §Decision 2 (Knowledge Agent)
- **Issue:** Anvil's Step 1b "Recall" queries a `session_store` SQLite database for past regressions, reverts, and failures on the files about to be changed. This is a regression-prevention mechanism that uses historical data to inform current implementation. The design replaces this with VS Code `store_memory` for cross-session knowledge, but `store_memory` is a key-value store for broad patterns — it doesn't provide per-file regression history queryable by file path.

  The design's Knowledge Agent captures knowledge at Step 8 (end of pipeline), but there's no mechanism for Implementers to QUERY past session data at the start of their work. Knowledge capture ≠ knowledge retrieval at implementation time.

- **Impact:** An Implementer modifying `auth.ts` won't know that the last 3 pipeline runs also modified `auth.ts` and two of them caused regressions. The historical context that Anvil uses to prevent repeated mistakes is lost.
- **Suggested fix:** Either (a) add a `session_store` SQLite table that the Knowledge Agent writes to at Step 8, and that Implementers query at the start of each task (add to Implementer tool list and workflow), or (b) have the Planner embed relevant historical context from `store_memory` into each task's `relevant_context` field, or (c) accept this as a conscious simplification and document it as a known capability regression.

---

### [Finding 10]: Who performs reversion on 2 failed fix attempts?

- **Severity:** High
- **Area:** §Decision 6 (Verification-Replan Loop), §Decision 3 (Steps 5-6), FR-4.7
- **Issue:** FR-4.7 requires: "After 2 failed fix attempts for a verification check, the system MUST revert changes for that check and record the failure." Design §Decision 6 says "revert changes for that check, record the failure in the ledger." But no agent is assigned the revert responsibility:
  - The Orchestrator can't modify files (tool restriction).
  - The Verifier is read-only for source code.
  - The Planner creates task schemas but doesn't touch code.
  - The Implementer has already failed twice — is it supposed to revert itself?

  There's also no specification of HOW to revert: `git checkout -- <files>`? `git revert`? Manual file restoration?

- **Impact:** On the third failed verification iteration, the pipeline "proceeds with findings documented" but the broken code remains in the working tree. If the feature has other passing tasks, the broken code from the failed task contaminates the final output. The revert requirement from FR-4.7 is unimplementable as currently designed.
- **Suggested fix:** Assign revert responsibility explicitly to the Implementer as part of its re-dispatch in the replan loop: "On third replan iteration, Implementer receives instruction to revert its changes via `git checkout -- <files>` before proceeding." Alternatively, have the Planner's replan output include a `revert_scope` field listing files to restore, and add revert as the first step of the re-implementation dispatch.

---

### [Finding 11]: Orchestrator context window at 20+ tasks

- **Severity:** Medium
- **Area:** §Pipeline State Model, §Decision 3 (Scaling Note), NFR-3
- **Issue:** The orchestrator tracks ALL pipeline state in-context: completed steps, error logs, task statuses, routing decisions, telemetry observations. For a 20-task feature with 54 dispatches (happy path) or 120+ dispatches (worst case with replan loops), the orchestrator accumulates:
  - 20 implementation report routing decisions
  - 20 verification gate assessments (each requiring SQL output parsing)
  - 6 adversarial review verdict assessments
  - Error logs, retry decisions, approval gate records
  - Context from reading upstream outputs for routing

  The design acknowledges linear scaling (§Dispatch Count) but doesn't address the orchestrator's context accumulation. The orchestrator has no pruning, summarization, or context-offloading strategy.

- **Impact:** For complex features (NFR-3 requires handling 10+ tasks, 50+ files), the orchestrator itself may hit context limits and start dropping early-run state from its window, causing incorrect routing for late-run tasks.
- **Suggested fix:** Add a context management strategy for the orchestrator: (a) periodically summarize completed steps rather than retaining full routing logs, (b) offload completed-step records to a SQLite `pipeline_telemetry` table (the design already defines this table — make the orchestrator write to it for offloading, not just knowledge capture), or (c) define a maximum task count per pipeline invocation with a recommendation to split larger features.

---

### [Finding 12]: Mermaid diagram contains stale "init manifest" reference

- **Severity:** Medium
- **Area:** §Decision 10 (Mermaid Diagram)
- **Issue:** The Mermaid diagram reads `Step 0: Setup<br/>Load schemas, init manifest, check prerequisites`. The pipeline manifest was eliminated in the CT v2 revision (Finding #1). "init manifest" no longer corresponds to any design element.
- **Impact:** Minor but causes implementer confusion — they'll look for manifest initialization code that shouldn't exist. Any literal implementation of the Mermaid description will create a phantom feature.
- **Suggested fix:** Change the Step 0 node text to `Step 0: Setup<br/>Check prerequisites, scan for existing outputs` to match the actual v3 setup description.

---

### [Finding 13]: Design review evidence gate query is unfiltered — fragile for re-runs

- **Severity:** Medium
- **Area:** §Decision 3 (Step 3b), §Decision 6 (Evidence Gate Queries)
- **Issue:** The Step 3b evidence gate query is:
  ```sql
  SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review';
  ```
  The Step 7 evidence gate query is:
  ```sql
  SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND check_name LIKE 'review-code-%';
  ```
  The Step 3b query counts ALL `phase='review'` records. Currently this works because design reviews precede code reviews. But if the pipeline ever needs to re-run design review after a code review round (e.g., in a future extension), the unfiltered query would count code review verdicts toward the design review gate. The Step 7 query is properly filtered; Step 3b is not.
- **Impact:** Low for current design, but creates a latent bug for any future pipeline modification that allows re-running design review. Asymmetric query patterns also create implementation confusion.
- **Suggested fix:** Add `check_name LIKE 'review-design-%'` to the Step 3b query for consistency and future-proofing:
  ```sql
  SELECT COUNT(*) FROM anvil_checks WHERE task_id='{task_id}' AND phase='review' AND check_name LIKE 'review-design-%';
  ```

---

### [Finding 14]: `anvil_checks` schema columns meaningless for review verdicts

- **Severity:** Medium
- **Area:** §Data Storage Strategy (SQLite Schema), §Decision 7 (Adversarial Review)
- **Issue:** The `anvil_checks` table has columns `command TEXT`, `exit_code INTEGER`, `output_snippet TEXT` that are meaningful for tool-based verification checks (e.g., `command='tsc --noEmit'`, `exit_code=0`). But adversarial review verdicts INSERT into the same table. What goes in `command` and `exit_code` for a review verdict? The design never specifies. `command` could be NULL (allowed by schema), `exit_code` could be NULL, but then the schema is semantically overloaded — different rows have fundamentally different column interpretations.
- **Impact:** Implementers will handle review INSERT columns inconsistently. Queries that join or filter on `command`/`exit_code` will produce confusing results for review rows. The "one table for all evidence" simplification creates semantic ambiguity.
- **Suggested fix:** Either (a) document explicit column values for review verdicts (e.g., `command='adversarial-review'`, `exit_code=NULL`, `output_snippet='approve|needs_revision|blocker'`), or (b) add a `record_type` column (`CHECK(record_type IN ('verification', 'review'))`) to disambiguate, or (c) accept the overload but document it explicitly in the `anvil_checks` schema description with examples for both verification and review rows.

---

### [Finding 15]: Knowledge Agent's governed instruction updates — undefined safety filter

- **Severity:** Medium
- **Area:** §Decision 2 (Knowledge Agent detail)
- **Issue:** The design states: "Governed updates: Instruction file modifications require explicit approval in interactive mode; logged in autonomous mode." It also mentions the Knowledge Agent inherits from "R-Knowledge + Anvil Learn step" and has a "safety constraint filter prevents weakening existing security checks."

  However, the safety filter is never defined. What constitutes "weakening"? What counts as a "security check"? In autonomous mode, governed updates are merely "logged" — there's no gating mechanism to prevent a Knowledge Agent from modifying instruction files in ways that weaken the pipeline's own security posture.

- **Impact:** In autonomous mode, the Knowledge Agent could modify `.github/instructions/` files to remove security constraints, weaken review requirements, or alter agent behavior — with only a log entry as evidence. This is a self-modification vector with no runtime guard.
- **Suggested fix:** Define the safety filter explicitly: (a) list patterns that constitute "weakening" (removing severity escalations, relaxing retry budgets, removing tool restrictions), (b) in autonomous mode, restrict the Knowledge Agent to append-only operations on instruction files (no deletions, no modifications of existing lines), or (c) restrict instruction file modifications to interactive mode only (require approval always, not just in interactive mode).

---

## Requirement Coverage Assessment

| Requirement                         | Status                      | Notes                                                        |
| ----------------------------------- | --------------------------- | ------------------------------------------------------------ |
| CR-1 (Deterministic)                | ✅ Covered                  | Pipeline is deterministic                                    |
| CR-2 (Completion contracts)         | ✅ Covered                  | All 9 agents have contracts                                  |
| CR-3 (Parallel dispatch)            | ✅ Covered                  | Pattern A with cap of 4                                      |
| CR-4 (Adversarial review)           | ⚠️ **Overridden**           | Always-3 contradicts CR-4's "1 for standard" — see Finding 1 |
| CR-5 (Evidence gating)              | ⚠️ **Partially overridden** | "1 for standard" in review counts — see Finding 1            |
| CR-6 (Risk classification)          | ✅ Covered                  | Per-file 🟢/🟡/🔴                                            |
| CR-7 (Justification scoring)        | ✅ Covered                  | Decision records with confidence                             |
| CR-8 (Approval mode)                | ✅ Covered                  | 2 gates with structured prompts                              |
| CR-9 (No pending step)              | ✅ Covered                  | No pending steps                                             |
| CR-10 (Output directory)            | ✅ Covered                  | NewAgents/.github/                                           |
| CR-11 (Self-verification)           | ✅ Covered                  | Agent template includes it                                   |
| CR-12 (Anti-drift anchors)          | ✅ Covered                  | Agent template includes it                                   |
| CR-13 (Unified severity)            | ✅ Covered                  | Blocker/Critical/Major/Minor                                 |
| CR-14 (Pushback)                    | ✅ Covered                  | Spec agent with ask_user                                     |
| CR-15 (Bounded retries)             | ✅ Covered                  | All loops bounded                                            |
| FR-1.6 (Orchestrator tools)         | ❌ **Violated**             | Design adds run_in_terminal — see Finding 1                  |
| FR-4.7 (Revert on failure)          | ❌ **Unimplementable**      | No agent assigned revert responsibility — see Finding 10     |
| FR-5.3 (Risk-driven reviewer count) | ❌ **Overridden**           | Always 3 instead of 1/3 — see Finding 1                      |
| NFR-3 (Scalability)                 | ⚠️ **Risk**                 | Context window concerns at 20+ tasks — see Findings 6, 11    |

---

## Summary Statistics

| Severity  | Count  |
| --------- | ------ |
| Critical  | 4      |
| High      | 6      |
| Medium    | 5      |
| **Total** | **15** |

The critical issues (Findings 1-4) must be resolved before implementation. Findings 2, 3, and 4 will cause runtime failures in the happy path. Finding 1 is a normative contradiction that will paralyze implementation decisions.

The high issues (Findings 5-10) represent missing specifications that will require implementer judgment calls, leading to inconsistent results across agents.

The medium issues (Findings 11-15) are design debts that can be deferred but should be tracked.
