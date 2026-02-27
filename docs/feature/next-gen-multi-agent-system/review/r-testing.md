# Review: Testing

## Review Tier

**Full** — The system is a deterministic multi-agent pipeline with 9 agents, 10 YAML schemas, 2 SQLite tables, and SQL-based evidence gates. Verification is a core architectural concern (not a secondary concern), so full-depth testing review is warranted.

---

## Findings

### [Severity: Major] F-1: No EC-4 (SQLite Unavailable) Fallback Implementation in Any Agent

- **What:** EC-4 specifies that when SQLite is unavailable, the system must fall back to structured YAML evidence records. Neither the Orchestrator, Verifier, nor any other agent defines this fallback path. The Verifier includes a "safety net" `CREATE TABLE IF NOT EXISTS` but no YAML fallback. The Orchestrator has no conditional logic for SQLite absence.
- **Where:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) (Step 0), [verifier.agent.md](NewAgents/.github/agents/verifier.agent.md) (Safety Net Initialization), [schemas.md](NewAgents/.github/agents/schemas.md) (SQLite Schemas)
- **Why:** feature.md EC-4 explicitly requires: "If SQL was never available: use YAML throughout and log at pipeline start." No agent defines how YAML evidence records would be structured, stored, or queried as a substitute. This means the system cannot self-verify in environments where SQLite is unavailable.
- **Suggested Fix:** Add a YAML evidence fallback path in the Orchestrator (Step 0 detection) and Verifier (alternate recording mode). Define a YAML evidence record schema in schemas.md.
- **Affects Tasks:** Task 01 (schemas.md), Task 04 (orchestrator), Task 09 (verifier)

### [Severity: Major] F-2: No EC-2 (Approval Mode Timeout) Handling Defined

- **What:** EC-2 specifies that on interactive approval timeout, the pipeline must NOT auto-proceed, must preserve state, and must re-present the prompt on resume. The orchestrator's approval gate sections (Steps 1a, 4a) define interactive vs. autonomous modes but contain no timeout handling, state persistence for interrupted sessions, or re-prompt logic.
- **Where:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md#L218-L236) (Step 1a, 4a approval gates)
- **Why:** Without explicit timeout handling, the pipeline behavior on approval timeout is undefined — it could hang indefinitely or auto-proceed, both violating EC-2.
- **Suggested Fix:** Add timeout handling to each approval gate in the orchestrator: detect timeout condition, persist pipeline state to disk, and define resume behavior that re-presents the prompt.
- **Affects Tasks:** Task 04 (orchestrator)

### [Severity: Major] F-3: No EC-8 (Context Window Exceeded) Handling Defined

- **What:** EC-8 specifies that when an agent's context window is exceeded, the system must use memory-first reading, artifact indexes, and request summarized context from the orchestrator. While agents mention "context-efficient reading" in their operating rules, no agent defines: (a) how to detect context window exceeded, (b) how to request summarized context from orchestrator, or (c) how the orchestrator would provide summarized context.
- **Where:** All agent definitions (Operating Rules sections)
- **Why:** Context window limits are a real risk (noted as Medium likelihood, High impact in feature.md Risks). Without a defined escalation path, agents will silently truncate or fail.
- **Suggested Fix:** Define a context-exceeded detection mechanism (e.g., tool error response patterns) and an orchestrator protocol for providing compressed context. Add to dispatch-patterns.md as a cross-cutting operational rule.
- **Affects Tasks:** Task 02 (dispatch-patterns), Task 04 (orchestrator)

### [Severity: Major] F-4: No EC-9 (Model Routing Unavailable) Fallback in Adversarial Reviewer or Orchestrator

- **What:** EC-9 specifies fallback to same-model-different-prompt review when model routing is unavailable. The adversarial reviewer accepts a `model` parameter but has no fallback logic. The orchestrator dispatches with intended model assignments (gpt-5.3-codex, gemini-3-pro-preview, claude-opus-4.6) but defines no detection of routing unavailability or fallback dispatch.
- **Where:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) (Steps 3b, 7), [adversarial-reviewer.agent.md](NewAgents/.github/agents/adversarial-reviewer.agent.md) (Input Schema)
- **Why:** Model routing is noted as platform-dependent with Medium likelihood of unavailability. Without fallback, the pipeline would halt or produce undefined behavior when the platform can't route to specific models.
- **Suggested Fix:** Add model routing detection in Step 3b/7 of the orchestrator. Define fallback: dispatch 3 same-model instances with distinct review_focus values. Log degradation warning.
- **Affects Tasks:** Task 04 (orchestrator), Task 10 (adversarial-reviewer)

### [Severity: Minor] F-5: Orchestrator Decision Table Missing EC-6 (Insufficient Research) Recovery Path Detail

