# Adversarial Review: design — pragmatic-verifier

## Review Metadata

- **Perspective:** pragmatic-verifier
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-04T00:00:00Z
- **Feature:** tdd-e2e-enforcement
- **Task ID:** tdd-e2e-enforcement-design-review
- **Artifacts Reviewed:** `design-output.yaml` (744 lines), `design.md` (613 lines), cross-referenced against `spec-output.yaml` (1197 lines), `feature.md` (683 lines), and 5 current agent definition files

---

## Security Analysis

**Category Verdict:** approve

### Finding S-1: TDD RED Phase Evidence Remains Partially Self-Attested

- **Severity:** Major
- **Description:** The design's TDD compliance verification (D-13) uses structural analysis: verify test files exist, import production modules, and cross-check `initial_run_failures > 0` from the implementation report. However, the verifier cannot independently reproduce the RED phase. At verification time, both tests and production code are already committed in a single `git add -A` operation. The verifier reads the `initial_run_failures` count from the implementation report — a value that the implementer self-reports. A dishonest or malfunctioning implementer could write tests that pass immediately, then claim `initial_run_failures=3`. The structural check (tests import production modules) verifies test quality but not temporal ordering. DR-2 acknowledges this deviation from FR-1.4's "cross-check against git diff" but the alternative provides weaker assurance.
- **Affected artifacts:** `design-output.yaml` decision D-13, `design.md` §6.2 TDD Enforcement Sequence, DR-2
- **Recommendation:** Add a secondary heuristic: the verifier should check that `tdd_red_green.initial_run_exit_code` is a plausible non-zero value (not fabricated like 999999), and cross-verify that the test command recorded in the implementation report is consistent with the project's actual test runner. Consider requiring the implementer to capture a hash of the terminal output from the RED phase run, which the verifier can compare against pattern expectations. Acknowledge in the design that RED-phase timing verification is a known assurance gap.
- **Evidence:** FR-1.4 requires "cross-check against git diff showing test files committed before or alongside production files." DR-2 deviates from this. D-13 alternative rejected ("Trust implementer self-report") is functionally close to what the structural analysis achieves for the RED-phase count.

### Finding S-2: E2E Evidence Artifact Authenticity Not Verified

- **Severity:** Minor
- **Description:** The design specifies screenshot capture (FR-4.7) and evidence manifests but provides no mechanism for downstream consumers (adversarial reviewer, knowledge agent) to verify that screenshots are authentic — i.e., actually captured from a browser rendering the application, not placeholder images. Since the verifier is an LLM agent executing via `run_in_terminal`, the screenshots are written by the Playwright CLI process, which provides reasonable assurance. However, the design does not specify any integrity check (e.g., file size minimum, image format validation, or metadata verification).
- **Affected artifacts:** `design.md` §7.1, §6.1 Phase 5 — Teardown (evidence manifest generation)
- **Recommendation:** Add a minimal evidence integrity check in the evidence manifest generation step: verify captured screenshots are valid PNG/JPEG files with non-zero dimensions. This is a low-cost addition that catches gross failures (empty files, truncated captures).
- **Evidence:** FR-4.7 specifies structured evidence capture. AC-38 requires evidence proving "agents actually interacted with the app." No integrity validation is specified.

### Finding S-3: Secrets Exposure Risk in Evidence Capture Not Fully Mitigated

- **Severity:** Minor
- **Description:** The design (§7.1) correctly identifies that screenshots could capture sensitive data and says "flagged as risk if not" for masking. API response logs capture full bodies (truncated to 10KB per FR-4.7). However, the design provides no concrete mechanism for sanitization. A login flow screenshot capturing a session token in the URL bar, or an API response containing auth tokens, would violate NFR-2. The mitigation is post-hoc adversarial review, not prevention.
- **Affected artifacts:** `design.md` §7.1 Secrets in E2E Artifacts, `design-output.yaml` agent_modification_map for adversarial-reviewer
- **Recommendation:** Add a sanitization step to the Tier 5 teardown phase: before writing evidence manifests, scan response logs for common secret patterns (Bearer tokens, API keys, session IDs). Mark screenshots from auth-related skills as "potentially sensitive" in the evidence manifest. This is defense-in-depth alongside the adversarial review.
- **Evidence:** NFR-2: "Screenshots MUST NOT capture sensitive data visible on screen." CR-7: "Secrets...MUST NOT appear in...screenshots."

---

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: Browser Process Lifecycle Ambiguity Between E2E Phases

