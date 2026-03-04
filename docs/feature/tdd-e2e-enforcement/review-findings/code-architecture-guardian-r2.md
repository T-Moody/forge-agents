# Adversarial Review: code — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🔴
- **Round:** 2
- **Run ID:** 2026-03-04T00:00:00Z
- **Schema Version:** 1.0

## Round 1 Resolution Status

All 3 Major findings from Round 1 have been verified as **fully resolved**:

| R1 ID | Finding | Status | Evidence |
|-------|---------|--------|----------|
| A-1 | EG-10 phantom check_names (`baseline-verified`, `integration-tests-passed`) | **Resolved** | Replaced with `baseline-captured` and `behavioral-coverage` in `sql-templates.md` lines 642–679. Both check_names now present in verifier check_name table (lines 114, 121) with documented production paths (lines 165, 203). Zero remaining references to old names: `grep_search` for `baseline-verified\|integration-tests-passed` across `NewAgents/.github/agents/**` returns 0 matches. |
| A-2 | E2E concurrency cap contradiction (orchestrator: 2–4 vs. dispatch-patterns: 1) | **Resolved** | `orchestrator.agent.md` line 486 now states "maximum 1 E2E-enabled task runs at a time (sequential sub-wave queuing)". `e2e-integration.md` line 684 clarifies `max_concurrent_instances` controls browser contexts within a single verifier, "NOT cross-task dispatch concurrency. Cross-task E2E dispatch is capped at 1". All three documents (`orchestrator.agent.md`, `dispatch-patterns.md`, `e2e-integration.md`) now agree on cap=1 for cross-task dispatch. |
| C-1 | EG-10 gates are dead code (queries return 0 for all tasks) | **Resolved** | Same fix as A-1. EG-10 queries now reference producible check_names. `baseline-captured` produced at verifier Tier 0 (line 165), `tdd-compliance` and `behavioral-coverage` at Tier 2 (lines 220, 203), `e2e-test-execution` at Tier 5 (line 348). Producer comments added to each EG-10 query variant in `sql-templates.md`. |

## Security Analysis

**Category Verdict:** approve

No security findings. Re-review confirms Round 1 assessment remains valid:

- **Implementer E2E prohibition** unchanged in `implementer.agent.md` — clear Anti-Drift Anchor enforcement.
- **Command allowlist tightened** (per fix report): Pattern #4 scoped to local file paths (`^node \.[\\/].+\.m?[jt]s$`), Pattern #7 scoped to localhost-only (`^curl -sf?o? https?://localhost[:/]`), Pattern #8 scoped to numeric PIDs. These are security improvements, not regressions.
- **Evidence gate ordering** (D-5 Tier 5 after Tiers 1–4) unchanged.
- **Trust levels** (three-tier model) unchanged.

## Architecture Analysis

**Category Verdict:** approve

### Round 1 Finding A-1: EG-10 phantom check_names — RESOLVED

The producer-consumer contract is now consistent:

- **sql-templates.md** EG-10 queries reference: `baseline-captured`, `tdd-compliance`, `behavioral-coverage`, `e2e-test-execution`
- **verifier.agent.md** check_name table (lines 114–139) includes all four check_names with tier assignments
- **verifier.agent.md** documents explicit INSERT paths for each:
  - `baseline-captured` → Tier 0, step 4, line 165 (`passed=1` when no discrepancies)
  - `tdd-compliance` → Tier 2, line 220 (`passed=1` when all primary checks pass)
  - `behavioral-coverage` → Tier 2, line 203 (BLOCKING for code tasks)
  - `e2e-test-execution` → Tier 5, line 348 (composite: all E2E sub-phases passed)
- **Producer comments** added to each EG-10 query variant (lines 642, 658, 674), documenting which tier produces each check_name.
- **No remaining phantom references**: `grep_search` for `baseline-verified` or `integration-tests-passed` across `NewAgents/.github/agents/**` returns 0 matches.

### Round 1 Finding A-2: E2E concurrency cap contradiction — RESOLVED

All three authoritative documents now agree on a single, unambiguous concurrency model:

| Document | Statement | Cross-reference |
|----------|-----------|-----------------|
| `dispatch-patterns.md` line 187 | "Maximum 1 E2E-enabled task running at a time" (concurrency cap = 1) | Canonical source |
| `orchestrator.agent.md` line 486 | "maximum 1 E2E-enabled task runs at a time (sequential sub-wave queuing per dispatch-patterns.md §E2E Concurrency)" | References dispatch-patterns |
| `e2e-integration.md` line 684 | "`max_concurrent_instances` [...] controls how many browser contexts a **single** verifier task may spawn — it does NOT control cross-task dispatch concurrency. Cross-task E2E dispatch is capped at 1" | References dispatch-patterns |

The semantic distinction is now clear: `max_concurrent_instances` = browser-context parallelism within a single verifier task; cross-task E2E dispatch = strictly serial (1 at a time).

### Round 1 Finding A-3 (Minor): Verifier Tier 5 scope expansion — UNCHANGED

Status: Acknowledged as accepted tradeoff. The conditional gate (D-24 `e2e_required=true`) continues to isolate the expanded scope. No regression. Retained as known architecture note.

No new architecture findings.

## Correctness Analysis

**Category Verdict:** approve

### Round 1 Finding C-1: EG-10 dead code — RESOLVED

Same fix as A-1. The EG-10 lane-aware verification gates now query check_names that the verifier actively produces:

- **unit-only** (`baseline-captured` + `tdd-compliance`): Both produced by verifier Tiers 0 and 2. Expected count: 2.
- **unit-integration** (`baseline-captured` + `tdd-compliance` + `behavioral-coverage`): All produced by verifier Tiers 0–2. Expected count: 3.
- **full-tdd-e2e** (`baseline-captured` + `tdd-compliance` + `behavioral-coverage` + `e2e-test-execution`): All produced by verifier Tiers 0–5. Expected count: 4.

The `phase='after'` filter in EG-10 queries is consistent with the verifier's self-verification checklist (line 487: "All INSERTs use `phase='after'`").

### Round 1 Finding C-2 (Minor): TASK-004 line count inaccuracy — UNCHANGED

Status: Acknowledged. Verifier correctly caught the discrepancy. No regression. Retained as known correctness note.

No new correctness findings.

## Summary

All 3 Major findings from Round 1 are fully resolved. The EG-10 phantom check_names have been replaced with real, verifier-produced check_names (`baseline-captured`, `behavioral-coverage`), with producer comments and explicit INSERT paths documented. The E2E concurrency cap contradiction is eliminated — all three authoritative documents now consistently specify cap=1 for cross-task dispatch, with `max_concurrent_instances` clearly scoped to browser-context parallelism within a single verifier. No new issues introduced by the fixes. The 2 Minor findings from Round 1 (A-3 verifier scope, C-2 line count) remain acknowledged and unchanged.
