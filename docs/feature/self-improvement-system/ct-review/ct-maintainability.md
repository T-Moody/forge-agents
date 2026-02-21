# Critical Review: Maintainability (Revision 1 Re-Review)

> **Re-review** of revised design addressing 1 Critical + 6 High findings from the CT cluster. Key design changes: shared evaluation schema (CT-4), context-based telemetry (CT-3/CT-7), write-tool restriction separated as Track B (CT-2), orchestrator retains read tools (CT-1), PostMortem quantitative-only output (CT-6), memory-first reading pattern (CT-5).

---

## Previous Findings Disposition

| #   | Previous Finding                                                       | Previous Severity | Disposition             | Notes                                                                                                                                                                   |
| --- | ---------------------------------------------------------------------- | ----------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| F1  | Evaluation schema duplicated across 14 agents                          | High              | **RESOLVED**            | Schema centralized in `.github/agents/evaluation-schema.md`. Single-point evolution. See residual in F-N2.                                                              |
| F2  | Orchestrator complexity explosion (~10 new concerns)                   | High              | **PARTIALLY MITIGATED** | Telemetry section in memory.md removed, "never prune" exception removed, tool restriction separated to Track B. Track A still adds 6–7 concerns. Downgraded — see F-R1. |
| F3  | `memory` tool scope — foundation for telemetry, Step 0, merge          | High              | **RESOLVED**            | Telemetry no longer depends on memory tool. Residual uses (Step 0 init, merges) are pre-existing behaviors with existing fallbacks.                                     |
| F4  | Compact telemetry DSL — custom mini-DSL with implicit parsing contract | High              | **RESOLVED**            | Telemetry delivered as structured Markdown tables in orchestrator dispatch context. No custom DSL, no parsing contract.                                                 |
| F5  | "Never prune" telemetry in memory.md — unbounded growth                | Medium            | **RESOLVED**            | Telemetry not stored in memory.md. Zero shared-memory impact.                                                                                                           |
| F6  | PostMortem massive read surface — fragile coupling                     | Medium            | **PARTIALLY MITIGATED** | Memory-first reading with targeted grep extraction. Coupling surface remains but read strategy is context-efficient. Downgraded — see F-R2.                             |
| F7  | First YAML output in codebase — no validation pipeline                 | Medium            | **UNRESOLVED**          | No changes. 14 agents producing YAML with skip-on-corrupt as only defense. See F-R3.                                                                                    |
| F8  | Collision avoidance — agents need file-existence checking              | Medium            | **IMPROVED**            | Consistent sequence suffix strategy across all file types. Downgraded — see F-N4.                                                                                       |
| F9  | Workflow step insertion numbering ambiguity                            | Medium            | **UNRESOLVED**          | "N." placeholder unchanged. Downgraded to Low — cosmetic. See F-N5.                                                                                                     |
| F10 | PostMortem / R-Knowledge boundary blur                                 | Medium            | **RESOLVED**            | `improvement_recommendations` removed. Anti-Drift Anchor enforces "quantitative metrics only."                                                                          |
| F11 | Per-feature scoping limits cross-feature analysis                      | Low               | **UNCHANGED**           | By design — follows existing conventions. Accepted limitation.                                                                                                          |

**Summary:** 4 of 11 findings fully resolved (F1, F3, F4, F5). 1 resolved with clean structural change (F10). 2 partially mitigated and downgraded (F2, F6). 2 unresolved but downgraded (F8→Low, F9→Low). 1 unresolved at original severity (F7). 1 unchanged/accepted (F11).

---

## Findings

### Residual Findings (Carried Forward, Re-assessed)

### [Severity: Medium] F-R1: Orchestrator Cognitive Burden — Telemetry Accumulation as Implicit State

