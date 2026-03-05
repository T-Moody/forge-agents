# Adversarial Review: code — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-05T12:00:00Z

## Security Analysis

**Category Verdict:** approve

### Finding S-1: fetch_webpage URL scope restriction is instruction-only

- **Severity:** Minor
- **Description:** The `fetch_webpage` tool is granted to 3 agents (Researcher, Designer, Spec) with the scope restriction "NOT for fetching arbitrary URLs or scraping." This restriction is entirely instruction-based — there is no domain allowlist or URL pattern filter. A malicious or manipulated `initial-request.md` could embed URLs that lead to data exfiltration or SSRF-like scenarios via the agent's HTTP requests.
- **Affected artifacts:** [tool-access-matrix.md](NewAgents/.github/agents/tool-access-matrix.md) §3, §4, §5; [researcher.agent.md](NewAgents/.github/agents/researcher.agent.md) §1.8; [designer.agent.md](NewAgents/.github/agents/designer.agent.md) §1.5; [spec.agent.md](NewAgents/.github/agents/spec.agent.md) §1.5
- **Recommendation:** Acceptable given the system's enforcement model (all tool access is instruction-based per tool-access-matrix.md §11). VS Code's user-approval-per-invocation gate provides technical enforcement in interactive mode. No action required — noting for awareness.
- **Evidence:** tool-access-matrix.md §3 states `fetch_webpage 🔒 — Interactive mode only. Requires user approval per invocation in VS Code.` This approval gate is the actual enforcement; instruction-level "NOT for arbitrary URLs" is advisory only.

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: DR-1 scope defined in two places without synchronization

- **Severity:** Minor
- **Description:** The orchestrator's `run_in_terminal` scope (DR-1) is defined in two places that are now out of sync. `tool-access-matrix.md` §2 was updated to include "verification-ledger.db archive rename" as a permitted operation, but the orchestrator's own DR-1 text at lines 43 and 537 was NOT updated. This violates the single-source-of-truth principle — tool-access-matrix.md §1 is supposed to be authoritative (CR-4), but the orchestrator also has its own DR-1 constraint that an agent reads first.
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) lines 43, 537; [tool-access-matrix.md](NewAgents/.github/agents/tool-access-matrix.md) §2
- **Recommendation:** Amend the orchestrator DR-1 text at both locations (line 43 and line 537 anti-drift) to include "verification-ledger.db archive rename" alongside existing SQLite and git operations. Alternatively, make the orchestrator DR-1 text a cross-reference to tool-access-matrix.md §2 only, eliminating the duplication.
- **Evidence:** Line 43: `MUST NOT use for builds, tests, code execution, or file modification.` — "file modification" explicitly prohibits the rename required by Step 0 3.a. Line 537 anti-drift: `ONLY for SQLite reads (SELECT), DDL (Step 0), telemetry INSERT, git reads, git staging/commit (Step 9).` — enumerated list does not include file rename. Compare with tool-access-matrix.md §2 which adds: `Archive rename scope: Rename-Item/mv permitted ONLY for renaming verification-ledger.db`.

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Orchestrator references non-existent sql-templates.md §11 — should be §1.1

- **Severity:** Major
- **Description:** Orchestrator Step 0 item 3.a contains the cross-reference `[sql-templates.md](sql-templates.md) §11` for the archive check query. However, sql-templates.md sections end at §9 (Schema Evolution Strategy). There is no §11. The actual archive check query was added as §1.1 (`### §1.1 Archive Check Query`). The "§11" is a typographic confusion of "§1.1" vs "§11" — the dot was lost. This means an agent following the §11 reference will fail to locate the query template.
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) line 111
- **Recommendation:** Change `§11` to `§1.1` at orchestrator.agent.md line 111: `query per [sql-templates.md](sql-templates.md) §1.1`.
- **Evidence:** `grep_search` for section headers in sql-templates.md shows: §0, §1, §1.1, §2, §2a, §3, §4, §5, §6, §7, §8, §9. No §10, §11, or higher. The archive check query is at §1.1 (line 199). Orchestrator line 111 says `§11`.

### Finding C-2: DR-1 text explicitly prohibits the file rename required by Step 0 DB archive

- **Severity:** Major
- **Description:** The orchestrator's DR-1 constraint at line 43 states: `MUST NOT use for builds, tests, code execution, or file modification.` Step 0 item 3.a requires a file rename (`Rename-Item`/`mv` for verification-ledger.db archiving) — which IS a file modification. The anti-drift anchor at line 537 also omits file rename from the permitted operations list. An agent strictly following DR-1 would refuse to perform the archive rename instructed by Step 0 3.a. This creates an intra-file contradiction. AC-15 requires DR-1 to "explicitly include 'verification-ledger.db archive rename' as permitted" — this is only done in tool-access-matrix.md §2, not in the orchestrator's own DR-1 statements.
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) lines 43, 537
- **Recommendation:** Update both DR-1 statements: (1) At line 43, amend to: `ONLY for SQLite queries (SELECT, DDL at Step 0), telemetry INSERT, git read operations, git staging/commit at Step 9, and verification-ledger.db archive rename at Step 0. MUST NOT use for builds, tests, code execution, or other file modification.` (2) At line 537 anti-drift, add `DB archive rename (Step 0)` to the permitted list.
- **Evidence:** Line 43: `MUST NOT use for builds, tests, code execution, or file modification.` Line 537: `ONLY for SQLite reads (SELECT), DDL (Step 0), telemetry INSERT, git reads, git staging/commit (Step 9).` Neither mentions archive rename. Step 0 3.a: `rename to verification-ledger-{timestamp}.db`.

