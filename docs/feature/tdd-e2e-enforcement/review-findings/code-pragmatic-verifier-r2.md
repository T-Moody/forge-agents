# Adversarial Review: code — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🔴
- **Round:** 2
- **Run ID:** 2026-03-04T00:00:00Z

## Security Analysis

**Category Verdict:** approve

No security findings. The tightened allowlist patterns in tool-access-matrix.md §8.2 and e2e-integration.md §5 are correctly scoped:

- **`node` pattern** (`^node \.[\\/].+\.m?[jt]s$`): Requires local-path prefix (`.`) and JS/TS file extension. Prevents `-e` arbitrary code execution. Anchored with `$`.
- **`curl` pattern** (`^curl -sf?o? https?://localhost[:/]`): Restricted to localhost only. The `-s` flag is mandatory, preventing verbose output leaks.
- **`kill` pattern** (`^(kill (-[0-9]+ )?[0-9]+|taskkill /F /PID [0-9]+)$`): Anchored with `$`, accepts only numeric PIDs. No arbitrary process name targeting.

All 8 allowlist patterns are identical between the canonical source (e2e-integration.md §5) and the reference copy (tool-access-matrix.md §8.2). No security regression from round 1 fixes.

## Architecture Analysis

**Category Verdict:** approve

### Round 1 Finding A-2 (Minor): check_name Cross-File Consistency — RESOLVED

Round 1 flagged that EG-10 gate queries referenced check_names with no documented producer. This is now fully resolved:

- EG-10 queries use `baseline-captured`, `tdd-compliance`, `behavioral-coverage`, `e2e-test-execution` — all present in verifier.agent.md check_name table (lines 114, 121, 122, 139).
- Each EG-10 SQL comment includes a `-- Producer: verifier (...)` annotation with tier references, documenting the producer/consumer contract inline.
- Verifier.agent.md L165 explicitly instructs: "No discrepancies: INSERT with `check_name='baseline-captured'`, `passed=1`. This positive record is required by the EG-10 lane-aware verification gate."

**Evidence:** `check-names-consistent` verification check passed with `passed=1`, output: "All 12 new check_names in INSERT templates match those referenced in EG-8/EG-9/EG-10 queries."

### Round 1 Finding A-1 (Minor): Timeout Sum Exceeds Hard Cap — UNCHANGED

This was accepted as a known issue in round 1 and is not in scope for the round 2 fix. Remains as-is. No re-report.

No new architecture findings.

## Correctness Analysis

**Category Verdict:** approve

### Round 1 Finding C-1 (Critical): EG-10 Queries Reference Non-Existent check_names — FULLY RESOLVED

**Status:** Fixed. All three EG-10 lane variants now reference check_names that the verifier actually produces:

| EG-10 Variant         | check_names in IN clause                                                           | Verifier Table Match              |
| --------------------- | ---------------------------------------------------------------------------------- | --------------------------------- |
| unit-only (≥2)        | `baseline-captured`, `tdd-compliance`                                              | L114 (Tier 0), L122 (Tier 2) ✓    |
| unit-integration (≥3) | `baseline-captured`, `tdd-compliance`, `behavioral-coverage`                       | L114, L122, L121 (Tier 2) ✓       |
| full-tdd-e2e (≥4)     | `baseline-captured`, `tdd-compliance`, `behavioral-coverage`, `e2e-test-execution` | L114, L122, L121, L139 (Tier 5) ✓ |

**Producer path for `baseline-captured`:** Verifier.agent.md L164-165 — "No discrepancies: INSERT with `check_name='baseline-captured'`, `passed=1`." This closes the producer/consumer gap that was the root cause of C-1.

**Verification evidence:** The verifier's own `eg-10-syntax-valid` check confirmed: "3 lane variants present. unit-only(≥2 checks), unit-integration(≥3), full-tdd-e2e(≥4 incl e2e-test-execution). All use SELECT COUNT(\*) with IN clause and passed=1."

**Cross-check:** `check-names-consistent` check confirmed: "All 12 new check_names in INSERT templates match those referenced in EG-8/EG-9/EG-10 queries. Cross-verified: tdd-compliance(EG-8,EG-10), e2e-test-execution(EG-9,EG-10), suite/exploratory/adversarial-composite(EG-9 sub-phase)."

**Verdict:** The Critical defect that would have made all EG-10 gates unsatisfiable is now corrected. All three lane variants can be satisfied when the verifier produces the expected check records.

### Round 1 Finding C-2 (Major): 9/10 Implementation Reports Missing verify_phase — ACCEPTED (Known Issue)

**Status:** Unchanged and accepted. These reports were created before the Schema 7 extensions were implemented. Not a regression — pre-existing state. Logged as known issue per round 1 disposition.

### Round 1 Finding C-3 (Major): e2e-integration.md 697 Lines vs 550 Target — ACCEPTED (Known Issue)

**Status:** Unchanged and accepted. Content is valid and comprehensive. The overage is cosmetic and does not affect functional correctness. Logged as known issue per round 1 disposition.

### Round 1 Findings C-4, C-5 (Minor): TDD Fallback Edge Case, Line Count Discrepancy — ACCEPTED

**Status:** Unchanged. Low impact. Not re-reported.

### New Issues Introduced by Round 2 Fixes: None

Reviewed all four modified files. No new correctness, logic, or contract issues introduced. The changes are narrowly scoped to the identified fixes.

## Summary

The Critical defect from round 1 (C-1: EG-10 phantom check_names) is fully resolved. All three lane variants now reference verifier-produced check_names with documented producer paths. The `baseline-captured` positive record is explicitly mandated in the verifier workflow. The allowlist tightening is correctly applied with identical patterns across both source files. Concurrency semantics are clarified. No new issues were introduced by the fixes. Remaining known issues (C-2 Major: missing verify_phase in 9/10 reports, C-3 Major: 147-line overage) are pre-existing conditions accepted in round 1. Overall verdict: approve.
