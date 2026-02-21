# Review: Security

## Review Tier

**Full** — This feature modifies agent definitions that govern pipeline execution behavior, adds a new agent with broad read access (PostMortem), modifies the orchestrator's tool restriction and telemetry capabilities, and changes pipeline control flow. These are core architecture changes that warrant Full-tier security review even though all artifacts are Markdown-only.

## Findings

### [Severity: Minor] PostMortem Broad Read Scope Lacks Structural Data Classification Guard

- **What:** The PostMortem agent has the broadest read scope of any single agent: all `memory/*.mem.md` files, all `artifact-evaluations/*.md` files, `memory.md`, and the evaluation schema reference. While the design's threat model asserts evaluation data "contains no secrets, credentials, or PII," this is a current-state assertion, not a structural guarantee. If the pipeline ever processes a feature involving credentials or internal security configurations, agent memories or evaluations could reference sensitive details, and PostMortem would aggregate derived insights into `post-mortems/` — an append-only, less-compartmentalized location.
- **Where:** `.github/agents/post-mortem.agent.md` §Inputs (lines 22–27), §Reading Strategy (lines 55–65)
- **Why:** The lack of a data classification filter between evaluation inputs and PostMortem aggregation means sensitive information could propagate from compartmentalized agent memories into a single aggregated report. This is low-probability in the current Markdown-only context but represents a design gap for future extensibility.
- **Suggested Fix:** Add a note in the PostMortem agent's Operating Rules to explicitly exclude or redact any content matching secret/credential patterns (e.g., strings resembling API keys, tokens, connection strings) from aggregated output. Alternatively, add a design-level constraint that evaluation free-text fields must not reference literal credential values.
- **Affects Tasks:** Task 02 (PostMortem agent creation)

### [Severity: Minor] Evaluation Free-Text Fields as Indirect Prompt Injection Surface

- **What:** The evaluation schema contains five free-text list fields (`useful_elements`, `missing_information`, `information_not_used`, `inaccuracies`, `impact_on_work`). These are authored by pipeline agents (not external users), but their content flows as input to the PostMortem agent. If a producing agent were compromised or produced adversarial content, these fields could carry prompt injection payloads that influence PostMortem's behavior.
- **Where:** `.github/agents/evaluation-schema.md` §Schema (lines 18–33), `.github/agents/post-mortem.agent.md` §Workflow Step 5 (line 119)
- **Why:** In the current closed-loop pipeline (agent-to-agent, no external user input into evaluations), this is very low risk. However, the design lacks explicit sanitization or structural validation of free-text content beyond the "≥1 entry" constraint. The PostMortem agent's File Boundaries and Anti-Drift Anchor provide defense-in-depth that limits the blast radius.
- **Suggested Fix:** No immediate action required. For future hardening, consider adding a rule to the evaluation schema that free-text fields must not exceed a maximum length (e.g., 500 characters per entry) to limit injection surface area.
- **Affects Tasks:** Task 01 (evaluation schema), Task 02 (PostMortem agent)

## Secrets/PII Scan Results

**Patterns scanned:** `password`, `secret`, `api_key`, `apikey`, `token`, `Bearer`, `private_key`, `AWS_`, `AZURE_`, `connection_string`, SSN patterns, credit card patterns, email patterns, phone number patterns.

**Files scanned:** All 18 changed files across `.github/agents/` and `.github/prompts/`, plus all files in `docs/feature/self-improvement-system/`.

**Result:** No secrets or PII detected. All matches for secret-related keywords are documentation/instruction references (e.g., "Flag immediately with severity: critical" in error handling rules, "contains no secrets" in design security section), not actual credentials or sensitive data.

## OWASP Top 10 Results

| #   | Category                  | Result           | Notes                                                                                                                                                                    |
| --- | ------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Injection                 | N/A              | No SQL, command execution, or XSS — Markdown-only repository                                                                                                             |
| 2   | Broken Authentication     | N/A              | No authentication system — single-user Copilot session                                                                                                                   |
| 3   | Sensitive Data Exposure   | Pass             | No secrets/PII in any artifact; telemetry contains only agent names, statuses, and retry counts                                                                          |
| 4   | XML External Entities     | N/A              | No XML parsing                                                                                                                                                           |
| 5   | Broken Access Control     | Pass             | File boundary rules are explicit and reinforced in all 18 files; orchestrator tool restriction is triple-stated (Rule 5, Anti-Drift Anchor, feature-workflow prompt)     |
| 6   | Security Misconfiguration | Pass             | No debug mode flags, no default credentials, no production config                                                                                                        |
| 7   | Cross-Site Scripting      | N/A              | No web output                                                                                                                                                            |
| 8   | Insecure Deserialization  | Pass (with note) | PostMortem parses YAML from evaluation files; design includes graceful skip for corrupted YAML (EC-5) — adequate defense                                                 |
| 9   | Known Vulnerabilities     | N/A              | No runtime dependencies — Markdown-only repository                                                                                                                       |
| 10  | Insufficient Logging      | Pass             | Telemetry tracking captures agent name, step, pattern, status, retries, failure reason, iteration, human intervention, timestamp; PostMortem produces structured run-log |

**Prompt injection review:** Searched all changed files for prompt injection patterns (`ignore previous`, `ignore above`, `disregard`, `you are now`, `system prompt`, `jailbreak`). No prompt injection vectors found. All "override" references are legitimate pipeline logic (R-Security blocker override rule).

## Dependency Vulnerabilities

No dependency manifests exist (`package.json`, `requirements.txt`, `Cargo.lock`, `go.sum`, `*.csproj`). This is a Markdown-only agent-definition repository with no runtime dependencies. No dependency vulnerability check applicable.

## Cross-Cutting Observations

1. **Quality (R-Quality scope):** The evaluation schema document (`.github/agents/evaluation-schema.md`) uses the `chatagent` code fence wrapper but is explicitly marked as "not an agent definition" — a shared reference document. This is a minor naming/classification inconsistency but does not affect security.

2. **Quality (R-Quality scope):** The PostMortem agent's collision avoidance uses date-based filenames (`<run-date>-run-log.md`) with sequence suffixes for same-day collisions. This is adequate but less granular than timestamp-based naming if multiple runs occur on the same date.

## Summary

**Full-tier review completed. 2 findings: 0 blockers, 0 major, 2 minor.**

All 18 changed files were scanned. Security posture is strong:

- **File boundaries:** All 14 modified agents and the new PostMortem agent declare explicit file boundaries with enforcing rules.
- **Read-only enforcement:** PostMortem has triple-layered read-only constraints (File Boundaries section, Read-Only Enforcement section, Anti-Drift Anchor).
- **Tool restriction:** Orchestrator's write-tool prohibition is triple-stated (Operating Rule 5, Anti-Drift Anchor, feature-workflow prompt rule).
- **Non-blocking safety:** PostMortem failure cannot leave the system in an inconsistent state — ERROR is non-blocking, pipeline outcome is determined at Step 7, and memory merge is skipped on ERROR.
- **No secrets exposure:** No secrets, credentials, or PII in any changed file.
- **Prompt injection:** No vectors detected; closed-loop agent-to-agent pipeline limits external injection risk.