- **What:** The revised design moves telemetry out of memory.md (good) but into the orchestrator's "working context" — meaning the LLM must mentally track and maintain a growing telemetry table across 20+ sequential `runSubagent` dispatches. After each dispatch, the orchestrator must observe the completion contract, read the memory file, and "note the dispatch metadata" in its working context. There is no explicit mechanism for how the orchestrator persists this growing dataset across turns. An LLM does not have persistent mutable state — it has a conversation history. If the orchestrator doesn't actively write out the accumulated telemetry table in its responses, it may silently drop or mangle entries. Over a full pipeline run with 20+ agents (plus Pattern C replan iterations potentially tripling V-cluster entries), this is a non-trivial accumulation task. The design specifies a Markdown table format for the Step 8 dispatch prompt but doesn't instruct the orchestrator to build this table incrementally. The instructions say "accumulates in working context" — which is an implementation-level gap, not a design specification.
- **Where:** design.md §Telemetry Accumulation — "The orchestrator accumulates telemetry data in its working context throughout the pipeline run"; design.md §Telemetry dispatch prompt format — the table the orchestrator must produce at Step 8.
- **Likelihood:** Medium — LLMs can maintain lists across many turns, but accuracy degrades with length; subtle field omissions (e.g., dropping `iteration_number` for a later dispatch) would go undetected until PostMortem processes the data.
- **Impact:** Medium — lossy telemetry undermines the diagnostic value of the run-log. The design's fallback (agent memories contain completion statuses) is valid but covers only a subset of telemetry fields (status, not retry count, pattern, timestamps, etc.).
- **Assumption at risk:** That an LLM can reliably maintain a structured, growing dataset across 20+ tool call cycles without an explicit persistence mechanism.

---

### [Severity: Medium] F-R2: PostMortem Coupling Surface Remains Proportional to Agent Count

- **What:** The memory-first reading pattern (CT-5) improves context efficiency but does not reduce the coupling surface. PostMortem still depends on: (a) the evaluation YAML schema across 14 agent-produced files, (b) the memory file format across ~19 agent-produced `.mem.md` files, (c) the orchestrator-provided telemetry table format. Changes to any of these formats require PostMortem awareness. The memory-first pattern makes the reads smarter but doesn't decouple PostMortem from the number or structure of upstream outputs. As the pipeline grows (new agents, new evaluation targets), PostMortem's coupling surface grows linearly.
- **Where:** design.md §PostMortem Agent Definition → Inputs; design.md §Memory-First Reading Pattern.
- **Likelihood:** Medium — format changes to memory files or evaluation schemas are likely as the system matures.
- **Impact:** Low — PostMortem is non-blocking, so failures don't break the pipeline. Downgraded from Medium because memory-first pattern makes PostMortem more resilient to individual file issues (targeted reads skip problematic files gracefully).
- **Assumption at risk:** That evaluation file format, memory file format, and telemetry table format will remain stable or evolve in lockstep with PostMortem updates.

---

### [Severity: Medium] F-R3: YAML Validation Gap — Still No Enforcement Pipeline

- **What:** Unchanged from v0 review. This is the first structured YAML output in the codebase (confirmed: research/patterns.md §YAML/Structured Output Patterns states no YAML exists). 14 agents will produce YAML for the first time with no schema validator, no linter, and no test harness. The shared evaluation-schema.md is a reference document, not a runtime validator. PostMortem's skip-on-corrupt behavior (EC-5) is the only defense. The `evaluation_error` fallback is itself YAML — if YAML generation is the failure mode, the fallback is fragile. The design adds TS-2 (schema field validation) and PostMortem input validation (checking required fields, score ranges) — but these are post-hoc checks in PostMortem, not pre-write validation by the producing agent.
- **Where:** design.md §Data Models & DTOs → Artifact Evaluation Schema; design.md §Security → Input Validation; all 14 evaluating agent files.
- **Likelihood:** High — LLMs frequently produce YAML formatting errors (indentation, missing quotes, list/scalar confusion).
- **Impact:** Medium — each malformed evaluation is one fewer data point. With 14 producers and no validation, a 10% error rate means 1–2 lost evaluations per run. Systematic errors from specific agents may go undetected.
- **Assumption at risk:** That LLMs produce consistently valid YAML without runtime validation.

---

### New Findings (Introduced by Revision)

### [Severity: Medium] F-N1: Feature Spec / Design Conflict on AC-5 (Tool Restriction Scope)

