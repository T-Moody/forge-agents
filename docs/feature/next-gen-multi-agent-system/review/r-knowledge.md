# Knowledge Evolution Analysis

> **Feature:** Next-Generation Deterministic Multi-Agent Pipeline
> **Analyzed by:** R-Knowledge Agent
> **Date:** 2026-02-26
> **Scope:** All files in `NewAgents/`, plus design history in `docs/feature/next-gen-multi-agent-system/`

---

## Check Results

### 1. Decision Logging (Append-Only) — PASS

The Knowledge Agent definition ([knowledge-agent.agent.md](../../../NewAgents/.github/agents/knowledge-agent.agent.md)) correctly implements append-only decision logging:

- **Workflow Step 4** (lines 155–190): Explicitly reads existing `decisions.yaml`, notes last entry ID, and appends. States: _"NEVER modify or delete existing entries — this is an append-only log."_
- **Operating Rule 6**: _"Append-only decision log: NEVER modify or delete existing entries in `decisions.yaml`. Read existing content first, note the last entry, and append only."_
- **Self-Verification Checklist**: Includes three dedicated checks — (1) prior entries preserved, (2) sequential IDs, (3) all required fields present (`id`, `title`, `rationale`, `confidence`).
- **Anti-Drift Anchor**: Reinforces _"maintain the append-only decision log"_ as core identity.

No weaknesses found.

### 2. Evidence Bundle — All 6 Components Present — PASS