- **What:** The decision table has "Research outputs | <2 DONE after retry | ERROR — insufficient research" but EC-6 specifies a richer behavior: "present available findings to the user with a warning... and offer to proceed or abort (in interactive mode). In autonomous mode, proceed with available findings and log the gap." The decision table routes directly to ERROR rather than the nuanced EC-6 behavior.
- **Where:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) (Orchestrator Decision Table)
- **Why:** The decision table is the authoritative routing reference. Its simplified routing contradicts the richer EC-6 recovery expected by the spec.
- **Suggested Fix:** Expand the decision table entry for `<2 DONE after retry` to differentiate interactive (offer proceed/abort) vs. autonomous (proceed with warning) modes, matching EC-6.
- **Affects Tasks:** Task 04 (orchestrator)

### [Severity: Minor] F-6: `output_snippet` Length Enforcement is Documentation-Only

- **What:** The `anvil_checks` schema has `CHECK(LENGTH(output_snippet) <= 500)` and agents are instructed to truncate. But no agent self-verification checklist actually validates this constraint before INSERT. If an agent inserts >500 characters, the SQL CHECK constraint would reject the INSERT — but the agent would receive a SQL error with no recovery defined.
- **Where:** [verifier.agent.md](NewAgents/.github/agents/verifier.agent.md) (Operating Rule 4), [schemas.md](NewAgents/.github/agents/schemas.md) (anvil_checks table)
- **Why:** SQL CHECK constraint rejection on INSERT is a deterministic error that should not be retried. Without explicit truncation validation in self-verification, this becomes a silent failure mode.
- **Suggested Fix:** Add `output_snippet ≤ 500 chars verified` to the Verifier and Adversarial Reviewer self-verification checklists.
- **Affects Tasks:** Task 09 (verifier), Task 10 (adversarial-reviewer)

### [Severity: Minor] F-7: Verifier Tier 1 Description in README Mismatches Agent Definition

- **What:** The README describes Tier 1 as "Schema validation, lint checks" but the Verifier agent defines Tier 1 as "IDE Diagnostics + Syntax/Parse Check." Lint is actually Tier 2 in the Verifier. This creates a misleading documentation trail for anyone verifying the system's test behavior.
- **Where:** [README.md](NewAgents/README.md) (Per-Task Verifier with 4-Tier Cascade), [verifier.agent.md](NewAgents/.github/agents/verifier.agent.md) (Tier 1, Tier 2)
- **Why:** Mismatched documentation could cause incorrect expectations about what each verification tier covers.
- **Suggested Fix:** Update README tier descriptions to match the Verifier agent definition exactly.
- **Affects Tasks:** Task 13 (README)

### [Severity: Minor] F-8: Dispatch Patterns Evidence Gate SQL Missing task_id Filter

- **What:** The evidence gate SQL queries in dispatch-patterns.md (§Evidence Gating) omit the `task_id` filter, using only `run_id` and `round`. The equivalent queries in orchestrator.agent.md correctly include `task_id`. This inconsistency could mislead an implementer who references dispatch-patterns.md instead of the orchestrator.
- **Where:** [dispatch-patterns.md](NewAgents/.github/agents/dispatch-patterns.md) (Evidence Gating section), vs. [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) (Evidence Gate SQL Query Templates)
- **Why:** dispatch-patterns.md is a reference document. Inconsistent queries between it and the orchestrator could cause bugs if an agent author copies from the wrong source.
- **Suggested Fix:** Add `AND task_id = '{task_id}'` to the evidence gate queries in dispatch-patterns.md to match the orchestrator.
- **Affects Tasks:** Task 02 (dispatch-patterns)

### [Severity: Minor] F-9: Spec Agent Self-Verification Missing Edge Case Coverage Check for All EC-1 through EC-10

- **What:** The Spec Agent's self-verification includes "Edge case coverage: Edge cases cover failure modes and error scenarios, not just happy paths" which is generic. It doesn't verify that the produced spec actually covers all 10 edge cases defined in the feature.md spec (EC-1 through EC-10). Since the Spec is the agent that _produces_ the edge cases, its self-check should confirm completeness.
- **Where:** [spec.agent.md](NewAgents/.github/agents/spec.agent.md) (Self-Verification section, item 4)
- **Why:** Without explicit EC coverage validation, a spec run could produce fewer edge cases than required.
- **Suggested Fix:** Add a self-verification item: "All edge cases from initial-request.md are addressed (mapped or explicitly not applicable)."
- **Affects Tasks:** Task 07 (spec)

---

## Coverage Assessment

- **Changed files with tests (self-verification sections):** 9 of 9 agents have self-verification sections
- **Coverage gaps:**
  - EC-2 (approval timeout): No implementation in any agent
  - EC-4 (SQLite fallback): No YAML fallback path defined
  - EC-8 (context window): No detection/escalation protocol
  - EC-9 (model routing): No fallback dispatch logic
  - EC-6 (insufficient research): Decision table oversimplifies vs. spec
- **Edge cases with test coverage:**