- **What:** Feature.md AC-5 explicitly states the orchestrator allowed tools are `[agent, agent/runSubagent, memory]` only, and that "No instructions in the file reference using file-write, file-read, file-create, `grep_search`, `semantic_search`, `read_file`, `run_in_terminal`, or any other tools directly." The revised design deliberately deviates by retaining `read_file`, `grep_search`, `semantic_search`, `file_search`, `list_dir`, and `get_errors`. The design acknowledges this as a "FR-6 deviation" but the feature spec has not been updated. An implementer following the spec literally (AC-5 pass criteria) would remove read tools, breaking the pipeline — exactly the Critical issue that CT-strategy identified. This creates a normative conflict: which document is authoritative for implementation?
- **Where:** feature.md AC-5 — "No instructions in the file reference using...`read_file`...or any other tools directly"; design.md §Write-Tool Restriction — "The orchestrator retains read-only tools"; design.md §FR-6 deviation.
- **Likelihood:** Medium — the design is explicit about the deviation, but the spec is the normative implementation reference. A planner/implementer reading the spec first may not see the design's rationale.
- **Impact:** Medium — if implemented per spec, the pipeline's cluster decision flows break (Critical finding from v0). If implemented per design, AC-5 technically fails.
- **Assumption at risk:** That the design's acknowledged deviation will be understood and followed by the implementer over the spec's literal acceptance criteria.

---

### [Severity: Medium] F-N2: Shared Schema Doc Not in Agent Inputs — Reference-by-Instruction Only

- **What:** The evaluation workflow step template tells agents to "produce one `artifact_evaluation` YAML block following the schema defined in `.github/agents/evaluation-schema.md`." However, `evaluation-schema.md` is not listed in any evaluating agent's Inputs section — only in PostMortem's Inputs. The 14 evaluating agents reference the schema by instruction only. Whether the agent actually reads the file depends on LLM behavior. If an agent doesn't read the schema document, it produces YAML based on the brief workflow step instruction alone. The single-source-of-truth benefit of the CT-4 resolution only works if agents actually read the document. Without the schema in context, agents fall back to generating YAML from instruction text, which may drift from the schema over time — partially recreating the duplication problem (as duplicated understanding rather than duplicated text).
- **Where:** design.md §Evaluating Agent Changes → Workflow Step Addition — template references schema but doesn't mandate reading; design.md §Evaluating Agent Changes → Outputs Section Addition — evaluation-schema.md not in agent Inputs; contrast design.md §PostMortem Inputs — explicitly lists evaluation-schema.md.
- **Likelihood:** Medium — some LLMs will read a referenced file proactively; others may not, especially if context budget is tight after primary work.
- **Impact:** Medium — without reading the schema doc, agents produce YAML from a one-line instruction. This weakens schema compliance guarantees and partially undermines the CT-4 resolution's value.
- **Assumption at risk:** That agents will read a referenced file even when it isn't listed in their Inputs or explicitly required by their workflow step.

---

### [Severity: Low] F-N3: Feature Spec References Removed `improvement_recommendations` Field

- **What:** The revised design removes `improvement_recommendations` from the PostMortem report schema (CT-6 resolution). However, feature.md still references this field in FR-3.3 (report schema includes the field) and US-3 ("Developer uses `improvement_recommendations` to decide which agent definitions to update"). The design's CT Revision Summary table explains the change clearly, but the feature spec hasn't been updated. Testers using AC-8 ("has all top-level keys from FR-3.3") may flag the missing field as a failure.
- **Where:** feature.md FR-3.3 — schema includes `improvement_recommendations`; feature.md US-3 — references the field; design.md §Post-Mortem Report Schema — field removed with CT-6 rationale.
- **Likelihood:** High — the inconsistency exists in the current documents.
- **Impact:** Low — the design's CT Revision Summary clearly explains the change. An attentive implementer will follow the design. But testers may encounter confusion.
- **Assumption at risk:** That implementers and testers will prioritize the revised design over the unrevised spec for schema compliance.

---

### [Severity: Low] F-N4: Evaluation Step Template Lacks Explicit File-Existence Check

