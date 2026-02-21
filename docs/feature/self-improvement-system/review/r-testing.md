# Review: Testing

## Review Tier

Full — This is a Markdown-only agent-definition repository with no test framework, no runtime code, and no build system. All "testing" is structural validation performed by the V cluster (V-Build, V-Tests, V-Tasks, V-Feature). The review evaluates whether that structural validation is sufficient to establish confidence in the 18 changed files (2 new, 16 modified) across 14 acceptance criteria.

## Findings

### [Severity: Minor] Evaluation Target Deviation — r-testing.agent.md

- **What:** The r-testing evaluation step targets both `design.md` and `feature.md`, but the design document (design.md §Per-Agent Evaluation Targets, line 359) and task specification (tasks/09-agent-eval-r-cluster.md, line 20) specify only `feature.md`. Additionally, `design.md` is not listed in r-testing.agent.md's Inputs section.
- **Where:** `.github/agents/r-testing.agent.md`, lines 163–164
- **Why:** This is a deviation from the authoritative design spec. While additive and non-harmful (more evaluation data), it creates an inconsistency: the agent evaluates an artifact it doesn't declare as an input. V-Tests and V-Tasks both verified the presence of `feature.md` as a target but did not catch the extra `design.md` because they checked for **inclusion**, not **exact match**.
- **Suggested Fix:** Either (a) remove `design.md` from the evaluation targets to match the design spec, or (b) add `design.md` to the r-testing Inputs section and update design.md §Per-Agent Evaluation Targets to reflect the change. Option (b) is preferred since r-testing does reference design context.
- **Affects Tasks:** Task 09

### [Severity: Minor] V-Tests Did Not Validate Evaluation Target Correctness Per Agent

- **What:** V-Tests' 22 structural checks confirmed that all 14 agents have evaluation steps, reference evaluation-schema.md, and include non-blocking clauses. However, no check verified that each agent's evaluation targets **exactly match** the design.md Per-Agent Evaluation Targets table.
- **Where:** `verification/v-tests.md` — §Cross-Reference Validation section
- **Why:** Without target correctness validation, deviations like the r-testing finding above pass undetected. The V-Tests checks verify **structure** (evaluation step exists) but not **content accuracy** (evaluation targets are correct per design). A cross-reference check against the design table would close this gap.
- **Suggested Fix:** Add a structural check that compares each agent's declared evaluation targets against the authoritative design.md §Per-Agent Evaluation Targets table. This could be a 14-row table check in V-Tests.
- **Affects Tasks:** N/A (verification methodology, not implementation)

### [Severity: Minor] No Runtime Behavioral Validation

- **What:** All 22 V-Tests checks and all 14 V-Feature AC verifications are static structural analysis of agent definition files. No actual pipeline run was performed, so edge cases (EC-1 through EC-11), actual YAML generation, telemetry accumulation, PostMortem analysis quality, and graceful degradation behavior remain unverified at runtime.
- **Where:** Entire verification suite (v-build.md, v-tests.md, v-tasks.md, v-feature.md)
- **Why:** This is an inherent limitation of the Markdown-only repository — there is no test framework or runtime to execute. The structural checks verify **intent** (agent instructions are correct) rather than **behavior** (agents produce correct outputs). This gap is documented and expected per constraint C-6.
- **Suggested Fix:** No immediate fix possible. Long-term: consider adding a CI-compatible linting/validation script (see Suggestions section) or a "dry run" integration test that invokes agents on a synthetic feature to verify output compliance.
- **Affects Tasks:** N/A (infrastructure gap)

### [Severity: Minor] post-mortem.agent.md Code Fence Format (Inherited)