| Edge Case                    | Addressed in Implementation? | Where                                                                              |
| ---------------------------- | ---------------------------- | ---------------------------------------------------------------------------------- |
| EC-1 (Review disagreement)   | ✅ Fully                     | Orchestrator decision table, adversarial-reviewer disagreement resolution protocol |
| EC-2 (Approval timeout)      | ❌ Not addressed             | No agent defines timeout handling                                                  |
| EC-3 (Invalid output)        | ✅ Fully                     | Orchestrator error classification (schema violation → retry once → ERROR)          |
| EC-4 (SQLite unavailable)    | ❌ Not addressed             | No fallback path defined                                                           |
| EC-5 (Partial failure)       | ✅ Fully                     | Orchestrator Pipeline State Recovery section with `list_dir`-based scanning        |
| EC-6 (Insufficient research) | ⚠️ Partial                   | Decision table says ERROR; spec says interactive/autonomous nuance                 |
| EC-7 (Replan loop exhaust)   | ✅ Fully                     | Pattern B max 3 iterations → proceed with Low confidence                           |
| EC-8 (Context window)        | ❌ Not addressed             | Only generic "context-efficient reading" advice                                    |
| EC-9 (Model routing)         | ❌ Not addressed             | No detection or fallback logic                                                     |
| EC-10 (Concurrency limit)    | ✅ Fully                     | Sub-wave partitioning in dispatch-patterns.md and orchestrator                     |

## Test Quality Assessment

- **Behavioral tests (self-verification):** All 9 agents have behavioral self-verification checklists that test observable outputs (schema compliance, file existence, field completeness). These are well-designed — they check _what_ was produced, not _how_ it was produced.
- **Implementation-detail tests:** 0 — no self-verification checks test implementation internals.
- **Test data quality:** Good — schemas.md provides complete examples for all 10 schemas with realistic sample data, correct field types, and meaningful values. SQL INSERT examples in the Verifier are thorough with 6 concrete examples covering all tiers.

## Missing Test Scenarios

1. **SQLite fallback path (EC-4):** No agent defines how to verify behavior when SQLite is unavailable. Need: detection mechanism + YAML evidence format + queries on YAML.
2. **Approval timeout (EC-2):** No pipeline state persistence for interrupted interactive sessions. Need: state serialization + resume detection + re-prompt.
3. **Context window exceeded (EC-8):** No detection protocol or escalation to orchestrator for summarized context. Need: error detection + orchestrator protocol.
4. **Model routing fallback (EC-9):** No detection of model routing unavailability or same-model-different-prompt fallback. Need: detection + fallback dispatch + degradation logging.
5. **Cross-agent schema violation propagation:** What happens if a Researcher outputs a valid YAML file but with wrong field names that pass structural parsing but fail semantic expectations? The Spec agent's reading logic has no validation of upstream schema.
6. **Concurrent SQLite writes under load:** While WAL + busy_timeout are mandated, no verification step confirms WAL mode was actually set. The Verifier could check `PRAGMA journal_mode` as part of its baseline.
7. **git tag -f race condition:** If two pipeline runs start simultaneously on the same branch, `git tag -f pipeline-baseline-{run_id}` could collide. No mitigation defined.

## Cross-Cutting Observations

- **Strength:** The SQL-first evidence approach with INSERT-before-report is a robust self-verification pattern. Every verification check produces a durable record that the orchestrator independently queries. This is a genuine anti-hallucination mechanism.
- **Strength:** Self-verification checklists in all 9 agents are consistently structured and test behavioral outputs. The checklists are actionable — each item can be checked mechanically.
- **Strength:** The 4-tier verification cascade is well-defined with clear progression (always → conditional → fallback → operational readiness). Each tier has specific check_name conventions that enable SQL LIKE filtering.
- **Strength:** The completion contract system (DONE/NEEDS_REVISION/ERROR) with embedded evidence_summary enables automated routing without parsing prose.
- **Gap:** No agent validates upstream schema compliance on _read_. Each agent self-validates its own output, but no agent validates that the document it is _consuming_ matches the expected schema. A malformed upstream output could silently propagate.
- **Gap:** The orchestrator is described as performing evidence gate _verification_ via SQL queries, but it is not described as performing schema _validation_ of agent outputs. The orchestrator explicitly says "You NEVER perform schema validation (agents self-validate)." This creates a trust gap — if an agent's self-validation fails silently, the orchestrator won't catch it.

## Summary

- **Blocker:** 0
- **Critical:** 0
- **Major:** 4 (F-1: SQLite fallback, F-2: approval timeout, F-3: context window, F-4: model routing)
- **Minor:** 5 (F-5: decision table simplification, F-6: output_snippet enforcement, F-7: README tier mismatch, F-8: dispatch-patterns SQL inconsistency, F-9: spec EC coverage)
- **Total:** 9 findings (4 Major, 5 Minor)
- **EC coverage:** 6/10 edge cases fully addressed, 1 partially, 3 not addressed
- **Self-verification quality:** All 9 agents have behavioral self-verification — strong coverage of output correctness
- **Schema validation quality:** schemas.md is comprehensive (10 schemas, 2 SQLite tables, complete examples, evolution strategy). Sufficient for output validation. Missing upstream _input_ validation.