The evidence bundle assembly (Step 8b) in [knowledge-agent.agent.md](../../../NewAgents/.github/agents/knowledge-agent.agent.md#L199-L326) documents all 6 mandatory components:

| Component                               | Description                                                                  | Verified |
| --------------------------------------- | ---------------------------------------------------------------------------- | -------- |
| **(a) Overall Confidence Rating**       | High/Medium/Low with criteria table mapping pipeline outcomes                | ✅       |
| **(b) Aggregated Verification Summary** | Per-task pass/fail/regression table from verification reports                | ✅       |
| **(c) Adversarial Review Summary**      | Per-model verdicts for both design review (Step 3b) and code review (Step 7) | ✅       |
| **(d) Rollback Command**                | `git revert --no-commit pipeline-baseline-{run_id}..HEAD`                    | ✅       |
| **(e) Blast Radius**                    | Files modified + per-risk-level breakdown + regression count                 | ✅       |
| **(f) Known Issues**                    | Severity-rated issue table or explicit "No known issues"                     | ✅       |

Self-verification checklist (lines 443–449) independently verifies all 6 components are present in the output, plus rollback command format and confidence rating validity.

Schema 10 in [schemas.md](../../../NewAgents/.github/agents/schemas.md#L825-L930) mirrors this with structured fields for `evidence_bundle.overall_confidence`, `verification_summary`, `review_summary`, `rollback_command`, `blast_radius`, and `known_issues`.

### 3. Cross-Session Learning via store_memory — Properly Scoped — PASS

The Knowledge Agent (Workflow Step 3, lines 155–180) defines explicit inclusion and exclusion criteria for `store_memory`:

**What to store:**

- Build/test commands verified to work
- Codebase conventions discovered during implementation
- Patterns that recur across multiple tasks
- Toolchain configurations or environment-specific findings

**What NOT to store:**

- Secrets, API keys, tokens, or credentials
- Personally identifiable information (PII)
- Ephemeral pipeline state (step counts, timing for specific run)
- Information that changes frequently and would become stale

**Scoping mechanisms:**

- Each knowledge update has `stored_via` field: `store_memory` | `decisions.yaml` — routing is explicit
- Self-verification includes: _"No secrets, PII, or ephemeral state stored via `store_memory`"_
- Safety Constraint Filter (Step 1 of filter application): _"Before persisting any knowledge update via `store_memory`, verify it does not encode a weakening of safety."_

### 4. Governed Updates — Interactive Approval + Safety Filter — PASS

**Interactive Approval** ([knowledge-agent.agent.md](../../../NewAgents/.github/agents/knowledge-agent.agent.md) §Governed Updates):

- Interactive mode: _"Instruction file modifications require explicit user approval before being applied. Present proposed changes and wait for confirmation."_
- Autonomous mode: _"Instruction file modifications are logged in `knowledge-output.yaml` as suggestions but NOT automatically applied."_
- 4 rules codified: (1) never auto-apply in autonomous mode, (2) present clearly in interactive mode, (3) log all proposals, (4) narrow scope.

**Safety Constraint Filter** (§Safety Constraint Filter):

- 8 explicit rejection categories: removing error handling/try-catch, weakening input validation, removing security scanning, removing anti-drift anchors, weakening completion contracts, removing read-only enforcement, removing verification tiers, removing evidence gating checks.
- 3-step application: (1) check before `store_memory`, (2) check before `decisions.yaml` append, (3) log rejected proposals with `stored_via: "rejected — safety constraint"`.

### 5. Architectural Decisions from Design Process — CAPTURED BELOW

10 major design decisions (D1–D10) plus v4 revision decisions are documented in the design history. These are captured in `decisions.md` alongside this analysis.

---

## Instruction Suggestions

### [Category: instruction-update] Agent Definition Authoring Convention

- **File:** `.github/instructions/agent-definitions.instructions.md` (new)
- **Change:** Add
- **Rationale:** All 9 agent definitions follow a consistent structure (Role & Purpose → Input Schema → Output Schema → Workflow → Self-Verification → Completion Contract → Operating Rules → Anti-Drift Anchor). This convention should be codified for future agent creation or modification to ensure structural consistency.
- **Before:** (no instruction exists)
- **After:** See knowledge-suggestions.md §Suggestion 1

### [Category: instruction-update] Schema Reconciliation Convention

- **File:** `.github/instructions/schema-reconciliation.instructions.md` (new)
- **Change:** Add
- **Rationale:** The design process revealed dual schema definitions (`anvil_checks` in §Decision 6 vs. §Data Storage) creating conflicts (output_snippet length, severity enum casing). Multiple implementers independently resolved this by choosing §Decision 6 as canonical. This convention prevents future schema definition drift.
- **Before:** (no instruction exists)
- **After:** See knowledge-suggestions.md §Suggestion 2

## Skill Suggestions

None identified — agent capabilities are well-matched to their responsibilities.

## Pattern Captures

### [Category: pattern-capture] Typed YAML Schema Boundary Pattern

- **Pattern:** Every agent-to-agent communication boundary is governed by a typed YAML schema with defined fields, types, allowed values, and a single-example-per-schema reference document. Machine-readable YAML is primary; human-readable Markdown companions are generated from the same data.
- **Evidence:** [schemas.md](../../../NewAgents/.github/agents/schemas.md) defines 10 schemas. Every agent definition references its input/output schemas by number. The Producer/Consumer Dependency Table (schemas.md lines 40–55) maps all data flows.
- **Applicability:** Any multi-agent pipeline where agents communicate via structured artifacts. Particularly effective when the orchestrator needs deterministic routing based on agent output fields.

### [Category: pattern-capture] SQL-Primary Verification Ledger

- **Pattern:** All verification evidence is recorded as SQL INSERTs into a single `anvil_checks` table. Evidence gates use COUNT queries with WHERE filters on `run_id`, `round`, `verdict`, `phase`, and `check_name`. The SQL ledger is the source of truth; YAML reports are summaries for routing.
- **Evidence:** [schemas.md §SQLite Schemas](../../../NewAgents/.github/agents/schemas.md#L942), [orchestrator.agent.md §Evidence Gate SQL Query Templates](../../../NewAgents/.github/agents/orchestrator.agent.md#L569-L640), [verifier.agent.md §SQL Evidence Recording](../../../NewAgents/.github/agents/verifier.agent.md#L109-L170).
- **Applicability:** Any pipeline requiring tamper-resistant, queryable verification evidence. The `check_name` naming convention pattern is critical — evidence gate LIKE patterns depend on consistent naming.

### [Category: pattern-capture] Non-Blocking Terminal Agent

- **Pattern:** The Knowledge Agent (Step 8) is explicitly non-blocking — its ERROR does not halt the pipeline. The orchestrator proceeds to Step 9 regardless. This pattern protects the primary deliverable (implemented code) from supplementary analysis failures.
- **Evidence:** [knowledge-agent.agent.md §Non-Blocking Behavior](../../../NewAgents/.github/agents/knowledge-agent.agent.md#L346-L355), [orchestrator.agent.md §Step 8](../../../NewAgents/.github/agents/orchestrator.agent.md#L483-L510).
- **Applicability:** Any pipeline step whose output is valuable but not critical to the primary deliverable. Good for telemetry aggregation, documentation generation, or knowledge capture.

### [Category: pattern-capture] Governed Update Protocol

- **Pattern:** Two-mode governance for configuration changes: Interactive mode requires explicit user approval before applying; Autonomous mode logs proposals without applying. All proposals are recorded regardless of approval status.
- **Evidence:** [knowledge-agent.agent.md §Governed Updates](../../../NewAgents/.github/agents/knowledge-agent.agent.md#L356-L397).
- **Applicability:** Any agent that may modify shared configuration (instruction files, skill definitions, environment configs). Prevents autonomous agents from silently changing system behavior.

### [Category: pattern-capture] Deviation Record Pattern

- **Pattern:** When design intentionally diverges from specification requirements, formal Deviation Records (DR-1, DR-2, ...) document the spec requirement, the deviation, and the rationale. These are traceable artifacts linking design choices to spec constraints.
- **Evidence:** [design.md §Deviation Records](../../../docs/feature/next-gen-multi-agent-system/design.md) — DR-1 (orchestrator `run_in_terminal` vs FR-1.6), DR-2 (always-3 reviewers vs FR-5.3/CR-4). Both are reflected in [orchestrator.agent.md §DR-1](../../../NewAgents/.github/agents/orchestrator.agent.md#L60-L73).
- **Applicability:** Any design process where requirements are formal (numbered FRs/CRs) and deviations must be tracked for auditability.

### [Category: pattern-capture] Baseline Cross-Check via Git Tag

- **Pattern:** Before implementation, create a git tag (`pipeline-baseline-{run_id}`). The Verifier independently cross-checks Implementer baseline claims using `git show pipeline-baseline-{run_id}:<filepath>`. This solves the physical impossibility of re-running diagnostics on pre-implementation state.
- **Evidence:** [orchestrator.agent.md §Steps 5–6](../../../NewAgents/.github/agents/orchestrator.agent.md#L380-L382) (`git tag -f pipeline-baseline-{run_id}`), [verifier.agent.md §Baseline Cross-Check](../../../NewAgents/.github/agents/verifier.agent.md#L200-L220).
- **Applicability:** Any pipeline where post-implementation verification needs to compare against pre-implementation state, and re-running the original diagnostics is not possible.

## Decision Log Entries

### D-K1: Synthesized Architecture (A+D+B)

- **Context:** 5 architectural directions evaluated for next-gen multi-agent system.
- **Decision:** Start from Direction A's lean pipeline, enhanced with typed schemas from D and selective specialization from B. 9 agents, zero-merge typed YAML, SQL-primary verification.
- **Rationale:** A provides minimal orchestration overhead; D provides structural rigor at every boundary; B retains Verifier specialization and Knowledge Agent capability. Rejected C (mega-agents, context overflow), E (event-driven, over-engineered), pure-B (doesn't fix root problems), pure-A (unified verifier too complex).
- **Scope:** Project-wide.
- **Affected Components:** All agent definitions, schemas.md, orchestrator.agent.md.

### D-K2: SQL-Primary Verification with WAL

- **Context:** Need tamper-resistant, concurrent-safe verification evidence storage.
- **Decision:** SQLite `anvil_checks` table as primary verification ledger. WAL mode + `busy_timeout=5000` mandatory. YAML reports are summaries, SQL is source of truth.
- **Rationale:** Anvil's proven pattern adapted for multi-agent parallel writes. WAL prevents SQLITE_BUSY with ≤4 concurrent agents. Evidence gates use COUNT queries with WHERE filters on run_id/round/verdict.
- **Scope:** Project-wide.
- **Affected Components:** orchestrator.agent.md (Step 0 init, evidence gates), verifier.agent.md (SQL INSERT), adversarial-reviewer.agent.md (SQL INSERT), schemas.md §SQLite Schemas.

### D-K3: Non-Blocking Knowledge Capture

- **Context:** Evidence bundle assembly is valuable but should not block the primary deliverable.
- **Decision:** Knowledge Agent is non-blocking. Its ERROR doesn't halt the pipeline. Evidence bundle silently disappears on failure.
- **Rationale:** Blocking knowledge capture creates a fragile dependency on a supplementary analysis step. The tradeoff (silent disappearance of evidence bundle) is acceptable because the primary deliverable (implemented code) is independently verified.
- **Scope:** Feature-specific (Knowledge Agent + orchestrator routing).
- **Affected Components:** knowledge-agent.agent.md, orchestrator.agent.md §Step 8.

### D-K4: Governed Update Protocol for Knowledge Agent

- **Context:** Knowledge Agent identifies instruction improvements but should not silently modify shared configuration.
- **Decision:** Interactive mode requires explicit user approval; autonomous mode logs but does not apply. Safety constraint filter rejects any proposal that weakens safety.
- **Rationale:** Prevents autonomous agents from silently changing behavior. User retains control. Safety filter prevents gradual erosion of verification and error handling.
- **Scope:** Project-wide (instruction modification governance).
- **Affected Components:** knowledge-agent.agent.md §Governed Updates, §Safety Constraint Filter.

### D-K5: Deviation Records for Spec Divergences

- **Context:** Design intentionally diverges from spec in two places: orchestrator tool access (DR-1) and always-3 reviewers (DR-2).
- **Decision:** Formal Deviation Records document each divergence with spec requirement ID, deviation description, and rationale.
- **Rationale:** Maintains traceability between spec and design. Reviewers can evaluate whether deviations are justified rather than discovering undocumented changes.
- **Scope:** Feature-specific.
- **Affected Components:** design.md §Deviation Records, orchestrator.agent.md §DR-1.

## Summary

- **Total suggestions:** 8
- **Instruction updates:** 2
- **Skill updates:** 0
- **Pattern captures:** 6
- **Workflow improvements:** 0
- **Rejected (safety filter):** 0
