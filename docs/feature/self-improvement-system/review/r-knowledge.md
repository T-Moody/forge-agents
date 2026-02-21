# Knowledge Evolution Analysis

## Instruction Suggestions

### [Category: instruction-update] Create Cross-Cutting Concern Pattern Instruction

- **File:** `.github/instructions/cross-cutting-concerns.instructions.md`
- **Change:** Add (new file)
- **Rationale:** The self-improvement feature demonstrated a scalable pattern for adding a cross-cutting capability to multiple agents: (1) define the schema once in a shared reference document, (2) add a consistent workflow step to each affected agent referencing the shared document. This pattern was applied successfully to 14 agents with zero inconsistencies. Future cross-cutting features (e.g., adding cost tracking, adding quality gates) would benefit from this documented pattern rather than re-discovering it.
- **Before:** No `.github/instructions/` directory exists.
- **After:** New instruction document describing the shared-reference-document + per-agent-step pattern, including naming conventions, non-blocking rules, and the template structure used across this feature.

### [Category: instruction-update] Create Non-Blocking Agent Pattern Instruction

- **File:** `.github/instructions/non-blocking-agent.instructions.md`
- **Change:** Add (new file)
- **Rationale:** Two non-blocking agents now exist (R-Knowledge, PostMortem) with a clear pattern: two-state completion contract (DONE/ERROR only, no NEEDS_REVISION), explicit anti-drift anchor, pipeline continues on ERROR, and the orchestrator logs a warning and skips memory merge on failure. Documenting this pattern ensures future additive agents follow the same convention.
- **Before:** No instruction exists; the pattern is implicit in `r-knowledge.agent.md` and `post-mortem.agent.md`.
- **After:** Instruction covering: when to use non-blocking vs. three-state contracts, required sections (File Boundaries, Read-Only Enforcement, Anti-Drift Anchor with non-blocking statement), orchestrator integration (non-blocking ERROR handling, skip-merge behavior).

### [Category: instruction-update] Create Agent Definition Format Instruction

- **File:** `.github/instructions/agent-definition-format.instructions.md`
- **Change:** Add (new file)
- **Rationale:** All 20 active agents follow a consistent format: bare `---` YAML frontmatter (not code-fenced), standard section order (Title/Role, Inputs, Outputs, Operating Rules ×6, Workflow, Completion Contract, Anti-Drift Anchor). The `post-mortem.agent.md` format warning (code fence wrapper instead of bare `---`) demonstrates that without a documented format standard, implementers can deviate. An explicit instruction would prevent this class of error.
- **Before:** Format convention is implicit — inferred only by reading existing agents.
- **After:** Instruction specifying: bare `---` frontmatter (never code-fenced), required `name` and `description` fields, required section order, Operating Rules template (6 rules), and naming conventions (`<name>.agent.md`, lowercase hyphen-separated).

## Skill Suggestions

### [Category: skill-update] Implementer: Verify Format Consistency with Existing Files

