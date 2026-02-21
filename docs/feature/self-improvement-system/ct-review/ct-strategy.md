# Critical Review: Strategy (Re-Review — Revision 1)

## Previous Findings Resolution

| #   | Previous Finding                                              | Previous Severity | Status                              | Notes                                                                                                                |
| --- | ------------------------------------------------------------- | ----------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| F1  | Tool restriction removes read tools, breaks cluster decisions | Critical          | **RESOLVED**                        | Design retains all read tools; only write/execute tools removed. §Orchestrator Write-Tool Restriction                |
| F2  | Bundling restrictive + additive changes compounds risk        | High              | **RESOLVED**                        | Tool restriction separated into independent Track B. §Implementation Checklist                                       |
| F3  | memory.md telemetry bloat (never pruned, ~3.5-7KB/run)        | High              | **RESOLVED**                        | Telemetry moved entirely out of memory.md; orchestrator accumulates in context. §Telemetry Accumulation              |
| F4  | LLM self-evaluation produces low-signal scores                | High              | **RETAINED (downgraded to Medium)** | Fundamental value-proposition risk not addressable by design changes. See Finding 1.                                 |
| F5  | improvement_recommendations blurs R-Knowledge boundary        | Medium            | **RESOLVED**                        | Field removed from PostMortem schema. §Post-Mortem Report Schema                                                     |
| F6  | Collision avoidance inconsistent across file types            | Medium            | **PARTIALLY RESOLVED**              | Consistent sequence suffix adopted, but instructions missing from evaluating agent workflow template. See Finding 3. |
| F7  | System doesn't close the feedback loop                        | Medium            | **RETAINED**                        | By design — no feedback mechanism added. See Finding 2.                                                              |
| F8  | Orchestrator contradicts its own tool usage rules             | Medium            | **RESOLVED**                        | Track B explicitly resolves Step 0/Rule 5 contradiction. §Step 0 Modifications                                       |
| F9  | Telemetry provides minimal incremental value                  | Low               | **RETAINED**                        | Fundamental value question; not addressable by design changes.                                                       |
| F10 | Spec agent evaluates 4 files disproportionately               | Low               | **RETAINED**                        | Design choice; low impact.                                                                                           |

**Summary:** 5 of 10 findings fully resolved, 1 partially resolved, 4 retained (mostly fundamental value concerns). The 1 Critical and 2 of 3 High findings are resolved. No new Critical or High findings identified.

---

## Findings

### [Severity: Medium] Finding 1: LLM Self-Evaluation Quality Remains the Core Value-Proposition Risk

- **What:** The system's value depends on LLMs producing calibrated, differentiated evaluation scores. LLMs tend toward sycophantic scoring (clustering 7-8 on a 1-10 scale) and generic feedback. The revised PostMortem — now purely quantitative with no `improvement_recommendations` — is even more dependent on score quality since qualitative recommendations were removed as the compensating narrative layer. If scores are uniformly high and non-differentiated, the post-mortem report becomes a bureaucratic artifact with flat `agent_accuracy_scores` and empty `recurring_issues`. The free-text list fields (`missing_information`, `inaccuracies`) likely carry more signal than numeric scores, but PostMortem analysis (workflow steps 5-7) prioritizes numeric aggregation via `grep_search` extraction.
- **Where:** design.md §Shared Evaluation Schema, design.md §PostMortem Workflow steps 5-7, feature.md §FR-1.2
- **Likelihood:** High
- **Impact:** Medium — system operates correctly but may produce uniformly positive reports that don't drive improvement
- **Assumption at risk:** That integer 1-10 scores produce sufficient variance and accuracy to meaningfully differentiate producer agent quality.

### [Severity: Medium] Finding 2: No Feedback Loop — "Self-Improvement" Remains a Misnomer

- **What:** The revised design still ends at report generation. Feature.md US-3 (line 382) says "Developer uses `improvement_recommendations` to decide which agent definitions to update" — but that field no longer exists. The developer now receives only raw metrics with no actionable suggestions. The gap between the feature name and its deliverables is wider in this revision: v0 at least provided recommendations as a bridge to action, while the revision (correctly) removes them. The initial request says "agents can improve over time" but no mechanism feeds insights back into the pipeline.
- **Where:** feature.md §US-3 (line 382 references removed field), design.md §Post-Mortem Report Schema (field removed), initial-request.md §PART 3
- **Likelihood:** N/A — by design
- **Impact:** Low — correct design choice (auto-applying changes would be dangerous), but the value story needs recalibration
- **Assumption at risk:** That humans will manually analyze quantitative-only reports and derive actionable improvements from raw scores and frequency counts.

