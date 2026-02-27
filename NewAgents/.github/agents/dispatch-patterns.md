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

**Adversarial Reviewers ×3 (Steps 3b, 7):**

```
Dispatch: adversarial-reviewer ×3 with distinct review_focus
  Reviewer 1: review_focus='security'
  Reviewer 2: review_focus='architecture'
  Reviewer 3: review_focus='correctness'
Concurrency: 3 concurrent (within cap)
Gate: ≥2 of 3 verdict='approve' AND 0 verdict='blocker'
Retry: Failed reviewers retried once
Result: Review verdicts + findings forwarded; any blocker → pipeline ERROR
```

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

Review steps (3b, 7) use SQL-based evidence gates. The orchestrator verifies verdicts independently via queries on the `anvil_checks` table:

```sql
-- All reviewers submitted (round-specific)
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND round = {r} AND verdict IS NOT NULL;
-- Expected: >= 3

-- No security blockers
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND round = {r} AND verdict = 'blocker';
-- Expected: = 0

-- Majority approve
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND round = {r} AND verdict = 'approve';
-- Expected: >= 2
```

All gate queries filter on `run_id` and `round` to prevent cross-contamination between pipeline runs and review rounds.