- **Agent:** implementer
- **Change:** Update (behavioral guidance addition)
- **Rationale:** Task 02 (PostMortem agent creation) introduced a format inconsistency — the file was wrapped in a ` ```chatagent ` code fence instead of using bare `---` YAML frontmatter like all 19 other active agents. This was caught by V-Build but persisted through the pipeline as a low-severity warning. Adding explicit format-consistency verification to the implementer's self-reflection step would catch this earlier.
- **Before:** Self-Reflection step 7 checks: "All task requirements addressed, tests cover ACs, no omissions, code follows conventions."
- **After:** Self-Reflection additionally checks: "For new `.agent.md` files — does the file format (frontmatter style, section order) match existing agents in the same directory? Verify by comparing with an existing agent file."

### [Category: skill-update] V-Build: Add Frontmatter Format Validation as a Blocking Check

- **Agent:** v-build
- **Change:** Update (promote format warning to structured check)
- **Rationale:** V-Build detected the post-mortem.agent.md format issue but classified it as a non-blocking warning. Since chatagent parser compatibility directly affects whether an agent is functional, frontmatter format deviation should be a more prominent structural check — potentially blocking if the file would not be parsed correctly.
- **Before:** V-Build checks for file existence and YAML frontmatter presence but treats code-fence wrapping as a warning.
- **After:** V-Build includes an explicit structural check: "All `.agent.md` files in `.github/agents/` must start with bare `---` YAML frontmatter at line 1 (not wrapped in code fences)." Deviation is flagged as a finding (not just a warning) with recommended fix.

## Pattern Captures

### [Category: pattern-capture] Shared Reference Document for Cross-Agent Schemas

- **Pattern:** When introducing a structured data format used by multiple agents, define the schema once in a shared reference document placed in `.github/agents/` alongside agent definitions. Each consuming agent references the document by path rather than inlining the schema. The reference document includes: schema definition, field-level constraints, error fallback format, behavioral rules (non-blocking, secondary to primary output), collision avoidance, and version identifier.
- **Evidence:** `.github/agents/evaluation-schema.md` — referenced by 14 evaluating agents via `evaluation-schema.md` path reference in their Inputs and workflow steps. Zero inconsistencies across all 14 agents. Schema changes require updating only one file.
- **Applicability:** Any future feature requiring a shared structured format across multiple agents (e.g., cost-tracking schema, performance metrics schema, agent health-check schema). Also applicable if the evaluation schema itself needs versioning — the `v1` identifier enables future `v2` migration.

### [Category: pattern-capture] Telemetry-in-Context Accumulation

- **Pattern:** For pipeline-wide metadata that must be collected across all steps but only consumed once at the end, accumulate data in the orchestrator's working context (not in files or shared memory) and pass it as dispatch context to the consuming agent. This avoids intermediate file writes, prevents shared memory bloat, and eliminates tool-dependency risks for the orchestrator.
- **Evidence:** Orchestrator Global Rule 13 accumulates telemetry after each `runSubagent` return; Step 8.1 passes the full dataset as a structured Markdown table in the PostMortem dispatch prompt. This pattern resolved 3 CT findings (CT-3, CT-7: memory.md bloat; CT-maintainability F4: parsing contract).
- **Applicability:** Any future pipeline-wide metric or audit trail that is accumulated incrementally and consumed as a batch at the end. Constraint: data volume must fit in the orchestrator's context window alongside all other pipeline activity (~15-25KB for telemetry in a full run with 25-40 dispatches).

### [Category: pattern-capture] Dual-Track Implementation for Shared File Modifications

- **Pattern:** When two independent changes target the same file, assign them to separate implementation tracks with an explicit dependency ordering. Track B (smaller/foundational change) is sequenced before Track A (larger/additive change). The later task reads the file post-earlier-task changes. This prevents merge conflicts without requiring fine-grained line-level coordination.
- **Evidence:** Task 03 (Track B: tool restriction) modified `orchestrator.agent.md` first; Task 10 (Track A: Step 8 + telemetry) depended on Task 03 and applied additive changes on top. Both tasks completed successfully with no conflicts (confirmed by V-Tasks and V-Feature verification).
- **Applicability:** Any feature where multiple implementation tasks modify the same large file. The key constraint is that the dependency must be explicitly declared in the task `depends_on` field and the planner must sequence accordingly.

### [Category: pattern-capture] Non-Blocking Evaluation Wrapper for Additive Agent Capabilities

- **Pattern:** When adding a secondary capability to existing agents (e.g., evaluation, metrics collection), wrap it as a non-blocking workflow step: (1) place after primary work completion, (2) include explicit "failure MUST NOT cause ERROR" clause, (3) provide a graceful degradation fallback (e.g., `evaluation_error` block), (4) mark as "secondary, non-blocking" in the Outputs section. This ensures the additive capability cannot regress existing agent behavior.
- **Evidence:** All 14 evaluating agents follow this pattern — each has "evaluation failure MUST NOT cause your completion status to be ERROR" and the evaluation-schema.md provides the `evaluation_error` fallback block. V-Feature verified AC-13 (evaluation non-blocking) across all 14 agents.
- **Applicability:** Any future additive capability added to existing agents where the primary output must remain unaffected. The pattern is generalizable beyond evaluations to any secondary data-collection or reporting step.

### [Category: pattern-capture] Memory-First Reading for Context-Window-Constrained Agents

- **Pattern:** Agents that must process many files (e.g., PostMortem reading 14+ evaluation files) should use a memory-first reading strategy: (1) read compact memory files for orientation, (2) identify which files need detailed examination, (3) use grep_search for score/metric extraction, (4) read full content only for problem files. This prevents context window saturation.
- **Evidence:** PostMortem agent definition includes a 5-step Reading Strategy based on this pattern (design.md CT-5 resolution). The orchestrator's memory-first protocol is the existing precedent for this pattern.
- **Applicability:** Any agent that must process a variable number of input files that could grow with pipeline complexity. Particularly relevant for analysis/reporting agents.

## Decision Log Entries

### [Decision: Centralized Schema Reference vs. Inline Schema]

- **Context:** The self-improvement system requires 14 agents to produce artifact evaluations in a consistent YAML format. The initial request specified the schema inline. CT-maintainability identified that duplicating the schema across 14 agent files creates a maintenance burden and divergence risk.
- **Decision:** Define the evaluation schema once in a shared reference document (`.github/agents/evaluation-schema.md`) that all 14 agents reference by path. The document includes the full schema, field constraints, error fallback, behavioral rules, and a version identifier.
- **Rationale:** Single-source-of-truth eliminates 14-file duplication. Schema evolution requires updating only one file. Version identifier (`v1`) enables future migration. The pattern is consistent with the existing `dispatch-patterns.md` reference document in the same directory.
- **Scope:** Project-wide (establishes a pattern for future shared schemas).
- **Affected Components:** `.github/agents/evaluation-schema.md` (new), 14 evaluating agent `.agent.md` files (all reference it), `.github/agents/post-mortem.agent.md` (references it).

### [Decision: Telemetry in Orchestrator Context vs. File-Based Storage]

- **Context:** The initial request (PART 2) specified orchestrator telemetry tracking with a structured run log. The original design stored telemetry in a `## Telemetry` section of `memory.md`. CT cluster identified this as problematic: unbounded growth in shared memory (CT-scalability F1), memory tool dependency (CT-security, CT-maintainability), and an implicit parsing contract between orchestrator and PostMortem (CT-maintainability F4).
- **Decision:** Orchestrator accumulates telemetry in its working context (not in any file) during Steps 1–7. At Step 8, it passes the full dataset as structured Markdown in the PostMortem dispatch prompt. PostMortem writes the run-log file.
- **Rationale:** Eliminates memory.md growth risk, removes memory tool dependency for telemetry, and makes the data flow explicit (orchestrator → PostMortem dispatch prompt → PostMortem writes). Trade-off: telemetry is not persisted during Steps 1–7, so a crash loses the data. Accepted because modern context windows (100K+) accommodate the volume (~15-25KB) and the alternative (file-based storage) conflicted with the write-tool restriction.
- **Scope:** Feature-specific (telemetry architecture), with project-wide implication (establishes the in-context accumulation pattern).
- **Affected Components:** `orchestrator.agent.md` (Global Rule 13, Step 8.1), `post-mortem.agent.md` (Workflow Steps 1-2).