- **What:** The collision avoidance strategy now uses consistent sequence suffixes (improved from v0). However, the evaluation workflow step template doesn't instruct agents to check whether the evaluation file already exists before writing. The template says "Write all blocks to: `docs/feature/<feature-slug>/artifact-evaluations/<agent-name>.md`" with no mention of checking for existing files or using sequence suffixes. The collision avoidance logic in §Append-Only Semantics describes the `-2`, `-3` suffix pattern but this information isn't in the agent-facing template that will be in their prompt context.
- **Where:** design.md §Evaluating Agent Changes → Workflow Step Addition — template lacks existence check; design.md §Append-Only Semantics — collision rules defined outside agent-facing template.
- **Likelihood:** Low — same-feature re-runs are uncommon.
- **Impact:** Low — file overwrite risk in rare edge case.
- **Assumption at risk:** That agents will implement collision avoidance from rules specified outside their workflow template.

---

### [Severity: Low] F-N5: Step Numbering Remains Ambiguous Across 14 Agents

- **What:** Unchanged from v0. The evaluation step uses "N." as a placeholder. Each of 14 agents has a different step count (spec: 7 steps, designer: 13 steps, CT agents: ~11 steps). The design doesn't specify the concrete step number for each agent. The migration section claims "No existing steps are renumbered" but inserting a step between existing steps (e.g., between spec step 5 and step 6) inherently requires either renumbering or non-integer numbering. Both contradict existing conventions.
- **Where:** design.md §Evaluating Agent Changes → Workflow Step Addition; design.md §Migration & Backwards Compatibility.
- **Likelihood:** High — implementer must decide per agent.
- **Impact:** Low — functional correctness unaffected; cosmetic inconsistency.
- **Assumption at risk:** That "additive step insertion" and "no renumbering" can coexist in numbered workflows.

---

## Cross-Cutting Observations

- **CT-Strategy scope:** The feature spec / design conflict on AC-5 (F-N1) is a strategic concern — the spec's acceptance criteria form the implementation contract, and the design deviates without spec amendment. The planner needs clear guidance on which document takes precedence.

- **CT-Scalability scope:** Context-window telemetry accumulation (F-R1) has a scalability dimension — as pipeline agent count grows, the telemetry table grows linearly in the orchestrator's context. For pipelines with Pattern C replan loops, the growth could be 2–3× the base agent count.

- **CT-Security scope:** The reference-by-instruction pattern for evaluation-schema.md (F-N2) creates a soft compliance boundary. If agents don't read the schema and produce unexpected YAML structures, PostMortem's input validation is the sole defense against data integrity issues.

---

## Requirement Coverage

| Requirement                   | Coverage Status             | Notes                                                                                                        |
| ----------------------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------ |
| FR-1 (Artifact Evaluation)    | Covered — residual concerns | Schema reference-by-instruction gap (F-N2); YAML validation gap (F-R3); step numbering (F-N5)                |
| FR-2 (Orchestrator Telemetry) | Covered — new concern       | Context-window accumulation reliability (F-R1) replaces v0 concerns (DSL, memory.md bloat) — net improvement |
| FR-3 (PostMortem Agent)       | Covered — residual concern  | Coupling surface proportional to agent count (F-R2), mitigated by memory-first reads                         |
| FR-4 (Storage Design)         | Adequately addressed        | Collision avoidance instruction gap (F-N4) is minor                                                          |
| FR-5 (Pipeline Extension)     | Adequately addressed        | Orchestrator gains fewer concerns than v0 — Track B separation is material improvement                       |
| FR-6 (Tool Restriction)       | Spec/design conflict        | AC-5 contradicts design's read-tool retention (F-N1) — needs reconciliation                                  |
| FR-7 (Safety)                 | Well addressed              | Additive-only approach sound; non-blocking pattern correct                                                   |
| NFR-1 (Additive-Only)         | Minor tension               | Step numbering ambiguity (F-N5) — cosmetic, not functional                                                   |
| NFR-2 (Structured Output)     | Covered with caveat         | YAML validation gap remains (F-R3)                                                                           |
| NFR-4 (Non-Blocking)          | Well addressed              | Both evaluation and PostMortem correctly non-blocking                                                        |
| NFR-6 (Graceful Degradation)  | Well addressed              | Memory-first pattern, skip-on-corrupt, fallback hierarchy all sound                                          |
| NFR-7 (Per-Feature Isolation) | Addressed                   | Accepted limitation for cross-feature analysis (unchanged)                                                   |
