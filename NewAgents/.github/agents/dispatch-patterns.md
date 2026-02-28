---
name: dispatch-patterns
description: Dispatch pattern definitions for pipeline agent coordination
---

# Dispatch Patterns — Reference

This is a reference document for the orchestrator. Contains full definitions of reusable
dispatch patterns for subagent invocations and the replan loop.

---

## Pattern A — Fully Parallel

Used by: Research (Step 1), Design Review (Step 3b), Implementation waves (Step 5), Verification waves (Step 6), Code Review (Step 7).

1. Dispatch N subagents in parallel (N ≤ concurrency cap of 4).
2. Wait for all N to return.
3. Handle individual errors: retry once per failed agent (1 retry per agent, orchestrator-level).
4. Evaluate gate condition: **≥M of N must return `status: DONE`** (M varies by step — see Pattern Usage Table).
5. If gate condition met → pattern DONE; orchestrator proceeds.
6. If gate condition NOT met after retries → pattern ERROR for that step.

### Gate Condition Details

| Dispatch         | N (total) | M (required DONE) | Gate Expression              |
| ---------------- | --------- | ----------------- | ---------------------------- |
| Researchers ×4   | 4         | 2                 | ≥2 of 4 DONE                 |
| Design Review ×3 | 3         | 2                 | ≥2 of 3 approve + 0 blockers |
| Implementers     | ≤4/wave   | all               | all DONE                     |
| Verifiers        | ≤4/wave   | all               | all DONE (per-task gate)     |
| Code Review ×3   | 3         | 2                 | ≥2 of 3 approve + 0 blockers |

### Retry Policy

- **Budget:** 1 retry per failed subagent (orchestrator-level).
- **Trigger:** Agent returns `status: ERROR` or fails to return within timeout.
- **Schema violation:** Treated as transient — retry once. Second schema violation → ERROR.
- **No retry on:** Security Blocker verdicts (immediate pipeline ERROR — see severity-taxonomy.md).

### Examples

**Researchers ×4 (Step 1):**

```
Dispatch: researcher ×4 (architecture, impact, dependencies, patterns)
Concurrency: 4 concurrent (at cap)
Gate: ≥2 of 4 DONE
Retry: Failed researchers retried once
Result: 2–4 research outputs forwarded to Spec agent
```

**Adversarial Reviewers ×3 (Steps 3b, 7) — Perspective-Based:**

Each reviewer reviews **all** categories (security, architecture, correctness) through a distinct perspective lens. 3 perspectives × 3 categories = 9 review dimensions per round. See [review-perspectives.md](review-perspectives.md) for full perspective definitions.

```
Dispatch: adversarial-reviewer ×3 with distinct review_perspective
  Instance 1: review_perspective = "security-sentinel"
  Instance 2: review_perspective = "architecture-guardian"
  Instance 3: review_perspective = "pragmatic-verifier"

  Parameters per instance:
    review_scope: "design" (Step 3b) or "code" (Step 7)
    review_perspective: <perspective-id>
    run_id: {run_id}
    round: {round}
    feature_slug: {feature_slug}

Concurrency: 3 concurrent (within cap)
Gate: All 3 reviewers submitted (9 SQL rows) AND 0 verdict='blocker'
      AND ≥2 of 3 reviewers fully approve across all categories
Retry: Failed reviewers retried once
Result: Review verdicts + findings forwarded; any blocker → pipeline ERROR
```

> **Note:** Each reviewer produces 3 SQL INSERT records (one per category) and 1 verdict YAML file with per-category sub-verdicts. The evidence gate validates all-category coverage, not single-category verdicts.

---

## Pattern B — Sequential with Replan Loop

Used by: Implementation-Verification cycle (Steps 5–6 iteration).

1. **Dispatch** implementation wave (Pattern A with ≤4 concurrent implementers).
2. Wait for all implementers to return.
3. **Verify** each completed task (Pattern A with ≤4 concurrent verifiers, 1 per task).
4. Evaluate verification results:
   - If all tasks pass → DONE, proceed to next pipeline step.
   - If any task returns `NEEDS_REVISION` → **replan**.
