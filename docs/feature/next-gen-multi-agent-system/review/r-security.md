# Review: Security

## Review Tier

**Standard** — The 14 files under `NewAgents/` are agent definition Markdown documents and shared reference files. They contain no runtime code, no authentication logic, no data storage implementation, and no dependency manifests. The security surface is instruction-level constraints, not executable code. Standard tier applies: full secrets/PII scan plus standard security review for logic correctness of the documented security controls.

## Findings

### [Severity: Minor] Verifier Safety-Net Init Lacks WAL Pragma

- **What:** The Verifier's safety-net `CREATE TABLE IF NOT EXISTS` block (for when orchestrator Step 0 didn't run) omits `PRAGMA journal_mode=WAL` and `PRAGMA busy_timeout=5000`.
- **Where:** [verifier.agent.md](../../../NewAgents/.github/agents/verifier.agent.md#L128-L145) — §Safety Net Initialization
- **Why:** If the orchestrator's Step 0 init fails or is skipped, and the Verifier's safety net fires, the database is created in the default `DELETE` journal mode. With ≤4 concurrent agents writing, this risks `SQLITE_BUSY` errors. Not a security vulnerability, but undermines the reliability of the evidence ledger that security decisions depend on.
- **Suggested Fix:** Add `PRAGMA journal_mode=WAL;` and `PRAGMA busy_timeout=5000;` to the Verifier's safety-net SQL block, matching the orchestrator's Step 0 init.
- **Affects Tasks:** Task 11 (verifier.agent.md)

### [Severity: Minor] SQL Escaping Instruction Missing from Adversarial Reviewer

- **What:** The Verifier includes Operating Rule #9 ("Escape single quotes in `output_snippet` and `command` values before INSERTing"), but the Adversarial Reviewer — which also performs SQL INSERTs into `anvil_checks` via `run_in_terminal` — has no equivalent escaping instruction.
- **Where:** [adversarial-reviewer.agent.md](../../../NewAgents/.github/agents/adversarial-reviewer.agent.md#L119-L134) — §SQL INSERT Format
- **Why:** The `output_snippet` field is populated from the `{first_500_chars_of_summary}` which is free-text authored by the reviewing agent. While the content is agent-generated (not external user input), review summaries could contain single quotes that would break the INSERT statement. This is a robustness concern more than a classical SQL injection risk, since the data source is the agent itself, not an untrusted external party.
- **Suggested Fix:** Add an operating rule to the Adversarial Reviewer: "SQL escaping: Escape single quotes in `output_snippet` values before INSERTing (replace `'` with `''`)."
- **Affects Tasks:** Task 08 (adversarial-reviewer.agent.md)

### [Severity: Minor] Verifier Tool Table / Restriction Inconsistency

- **What:** The Verifier's Restrictions section states "You MUST NOT use `create_file`" and `create_file` is absent from the Tool Access table. However, the Output Schema says the Verifier writes `verification-reports/<task-id>.yaml`, and the Restrictions section says "The ONLY file you may write is `verification-reports/<task-id>.yaml`."
- **Where:** [verifier.agent.md](../../../NewAgents/.github/agents/verifier.agent.md#L393-L412) — §Tool Access and §Restrictions
- **Why:** It's unclear how the Verifier creates its output file without `create_file`. This creates an ambiguous tool boundary. From a security perspective, this is conservative (denying write access rather than over-granting), but the functional gap may lead to confusion or workarounds (e.g., using `run_in_terminal` to write files, which would bypass the intent of the restriction).
- **Suggested Fix:** Add `create_file` to the Verifier's Tool Access table with a note restricting it to `verification-reports/<task-id>.yaml` only. Update the Restrictions section to say: "You MUST NOT use `create_file` for any file other than `verification-reports/<task-id>.yaml`."
- **Affects Tasks:** Task 11 (verifier.agent.md)

### [Severity: Minor] `git add -A` Relies on Soft Control for Sensitive File Exclusion

- **What:** The Implementer uses `git add -A` which stages ALL untracked/modified files. The only protection against staging secrets is a textual instruction to "visually confirm you have not created files containing secrets, API keys, or credentials."
- **Where:** [implementer.agent.md](../../../NewAgents/.github/agents/implementer.agent.md#L192-L197) — §Git Staging
- **Why:** `.gitignore`-based exclusion is the standard Git mechanism but offers no defense-in-depth if `.gitignore` is incomplete or absent. The "visual confirm" instruction is a soft, instruction-level control with no enforcement mechanism. This was already identified in adversarial review R2 (lesson: "git add -A relies entirely on .gitignore for sensitive file exclusion; absent or incomplete .gitignore risks leaking secrets into version control").
- **Suggested Fix:** No change required — this is an accepted design risk with documented mitigation. For additional defense-in-depth, consider adding an instruction for the Verifier's Tier 4 secrets scan to also check `git diff --staged` for sensitive patterns before the orchestrator commits at Step 9.
- **Affects Tasks:** Task 09 (implementer.agent.md)

### [Severity: Minor] Illustrative Fake Credential in Verifier Example

- **What:** The Verifier's Tier 4 secrets scan example SQL contains `apiKey = "sk-..."` as an illustrative sample of what a failed scan looks like.
- **Where:** [verifier.agent.md](../../../NewAgents/.github/agents/verifier.agent.md#L504-L509) — §SQL INSERT Examples, Tier 4 Secrets Scan
- **Why:** This is clearly a truncated placeholder (ending in `...`) in a documentation example, not a real credential. However, automated secrets scanning tools (GitHub secret scanning, Gitleaks, TruffleHog) may flag it as a potential leaked key prefix. Low practical risk but could generate false positives in CI/CD scanning pipelines.
- **Suggested Fix:** Replace `apiKey = "sk-..."` with a more obviously fake value like `apiKey = "<EXAMPLE-REDACTED>"` to avoid triggering automated scanners.
- **Affects Tasks:** Task 11 (verifier.agent.md)

## Secrets/PII Scan Results

**Patterns checked:** `password`, `secret`, `api_key`, `apikey`, `token`, `Bearer`, `private_key`, `AWS_`, `AZURE_`, `connection_string`, real key patterns (`sk-[a-zA-Z0-9]{20,}`, `ghp_`, `gho_`, `AKIA`), email addresses, phone numbers, SSNs.

**Files scanned:** All 14 files in `NewAgents/`:

- 9 agent definitions (`.agent.md`)
- 3 shared reference documents (`schemas.md`, `dispatch-patterns.md`, `severity-taxonomy.md`)
- 1 prompt file (`feature-workflow.prompt.md`)
- 1 README (`README.md`)

**Results:** No secrets or PII detected. All matches for sensitive keywords are references to security concepts (instructions to avoid secrets, descriptions of what to scan for) or illustrative examples. The single email address (`test@example.com`) is a safe RFC 2606 example domain. The `sk-...` is a truncated documentation example.

## OWASP Top 10 Results

N/A — Standard tier. No executable code in scope.

## Dependency Vulnerabilities

No dependency manifests found in `NewAgents/`. All 14 files are Markdown documents (agent definitions and reference documentation). No `package.json`, `requirements.txt`, `Cargo.toml`, `.csproj`, `go.mod`, or any other dependency file exists. No runtime dependencies to assess.

## Cross-Cutting Observations

1. **Quality (R-Quality scope):** The Verifier's `create_file` tool gap (Finding 3) is also a functional/quality issue — the agent may not be able to produce its designated output file. This should be cross-checked during quality review.

2. **Testing (R-Testing scope):** The `git add -A` + `.gitignore` reliance (Finding 4) could benefit from an integration test that verifies no sensitive file patterns are staged after implementation. This is a testing concern, not purely security.

## Summary

**Review Tier:** Standard
**5 findings: 0 Blocker, 0 Critical, 0 Major, 5 Minor.**

The security posture of the 14 NewAgents files is strong. All 9 critical security control areas from the review checklist are properly addressed:

| Check                              | Status  | Notes                                                                   |
| ---------------------------------- | ------- | ----------------------------------------------------------------------- |
| 1. Orchestrator tool restrictions  | ✅ Pass | Allowlist + denylist + anti-drift anchor                                |
| 2. Security blocker policy         | ✅ Pass | Consistent across orchestrator, adversarial-reviewer, severity-taxonomy |
| 3. Implementer credentials/git     | ✅ Pass | No credential handling; git ops safe with soft control for staging      |
| 4. Verifier read-only              | ✅ Pass | Read-only enforced with tool restrictions (minor tool table gap)        |
| 5. Adversarial reviewer read-only  | ✅ Pass | Cannot modify reviewed code; explicit tool restrictions                 |
| 6. Knowledge agent safety filter   | ✅ Pass | Comprehensive safety constraint filter with rejection logging           |
| 7. SQLite WAL + no injection       | ✅ Pass | WAL mandated in Step 0; minor: safety-net init missing WAL              |
| 8. No secrets/PII                  | ✅ Pass | Clean scan across all 14 files                                          |
| 9. DR-1 run_in_terminal constraint | ✅ Pass | Thoroughly documented with allowlist + denylist + anti-drift            |

No pipeline-blocking issues found. All findings are Minor-severity improvements that strengthen defense-in-depth without indicating current exploitable vulnerabilities.
