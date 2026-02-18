# CT-Maintainability Review: Agent-Isolated Memory System

**Verdict:** The design is structurally sound but underestimates orchestrator complexity growth and silently drops an existing escalation pathway. Several maintainability risks need attention before planning.

**Overall Risk Level:** Medium — No blocking design flaws, but multiple maintainability concerns compound to create meaningful long-term burden if unaddressed.

---

## Fundamental Concerns

### 1. The "+10 Lines Net" Estimate Is Significantly Undercooked

**What:** The design (NFR-1) estimates the orchestrator grows by only ~+10 lines net (+30 for decision logic, −20 for removed aggregator references). This is implausibly optimistic. The actual additions include:

- CT decision flow: 6 lines (design's own estimate)
- V decision table: 10-row table + surrounding prose = ~15 lines
- R decision flow: 9 lines (design's own estimate)
- Memory merge protocol (5 lines per the design, but needs per-cluster invocations across Steps 1.1m, 2m, 3m, 3bm, 4m, 5m, 6.3m, 7m = at least 8 inline references)
- `memory/` directory creation at Step 0: ~2 lines
- Updated NEEDS_REVISION routing table: modified but not shorter
- Updated Orchestrator Expectations table: modified rows ≠ removed rows
- Eight new merge sub-steps (1.1m through 7m) in the pipeline flow

Conservative estimate: +50–80 lines net, pushing the orchestrator from 399 to ~450–480 lines. The design acknowledges a 450-line threshold for triggering extraction to a reference file but doesn't proactively plan for it.

**Where:** [design.md](design.md), NFR-1 section; orchestrator Workflow Steps
**Likelihood:** High
**Impact:** Medium — An oversized orchestrator prompt risks degraded LLM reliability and makes the file harder to maintain
**Assumption at risk:** That decision logic + merge protocol fits in ~+10 lines net

### 2. "Unresolved Tensions" Escalation Path Silently Dropped

**What:** The current orchestrator (Step 3b.3, NEEDS_REVISION routing table) has an explicit escalation path: when CT revision loop is exhausted and NEEDS_REVISION persists, the orchestrator "forwards Unresolved Tensions as planning constraints" to the planner. This capability comes from the CT-Aggregator, which detects contradictions between sub-agents and structures them as Unresolved Tensions.

The design removes aggregators and acknowledges "Loss of Unresolved Tensions Detection" (Tradeoff 2, Assumption A-5) but does **not** update or remove the orchestrator's existing references to forwarding Unresolved Tensions. The current [orchestrator.agent.md](../../NewAgentsAndPrompts/orchestrator.agent.md) at lines 208 and 321 explicitly says "forward Unresolved Tensions as planning constraints" — this phrasing has no source once aggregators are gone, yet the design's orchestrator changes don't address it.

**Where:** Orchestrator Step 3b.3, NEEDS_REVISION Routing Table (line 321 of current orchestrator); design.md Tradeoff 2
**Likelihood:** High — the reference will remain unless explicitly removed
**Impact:** Medium — The orchestrator will instruct itself to forward something that no longer exists, causing confusion or silent no-ops during CT escalation
**Assumption at risk:** That removing aggregators cleanly removes all downstream references to aggregator-produced artifacts (Unresolved Tensions is a conceptual artifact, not a file, so it's easy to miss)

---

## Risks by Category

### Maintainability Concerns

#### M-1: 21 Files With Identical "Write Isolated Memory" Step Creates Duplication Debt

**What:** Every active agent file gets an identical "Write Isolated Memory" workflow step (same template, same field structure). If the memory template ever changes (e.g., adding a `Duration` field, changing the severity taxonomy, or adjusting the ≤5 bullet limit), all 21 files must be updated in lockstep. This is the same class of problem that Operating Rules 1–4 already exhibit but is acknowledged as necessary for convention consistency. The design does not consider extracting this step as a shared pattern reference (the way dispatch patterns are referenced from a shared file).

**Where:** Design Tier 4 (14 sub-agents), Tier 3 (5 files), Tier 2 (2 files) — all receive the same template
**Likelihood:** Medium — memory format is new and likely to evolve
**Impact:** Medium — Convention drift across 21 files during future changes
**Assumption at risk:** That the memory template is stable enough to duplicate verbatim across 21 files without a shared reference

#### M-2: Planner Replan Mode Gains Significant Unrequested Complexity

**What:** The planner's replan mode currently reads a single `verifier.md` file that contains a pre-compiled Actionable Items section with task ID failure mapping. The v-aggregator currently performs sophisticated cross-referencing (Steps 4–5 of [v-aggregator.agent.md](../../NewAgentsAndPrompts/v-aggregator.agent.md)): extract failing task IDs from `v-tasks.md`, cross-reference with test failures from `v-tests.md`, cross-reference with feature-level gaps from `v-feature.md`, and produce a prioritized Actionable Items list.

Post-change, the planner must read 3 individual V artifact files and self-assemble this cross-referencing. The design (Tradeoff 3) says "the planner may need enhanced workflow instructions for self-assembly" but doesn't include those enhanced instructions in the implementation checklist. The planner's Tier 3 changes list only replacing `verifier.md` with individual V artifacts — no new cross-referencing workflow steps.

**Where:** Design Tier 3 (planner.agent.md changes), Tradeoff 3, current [planner.agent.md](../../NewAgentsAndPrompts/planner.agent.md) line 59
**Likelihood:** High — the cross-referencing gap is structural
**Impact:** High — Untargeted replanning degrades pipeline efficiency; was flagged as High risk in feature.md (EC-11)
**Assumption at risk:** A-6 — that the planner can simply read individual V artifacts and self-assemble failure mapping without enhanced instructions

#### M-3: Sequential Agent Memory Indirection Adds Complexity Without Safety Benefit

**What:** Sequential agents (spec, designer, planner) currently write directly to `memory.md`, which is safe because they run with no concurrent writers. The design forces them through an indirection: write to isolated memory file → orchestrator merges. This adds: (a) an extra file per sequential step, (b) a merge sub-step after every sequential agent (Steps 2m, 3m, 4m), (c) a delay before memory is available for the next agent.

The design justifies this as "uniform model" (Tradeoff 4, Assumption A-2) and "simplifies the memory write safety rule." But the simplification is marginal — the rule changes from "orchestrator + sequential agents write; parallel agents don't" to "orchestrator is sole writer." The added indirection costs more maintenance overhead than the rule simplification saves.

**Where:** Design Tradeoff 4, Sequential Agent Memory Model Migration section
**Likelihood:** High — this is a definite added complexity
**Impact:** Low — The extra files and merge steps are small but they add noise to every sequential step
**Assumption at risk:** A-2 — that uniform model justifies the indirection for sequential agents

#### M-4: Orchestrator Becomes a God Object for Cluster Decision Logic

**What:** The orchestrator currently delegates cluster completion logic to three specialized aggregator agents, each with clear single responsibility. Post-change, the orchestrator absorbs all three decision tables plus memory merge logic. This concentrates four distinct responsibilities in one file:

1. Pipeline coordination (current)
2. CT cluster severity evaluation (new)
3. V cluster decision table evaluation (new)
4. R cluster security-override evaluation (new)

The design mentions a deferred alternative ("Extract Decision Logic to Separate Reference File") but defers it without a triggering condition other than "if the orchestrator exceeds ~450 lines." There's no intermediate option — either everything stays in the orchestrator or it gets extracted to a reference file.

**Where:** Orchestrator-Specific Changes, Alternative Considered: Extract Decision Logic
**Likelihood:** Medium — the orchestrator will be maintainable initially but less so as it grows
**Impact:** Medium — Harder to reason about the orchestrator's behavior; harder to test one decision table without reading the full prompt
**Assumption at risk:** That embedding three decision tables in one file doesn't degrade LLM comprehension of the orchestrator prompt

### Edge Cases Not Covered

#### E-1: Memory Template Severity Field Ambiguity for Non-Cluster Agents

**What:** The memory template specifies `Highest Severity` with cluster-specific comments (CT: Critical/High/Medium/Low; R: Blocker/Major/Minor; V: PASS/FAIL; Others: N/A). But "Others" (spec, designer, planner, implementer, documentation-writer) are told to use "N/A." The template shows this only as an HTML comment, not as a structured constraint. An agent that doesn't read comments carefully could put arbitrary text in this field. If the orchestrator ever starts using severity fields from non-cluster agents for routing, malformed values would trigger worst-case fallback.

**Where:** Design Data Models section, memory template `Highest Severity` field
**Likelihood:** Low — non-cluster agents aren't read for severity routing
**Impact:** Low — Currently harmless but creates inconsistency
**Assumption at risk:** That all agents will correctly interpret a comment-embedded taxonomy as an instruction

#### E-2: Memory File Accumulation Across Pipeline Runs

**What:** The `memory/` directory accumulates `*.mem.md` files for every agent that runs. The design doesn't specify cleanup. If a pipeline re-runs (e.g., replan loops create new implementer memories like `implementer-T01.mem.md` for each iteration), stale memory files from previous iterations persist alongside current ones. The orchestrator reads "all isolated memories from the cluster" — does it read stale ones too?

**Where:** Design Documentation Structure, Memory Lifecycle Actions
**Likelihood:** Medium — replan loops are the common trigger
**Impact:** Medium — Stale memory files could mislead the orchestrator's routing decisions
**Assumption at risk:** That the `memory/` directory is cleanly scoped to a single pipeline iteration

### Backwards Compatibility Risks

#### B-1: `r-knowledge.agent.md` References `verifier.md` Outside Its Cluster

**What:** R-Knowledge (line 25, 78 of current [r-knowledge.agent.md](../../NewAgentsAndPrompts/r-knowledge.agent.md)) reads `verifier.md` via Artifact Index navigation for cross-feature architectural analysis. This is the only agent outside the V cluster that reads `verifier.md`. The design's Tier 4 table notes r-knowledge needs its `verifier.md` reference removed, but the replacement isn't specified — does r-knowledge now read individual `verification/v-*.md` files? The Artifact Index won't contain a `verifier.md` entry anymore, so the navigation instruction becomes a dead reference.

**Where:** Design Tier 4 table (r-knowledge row), current r-knowledge.agent.md lines 25 and 78
**Likelihood:** High — the reference exists and is explicit
**Impact:** Low — R-Knowledge is non-blocking, but it will fail to find `verifier.md` and fall back to a gap
**Assumption at risk:** That removing `verifier.md` from the Artifact Index cleanly handles all consumers

---

## Requirement Coverage Gaps

### Gap 1: No Enhanced Planner Replan Instructions (FR-6.2 Partial)

FR-6.2 states the planner "MUST read individual V sub-agent artifacts directly." The design's Tier 3 planner changes replace the `verifier.md` input with individual V artifacts but don't add workflow instructions for cross-referencing task failures across files — the step that the v-aggregator previously performed. The feature spec (EC-11) flags this as High severity if missed. The design acknowledges the gap in Tradeoff 3 ("planner may need enhanced workflow instructions") but doesn't include those instructions in the implementation checklist.

### Gap 2: Unresolved Tensions Escalation Path Not Addressed (FR-8.1 Partial)

FR-8.1 requires updating the NEEDS_REVISION Routing Table. The design updates the table to replace aggregator names with orchestrator evaluation, but doesn't address the existing "forward Unresolved Tensions as planning constraints" escalation text. This text becomes meaningless without aggregators, but the design's routing table changes don't explicitly remove or replace it.

### Gap 3: Feature-Workflow Approval Gate Timing (FR-14 Partial)

The current `feature-workflow.prompt.md` says approval gates are "after research synthesis and after planning." Post-change, there's no synthesis step. The design (Tier 3, feature-workflow changes) mentions "Update approval gate reference: 'after research completion' (not 'research synthesis')" but this is buried in the implementation checklist. The approval gate timing is user-facing behavior (APPROVAL_MODE variable description) and should be prominently documented.

---

## Key Assumptions

| #   | Assumption                                                              | Risk If Wrong                                                                                                                                       | Validated?                                                                                                 |
| --- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| A-2 | Sequential agents using isolated memory is worth the uniformity benefit | Adds unnecessary files and merge steps for no safety gain                                                                                           | Partially — design argues uniformity but doesn't quantify the benefit vs. cost                             |
| A-4 | Loss of deduplication is acceptable                                     | Duplicate findings in CT/R artifacts inflate downstream reading burden                                                                              | Not validated — no analysis of typical duplicate rates                                                     |
| A-5 | Loss of Unresolved Tensions detection is acceptable                     | Contradictions between sub-agents go undetected; existing escalation path ("forward Unresolved Tensions as planning constraints") becomes dead code | Partially — design acknowledges loss but misses the existing orchestrator reference to forwarding tensions |
| A-6 | Planner can self-assemble task-failure mapping from raw V artifacts     | Replanning becomes untargeted, degrading pipeline efficiency                                                                                        | Not validated — design explicitly says "may need enhanced instructions" but doesn't provide them           |

---

## Strategic Considerations

### The Orchestrator Is Becoming the System

Every responsibility removed from a specialized agent lands in the orchestrator. Today it's aggregator logic; tomorrow it could be validation logic, retry logic, or approval logic. The orchestrator was designed as a coordinator — a thin dispatch layer. Absorbing domain-specific decision tables (CT severity evaluation, V decision matrix, R security override) pushes it toward a domain expert role. This is a one-way door: once the decision logic is embedded, extracting it later requires re-creating specialized agents or a new reference-file pattern.

The design's deferred alternative ("extract to dispatch-patterns.md if >450 lines") should be the default plan, not a contingency. If the implementation lands at 460 lines, the extraction becomes an unplanned task. Proactively defining the extraction boundary now prevents scope growth during implementation.

### Memory Template as a Contract

The `*.mem.md` template is, functionally, an API contract between 21 agents and the orchestrator. But it's defined only in the design document — not in a shared reference file that agents can reference. Each agent will have its own inline copy of the template. If this template needs versioning or evolution, there's no single source of truth. Consider whether `dispatch-patterns.md` (or a new `memory-format.md`) should host the canonical template.

### Diminishing Returns on Uniformity

The uniform branch-merge model (Assumption A-2) applies the same pattern to fundamentally different execution contexts: parallel agents that need isolation (real problem), and sequential agents that already have safe write access (no problem). Applying the same solution to both adds complexity to the common case (sequential) for the sake of pattern consistency. Sometimes the right design decision is to have two paths when two contexts exist.

---

## Recommendations

1. **[High Priority]** Add enhanced replan workflow instructions to the planner's Tier 3 changes. The planner needs explicit steps for cross-referencing `v-tasks.md` failing task IDs with `v-tests.md` test failures and `v-feature.md` criteria gaps. Without this, Tradeoff 3 is an acknowledged gap without a mitigation.

2. **[High Priority]** Explicitly address the "forward Unresolved Tensions as planning constraints" text in the orchestrator's Step 3b.3 and NEEDS_REVISION routing table. Either remove it (since tensions are no longer detected) or replace it with a simpler escalation strategy (e.g., "forward all High/Critical findings from individual CT artifacts as planning constraints").

3. **[Medium Priority]** Revise the +10 lines net estimate in NFR-1 with a realistic count. If the estimate exceeds 450 lines, proactively include the extraction-to-reference-file as an implementation step, not a deferred contingency.

4. **[Medium Priority]** Define a memory file cleanup strategy: when does the orchestrator clear stale `memory/*.mem.md` files? At minimum, the Step 0 setup should clean any existing `memory/` directory from prior runs.

5. **[Low Priority]** Consider whether sequential agents (spec, designer, planner) should be exempt from the isolated memory model, keeping their existing direct-write to `memory.md`. This eliminates unnecessary indirection without harming the model's safety properties.

6. **[Low Priority]** Consider hosting the memory template in a shared reference location (e.g., a "Memory File Format" section in `dispatch-patterns.md`) so that all agents reference a single canonical template rather than 21 inline copies.