5. **Replan:** Invoke Planner in replan mode with verification findings → produce revised tasks.
6. **Re-implement** revised tasks only (Pattern A, ≤4 concurrent).
7. **Re-verify** revised tasks (Pattern A, ≤4 concurrent).
8. Repeat from step 4.
9. **Maximum 3 iterations.** After 3 iterations with unresolved issues → proceed with findings documented, `confidence: Low`.

### Routing Table

| Verification Result | Action                                       | Next Step            |
| ------------------- | -------------------------------------------- | -------------------- |
| All tasks DONE      | Proceed                                      | Step 7 (Code Review) |
| Any NEEDS_REVISION  | Planner replan → re-implement → re-verify    | Step 5 (iterate)     |
| Any ERROR           | Retry once → if still ERROR → pipeline ERROR | Pipeline ERROR       |
| Iteration 3 reached | Proceed with findings, confidence: Low       | Step 7 (Code Review) |

### Code Review Cycling (Step 7)

Code review has its own sub-loop within Pattern B:

1. Dispatch 3 adversarial reviewers in parallel (Pattern A).
2. Evaluate verdicts:
   - All approve → DONE.
   - Any `NEEDS_REVISION` → Implementer fixes → re-verify → re-review (round++).
   - Any `blocker` → pipeline ERROR (Security Blocker Policy).
3. **Maximum 2 review rounds.** After 2 rounds with remaining findings → known issues, `confidence: Low`.

---

## Concurrency Rules

### Concurrency Cap

**Maximum 4 concurrent subagents per wave.**

This cap applies to every Pattern A dispatch. No orchestrator step may dispatch more than 4 subagents simultaneously.

### Sub-Wave Partitioning

When a step requires dispatching more than 4 subagents (e.g., >4 implementation tasks), partition into sequential sub-waves of ≤4:

**Algorithm:**

1. Sort tasks by dependency order (independent tasks first).
2. Partition into sub-waves of size ≤4.
3. Execute sub-wave 1 (Pattern A, ≤4 concurrent).
4. Wait for sub-wave 1 to complete.
5. Execute sub-wave 2 (Pattern A, ≤4 concurrent).
6. Repeat until all sub-waves complete.

**Example — 9 implementation tasks:**

```
Sub-wave 1: Tasks 1, 2, 3, 4  (4 concurrent)  → wait for all
Sub-wave 2: Tasks 5, 6, 7, 8  (4 concurrent)  → wait for all
Sub-wave 3: Task 9             (1 concurrent)  → wait
```

**Rules:**

- Sub-waves are sequential relative to each other.
- Within a sub-wave, agents run fully in parallel (Pattern A).
- The gate condition applies per sub-wave, not across the full set.
- If a sub-wave fails its gate, the orchestrator handles the failure before proceeding to the next sub-wave.

---

## Pattern Usage Table

| Pipeline Step               | Pattern   | Max Concurrent | Gate Condition                         | Retry Budget     |
| --------------------------- | --------- | -------------- | -------------------------------------- | ---------------- |
| Step 0: Setup               | —         | 1              | Automatic (init checks pass)           | —                |
| Step 1: Research            | Pattern A | 4              | ≥2 of 4 DONE                           | 1 retry/agent    |
| Step 2: Specification       | —         | 1              | Sequential (DONE)                      | 1 retry          |
| Step 3: Design              | —         | 1              | Sequential (DONE)                      | 1 retry          |
| Step 3b: Design Review      | Pattern A | 3              | ≥2 approve + 0 blockers (SQL evidence) | 1 retry/reviewer |
| Step 4: Planning            | —         | 1              | Sequential (DONE)                      | 1 retry          |
| Steps 5–6: Implement+Verify | Pattern B | 4/wave         | All tasks DONE (max 3 iterations)      | 1 retry/agent    |
| Step 7: Code Review         | Pattern A | 3              | ≥2 approve + 0 blockers (SQL evidence) | 1 retry/reviewer |
| Step 8: Knowledge Capture   | —         | 1              | Non-blocking (ERROR does not halt)     | 1 retry          |
| Step 9: Auto-Commit         | —         | 1              | Automatic                              | —                |