- **What:** `post-mortem.agent.md` wraps its content in a ` ```chatagent ` code fence (line 1 and line 250) instead of bare `---` YAML frontmatter, diverging from all 19 other active `.agent.md` files.
- **Where:** `.github/agents/post-mortem.agent.md`, lines 1 and 250
- **Why:** VS Code's chatagent parser may not recognize the agent. This format inconsistency was identified by V-Build, propagated through V-Tests, V-Tasks, and V-Feature as a low-severity warning. All content is correct; only the wrapper format differs.
- **Suggested Fix:** Remove the ` ```chatagent ` wrapper (line 1) and closing ` ``` ` (line 250) so the file starts with `---` like all other agents.
- **Affects Tasks:** Task 02

## Coverage Assessment

- **Changed files with tests:** 18 of 18 — All changed files (2 new + 16 modified) were structurally validated by V-Build (file existence, format), V-Tasks (per-task AC), V-Tests (22 structural checks), and V-Feature (14 ACs).
- **Coverage gaps:**
  - Evaluation target correctness per agent was not cross-referenced against design.md (minor gap — 1 deviation found in r-testing)
  - Runtime behavior of edge cases EC-1 through EC-11 was not tested (inherent limitation of Markdown-only repos)
  - Telemetry accumulation in orchestrator context window: verified by instruction presence (Global Rule 13), but not by execution
  - PostMortem self-verification step (Step 9): verified by instruction presence, but quality of output cannot be assessed without a run
- **Edge cases missing:**
  - EC-2 (YAML generation failure): instructions verified, runtime behavior untested
  - EC-5 (corrupted YAML in evaluations): PostMortem has skip-and-note instructions, but no synthetic test
  - EC-6 (memory tool limitation fallback): delegation instructions present, but untested
  - EC-10 (missing directory at runtime): lazy creation instructions present, but untested

## Test Quality Assessment

- **Behavioral tests:** 0 (no test framework exists)
- **Structural validation checks:** 22 (V-Tests) + 14 ACs (V-Feature) + 11 task verifications (V-Tasks) + file integrity (V-Build) = comprehensive static coverage
- **Implementation-detail tests:** 0 (not applicable — all checks are behavioral in nature, verifying what the agent definitions declare rather than how they work internally)
- **Test data quality:** Good — V-Tests uses specific line numbers, grep pattern matches, and cross-file references. Evaluation sample data (implementer-09.md) demonstrates correct schema compliance with realistic scores and descriptive entries.

## Missing Test Scenarios

1. **[Priority: Medium] Cross-reference evaluation targets** — Verify each agent's evaluation targets match design.md §Per-Agent Evaluation Targets. Would catch the r-testing deviation.
2. **[Priority: Medium] Schema compliance of existing evaluation files** — Parse the YAML in `artifact-evaluations/*.md` to verify all fields present, correct types, scores in 1–10 range. Currently only spot-checked via V-Feature AC-2.
3. **[Priority: Low] Frontmatter format consistency** — Automated check that all `.agent.md` files start with `---` (not code fences). Would catch the post-mortem format issue.
4. **[Priority: Low] Evaluation step ordering verification** — Verify evaluation steps come after primary output generation and before memory writing in all 14 agents' workflow sequences.
5. **[Priority: Low] Tool restriction negative check** — Grep orchestrator.agent.md for any remaining references to prohibited tools in instructional text (not just in the restriction rule itself).

## Cross-Cutting Observations

1. **Verification depth is strong for a Markdown-only repo.** The 4-layer verification approach (V-Build → V-Tests → V-Tasks → V-Feature) provides overlapping coverage that compensates for the absence of automated tests. Each layer catches different classes of issues.

2. **Evaluation pattern consistency is excellent.** All 14 evaluating agents follow an identical template: evaluation-schema.md in Inputs, non-blocking clause, "secondary, non-blocking" in Outputs, error fallback guidance. This consistency makes future automated validation straightforward.

3. **Linting opportunity.** The structured nature of agent definitions (YAML frontmatter + Markdown sections + evaluation YAML) is highly amenable to automated linting. A simple script could validate: frontmatter format, required sections presence, evaluation target correctness, schema compliance, and tool restriction compliance. This would replace the manual V-cluster checks.

4. **PostMortem agent is the riskiest component for runtime behavior.** It depends on orchestrator context-passing (telemetry), file discovery (evaluation files), YAML parsing, and multi-step computation (averaging scores). None of these can be validated without a pipeline run. The self-verification step (Step 9) partially mitigates this.

## Summary

- **Blocker:** 0
- **Major:** 0
- **Minor:** 4 (1 evaluation target deviation, 1 V-Tests coverage gap, 1 runtime validation limitation, 1 inherited format warning)

All findings are non-blocking. The structural validation performed by the V cluster (22 checks + 14 ACs + 11 tasks) provides adequate confidence for a Markdown-only repository. The r-testing evaluation target deviation is the only concrete inconsistency found, and it is additive rather than destructive. The inherited post-mortem.agent.md format warning should be addressed but does not affect functional correctness.