### Finding C-3: Spec workflow step 2.3 uses old pushback option names, inconsistent with restructured Interactive Mode section

- **Severity:** Minor
- **Description:** The spec.agent.md workflow step 2.3 at line ~258 still says `present them via ask_questions with structured multiple-choice options (proceed / modify / abandon)`. These are the OLD aggregate options. The restructured Interactive Mode section above uses the NEW per-concern options: `Accept as known risk / Modify scope to address / Dismiss`. An agent reading both would encounter conflicting instruction. The detailed section takes precedence, but the summary creates confusion.
- **Affected artifacts:** [spec.agent.md](NewAgents/.github/agents/spec.agent.md) workflow step 2.3 (~line 258)
- **Recommendation:** Update step 2.3 to: `If concerns exist, present each concern as a separate question via ask_questions with per-concern options (accept / modify / dismiss). Wait for user responses and aggregate per the Interactive Mode rules above.`
- **Evidence:** Spec workflow line ~258: `structured multiple-choice options (proceed / modify / abandon)`. Compare with the restructured Interactive Mode section which documents `Accept as known risk / Modify scope to address / Dismiss`.

### Finding C-4: Self-verification checklists not updated for 5 of 7 modified agents (AC-22)

- **Severity:** Minor
- **Description:** AC-22 requires "All modified agent files have updated self-verification checklists covering new behaviors." Only planner.agent.md (check #11 for e2e_required) and verifier.agent.md (Tier 5 check) had their self-verification checklists updated. The remaining 5 agents with new behaviors did not update self-verification: researcher (fetch_webpage), designer (fetch_webpage), spec (fetch_webpage + per-concern pushback), implementer (no-redirect rule 11), and adversarial-reviewer/knowledge-agent (approval_mode). CR-7 uses "should" priority, so this is not blocking.
- **Affected artifacts:** [researcher.agent.md](NewAgents/.github/agents/researcher.agent.md), [designer.agent.md](NewAgents/.github/agents/designer.agent.md), [spec.agent.md](NewAgents/.github/agents/spec.agent.md), [implementer.agent.md](NewAgents/.github/agents/implementer.agent.md)
- **Recommendation:** Add self-verification checks for: (1) researcher/designer/spec: "fetch_webpage was only used in interactive mode" per FR-3.6. (2) implementer: "No terminal output was redirected to files" per FR-4.1.
- **Evidence:** Review of all staged diffs shows no self-verification checklist changes for these agents. Only `planner.agent.md` check #11 and `verifier.agent.md` Tier 5 checklist were modified.

### Finding C-5: Orchestrator archive procedure missing `-corrupt` suffix for corrupt DB case

- **Severity:** Minor
- **Description:** sql-templates.md §1.1 step 3 specifies that when the archive query fails due to corruption, the DB should be renamed to `verification-ledger-{timestamp}-corrupt.db` (with `-corrupt` suffix). However, the orchestrator Step 0 3.a says only `Query fails → archive unconditionally` without specifying the `-corrupt` suffix distinction. An agent following only the orchestrator text would use the standard timestamp naming for corrupt DBs, losing the diagnostic signal.
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) line 111; [sql-templates.md](NewAgents/.github/agents/sql-templates.md) §1.1 step 3
- **Recommendation:** Amend orchestrator 3.a to: `Query fails → archive unconditionally with -corrupt suffix (per §1.1 step 3).`
- **Evidence:** sql-templates.md §1.1 step 3: `rename the DB to verification-ledger-{timestamp}-corrupt.db`. Orchestrator 3.a: `Query fails → archive unconditionally` (no suffix specified).

## Summary

The implementation addresses all 8 issues across 14 modified files and 1 new prompt file. All 22 acceptance criteria have corresponding implementations, with 2 partially met (AC-15 missing DR-1 update in orchestrator, AC-22 missing self-verification updates in 5 agents). The two Major findings are both cross-reference/consistency issues within the orchestrator: a §11 typo that should be §1.1, and DR-1 text that prohibits the file rename it simultaneously requires at Step 0. These are practical issues that would cause an agent to either fail to find the archive query template or refuse to perform the archive rename. Seven minor findings cover instruction inconsistencies and missing "should"-priority self-verification updates. No security blockers; the fetch_webpage interactive-mode gate is sufficient. Overall: solid implementation that needs two targeted fixes.