### [Severity: Medium] Finding 3: Collision Avoidance Not Present in Evaluating Agent Workflow Template

- **What:** §Append-Only Semantics (design.md line 756) specifies that evaluation files use sequence suffixes on collision (`<agent>-2.md`, `<agent>-3.md`). The PostMortem agent definition (design.md line 596) includes collision instructions: "If a file with the same name exists, append a sequence suffix." However, the evaluating agent workflow template in §Workflow Step Addition (~line 330) simply says: "Write all blocks to: `docs/feature/<feature-slug>/artifact-evaluations/<agent-name>.md`" — no collision detection, no sequence suffix instruction. Agents follow their workflow template literally. Without explicit collision instructions, the 14 evaluating agents will attempt to write to the bare filename, potentially overwriting existing files during NEEDS_REVISION re-executions (designer runs twice per CT loop) or same-feature re-runs.
- **Where:** design.md §Workflow Step Addition (~line 330) vs. design.md §Append-Only Semantics (line 756)
- **Likelihood:** Medium — NEEDS_REVISION re-executions are expected and designed into the pipeline (CT and V loops)
- **Impact:** Medium — earlier evaluation data silently overwritten, violating append-only constraint (FR-4.3); PostMortem undercount
- **Assumption at risk:** That agents will independently implement collision detection without explicit workflow instructions.

### [Severity: Medium] Finding 4: Spec-Design Divergence on Tool Restriction Creates Testability Ambiguity

- **What:** Feature.md FR-6.1 specifies tools as "only: `agent`, `agent/runSubagent`, and `memory`." AC-5 explicitly says no instructions must reference `read_file`, `grep_search`, `semantic_search`, or any other tools. The revised design retains all these read tools based on the Critical finding (CT-1). The design acknowledges the deviation and provides sound rationale (§FR-6 deviation), but the spec hasn't been updated. AC-5 as written would FAIL the revised design. The design's TS-5 revision aligns with the design's approach, but it silently revises the acceptance criterion without updating the source spec. A verification agent evaluating AC-5 against feature.md will flag the retained read tools as non-compliant.
- **Where:** feature.md §FR-6.1 (~line 153), feature.md §AC-5 (~line 297), design.md §Write-Tool Restriction → FR-6 deviation (~line 399)
- **Likelihood:** High — will surface during V cluster verification
- **Impact:** Medium — verification agents may flag as non-compliant; forces human adjudication of whether the deviation is acceptable
- **Assumption at risk:** That verification agents will accept the design's CT-driven rationale for deviating from the spec's literal text.

### [Severity: Medium] Finding 5: Orchestrator Context Accumulation Has No Size Bound or Degradation Strategy

- **What:** The orchestrator accumulates all dispatch telemetry in its working context from Step 1 through Step 8. A full pipeline run (19+ dispatches, with Pattern C replan iterations adding 20+ more) produces ~25-40 telemetry observations, each with 10 fields. The Markdown table format in the PostMortem dispatch prompt (~line 462) is compact, but the orchestrator's working context by Step 8 also includes the 486-line agent definition, all step dispatch/return interactions, memory merge operations, and cluster decision logic. The design acknowledges the resilience trade-off (telemetry lost on crash) but specifies no size bound and no degradation strategy for context pressure. There is no fallback such as "if telemetry exceeds N entries, summarize early-pipeline entries" or "delegate periodic telemetry snapshot writes."
- **Where:** design.md §Telemetry Accumulation (~line 435), design.md §Step 8 Addition, design.md §Pattern C Replan Loop Telemetry
- **Likelihood:** Low — modern context windows (100K+) likely accommodate this; telemetry is ~15-25KB in table format
- **Impact:** Medium — if context pressure occurs, it degrades orchestrator reasoning at the critical Step 7-8 boundary (R cluster decision + PostMortem dispatch), which is hard to detect
- **Assumption at risk:** That accumulated telemetry alongside all pipeline state fits within the orchestrator's context window without reasoning degradation at later steps.

### [Severity: Low] Finding 6: Feature Spec Schema and User Story Reference Removed Field