- **Severity:** Major
- **Description:** The 5-phase E2E lifecycle (§6.1) is ambiguous about browser process management across phases. Phase 1 (Setup) says "Setup interaction tools (launch browser via Playwright CLI)." Phase 2 (Suite) runs test-suite skills via `run_in_terminal(skill.steps[].command)` — which typically launches its OWN browser instance (e.g., `npx playwright test` spawns browsers internally). Phase 3 (Exploratory) then performs step-by-step browser interaction. The design does not clarify:
  (a) Does Phase 2's test-suite runner use the same browser launched in Phase 1, or its own?
  (b) Is the Phase 1 browser still running during Phase 2, creating two browser processes?
  (c) When Phase 2 completes, is its browser closed before Phase 3 starts?
  (d) How does PID tracking (FR-5.6) handle browser PIDs spawned by a child process (the test runner)?
  This ambiguity creates risk of orphaned browser processes between phases, especially if Phase 2's test runner crashes.
- **Affected artifacts:** `design.md` §6.1 E2E Verification Sequence, `design-output.yaml` D-5 (Tier 5 architecture), D-11 (tool access)
- **Recommendation:** Explicitly specify browser lifecycle per phase: Phase 1 launches browser ONLY for Phases 3-4 (exploratory/adversarial). Phase 2 runs test-suite commands that manage their own browser processes. Phase 2's browser processes must be tracked and confirmed terminated before Phase 3 begins. Add a "Phase 2 cleanup" sub-step between Suite and Exploratory phases. Document which PIDs the verifier tracks (app process, Phase 1 browser, Phase 2 test runner child processes).
- **Evidence:** FR-5.6: "The verifier MUST track the PID of every app process AND every browser/interaction tool process it starts." The design's Phase 2 (`run_in_terminal(skill.steps[].command)`) spawns child processes whose browser PIDs are not directly trackable by the verifier.

### Finding A-2: Orchestrator Line Budget Estimate Dangerously Tight

- **Severity:** Major
- **Description:** The orchestrator is at 528/550 lines. The design estimates +20 lines added, +15 lines modified, resulting in ~548/550 lines (2-line margin). The additions include: E2E contract discovery at Step 0, contract path propagation, workflow lane enforcement, E2E concurrency management (tracking active E2E count, sub-wave queuing), three new evidence gates (EG-8, EG-9, EG-10), and reference to `e2e-integration.md`. Each of these is a non-trivial logic block. The sub-wave queuing logic alone (D-17) — tracking M active E2E verifiers, comparing against max_concurrent_instances, dispatching overflow — could easily consume 10+ lines. The estimate of 20 additional lines may be optimistic.
- **Affected artifacts:** `design-output.yaml` file_inventory for orchestrator.agent.md (+20 added, +15 modified), D-2 (extraction rationale references 528/550 budget)
- **Recommendation:** Re-estimate orchestrator line additions with specific per-feature line counts: Step 0 discovery (~3 lines), path propagation (~2 lines), lane reading (~3 lines), EG-8/EG-9/EG-10 checks (~9 lines), concurrency tracking (~8 lines), sub-wave queuing (~6 lines). If the total exceeds 22, identify additional content to extract to `e2e-integration.md` or `dispatch-patterns.md`. Consider extracting the EG-8/EG-9/EG-10 query definitions into `sql-templates.md` with only the orchestrator calling them by reference.
- **Evidence:** Pushback concern #1 (Major): "Orchestrator at 528/550 lines, E2E routing may exceed cap." D-2 rationale: "The orchestrator is at 528/550 lines." The 20-line estimate provides only 2 lines of margin.

### Finding A-3: Verifier Complexity Growth Risk

- **Severity:** Minor
- **Description:** The verifier grows from 343 lines to approximately 523+ lines (+180 added, +30 modified). Tier 5 adds a 5-phase lifecycle, contract parsing, skill execution engine, PID tracking, interaction logging, evidence capture, and evidence manifest generation. This is essentially an embedded test automation framework. While the design mitigates via reference to `e2e-integration.md`, the verifier's workflow section still needs the complete execution logic for all 5 phases. The verifier's cognitive load increases significantly — it now handles deterministic verification (Tiers 1-4) AND dynamic interactive testing (Tier 5).
- **Affected artifacts:** `design-output.yaml` file_inventory for verifier.agent.md (🔴 risk), D-5 (Tier 5), agent_modification_map for verifier (10 change items)
- **Recommendation:** Consider whether the Tier 5 execution logic could be further extracted. For example, the 5-phase sequence could be described as pseudocode in `e2e-integration.md` with the verifier referencing each phase, rather than inlining the full sequence in `verifier.agent.md`. This is consistent with how other complex logic is handled (dispatch-patterns.md for dispatch logic).
- **Evidence:** Verifier file_inventory: estimated_lines_added=180, estimated_lines_modified=30. Current verifier is 343 lines. 10 distinct changes listed in agent_modification_map.

