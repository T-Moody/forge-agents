# Adversarial Review: code — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-04T00:00:00Z
- **Schema Version:** 1.0

## Security Analysis

**Category Verdict:** approve

No security findings through the architecture lens. Security boundaries are properly maintained:

- **Implementer E2E prohibition** is instruction-enforced in `implementer.agent.md` (lines 356–374) with Anti-Drift Anchor reinforcement (line 458). The prohibition is clear: MUST NOT start apps, run E2E tests, launch browsers, make HTTP requests, or execute live interactions.
- **Evidence gate ordering** (D-5): Tier 5 executes AFTER Tiers 1–4 complete, preserving the security principle that basic validation passes before the verifier gains expanded process execution privileges.
- **Command allowlist** (tool-access-matrix.md §8.2): 8 regex patterns with mandatory pre-execution validation. Rejected commands are logged but never executed.
- **Trust levels** (e2e-integration.md §1): Three-tier trust model (Trusted/Semi-Trusted/Untrusted) properly stratifies contract structure, command executables, and skill content. Evidence sanitization pipeline (D-25) is documented with credential redaction rules.

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: EG-10 check_name producer-consumer contract mismatch

- **Severity:** Major
- **Description:** The EG-10 lane-aware verification queries in `sql-templates.md` reference two check_names that no agent in the pipeline ever produces: `baseline-verified` and `integration-tests-passed`. The verifier's check_name table (verifier.agent.md lines 115–134) does not include either check_name. The closest existing check_names are `baseline-discrepancy` (produced only on failure, with `passed=0`) and `tests` (Tier 2 test execution, not integration-specific). This breaks the producer-consumer schema contract — the orchestrator (consumer) queries for evidence that the verifier (producer) never inserts.
- **Affected artifacts:**
  - `NewAgents/.github/agents/sql-templates.md` lines 646, 661, 676 — EG-10 query variants reference phantom check_names
  - `NewAgents/.github/agents/verifier.agent.md` lines 115–134 — check_name table lacks `baseline-verified` and `integration-tests-passed`
  - `NewAgents/.github/agents/orchestrator.agent.md` line 293 — references EG-10 for lane verification without noting the check_name gap
- **Recommendation:** Either (a) add `baseline-verified` and `integration-tests-passed` to the verifier's check_name table with explicit INSERT templates in sql-templates.md §2a, defining when these records are produced, OR (b) rewrite the EG-10 queries to use existing check_names that the verifier actually produces (e.g., use `phase='baseline'` existence check for baseline, and `tests` or `smoke-execution` for integration). Option (a) is preferred as the semantics are clearer.
- **Evidence:** `grep_search` for `baseline-verified` in `NewAgents/.github/agents/verifier.agent.md` returns 0 matches. `grep_search` for `integration-tests-passed` returns 0 matches. These check_names appear exclusively in `sql-templates.md` EG-10 queries (lines 646, 661, 676) and nowhere else in the pipeline.

### Finding A-2: E2E concurrency cap contradiction between orchestrator and dispatch-patterns

- **Severity:** Major
- **Description:** Two authoritative documents define conflicting E2E concurrency rules:
  - `orchestrator.agent.md` line 486: "enforce `max_concurrent_instances` from contract (default 2, cap 4)"
  - `dispatch-patterns.md` line 187: "Maximum 1 E2E-enabled task running at a time within any wave (concurrency cap = 1 for E2E tasks)"
  - `e2e-integration.md` line 684: "Max concurrent: `max_concurrent_instances` from contract (default 2, hard cap 4)"

  The orchestrator says to respect the contract's `max_concurrent_instances` (allowing 2–4 parallel E2E tasks). `dispatch-patterns.md` mandates strict serial execution (1 at a time). These are irreconcilable — an agent following one source will violate the other. The dispatch-patterns document is the canonical source for dispatch logic, but the orchestrator's inline rule and the e2e-integration reference both contradict it.

- **Affected artifacts:**
  - `NewAgents/.github/agents/orchestrator.agent.md` line 486 — says use contract value (default 2)
  - `NewAgents/.github/agents/dispatch-patterns.md` line 187 — says always 1
  - `NewAgents/.github/agents/e2e-integration.md` line 684 — says use contract value (default 2)
- **Recommendation:** Reconcile to a single source of truth. Given that E2E tasks start app instances, bind ports, and drive browser sessions, the conservative `dispatch-patterns.md` rule of 1 at a time is safer and should be canonical. Update `orchestrator.agent.md` line 486 to reference `dispatch-patterns.md` for the actual concurrency cap, and update `e2e-integration.md` to clarify that `max_concurrent_instances` is a cap on the contract field value, not a dispatch concurrency directive.
- **Evidence:** Direct text comparison of the three files at the cited lines. All three are in the staged diff.

### Finding A-3: Verifier Tier 5 significantly expands single-responsibility scope

