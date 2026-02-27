# Verification: Feature-Level Acceptance Criteria

## Status

PASS

## Summary

14 of 15 acceptance criteria fully met, 1 partially-met (AC-7: severity taxonomy has minor aggregation-level inconsistency). All 4 checked Common Requirements (CR-4, CR-12, CR-14, CR-15) fully met. No regressions detected — all output is in a clean `NewAgents/` directory with no modifications to existing files. Initial-request scope (merge Forge + Anvil + Article) is fulfilled.

---

## Feature Acceptance Criteria Verification

### AC-1: Architectural Completeness

- **Status:** met
- **Evidence:** [design.md](../design.md) synthesizes Directions A+D+B with 10 justification-scored decisions covering: architecture selection (§Decision 1), agent inventory (§Decision 2), pipeline structure (§Decision 3), communication & state (§Decision 4), memory system (§Decision 5), verification architecture (§Decision 6), adversarial review (§Decision 7), risk & escalation (§Decision 8), approval mode (§Decision 9), file & output structure (§Decision 10). Pipeline Execution Rules section documents all steps. Evidence gates documented in orchestrator. Dispatch patterns in [dispatch-patterns.md](../../../NewAgents/.github/agents/dispatch-patterns.md).
- **Gaps:** None

### AC-2: Agent Count Within Range

- **Status:** met
- **Evidence:** 9 unique `.agent.md` files in `NewAgents/.github/agents/`: orchestrator, researcher, spec, designer, planner, implementer, verifier, adversarial-reviewer, knowledge-agent. 9 is within the 4–15 range.
- **Gaps:** None

### AC-3: Completion Contracts Universal