---

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: E2E Contract Validation Deferred From Step 0 to Tier 5 — Violates FR-2.5

- **Severity:** Major
- **Description:** FR-2.5 specifies: "The E2E contract MUST be validated at discovery time." The design's Step 0 (orchestrator) only discovers the contract's _existence_ and propagates the path. Actual field validation (required fields present, interaction_type matches app_type, port values valid, etc.) is deferred to Phase 1 of Tier 5 E2E verification — which runs at Step 6, after implementation (Step 5) is already complete. If the E2E contract has invalid fields, this won't be caught until a HIGH risk task reaches the verifier. Multiple implementation tasks may complete before the contract error surfaces, wasting pipeline time. The spec explicitly says "at discovery time" (Step 0), not "at first use" (Step 6).
- **Affected artifacts:** `design.md` §6.1 Phase 1 (validation in Tier 5), §6.3 Step 0 (only discovery, not validation), `design-output.yaml` D-3 (contract location), orchestrator agent_modification_map (no validation listed)
- **Recommendation:** Add contract validation to Step 0. The orchestrator can validate structure (required fields present, valid types) via a reference to validation rules in `e2e-integration.md`. Record `check_name='e2e-contract-validation'` at Step 0. If validation fails at Step 0, the orchestrator can surface the error immediately (interactive mode: prompt fix; autonomous mode: log and degrade to unit-only).
- **Evidence:** FR-2.5: "The E2E contract MUST be validated at discovery time. Validation checks: all required fields present...Validation failures MUST be recorded as check_name='e2e-contract-validation' with passed=0." The design's §4.6 check_name table lists `e2e-contract-validation` with Writer=Verifier and Phase=`after`, not Step 0.

### Finding C-2: Per-Variation Adversarial SQL Records Not Specified — Violates AC-11

- **Severity:** Major
- **Description:** AC-11 explicitly requires: "Each adversarial variation MUST produce a check_name='e2e-adversarial' SQL INSERT." The spec envisions per-variation granularity — if a skill has 3 adversarial variations (ADV-001, ADV-002, ADV-003), there should be 3 separate `e2e-adversarial` SQL records. The design's §6.1 Phase 4 shows "Record composite → record e2e-adversarial" — a single composite record per phase. The check_name table (§4.6) lists only one `e2e-adversarial` pattern. The design does not specify per-variation SQL INSERTs. This reduces diagnostic granularity: if the composite fails, the orchestrator and downstream agents can't determine which specific variation found a vulnerability without parsing the interaction log.
- **Affected artifacts:** `design.md` §6.1 Phase 4, §4.6 check_name patterns, `design-output.yaml` D-7 (SQL evidence strategy)
- **Recommendation:** Specify per-variation `e2e-adversarial` SQL INSERTs. Use a convention like `output_snippet` containing the variation ID (e.g., "ADV-001: SQL injection in email — PASSED"). Alternatively, define distinct check_names per variation (e.g., `e2e-adversarial-ADV-001`) — but this complicates the composite check. The simplest approach: INSERT one `e2e-adversarial` record per variation, then one composite `e2e-adversarial-composite` or keep the existing `e2e-test-execution` composite. Add this clarification to `sql-templates.md`.
- **Evidence:** AC-11: "Each adversarial variation MUST produce a check_name='e2e-adversarial' SQL INSERT." Contrast with design's §6.1: "Record composite → record e2e-adversarial."

### Finding C-3: Lane-to-Tier Execution Mapping Not Updated in Verifier

- **Severity:** Major
- **Description:** FR-6.1 specifies verification tiers per lane: LOW → Tiers 1-2, MEDIUM → Tiers 1-3, HIGH → Tiers 1-4 + E2E. The current verifier runs tiers based on task _size_ (Standard vs. Large), not workflow lane: Tier 3 is "required when Tiers 1-2 produce no runtime verification" (optional otherwise), and Tier 4 is "Large tasks only" (any 🔴 file). The design adds Tier 5 for `full-tdd-e2e` lane but does NOT modify the tier execution logic to be lane-aware. Consequences:
  (a) A `unit-integration` (MEDIUM) task with successful Tier 2 results will skip Tier 3 — violating FR-6.1's "Tiers 1-3" requirement for MEDIUM.
  (b) A `unit-only` (LOW) task that happens to be "Large" (🔴 file) will run Tier 4 — exceeding FR-6.1's "Tiers 1-2" scope.
  The discrepancy arises because the spec introduces lane-based tier gating while the existing verifier uses size-based tier gating, and the design doesn't reconcile these two systems.