---

## Evidence Gating (SQL)

Review steps (3b, 7) use SQL-based evidence gates. The orchestrator verifies verdicts independently via queries on the `anvil_checks` table. With perspective-based review, each reviewer produces **3 SQL records** (one per category), yielding **9 total records per round**. See [sql-templates.md](sql-templates.md) §6 for the full query templates.

```sql
-- (1) All reviewers submitted: 3 distinct perspective instances
SELECT COUNT(DISTINCT instance) FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND phase = 'review' AND round = {round};
-- Expected: 3 (security-sentinel, architecture-guardian, pragmatic-verifier)

-- (2) Each reviewer covered all 3 categories
SELECT instance, COUNT(DISTINCT check_name) AS cats
FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND phase = 'review' AND round = {round}
  AND check_name IN (
    'review-{scope}-security',
    'review-{scope}-architecture',
    'review-{scope}-correctness'
  )
GROUP BY instance HAVING cats = 3;
-- Expected: 3 rows (one per reviewer)

-- (3) Zero blockers
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND task_id = '{task_id}'
  AND phase = 'review' AND round = {round}
  AND verdict = 'blocker';
-- Expected: 0

-- (4) Majority fully-approving reviewers (≥2 of 3 approve ALL categories)
SELECT COUNT(*) FROM (
  SELECT instance FROM anvil_checks
  WHERE run_id = '{run_id}' AND task_id = '{task_id}'
    AND phase = 'review' AND round = {round}
    AND check_name IN (
      'review-{scope}-security',
      'review-{scope}-architecture',
      'review-{scope}-correctness'
    )
  GROUP BY instance
  HAVING COUNT(CASE WHEN verdict != 'approve' THEN 1 END) = 0
);
-- Expected: >= 2
```

All gate queries filter on `run_id`, `task_id`, and `round` to prevent cross-contamination between pipeline runs, review scopes (design vs code), and review rounds.

---

## Parallel Dispatch Notes

Pattern A expresses **parallel intent** — all N subagents are logically dispatched concurrently. However, true runtime concurrency depends on the VS Code Copilot platform:

- **VS Code `runSubagent` API:** As of the current platform version, `runSubagent` calls may be serialized by the runtime even when dispatched in parallel from the orchestrator.
- **Impact:** Wall-clock time may equal the sum of individual agent durations rather than the maximum. The dispatch pattern is correct regardless — it expresses the desired parallelism, and the platform will execute it as efficiently as it can.
- **Optimization:** When execution is serialized, the orchestrator should dispatch the fastest-expected agents first to minimize blocking time for downstream dependencies.
- **No orchestrator changes needed:** The dispatch pattern remains the same whether the platform runs agents in parallel or sequentially. If future VS Code versions enable true concurrency, the pipeline benefits automatically.

---

## Phase 2 Enhancement Path — Model Diversity

The current review design achieves diversity through **prompt perspectives** (security-sentinel, architecture-guardian, pragmatic-verifier). Each reviewer uses the same underlying model but applies a distinct persona that shapes its analysis across all categories.

If future research (FR-1) discovers a working VS Code API mechanism for model routing (e.g., specifying different models per `runSubagent` call), model diversity can be **layered on top of** prompt diversity without changing the dispatch pattern:

1. **Dispatch pattern unchanged:** 3 reviewers with `review_perspective` parameter remain the same.
2. **Add `model` parameter:** Each reviewer additionally receives a model identifier (e.g., `model: 'gpt-4o'`, `model: 'claude-sonnet'`, `model: 'gemini-pro'`).
3. **Multiplicative diversity:** Prompt perspective × model = stronger coverage. A security-sentinel on GPT-4o catches different vulnerabilities than a security-sentinel on Claude.
4. **Graceful degradation:** If model routing fails or is unavailable, the pipeline falls back to prompt-only diversity — no functional regression.

This is a Phase 2 enhancement. Phase 1 ships with prompt diversity only, which is sufficient for meaningful adversarial coverage.