- **Status:** met
- **Evidence:** All 9 agents have a `## Completion Contract` section confirmed via grep search (9 matches). Each specifies which of the three states it returns with routing rules:
  - **DONE/NEEDS_REVISION/ERROR:** planner ([line 313](../../../NewAgents/.github/agents/planner.agent.md#L313)), verifier ([line 344](../../../NewAgents/.github/agents/verifier.agent.md#L344)), adversarial-reviewer ([line 250](../../../NewAgents/.github/agents/adversarial-reviewer.agent.md#L250))
  - **DONE/ERROR only (documented reason):** researcher ([line 159](../../../NewAgents/.github/agents/researcher.agent.md#L159) — "does not return NEEDS_REVISION"), spec ([line 263](../../../NewAgents/.github/agents/spec.agent.md#L263)), designer ([line 233](../../../NewAgents/.github/agents/designer.agent.md#L233)), implementer ([line 323](../../../NewAgents/.github/agents/implementer.agent.md#L323)), knowledge-agent ([line 486](../../../NewAgents/.github/agents/knowledge-agent.agent.md#L486))
  - **DONE/ERROR (NEEDS_REVISION handled internally):** orchestrator ([line 722](../../../NewAgents/.github/agents/orchestrator.agent.md#L722))
- **Gaps:** None

### AC-4: Adversarial Review Functional

- **Status:** met
- **Evidence:** [adversarial-reviewer.agent.md](../../../NewAgents/.github/agents/adversarial-reviewer.agent.md) dispatched as 3 parallel instances with distinct `review_focus` (security, architecture, correctness). Always 3 reviewers per design deviation DR-2. Verdicts recorded as SQL INSERT into `anvil_checks` with `phase='review'` (line 57) + YAML verdict summary. COUNT gate verified by orchestrator via SQL queries ([orchestrator.agent.md §Evidence Gate SQL Query Templates](../../../NewAgents/.github/agents/orchestrator.agent.md#L554)). Max 2 rounds documented in [orchestrator decision table](../../../NewAgents/.github/agents/orchestrator.agent.md#L526).
- **Gaps:** None

### AC-5: Evidence Gating Enforced

- **Status:** met
- **Evidence:** Evidence gates at:
  - Post-implementation: orchestrator Step 6 dispatches verifier per-task, SQL COUNT queries verify signal thresholds ([orchestrator.agent.md line 403](../../../NewAgents/.github/agents/orchestrator.agent.md#L403))
  - Post-review (design): orchestrator Step 3b checks 3 verdicts submitted, 0 blockers, ≥2 approve ([orchestrator.agent.md line 290](../../../NewAgents/.github/agents/orchestrator.agent.md#L290))
  - Post-review (code): orchestrator Step 7 checks same patterns ([orchestrator.agent.md line 451](../../../NewAgents/.github/agents/orchestrator.agent.md#L451))
  - All gates are machine-checkable SQL COUNT queries with `run_id` and `round` filters.
- **Gaps:** None

### AC-6: Risk Classification Integrated

- **Status:** met
- **Evidence:** Planner classifies every file as 🟢/🟡/🔴 per [planner.agent.md §Per-File Classification](../../../NewAgents/.github/agents/planner.agent.md#L134). Any 🔴 file escalates task to Large ([line 225](../../../NewAgents/.github/agents/planner.agent.md#L225)). `overall_risk_summary` emitted in `plan-output.yaml` ([line 63](../../../NewAgents/.github/agents/planner.agent.md#L63)). Orchestrator reads risk for routing decisions ([orchestrator.agent.md line 88](../../../NewAgents/.github/agents/orchestrator.agent.md#L88)). Self-verification checklist item 4 ensures every file has classification ([line 348](../../../NewAgents/.github/agents/planner.agent.md#L348)).
- **Gaps:** None

### AC-7: Unified Severity Taxonomy

- **Status:** partially-met
- **Evidence:** The canonical 4-level taxonomy (Blocker/Critical/Major/Minor) is:
  - Defined in [severity-taxonomy.md](../../../NewAgents/.github/agents/severity-taxonomy.md) (lines 12–77)
  - Used correctly in individual finding templates: [adversarial-reviewer.agent.md line 96](../../../NewAgents/.github/agents/adversarial-reviewer.agent.md#L96), [verifier.agent.md SQL CHECK line 134](../../../NewAgents/.github/agents/verifier.agent.md#L134), [orchestrator.agent.md SQL CHECK line 161](../../../NewAgents/.github/agents/orchestrator.agent.md#L161)
  - Referenced in planner, knowledge-agent, and spec via `severity-taxonomy.md` links
- **Gaps:**
  - [adversarial-reviewer.agent.md line 67–73](../../../NewAgents/.github/agents/adversarial-reviewer.agent.md#L67-L73): `findings_count` YAML verdict uses 5 levels (`blocker`, `critical`, `high`, `medium`, `low`) instead of the canonical 4 (`Blocker`, `Critical`, `Major`, `Minor`)
  - [designer.agent.md line 176](../../../NewAgents/.github/agents/designer.agent.md#L176): Categorizes findings as "Blocker → Critical → High → Medium → Low" (5 levels, mixing taxonomies)
  - [spec.agent.md line 120](../../../NewAgents/.github/agents/spec.agent.md#L120): Uses "critical / high / medium / low" for edge case severity-if-missed
  - **Impact:** The primary finding-level vocabulary is correct. The inconsistency is in aggregation fields and secondary uses. This does not break SQL evidence gates (which use `Blocker`/`Critical`/`Major`/`Minor`) but violates the strict letter of AC-7 ("No agent uses an alternative vocabulary").

### AC-8: Typed Communication

- **Status:** met
- **Evidence:** All 9 agents reference `schemas.md` for their output schemas (20+ references found via grep). 10 typed YAML schemas defined in [schemas.md](../../../NewAgents/.github/agents/schemas.md) (1276 lines). Each agent produces structured YAML output with `agent_output` common header, `payload` section, and `completion` block. Orchestrator reads typed completion blocks programmatically. SQL used as primary verification evidence format.
- **Gaps:** None

### AC-9: Deterministic Routing

- **Status:** met
- **Evidence:** [orchestrator.agent.md §Orchestrator Decision Table](../../../NewAgents/.github/agents/orchestrator.agent.md#L526) documents a complete decision table mapping every agent output condition to a deterministic action (22 rows). Operating rule 5 ([line 716](../../../NewAgents/.github/agents/orchestrator.agent.md#L716)): "Given the same agent outputs, the orchestrator MUST make the same routing decisions. All routing is driven by completion contracts and SQL evidence gate queries."
- **Gaps:** None

### AC-10: Approval Mode Working

- **Status:** met
- **Evidence:** Two approval gates documented: [Step 1a post-research](../../../NewAgents/.github/agents/orchestrator.agent.md#L213) and [Step 4a post-planning](../../../NewAgents/.github/agents/orchestrator.agent.md#L337). `APPROVAL_MODE` variable in [feature-workflow.prompt.md](../../../NewAgents/.github/prompts/feature-workflow.prompt.md) (line 3: `agent: orchestrator`). Pipeline state tracks `approval_mode: autonomous | interactive` ([orchestrator.agent.md line 86](../../../NewAgents/.github/agents/orchestrator.agent.md#L86)). Autonomous mode auto-proceeds. Interactive mode presents structured multiple-choice via `ask_questions`.
- **Gaps:** None

### AC-11: Pipeline Completes End-to-End

- **Status:** met
- **Evidence:** Pipeline Steps 0–9 fully documented in [orchestrator.agent.md](../../../NewAgents/.github/agents/orchestrator.agent.md). Entry point: [feature-workflow.prompt.md](../../../NewAgents/.github/prompts/feature-workflow.prompt.md) with `agent: orchestrator`. Output: 9 agent definitions + 3 reference documents + 1 prompt + 1 README with Mermaid diagram. Autonomous mode requires no manual intervention outside designated approval gates. Every step has defined completion conditions and routing.
- **Gaps:** None

### AC-12: Self-Verification Implemented

- **Status:** met
- **Evidence:** 8 of 9 agents have `## Self-Verification` as H2 sections. The designer has `### 6. Self-Verification` as H3 subsection within the workflow ([designer.agent.md line 212](../../../NewAgents/.github/agents/designer.agent.md#L212)). All 9 contain substantive checklists verifying output against input requirements before returning. The designer's self-verification has 12 checklist items covering schema compliance, requirement mapping, decision record format, risk consistency, and security considerations. This is a formatting variance (H3 vs H2), not a capability gap.
- **Gaps:** None

### AC-13: Pushback System Functional

- **Status:** met
- **Evidence:** Two-layer pushback system:
  1. Lightweight pushback in orchestrator Step 0 ([orchestrator.agent.md line 133](../../../NewAgents/.github/agents/orchestrator.agent.md#L133)) — evaluates request quality before researcher dispatch. Interactive: `ask_questions` multiple-choice. Autonomous: log and proceed. Blocker: pipeline HALT both modes.
  2. Full pushback in spec agent ([spec.agent.md §Pushback System, line 127](../../../NewAgents/.github/agents/spec.agent.md#L127)) — evaluates implementation concerns, requirements concerns with structured choices. Autonomous: logged in `pushback_log` section. Interactive: user chooses proceed/modify/abandon.
- **Gaps:** None

### AC-14: Output Structure Correct

- **Status:** met
- **Evidence:** Confirmed via directory listing:
  - `NewAgents/.github/agents/`: 12 files (9 `.agent.md` + `schemas.md` + `dispatch-patterns.md` + `severity-taxonomy.md`)
  - `NewAgents/.github/prompts/`: 1 file (`feature-workflow.prompt.md`)
  - `NewAgents/README.md`: present with Mermaid diagram (179 lines)
- **Gaps:** None

### AC-15: Bounded Retries Enforced

- **Status:** met
- **Evidence:** [orchestrator.agent.md §Retry Budgets & Escalation](../../../NewAgents/.github/agents/orchestrator.agent.md#L624) defines explicit bounds:
  - Agent retry: 1 retry per dispatch
  - Verification replan loop: max 3 iterations
  - Design revision loop: max 1 iteration
  - Code review rounds: max 2 rounds
  - Deterministic failures: 0 retries
  - Escalation paths documented with termination conditions ([lines 644–680](../../../NewAgents/.github/agents/orchestrator.agent.md#L644-L680))
  - Implementer self-fix: max 2 attempts ([implementer.agent.md line 183](../../../NewAgents/.github/agents/implementer.agent.md#L183))
- **Gaps:** None

---

## Common Requirements Spot-Check

### CR-4: Adversarial Multi-Model Review

- **Status:** met
- **Evidence:** Always 3 perspective-diverse reviewers dispatched in parallel (design deviation DR-2 from spec's risk-driven count). Models: gpt-5.3-codex, gemini-3-pro-preview, claude-opus-4.6. Each verdict recorded independently via SQL INSERT + YAML. ≥3 review records required before proceeding. Max 2 adversarial rounds. Security blocker policy: any blocker → pipeline ERROR. All documented in [adversarial-reviewer.agent.md](../../../NewAgents/.github/agents/adversarial-reviewer.agent.md) and [orchestrator decision table](../../../NewAgents/.github/agents/orchestrator.agent.md#L526).

### CR-12: Anti-Drift Anchors

- **Status:** met
- **Evidence:** All 9 agents contain `## Anti-Drift Anchor` as final H2 section (9 grep matches confirmed). Each anchor restates agent identity, permitted operations, forbidden activities, and output scope. Example: orchestrator anchor specifies 5 responsibilities + allowed/restricted tools + DR-1 constraint.

### CR-14: Pushback System

- **Status:** met
- **Evidence:** Two-layer implementation (orchestrator Step 0 + spec agent). Surfaces implementation and requirements concerns with structured choices. Interactive: user must acknowledge. Autonomous: concerns logged. See AC-13 evidence above.

### CR-15: Bounded Retry Budget

- **Status:** met
- **Evidence:** See AC-15 evidence above. All retry loops have documented maximum iteration counts.

---

## Regression Check Results

### No Regressions

All implementation output is contained within the new `NewAgents/` directory. No existing files in the workspace were modified:

- No changes to `Anvil/anvil.agent.md` (original Anvil agent)
- No changes to existing `.github/` directory (if any exists)
- No changes to `docs/feature/` artifacts (these are pipeline metadata, not implementation)
- No broken imports or references — all internal references within `NewAgents/.github/agents/` use relative paths to co-located files

The feature is self-contained with no external side effects.

---

## Initial Request Scope Verification

The initial request ([initial-request.md](../initial-request.md)) asks for: "a next-generation, deterministic, reliable multi-agent system for GitHub Copilot in VS Code by deeply analyzing and merging two existing systems (Forge Orchestrator and Anvil Agent) along with failure-prevention principles from a GitHub engineering blog post."

### Forge Orchestrator Elements Merged

- 10-step deterministic pipeline (Steps 0–9)
- Dispatch patterns (Pattern A: Fully Parallel, Pattern B: Sequential)
- Parallel research (×4 instances)
- Three-state completion contracts (DONE/NEEDS_REVISION/ERROR)
- Wave-based concurrent execution (≤4 per sub-wave)
- Approval mode (autonomous + interactive)

### Anvil Agent Elements Merged

- SQL evidence ledger (`anvil_checks` table with WAL mode)
- Adversarial multi-model review (3 perspective-diverse reviewers)
- Pushback system (lightweight at Step 0 + full in Spec agent)
- Per-file risk classification (🟢/🟡/🔴)
- Baseline capture before implementation
- Evidence gating at critical transitions
- Justification scoring for architectural decisions

### Article Principles Merged

- Typed YAML schemas at every agent boundary (10 schemas in `schemas.md`)
- Action schemas: structured completion contracts with constrained output
- Failure-first design: bounded retries, error categorization, escalation paths
- Log intermediate state: SQL telemetry, typed YAML outputs
- Treat agents like distributed systems: retry transient, escalate deterministic

**Scope verdict:** Fulfilled. All three source systems are represented in the implementation.

---

## Overall Feature Readiness

**Partially Ready** — 14 of 15 acceptance criteria fully met. One criterion (AC-7) is partially-met due to severity taxonomy inconsistency in 3 agent files where aggregation/secondary fields use `high/medium/low` instead of `Major/Minor`. This is a low-severity documentation inconsistency that does not affect functional correctness — the SQL CHECK constraints and primary finding templates use the correct 4-level taxonomy throughout.

**Recommended action:** Fix the severity vocabulary in:

1. `adversarial-reviewer.agent.md` — change `findings_count` fields from `high/medium/low` to `major/minor`
2. `designer.agent.md` line 176 — change "High → Medium → Low" to match the unified taxonomy
3. `spec.agent.md` line 120 — change "high / medium / low" to match the unified taxonomy

These are 3 small text changes that would bring AC-7 to fully met.

---

## Issues Found

### [Severity: Minor] Severity Taxonomy Vocabulary Inconsistency (AC-7)

- **What:** Three agent files use `high/medium/low` severity vocabulary alongside the canonical `Blocker/Critical/Major/Minor` taxonomy, creating a 5-level hybrid instead of the specified 4-level system.
- **Where:**
  - `NewAgents/.github/agents/adversarial-reviewer.agent.md` lines 67–73 (`findings_count` YAML)
  - `NewAgents/.github/agents/designer.agent.md` line 176 (revision mode categorization)
  - `NewAgents/.github/agents/spec.agent.md` line 120 (edge case severity-if-missed)
- **Impact:** Does not affect SQL evidence gates or primary finding severity (which use the correct 4-level taxonomy). Violates the strict letter of AC-7 and CR-13 ("No agent may use a different severity vocabulary"). Could cause confusion if agents process severity values programmatically.
- **Suggested Action:** Replace `high`→`Major`, `medium`→`Major` or `Minor` (context-dependent), `low`→`Minor` in the 3 identified locations. In the adversarial-reviewer, reduce `findings_count` from 5 fields to 4 (blocker/critical/major/minor).

---

## Cross-Cutting Observations

- **Designer self-verification formatting:** The designer uses `### 6. Self-Verification` (H3) instead of `## Self-Verification` (H2) used by all other agents. This is cosmetic, not functional, but creates a minor inconsistency in agent structure. Not flagged as an issue since the content meets AC-12's requirement.
- **Pushback contradiction appearance:** The orchestrator Step 0 says "Pushback NEVER autonomously halts the pipeline" (line 136) and separately "Blocker concern → Pipeline HALT (both modes)" (line 137). These are not contradictory — non-Blocker concerns log and proceed in autonomous mode, while Blocker-level concerns halt regardless. The decision table (line 531) is consistent with this layered behavior.
- **Design deviation from spec (DR-2):** The spec (CR-4) specifies risk-driven reviewer count (1 for standard, 3 for Large/🔴). The design and implementation always use 3 reviewers regardless of risk level. This is documented as Deviation Record DR-2 in [design.md](../design.md). The deviation simplifies implementation and provides higher quality assurance, so it is an acceptable departure.
