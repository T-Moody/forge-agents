# Review: Security

## Review Tier

**Lightweight** — All 3 modified files are prompt/markdown text. No compiled code, no authentication logic, no data storage, no networking. Full secrets/PII scan performed; targeted security analysis of prompt injection vectors, privilege escalation, and memory tool disambiguation.

## Findings

### [Severity: Minor] Parallel Execution Summary uses stale "orchestrator merges" wording

- **What:** The Parallel Execution Summary (lines 516–525 of `orchestrator.agent.md`) still uses "orchestrator merges memories" shorthand at 8 occurrences, while all detailed merge steps (1.1m, 2m, 3m, 3b.2, 4m, between-waves, 6.3, 7.3) were correctly updated to "dispatches a subagent to merge."
- **Where:** `.github/agents/orchestrator.agent.md` lines 516–525 (Parallel Execution Summary section)
- **Why:** Inconsistency between abbreviated summary and corrected detailed steps. An LLM interpreting the summary literally could infer direct file-write capability. The risk is low because the detailed steps (which are authoritative) and the Anti-Drift Anchor unambiguously prohibit direct writes. The summary is a compact quick-reference and "merges" could be read as "causes merging via delegation."
- **Suggested Fix:** Change "orchestrator merges memories" to "orchestrator dispatches merge" or "orchestrator triggers merge" in the summary block for consistency. Alternatively, accept as-is since the detailed steps are authoritative.
- **Affects Tasks:** Task 01 (tool-restriction-prose)

### [Severity: Minor] YAML `tools:` field is advisory-only — no runtime enforcement guarantee

- **What:** The YAML `tools: [agent, agent/runSubagent, memory, read_file, list_dir]` field in the frontmatter is advisory. VS Code may not enforce tool restrictions at runtime. The design explicitly acknowledges this (§2.4) and mitigates with triple-layer prose enforcement (Global Rule 1, Operating Rule 5, Anti-Drift Anchor).
- **Where:** `.github/agents/orchestrator.agent.md` line 5 (YAML frontmatter)
- **Why:** This is a known limitation, not a bug — documented in design §2.4 and ct-security Finding 2. The prose restrictions are the primary enforcement mechanism. Noting here for completeness: if VS Code ever changes the `tools:` field to be enforced at runtime, the current configuration is correctly restrictive. No action needed.
- **Suggested Fix:** None required. The triple-layer prose enforcement (YAML + Operating Rule 5 + Anti-Drift Anchor) is the accepted mitigation. Monitor VS Code agent API for future enforcement updates.
- **Affects Tasks:** N/A (design decision, not implementation gap)

## Secrets/PII Scan Results

**Patterns checked:** `password`, `secret`, `api_key`, `apikey`, `token`, `Bearer`, `private_key`, `AWS_`, `AZURE_`, `connection_string`, email addresses, phone numbers, SSNs, credit card numbers.

**Files scanned:**
1. `.github/agents/orchestrator.agent.md` (545 lines) — No secrets or PII detected. "Token" references (0 hits in this file) are N/A.
2. `.github/prompts/feature-workflow.prompt.md` (59 lines) — No secrets or PII detected.
3. `docs/feature/orchestrator-tool-restriction/feature.md` (427 lines) — "Token" hits refer to LLM context tokens (NFR-3.1), not authentication tokens. No secrets or PII.

**Result: No secrets or PII detected.**

## OWASP Top 10 Results

N/A — Lightweight tier. However, targeted prompt-injection analysis was performed given the review focus:

- **Prompt injection vectors:** No new user-controllable input paths introduced. `{{USER_FEATURE}}` was pre-existing and unchanged. The Anti-Drift Anchor (line 540) provides strong adversarial resistance with explicit prohibited-tool enumeration and rationale.
- **Memory tool confusion:** All 5 pre-existing contradictions (where `memory` tool was conflated with pipeline file writes) have been corrected. Three disambiguation statements are now consistent across Global Rule 1 (L34), Operating Rule 5 (L93), and Anti-Drift Anchor (L540). Zero remaining "via `memory` tool" references for file operations.
- **Privilege escalation via delegation:** The orchestrator could theoretically instruct a subagent to use `grep_search` on its behalf. This is by design — the restriction prevents ad-hoc exploration by the orchestrator itself, not delegated research (which is the intended pipeline workflow). Not a vulnerability.

## Dependency Vulnerabilities

No dependency manifests found (`package.json`, `requirements.txt`, `Cargo.lock`, etc.). This is a pure markdown/prompt repository with no compiled dependencies. **No known vulnerabilities detected.**

## Cross-Cutting Observations

- **Quality concern (R-Quality scope):** The Parallel Execution Summary wording inconsistency (Minor finding above) is primarily a documentation quality issue. R-Quality should verify consistency between summary and detailed step wording.
- **Backwards compatibility (V-Feature scope):** Phase 1 preserves all `memory.md` architecture. The merge mechanism wording corrections ("via memory tool" → "via subagent delegation") fix pre-existing documentation bugs without behavioral changes. No pipeline flow regressions detected.
- **Feature.md scope annotation:** The Phase 1 Scope section (feature.md lines 250–274) correctly defers AC-4 through AC-10 and AC-13/14/15 to Phase 2. AC-12 Phase 1 interpretation is explicitly documented with clear rationale. No scope ambiguity.

## Summary

2 findings: 0 blockers, 0 major, 2 minor. No secrets, PII, or dependency vulnerabilities. All pre-existing memory-tool contradictions resolved. Tool restriction is consistently enforced across 3 prose locations + YAML frontmatter. Anti-Drift Anchor provides strong adversarial resistance. Phase 1 changes are backwards-compatible with zero behavioral impact on the pipeline.