### [Decision: PostMortem Quantitative-Only Boundary vs. R-Knowledge Qualitative Domain]

- **Context:** The initial request (PART 3) originally included `improvement_recommendations` in the PostMortem report. CT-strategy Finding F5 identified that this overlaps with R-Knowledge's qualitative improvement suggestions, creating scope ambiguity and potential contradictory outputs.
- **Decision:** PostMortem produces quantitative metrics only (scores, frequencies, retry rates, accuracy counts). It does NOT produce improvement_recommendations, knowledge-suggestions.md, or decisions.md. R-Knowledge remains the sole source of qualitative improvement suggestions.
- **Rationale:** Clear boundary: PostMortem answers "what happened" (quantitative); R-Knowledge answers "what should change" (qualitative). This prevents contradictory outputs and confusion about which agent's recommendations to follow. The design enforces this via three explicit prohibition statements in the PostMortem agent definition plus a self-verification check.
- **Scope:** Project-wide (establishes the quantitative/qualitative agent boundary pattern).
- **Affected Components:** `post-mortem.agent.md` (Outputs, Anti-Drift Anchor, Self-verification Step 9), `r-knowledge.agent.md` (unmodified — scope preserved).

### [Decision: Orchestrator Tool Restriction — Keep Read Tools per CT-1 Critical Finding]