- **Affected artifacts:** `design-output.yaml` agent_modification_map for verifier (no tier gating change listed), `design.md` §6.3 Workflow Lane Routing (specifies evidence gates but not tier execution)
- **Recommendation:** Add verifier agent modifications to make tier execution lane-aware: Tier 3 becomes mandatory for `unit-integration` and `full-tdd-e2e` lanes (regardless of Tier 2 results). Tier 4 is mandatory for `full-tdd-e2e` lane only (not size-based). Reconcile the size-based and lane-based models — the lane supersedes size for tier gating.
- **Evidence:** FR-6.1: "MEDIUM — Verification: Tiers 1-3." Current verifier §4: "Tier 3 — Required When Tiers 1–2 Produce No Runtime Verification." Design agent_modification_map for verifier lists 10 changes — none modify Tier 3/4 gating logic.

### Finding C-4: Database Isolation for Parallel E2E Not Automated

- **Severity:** Major
- **Description:** EC-10 specifies: "Each E2E instance uses unique DATABASE_URL or non-parallel mode; warn if not configured." The design addresses port isolation automatically (D-6: deterministic port offset) but provides no equivalent mechanism for database isolation. The E2E contract's `start_env` map can contain `DATABASE_URL`, but there's no mechanism to parameterize it per verifier instance. Two parallel E2E verifiers starting the same app with the same `DATABASE_URL` will cause data interference (one verifier's test data visible to the other, concurrent writes causing conflicts). While the design mentions EC-10 in §8.1, the listed recovery is just "warn if not configured" — but the design doesn't specify HOW or WHEN the warning is issued, or what "non-parallel mode" means in practice.
- **Affected artifacts:** `design.md` §8.1 EC-10 entry, §4.1 E2E Contract Schema (start_env field), D-6 (port allocation — no DB equivalent)
- **Recommendation:** Add database isolation guidance to `e2e-integration.md`: (a) If `start_env` contains a database-related variable (DATABASE_URL, DB_NAME, etc.), the verifier should parameterize it per instance (e.g., append `_task_ordinal`). (b) The E2E contract should support a `db_isolation_strategy` field: `per-instance` (append suffix), `shared-with-cleanup` (reset DB between runs), or `none` (user accepts risk). (c) At contract validation, warn if `supports_parallel=true` but no DB isolation is configured.
- **Evidence:** D-6 provides port isolation formula: `port_range_start + task_ordinal_index`. No equivalent formula exists for database isolation. EC-10 severity: "High."

### Finding C-5: on_failure='skip_remaining' Composite Pass/Fail Semantics Undefined

- **Severity:** Minor
- **Description:** FR-3.3 defines `on_failure` as `'fail'|'continue'|'skip_remaining'`. The design references this field in the skill step schema (§4.2) and failure handling (§8.1 EC-4, EC-13). However, the design does not specify how `skip_remaining` affects the composite skill pass/fail determination. If step 3 of 10 fails with `on_failure='skip_remaining'`: Are steps 4-10 recorded as "skip" in the interaction log? Is the skill overall "failed" or "partially passed"? Does the `e2e-exploratory` composite check record `passed=0` or `passed=1`? Without clear semantics, different implementers may handle this differently, breaking determinism (NFR-8).
- **Affected artifacts:** `design.md` §4.2 Step structure, §6.1 Phase 3 exploratory interaction
- **Recommendation:** Add a brief section to the design (or `e2e-integration.md`) specifying: `skip_remaining` causes all subsequent steps to be recorded as `result: "skip"` in the interaction log. The skill is recorded as `passed=0` if the failing step had an `assert` that failed. Steps without assertions that fail with `on_failure='skip_remaining'` mark the skill as `passed=0`.
- **Evidence:** FR-3.3: "on_failure ('fail'|'continue'|'skip_remaining' — behavior if step fails, default 'fail')." No composite semantics defined in design.

### Finding C-6: Per-Phase Timeout Arithmetic Exceeds Total Budget — Misleading Documentation

- **Severity:** Minor
- **Description:** D-9 defines per-phase limits: startup 60s + suite 300s + exploratory 180s/skill + adversarial 120s/skill + shutdown 30s = 690s minimum for a task with 1 skill per type. This exceeds the 600s total hard cap. The design correctly states "The total budget acts as a hard cap even if individual phases are within their limits," but the presentation implies all phases can run to their individual maximum, which is mathematically impossible for most tasks. An implementer or planner reading these limits might assume 300s is available for suite execution when in practice the suite only gets ~300s if no exploratory/adversarial skills exist.
- **Affected artifacts:** `design.md` §8.2 Timeout Hierarchy, `design-output.yaml` D-9
- **Recommendation:** Add a note to the timeout table: "Per-phase limits are maximums; the 600s total cap supersedes. For tasks with skills in all phases, the effective time per phase is reduced. Example budget for a typical task: startup 30s + suite 180s + 2 exploratory skills × 90s + 1 adversarial skill × 60s + shutdown 10s = 460s." This sets realistic expectations.
- **Evidence:** D-9: "600s total budget with per-phase limits." Phase sum: 60+300+180+120+30 = 690 > 600.

### Finding C-7: EG-10 Query Only Covers full-tdd-e2e Lane

- **Severity:** Minor
- **Description:** D-16 introduces EG-10 (lane-aware verification) and §5.3 shows the SQL query. However, the query shown only covers the `full-tdd-e2e` lane (total_passed >= 4 AND e2e_passed >= 1). There are no EG-10 variants for `unit-only` (total_passed >= 2) or `unit-integration` (total_passed >= 3). The design's §6.3 mentions lane-specific thresholds but the evidence gate architecture only provides one SQL query.
- **Affected artifacts:** `design.md` §5.3 EG-10 query, §6.3 Workflow Lane Routing (mentions per-lane thresholds)
- **Recommendation:** Specify EG-10 as a parameterized query or provide three variants in `sql-templates.md`. The orchestrator should select the appropriate variant based on `workflow_lane`.
- **Evidence:** §5.3 shows: "For full-tdd-e2e lane: ≥4 passed checks including e2e-test-execution." No query for unit-only or unit-integration lanes.

### Finding C-8: Interaction Step Retry Mechanism Delegated Without Specification

- **Severity:** Minor
- **Description:** EC-4 says "the step retries with its timeout_ms before failing." The skill step schema has `timeout_ms` (default 5000). But the design doesn't define what "retry" means for different interaction types. For browser interactions, Playwright has built-in auto-wait (retries element queries until timeout). For API interactions, a failed HTTP request could mean a connection error (retryable) or a 500 response (ambiguous). For CLI interactions, there's no inherent retry mechanism. The design's framework-agnostic approach (CR-4) means the retry behavior varies by tool.
- **Affected artifacts:** `design.md` §8.1 EC-4 entry, §4.2 step schema `timeout_ms` field
- **Recommendation:** Add to `e2e-integration.md`: For browser actions, `timeout_ms` maps to the tool's built-in wait/retry (e.g., Playwright's `timeout` option). For API actions, the verifier retries connection errors once within `timeout_ms` but does NOT retry non-2xx responses (those are assertion results). For CLI actions, the verifier waits up to `timeout_ms` for the command to complete.
- **Evidence:** EC-4 severity: "Medium." NFR-5: "Framework-agnostic — agents MUST NOT hardcode any tool-specific behavior."

---

## Summary

The design is comprehensive and well-structured, addressing all 14 FR groups and 38 acceptance criteria with 18 well-documented decisions. The architecture selection (Direction C) is sound. However, there are 7 Major findings across architecture and correctness that require revision:

1. **Browser lifecycle ambiguity** between E2E phases risks orphaned processes (A-1).
2. **Orchestrator line budget** is dangerously tight with only 2 lines of margin (A-2).
3. **Contract validation** is deferred from Step 0 to Tier 5, violating FR-2.5 (C-1).
4. **Per-variation adversarial SQL records** are not specified, violating AC-11 (C-2).
5. **Lane-to-tier execution mapping** is incomplete — verifier tier gating is still size-based, not lane-based (C-3).
6. **Database isolation** for parallel E2E is mentioned but not automated (C-4).
7. **TDD RED phase evidence** remains partially self-attested despite structural analysis (S-1).

The 7 Minor findings (S-2, S-3, A-3, C-5, C-6, C-7, C-8) are documentation clarifications and edge case refinements that can be addressed during implementation.

Overall verdict: **needs_revision** — the Major findings in correctness (spec compliance gaps) and architecture (process lifecycle ambiguity) require design iteration before implementation can proceed safely.