- **What:** Feature.md FR-3.3 (line 121) defines the `post_mortem_report` schema WITH `improvement_recommendations`. US-3 (line 382) describes a developer flow using this field. The revised design correctly removes it (CT-6 resolution). This creates a spec-design divergence: an implementer following FR-3.3 literally would include the field, contradicting the design. Similar to Finding 4, this is a documented deviation that the design handles well, but the spec remains stale.
- **Where:** feature.md §FR-3.3 (line 121), feature.md §US-3 (line 382), design.md §Post-Mortem Report Schema, design.md §CT Revision Summary row 6
- **Likelihood:** Medium — implementers may follow spec over design
- **Impact:** Low — design clearly documents the deviation; risk is implementer confusion, not architectural damage
- **Assumption at risk:** That implementers will follow the design's documented deviation rather than the spec's literal schema.

### [Severity: Low] Finding 7: PostMortem Self-Verification Tension with Memory-First Targeted Reading

- **What:** PostMortem self-verification (workflow step 9) requires "All discovered evaluation files were processed (or logged as skipped)." The Memory-First Reading Strategy says "Read full content only of evaluation files that indicate problems." These create ambiguity: does "processed" mean "had scores extracted via grep_search" or "had full content read and analyzed"? If the agent interprets "processed" strictly, it reads all files fully, defeating the memory-first optimization.
- **Where:** design.md §PostMortem Workflow step 9, design.md §Memory-First Reading Pattern
- **Likelihood:** Low — LLM agents will likely interpret "processed" loosely as "metrics extracted"
- **Impact:** Low — worst case is PostMortem reads all files fully (v0 behavior), wasting context but not causing errors
- **Assumption at risk:** That "processed" will be interpreted as "metrics extracted from" rather than "fully read and analyzed."

---

## Cross-Cutting Observations

- **CT-Maintainability concern:** The spec-design divergences on FR-3.3 (`improvement_recommendations`) and FR-6.1/AC-5 (read tools) create documentation debt. The deviations are documented in the design's CT Revision Summary table but are scattered rather than consolidated. Future spec updates must reconcile these. (Scope: ct-maintainability)

- **CT-Scalability concern:** Context-passing telemetry trades memory.md bloat (19 agents affected) for orchestrator context growth (1 agent affected). The trade is clearly favorable, but the new single-agent risk should have an explicit size bound or degradation strategy for Pattern C heavy-iteration scenarios. (Scope: ct-scalability)

- **CT-Security concern:** The evaluating agent workflow template instructs agents to write YAML but doesn't instruct them to validate their own output before writing. Malformed YAML is handled downstream by PostMortem (skip corrupted), but no upstream validation exists. This is defense-in-depth, not a critical gap. (Scope: ct-security)

---

## Requirement Coverage

| Requirement                       | Coverage Status                   | Notes                                                                                                                                                              |
| --------------------------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| PART 1 — Artifact Evaluation      | Fully Covered                     | 14 agents; shared schema (CT-4 resolved). Gap: collision avoidance missing from workflow template (Finding 3).                                                     |
| PART 2 — Orchestrator Telemetry   | Fully Covered                     | Context-based accumulation eliminates memory.md bloat. Minor spec deviation on run-log filename (`run-log.md` vs `<date>-run-log.md`) for append-only compliance.  |
| PART 3 — PostMortem Agent         | Fully Covered                     | Comprehensive agent spec; quantitative-only; memory-first reading; non-blocking. `improvement_recommendations` removed per CT-6 (justified deviation from FR-3.3). |
| PART 4 — Storage Design           | Fully Covered                     | Three directories; lazy creation; consistent sequence suffix collision avoidance.                                                                                  |
| PART 5 — Safety Requirements      | Fully Covered                     | Additive-only; non-blocking; Track A/B separation (CT-2 resolved).                                                                                                 |
| PART 6 — Deliverables             | Fully Covered                     | Implementation checklist with AC mapping; prompt update specified.                                                                                                 |
| Tool Restriction                  | Covered with Documented Deviation | Read tools retained (Critical finding resolution). FR-6.1/AC-5 deviation well-justified but creates V cluster testability ambiguity (Finding 4).                   |
| PostMortem / R-Knowledge Boundary | Fully Covered                     | Clean separation: quantitative-only, no recommendations. Anti-drift anchor and self-verification enforce.                                                          |
| Append-Only (FR-4.3)              | Mostly Covered                    | Consistent sequence suffixes. Gap: collision detection instructions missing from evaluating agent workflow template (Finding 3).                                   |
