# Adversarial Review: code — pragmatic-verifier (Round 2)

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🔴
- **Round:** 2
- **Run ID:** 2026-03-05T12:00:00Z

## Security Analysis

**Category Verdict:** approve

No security findings. Round 1 finding S-1 (fetch_webpage URL scope) was accepted as instruction-based enforcement — this remains appropriate. The VS Code user-approval-per-invocation gate provides the technical control layer. No new security concerns introduced by the R2 fixes.

## Architecture Analysis

**Category Verdict:** approve

No architecture findings. Round 1 finding A-1 (DR-1 two-place sync) has been fully resolved:

- **Line 43** now reads: `ONLY for SQLite queries (SELECT, DDL at Step 0), telemetry INSERT, git read operations, git staging/commit at Step 9, and verification-ledger.db archive rename at Step 0. MUST NOT use for builds, tests, code execution, or other file modification.`
- **Line 537** now reads: `ONLY for SQLite reads (SELECT), DDL (Step 0), DB archive rename (Step 0), telemetry INSERT, git reads, git staging/commit (Step 9).`

Both locations are consistent and include archive rename as a permitted operation. No new architectural concerns introduced.

## Correctness Analysis

**Category Verdict:** approve

All 5 Round 1 correctness findings have been fully resolved. Verification details:

### R1 Finding C-1 (Major): §11 → §1.1 cross-reference — **RESOLVED**

- **Evidence:** Orchestrator line 111 now reads: `query per [sql-templates.md](sql-templates.md) §1.1` — the typo is corrected. The cross-reference correctly points to the Archive Check Query section.

### R1 Finding C-2 (Major): DR-1 scope contradiction — **RESOLVED**

- **Evidence:** Line 43 now explicitly permits `verification-ledger.db archive rename at Step 0` and uses `other file modification` (instead of blanket `file modification`). Line 537 anti-drift anchor includes `DB archive rename (Step 0)` in the permitted operations list. No intra-file contradiction remains — Step 0 3.a's rename operation is authorized by both DR-1 statements.

### R1 Finding C-3 (Minor): Stale pushback options — **RESOLVED**

- **Evidence:** Spec workflow step 2.3 (line ~260) now reads: `present each concern as a separate question via ask_questions with per-concern options (accept / modify / dismiss)`. This matches the Interactive Mode section's `Accept as known risk / Modify scope to address / Dismiss` options. The anti-drift anchor also references `per-concern structured choices (accept / modify / dismiss)`.

### R1 Finding C-4 (Minor): Missing self-verification checklists — **RESOLVED**

- **Evidence:** All four agents now have updated self-verification checks:
  - Researcher (line 241): `fetch_webpage NOT used when approval_mode='autonomous'`
  - Designer (line 244): `fetch_webpage NOT used when approval_mode='autonomous'`
  - Spec (line ~332, check #9): `fetch_webpage guard: fetch_webpage NOT used when approval_mode='autonomous'`
  - Implementer (line 434): `No terminal output redirected to files`

### R1 Finding C-5 (Minor): Missing -corrupt suffix — **RESOLVED**

- **Evidence:** Orchestrator line 111 now reads: `Query fails → archive unconditionally with -corrupt suffix (per §1.1 step 3).` This matches sql-templates.md §1.1 step 3's specification of `verification-ledger-{timestamp}-corrupt.db`.

## Summary

All 7 Round 1 findings (2 Major, 5 Minor) have been fully resolved. The fixes are precise, correctly scoped, and introduce no new issues. DR-1 text at both locations (line 43 definition and line 537 anti-drift) is now consistent and correctly permits the archive rename operation. The §1.1 cross-reference is correct. Pushback options in the spec workflow match the Interactive Mode section. Self-verification checklists have been updated in all four agents that gained new behaviors. The -corrupt suffix is documented in the orchestrator's archive procedure. All categories approved.