- **Severity:** Minor
- **Description:** The verifier transforms from a read-only multi-tier static analysis agent to include process lifecycle management (starting apps, managing browser sessions via Playwright CLI, PID tracking, graceful/forced teardown). The Tier 5 addition is ~142 lines of new responsibility. While this is conditionally gated (`e2e_required=true` and `workflow_lane='full-tdd-e2e'`), it changes the verifier's fundamental character from observer to process-managing actor. The conditional gate mitigates context pressure for non-E2E tasks.
- **Affected artifacts:** `NewAgents/.github/agents/verifier.agent.md` lines 258–405
- **Recommendation:** This was acknowledged in the design review and accepted as a pragmatic tradeoff. The conditional gate (D-24) ensures non-E2E verification pays no context cost. Monitor verifier context window usage in post-mortem telemetry. Consider future extraction to a dedicated `e2e-verifier` sub-role if Tier 5 grows further.
- **Evidence:** Verifier diff shows +193 lines net, with the Tier 5 section representing the bulk of new content. Conditional gate at line 260 correctly isolates the expanded scope.

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: EG-10 gates are dead code — queries return 0 for all tasks

- **Severity:** Major
- **Description:** Because `baseline-verified` and `integration-tests-passed` check_names are never inserted into `anvil_checks` by any agent (see A-1), the EG-10 queries will always return COUNT(\*) = 0 for all tasks, regardless of workflow lane. This means:
  - `unit-only` tasks need ≥2 but get 0 (only `tdd-compliance` could match, yielding at most 1)
  - `unit-integration` tasks need ≥3 but get at most 1
  - `full-tdd-e2e` tasks need ≥4 but get at most 2 (`tdd-compliance` + `e2e-test-execution`)

  Every task in the pipeline would fail its lane verification gate. The EG-10 mechanism is architecturally sound in concept but the queries reference check_names that don't exist in the verifier's vocabulary.

- **Affected artifacts:**
  - `NewAgents/.github/agents/sql-templates.md` lines 629–678 — all three EG-10 query variants
  - `NewAgents/.github/agents/orchestrator.agent.md` lines 293–296 — gate table referencing EG-10
- **Recommendation:** Same as A-1. Define `baseline-verified` as a new check_name the verifier produces when baseline cross-check passes (complement to `baseline-discrepancy`). Define `integration-tests-passed` as a check the verifier produces when Tier 3 integration evidence is present. Add both to the verifier check_name table and provide INSERT templates in sql-templates.md §2a.
- **Evidence:** Cross-reference: sql-templates.md EG-10 queries (lines 646, 661, 676) vs. verifier check_name table (lines 115–134). The verification ledger (`anvil_checks`) for this run contains no records with `check_name='baseline-verified'` or `check_name='integration-tests-passed'` across all 16 tasks: `SELECT COUNT(*) FROM anvil_checks WHERE check_name IN ('baseline-verified','integration-tests-passed')` = 0.

### Finding C-2: TASK-004 implementation report line count inaccuracy

- **Severity:** Minor
- **Description:** The implementer's report for TASK-004 (e2e-integration.md) claimed 546 lines, but the actual file is 697 lines — a 151-line discrepancy. The verifier correctly caught this (`baseline-discrepancy` `passed=0` and `line-count-range` `passed=0`). While this is an implementation-level concern and the verifier handled it appropriately, the magnitude of the discrepancy (28% over-report) suggests the implementer was tracking an intermediate state rather than the final output.
- **Affected artifacts:**
  - `NewAgents/.github/agents/e2e-integration.md` — 697 lines actual vs. 546 claimed
  - `docs/feature/tdd-e2e-enforcement/implementation-reports/TASK-004.yaml` — inaccurate line count
- **Recommendation:** No action required — the verifier's verification chain correctly identified and recorded the discrepancy. The e2e-integration.md content quality is high despite exceeding the line target. Future implementation runs should perform line count validation as a self-check step.
- **Evidence:** `anvil_checks` records for TASK-004: `baseline-discrepancy` `passed=0` with output "implementer claimed 546 lines but actual file is 697 lines"; `line-count-range` `passed=0` with output "Actual: 697 lines. Target: 500 +/- 50 (450-550). Delta: +147 lines over upper bound".

## Summary

The TDD + E2E enforcement implementation is architecturally well-structured with proper separation of concerns, clean reference strategies, and appropriate conditional gating. Two Major issues require attention: (1) EG-10 lane-aware verification queries reference phantom check_names (`baseline-verified`, `integration-tests-passed`) that no agent produces, rendering these gates non-functional, and (2) the orchestrator and dispatch-patterns contradict on E2E concurrency caps (contract-driven 2–4 vs. strict 1). The orchestrator is well within its 550-line budget at 353 lines. Security boundaries are properly maintained. These issues are structural schema mismatches, not fundamental design flaws — they require reconciliation of cross-document references rather than architectural redesign.