- **Context:** The initial request and feature.md (FR-6.1, AC-5) specified restricting the orchestrator to `[agent, agent/runSubagent, memory]` only — removing all read tools. CT-strategy identified this as a Critical severity finding: removing `read_file`, `grep_search`, `semantic_search` would break the orchestrator's cluster decision flows (CT, V, R clusters all require the orchestrator to read sub-agent memory files for routing decisions).
- **Decision:** Keep all read tools (`read_file`, `grep_search`, `semantic_search`, `file_search`, `list_dir`, `get_errors`). Remove only write/execute tools (`create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`). This deviates from the literal AC-5 text but preserves the spirit of FR-6 (orchestrator delegates all writes).
- **Rationale:** The Critical finding was the highest-severity issue in the CT cluster and would have made the pipeline non-functional. The design revision (CT-1) explicitly documents this as an "FR-6 deviation" with full rationale. V-Feature verified AC-5 against the design revision (not the literal feature.md text).
- **Scope:** Project-wide (orchestrator tool access is a foundational constraint).
- **Affected Components:** `orchestrator.agent.md` (Operating Rule 5, Global Rule 1, Step 0, Anti-Drift Anchor).

### [Decision: Non-Blocking Evaluation Pattern]

- **Context:** Adding artifact evaluations to 14 existing agents created a risk of breaking existing behavior if evaluation generation fails (e.g., due to context window pressure, malformed upstream artifacts, or tool errors).
- **Decision:** Evaluation is always secondary and non-blocking. Evaluation failure must NEVER cause an agent's completion status to be ERROR. If evaluation fails, agents write an `evaluation_error` fallback block and proceed with their primary output intact.
- **Rationale:** The initial request (PART 5) requires "no breaking changes." Making evaluation blocking would mean that a failure in the new additive capability could regress existing pipeline behavior. The `evaluation_error` fallback provides diagnostic data even when full evaluation fails. This is consistent with the pipeline's existing non-blocking patterns (R-Knowledge, PostMortem).
- **Scope:** Project-wide (establishes precedent for how additive secondary outputs should behave).
- **Affected Components:** `evaluation-schema.md` (Rules 4-5), all 14 evaluating agent `.agent.md` files.

## Workflow Improvements

### [Category: workflow-improvement] Add Structural Linting Step to V-Build

- **What:** V-Build currently performs manual structural checks via grep/read. A lightweight linting script (e.g., PowerShell or Python) could automate frontmatter validation, section-order verification, and cross-reference checks.
- **Why:** The `post-mortem.agent.md` code-fence format issue persisted from implementation (Task 02) through V-Build (warning only) and was still present at final verification. An automated check would catch this during the build step, not as a late-stage warning.
- **Impact:** Would prevent format-consistency issues from reaching verification stages. Low implementation effort (read first 5 lines of each `.agent.md`, check for `---` at line 1).
- **Risk:** Low — adds validation without modifying any existing behavior.

### [Category: workflow-improvement] Spec-Design Reconciliation Step After CT Revisions

- **What:** When the design deviates from the spec due to CT cluster findings (as happened with AC-5/FR-6.1), the spec should be updated to reflect the CT-approved design revision. Currently, the spec remains unchanged, creating ambiguity during verification.
- **Why:** V-Feature had to verify AC-5 "per design revision, not literal AC-5 text" and CT-strategy Finding 4 explicitly flagged this as a testability ambiguity. A spec reconciliation step after design revision would eliminate this class of issue.
- **Impact:** Reduced ambiguity during V-cluster verification. Medium effort (designer or spec agent reconciles after CT revisions).
- **Risk:** Medium — adds a step to the pipeline but improves spec-implementation alignment.

## Summary

- **Total suggestions:** 9
- **Instruction updates:** 3
- **Skill updates:** 2
- **Pattern captures:** 5
- **Workflow improvements:** 2
- **Rejected (safety filter):** 0
