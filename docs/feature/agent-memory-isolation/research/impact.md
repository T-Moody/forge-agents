# Research: Impact

## Focus Area

impact

## Summary

All 24 active files in `NewAgentsAndPrompts/` are impacted: 3 files to remove, 2 files require major rewrites, 5 files need moderate changes, and 14 files need minor additions (isolated memory output). The orchestrator is the most heavily affected file with changes touching 20+ sections.

## Findings

### Per-File Impact Analysis

---

#### Tier 1 — Files to REMOVE (3 files)

##### 1. `ct-aggregator.agent.md` — REMOVE ENTIRELY

- **Rationale:** The orchestrator will read individual CT sub-agent memory files directly instead of invoking a merge-only aggregator.
- **Current role:** Reads `ct-review/ct-*.md`, produces `design_critical_review.md`, writes to `memory.md`.
- **Replacement:** Orchestrator reads CT sub-agent memories to determine if Critical/High issues exist, then routes to designer if needed. No `design_critical_review.md` produced.
- **Cascade:** Orchestrator must absorb the CT aggregator's decision logic (DONE/NEEDS_REVISION/ERROR determination based on severity levels), the design_critical_review.md references must be removed from orchestrator and designer.

##### 2. `v-aggregator.agent.md` — REMOVE ENTIRELY

- **Rationale:** The orchestrator will read individual V sub-agent memory files directly.
- **Current role:** Reads `verification/v-*.md`, produces `verifier.md`, writes to `memory.md`. Critical function: task ID failure mapping for planner replan.
- **Replacement:** Orchestrator reads V sub-agent memories to determine pass/fail. No `verifier.md` produced.
- **Cascade:** Orchestrator must absorb the V aggregator's decision logic (the completion contract decision table at lines 114-131), the `verifier.md` references must be removed from orchestrator and planner. The planner's replan mode currently depends on `verifier.md`'s Actionable Items section for task ID failure mapping — this critical function must be relocated (either to individual V sub-agent artifacts or to the orchestrator's merge logic).
- **Risk:** The task ID failure mapping (merging `v-tasks.md` failing task IDs with `v-tests.md` test failures and `v-feature.md` criteria gaps) is the most complex aggregation logic. Losing this without an equivalent replacement could degrade replan targeting.

##### 3. `r-aggregator.agent.md` — REMOVE ENTIRELY

- **Rationale:** The orchestrator will read individual R sub-agent memory files directly.
- **Current role:** Reads `review/r-*.md`, produces `review.md`, writes to `memory.md`. Enforces R-Security pipeline override. Surfaces knowledge suggestions.
- **Replacement:** Orchestrator reads R sub-agent memories to determine result. No `review.md` produced.
- **Cascade:** Orchestrator must absorb the R aggregator's decision logic (R-Security pipeline blocker override, the completion logic at lines 139-149), the `review.md` references must be removed from orchestrator. The R-Security pipeline override rule (Critical/Blocker findings = pipeline ERROR) must be preserved in the orchestrator.

---

#### Tier 2 — MAJOR Changes (2 files)

##### 4. `orchestrator.agent.md` — MAJOR REWRITE (399 lines, 20+ sections affected)

This is the most heavily impacted file. Virtually every section requires modification.

**Sections to change — Outputs (line 24):**

- Remove: `memory.md (lifecycle management only)` as sole non-coordination output
- Add: Orchestrator now also manages memory merging from isolated agent memory files

**Sections to change — Global Rule 6 (line 34): Memory-First Protocol:**

- Update pruning and invalidation logic to account for isolated memory files
- The merge step is new: after each agent completes, orchestrator reads agent's isolated memory and merges into shared `memory.md`

**Sections to change — Global Rule 12 (lines 40-43): Memory Write Safety:**

- Remove `ct-aggregator, v-aggregator, r-aggregator` from read-write list (line 42)
- Remove `researcher (synthesis mode)` from read-write list (line 42)
- All sub-agents now write to their own isolated memory file (not shared `memory.md`), so the read-only constraint on shared memory remains, but each agent has write access to its own memory file
- The orchestrator is the only entity that writes to shared `memory.md` (via merging)

**Sections to change — Documentation Structure (lines 46-70):**

- Remove: `analysis.md` (line 55)
- Remove: `design_critical_review.md # CT cluster aggregated output` (line 58)
- Remove: `verifier.md # V cluster aggregated output` (line 65)
- Remove: `review.md # R cluster aggregated output` (line 69)
- Add: `memory/` directory for isolated agent memory files (new)

**Sections to change — Cluster Dispatch Patterns (lines 87-104):**

- Pattern A (line 89): Remove "≥2 outputs required for aggregator" — no aggregator step
- Pattern B (line 90): Remove "forward to aggregator" — orchestrator reads memories directly
- Pattern C (lines 93-104): Remove "If aggregator DONE" — orchestrator reads V memories directly; update `verifier.md` references

**Sections to change — Step 1.2 (lines 154-163): Synthesize Research:**

- REMOVE ENTIRELY — no synthesis step, no `analysis.md` production
- Remove approval gate reference to "research synthesis" (line 164)
- Orchestrator reads research agent memories to track progress instead

**Sections to change — Step 2 (lines 168-172): Specification:**

- Change input from `analysis.md` to individual research files (`research/*.md`) (line 170)

**Sections to change — Step 3 (lines 175-180): Design:**

- Change inputs from `analysis.md` to individual research files (line 178)

**Sections to change — Step 3b.2 (lines 197-204): Invoke CT Aggregator:**

- REMOVE ENTIRELY — no ct-aggregator invocation
- Orchestrator reads CT sub-agent memories directly to determine severity

**Sections to change — Step 3b.3 (lines 206-210): Handle CT Result:**

- Update to read CT memories instead of `design_critical_review.md`
- Remove reference to `design_critical_review.md` routing to designer

**Sections to change — Step 4 (lines 212-220): Planning:**

- Change inputs from `analysis.md` to individual research files (line 214)
- Remove `design_critical_review.md` reference (line 214) — planning constraints come from CT memories
- Remove `verifier.md` (replan) dependency — comes from individual V memories

**Sections to change — Step 5.2 (lines 229-241): Execute Each Wave:**

- Update memory handling: orchestrator merges each agent's isolated memory after completion (line 240)
- Remove shared memory write restriction — agents write to isolated memory

**Sections to change — Step 6.3 (lines 270-274): Invoke V Aggregator:**

- REMOVE ENTIRELY — no v-aggregator invocation
- Orchestrator reads V sub-agent memories directly

**Sections to change — Step 6.4 (lines 276-279): Handle V Result:**

- Remove `verifier.md` references — use V memories to determine pass/fail
- Update Pattern C loop to use V memories instead of aggregator output

**Sections to change — Step 7.3 (lines 299-304): Invoke R Aggregator:**

- REMOVE ENTIRELY — no r-aggregator invocation
- Orchestrator reads R sub-agent memories directly
- Orchestrator must preserve R-Security pipeline override logic

**Sections to change — Step 7.4 (lines 306-316): Handle R Result:**

- Remove `review.md` reference — use R memories for decision
- Preserve knowledge-suggestions.md and decisions.md handling

**Sections to change — NEEDS_REVISION Routing Table (lines 319-327):**

- Replace "CT Aggregator" with "Orchestrator (CT cluster)"
- Replace "V Aggregator" with "Orchestrator (V cluster)"
- Replace "R Aggregator" with "Orchestrator (R cluster)"
- Remove `design_critical_review.md` and `verifier.md` references
- Update "All sub-agents route through their aggregator" row

**Sections to change — Orchestrator Expectations Per Agent (lines 329-354):**

- Remove rows: CT Aggregator, V Aggregator, R Aggregator
- Update CT sub-agent, V sub-agent, R sub-agent rows (no longer route through aggregator)

**Sections to change — Memory Lifecycle Actions (lines 358-366):**

- Add: "Merge" action — after each agent completes, merge isolated memory into shared memory
- Update "Validate" — no longer checking aggregator writes

**Sections to change — Parallel Execution Summary (lines 370-380):**

- Line 373: Remove "→ Synthesize → analysis.md"
- Line 375: Remove "→ CT-Aggregator → design_critical_review.md"
- Line 378: Remove "→ V-Aggregator → verifier.md"
- Line 379: Remove "→ R-Aggregator → review.md"

**Sections to change — Anti-Drift Anchor (line 394):**

- Remove "dispatch cluster patterns (Pattern A for CT/R, Pattern B+C for V)" or update
- Add reference to memory merging responsibility

##### 5. `researcher.agent.md` — MAJOR CHANGES (remove synthesis mode)

**Sections to change — Description (line 3):**

- Remove "Supports focused parallel research and synthesis modes" — only focused mode remains

**Sections to change — Agent description paragraph (line 10):**

- Remove "or synthesis mode (merging all partials)"

**Sections to REMOVE — Synthesis Mode Inputs (lines 26-30):**

- Remove entire section listing synthesis mode inputs

**Sections to REMOVE — Synthesis Mode Output (lines 38-40):**

- Remove `analysis.md` output reference

**Sections to change — Operating Rule 6 (line 54):**

- No changes needed (memory-first reading applies to focused mode too)

**Sections to REMOVE — Mode 2: Synthesis (lines 104-137):**

- Remove the entire "Mode 2: Synthesis" section including:
  - Synthesis Workflow (lines 106-114)
  - Analysis.md Contents (lines 116-128)

**Sections to change — Completion Contract (lines 130-137):**

- Remove "Synthesis Mode" completion contract section
- Only keep Focused Mode contract

**Sections to ADD:**

- Add isolated memory file as new output for focused mode
- Add workflow step to write key findings to isolated memory

---

#### Tier 3 — MODERATE Changes (5 files)

##### 6. `spec.agent.md`

**Sections to change — Inputs (lines 19-21):**

- Line 21: Replace `analysis.md` with individual research files: `research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md`

**Sections to change — Workflow step 3 (line 49):**

- Replace "Review `analysis.md` thoroughly" with reading individual research files

**Sections to change — feature.md Contents (line 73):**

- Replace "links to analysis.md" with "links to research files"

**Sections to ADD:**

- Add isolated memory file as new output
- Add workflow step to write key findings to isolated memory

##### 7. `designer.agent.md`

**Sections to change — Inputs (lines 17-21):**

- Line 19: Replace `analysis.md` with individual research files
- Line 21: Replace `design_critical_review.md` — during revision cycle, designer reads individual `ct-review/ct-*.md` files and/or CT sub-agent memories instead

**Sections to change — Workflow step 3 (line 49):**

- Replace "Read `analysis.md` and `feature.md`" with reading individual research files and `feature.md`

**Sections to change — design.md Contents (line 71):**

- Replace "references to analysis.md" with "references to research files"

**Sections to ADD:**

- Add isolated memory file as new output
- Add workflow step to write key findings to isolated memory

##### 8. `planner.agent.md`

**Sections to change — Inputs (lines 17-22):**

- Line 19: Replace `analysis.md` with individual research files
- Line 22: `(Replan mode) verifier.md` — replace with individual V sub-agent artifacts (`verification/v-tasks.md` for failing task IDs, `verification/v-tests.md` for test failures, etc.)

**Sections to change — Mode Detection step 2 (line 59):**

- Replace `verifier.md exists with failures` — check individual V artifacts instead

**Sections to change — Workflow step 4 (line 70):**

- Replace "Read `analysis.md`" with reading individual research files

**Sections to change — Workflow step 5 (line 71):**

- Replace "read `verifier.md`" with reading individual V sub-agent artifacts

**Sections to ADD:**

- Add isolated memory file as new output
- Add workflow step to write key findings to isolated memory

##### 9. `dispatch-patterns.md`

**Sections to change — Pattern A (lines 4-6):**

- Remove step 4: "invoke aggregator" — orchestrator reads memories directly
- Update step 5: "<2 outputs" threshold still applies but for orchestrator's merge, not aggregator
- Remove step 6: "Check aggregator completion contract"

**Sections to change — Pattern B (lines 8-13):**

- Remove step 3: "forward ERROR to aggregator"
- Remove step 6: "Invoke aggregator with all available outputs"
- Remove step 7: "Check aggregator completion contract"

##### 10. `feature-workflow.prompt.md`

**Sections to change — Rules (lines 15-27):**

- Line 25: "Cluster parallelization" — update to note no aggregator agents
- Line 26: Update research description — no synthesis step
- Add: Memory isolation model description

**Sections to change — Key Artifacts table (lines 33-38):**

- Remove any implied reference to `analysis.md`, `design_critical_review.md`, `verifier.md`, `review.md`
- Add: Agent memory files concept

---

#### Tier 4 — MINOR Changes (14 files — add isolated memory output)

All of these files currently read `memory.md` but do not write to it. Under the new system, each must create its own isolated memory file.

##### CT Sub-Agents (4 files)

**11. `ct-security.agent.md`**

- **Add to Outputs:** isolated memory file (e.g., `memory/ct-security.mem.md`)
- **Add workflow step:** Write key findings summary to isolated memory file (after step 8, before self-verification or after it)
- **Update Anti-Drift Anchor (line 138):** Remove "You do NOT write to `memory.md`" — reframe to "You write to your isolated memory file only"
- **Update references to aggregator:** Currently no explicit aggregator reference, but implicit via the cluster architecture

**12. `ct-scalability.agent.md`**

- Same pattern as ct-security: add isolated memory output, add write step
- **Update Anti-Drift Anchor (line ~148):** Remove "You do NOT write to `memory.md`"

**13. `ct-maintainability.agent.md`**

- Same pattern: add isolated memory output, add write step
- No explicit "do NOT write to memory.md" in anti-drift anchor

**14. `ct-strategy.agent.md`**

- Same pattern: add isolated memory output, add write step
- **Update Anti-Drift Anchor (line 149):** Remove "You do NOT write to `memory.md`"

##### V Sub-Agents (4 files)

**15. `v-build.agent.md`**

- **Add to Outputs:** isolated memory file
- **Replace "No Memory Write" step (line 117):** "The V Aggregator will consolidate..." → write own isolated memory
- Add workflow step to write key findings to isolated memory

**16. `v-tests.agent.md`**

- **Add to Outputs:** isolated memory file
- **Replace "No Memory Write" step (line 146):** "The V Aggregator will consolidate..." → write own isolated memory
- Add workflow step to write key findings to isolated memory

**17. `v-tasks.agent.md`**

- **Add to Outputs:** isolated memory file
- **Update line 53:** Remove "for the V-Aggregator and planner" → "for the orchestrator and planner"
- **Update line 59:** Remove "The V-Aggregator handles memory updates after all V sub-agents complete" → write own isolated memory
- **Update Anti-Drift Anchor (line 159):** Remove "You never write to `memory.md`" or reframe

**18. `v-feature.agent.md`**

- **Add to Outputs:** isolated memory file
- **Update line 52:** Remove "for the V-Aggregator and planner" → "for the orchestrator and planner"
- **Update line 58:** Remove "The V-Aggregator handles memory updates after all V sub-agents complete" → write own isolated memory
- **Update Anti-Drift Anchor (line 173):** Remove "You never write to `memory.md`" or reframe

##### R Sub-Agents (4 files)

**19. `r-quality.agent.md`**

- **Add to Outputs:** isolated memory file
- **Add workflow step:** Write key findings to isolated memory
- **Update "No Memory Write" step:** "The R Aggregator will consolidate..." → write own isolated memory
- **Update Anti-Drift Anchor (line 174):** Remove "You do NOT write to `memory.md`"

**20. `r-security.agent.md`**

- Same pattern: add isolated memory output, add write step
- **Update Anti-Drift Anchor (line 228):** Remove "You do NOT write to `memory.md`"

**21. `r-testing.agent.md`**

- Same pattern: add isolated memory output, add write step
- **Update "No Memory Write" step:** "The R Aggregator will consolidate..." → write own isolated memory

**22. `r-knowledge.agent.md`**

- **Add to Outputs:** isolated memory file
- **Update "No Memory Write" step (line ~280):** "The R Aggregator will consolidate..." → write own isolated memory
- Note: R-Knowledge already writes to `decisions.md` and `knowledge-suggestions.md` — these remain unchanged

##### Implementation/Documentation Agents (2 files)

**23. `implementer.agent.md`**

- **Add to Outputs:** isolated memory file
- **Add workflow step:** After step 6 (Update Task File), write key findings/decisions to isolated memory
- No anti-drift anchor changes needed (no memory-write restriction mentioned)

**24. `documentation-writer.agent.md`**

- **Add to Outputs:** isolated memory file
- **Update Workflow step 7 (line 67):** Replace "Do NOT write to `memory.md`" → write to own isolated memory file

---

#### Tier 5 — NO Changes (1 file)

##### 25. `critical-thinker.agent.md`

- Already marked as deprecated (line 1). No changes needed per initial-request.md.

---

### Cascade Effects

#### Cascade 1: Removing analysis.md triggers input changes in 4 downstream agents

- `orchestrator.agent.md` → removes Step 1.2 (synthesis) and all `analysis.md` input references
- `spec.agent.md` → input changes from `analysis.md` to `research/*.md`
- `designer.agent.md` → input changes from `analysis.md` to `research/*.md`
- `planner.agent.md` → input changes from `analysis.md` to `research/*.md`

#### Cascade 2: Removing aggregators triggers orchestrator absorption of decision logic

- CT aggregator removal → orchestrator must determine severity from CT memories (DONE/NEEDS_REVISION logic)
- V aggregator removal → orchestrator must determine pass/fail from V memories and handle task ID failure mapping (most complex)
- R aggregator removal → orchestrator must determine result from R memories and enforce R-Security pipeline override

#### Cascade 3: Removing design_critical_review.md triggers changes in 3 files

- `orchestrator.agent.md` → removes CT aggregator invocation, updates CT result handling, removes reference in Step 4 planner inputs
- `designer.agent.md` → updates revision cycle input from `design_critical_review.md` to individual CT review files
- `planner.agent.md` → removes planning constraints reference to `design_critical_review.md` (if any — currently passed through orchestrator)

#### Cascade 4: Removing verifier.md triggers changes in 2 files

- `orchestrator.agent.md` → removes V aggregator invocation, updates V result handling and Pattern C loop
- `planner.agent.md` → replan mode must read individual V artifacts instead of `verifier.md` for task ID failures

#### Cascade 5: Removing review.md triggers changes in 1 file

- `orchestrator.agent.md` → removes R aggregator invocation, updates R result handling

#### Cascade 6: Adding isolated memory output triggers changes in 14 sub-agent files

- Every sub-agent (CT ×4, V ×4, R ×4, implementer, documentation-writer) needs:
  1. New output in Outputs section
  2. New workflow step to write isolated memory
  3. Updates to anti-drift anchors and memory-write restriction text
  4. Updates to "No Memory Write" sections (V and R sub-agents)

#### Cascade 7: Memory merge responsibility moves to orchestrator

- `orchestrator.agent.md` → gains new "Merge" action in Memory Lifecycle
- All sub-agents → no longer have aggregators that consolidate their findings into memory
- Orchestrator's between-waves memory updates (Step 5.2, line 240) must be generalized

#### Cascade 8: dispatch-patterns.md changes cascade to orchestrator pattern references

- Pattern A, B, C definitions change → orchestrator sections referencing these patterns must align

### Risk Areas

#### Risk 1: Task ID Failure Mapping Loss (HIGH RISK)

- **Current state:** `v-aggregator.agent.md` performs critical cross-referencing: merges `v-tasks.md` failing task IDs with `v-tests.md` test failures and `v-feature.md` criteria gaps into `verifier.md`'s Actionable Items section. The planner depends on this exact mapping for targeted replanning.
- **Risk:** Removing V-Aggregator without reproducing this cross-referencing logic means the planner loses its primary input for targeted replan. The orchestrator must either (a) absorb this merging logic, or (b) rely on individual V artifacts — but then each V sub-agent's output format becomes more critical and may need enhancement to include task ID references.
- **Affected files:** `v-aggregator.agent.md` (removed), `orchestrator.agent.md` (must absorb), `planner.agent.md` (input format changes), `v-tasks.agent.md` (may need enhanced output)

#### Risk 2: R-Security Pipeline Override Preservation (HIGH RISK)

- **Current state:** `r-aggregator.agent.md` enforces the R-Security pipeline override (Critical/Blocker findings → pipeline ERROR). This is a safety-critical rule.
- **Risk:** If the orchestrator doesn't correctly absorb this override logic, security-critical findings could be downgraded or missed.
- **Affected files:** `r-aggregator.agent.md` (removed), `orchestrator.agent.md` (must absorb)

#### Risk 3: Orchestrator Complexity Explosion (MEDIUM RISK)

- **Current state:** The orchestrator is already 399 lines with significant complexity. Absorbing 3 aggregators' decision logic (CT severity determination, V pass/fail with task mapping, R security override) adds substantial new responsibilities.
- **Risk:** The orchestrator could become unwieldy, violating the original design principle of keeping merge-only logic separate. The memory merging responsibility adds another layer.
- **Affected files:** `orchestrator.agent.md`

#### Risk 4: Memory File Naming/Path Convention (MEDIUM RISK)

- **Current state:** No convention exists for isolated memory file paths.
- **Risk:** Without a clear naming convention (e.g., `memory/ct-security.mem.md` vs `ct-review/ct-security.mem.md`), implementations could be inconsistent. The initial request doesn't specify the exact path pattern.
- **Affected files:** All 14 sub-agent files, `orchestrator.agent.md` (must know where to find them)

#### Risk 5: Spec/Designer/Planner Input Expansion (LOW-MEDIUM RISK)

- **Current state:** These agents read a single `analysis.md` as consolidated research input.
- **Risk:** Switching to 4 individual research files (`research/architecture.md`, `research/impact.md`, `research/dependencies.md`, `research/patterns.md`) increases the input surface and reading burden for these agents. If a research file is missing (agent ERROR'd), the downstream agent must handle gracefully.
- **Affected files:** `spec.agent.md`, `designer.agent.md`, `planner.agent.md`

#### Risk 6: Pattern C (Replan Loop) Refactoring (MEDIUM RISK)

- **Current state:** Pattern C references `verifier.md` and the V aggregator's completion contract for loop decisions.
- **Risk:** Refactoring Pattern C to work with individual V memories and artifacts requires careful alignment. The loop exit condition ("aggregator DONE") must map to reading individual V memories.
- **Affected files:** `orchestrator.agent.md` (Pattern C section), `dispatch-patterns.md`

## File References

| File                            | Impact Level | Rationale                                                                                                  |
| ------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------- |
| `orchestrator.agent.md`         | Major        | Primary coordination hub; absorbs aggregator logic, removes analysis.md, adds memory merge                 |
| `researcher.agent.md`           | Major        | Remove synthesis mode, remove analysis.md output, add isolated memory                                      |
| `ct-aggregator.agent.md`        | Remove       | Deprecated — orchestrator reads CT memories directly                                                       |
| `v-aggregator.agent.md`         | Remove       | Deprecated — orchestrator reads V memories directly                                                        |
| `r-aggregator.agent.md`         | Remove       | Deprecated — orchestrator reads R memories directly                                                        |
| `spec.agent.md`                 | Moderate     | Input change: analysis.md → research/\*.md; add isolated memory                                            |
| `designer.agent.md`             | Moderate     | Input change: analysis.md → research/_.md, design_critical_review.md → ct-review/_.md; add isolated memory |
| `planner.agent.md`              | Moderate     | Input change: analysis.md → research/_.md, verifier.md → verification/_.md; add isolated memory            |
| `dispatch-patterns.md`          | Moderate     | Remove aggregator steps from Pattern A, B definitions                                                      |
| `feature-workflow.prompt.md`    | Moderate     | Update workflow description, key artifacts, memory model                                                   |
| `ct-security.agent.md`          | Minor        | Add isolated memory output and write step                                                                  |
| `ct-scalability.agent.md`       | Minor        | Add isolated memory output and write step                                                                  |
| `ct-maintainability.agent.md`   | Minor        | Add isolated memory output and write step                                                                  |
| `ct-strategy.agent.md`          | Minor        | Add isolated memory output and write step                                                                  |
| `v-build.agent.md`              | Minor        | Add isolated memory output; replace "No Memory Write" section                                              |
| `v-tests.agent.md`              | Minor        | Add isolated memory output; replace "No Memory Write" section                                              |
| `v-tasks.agent.md`              | Minor        | Add isolated memory output; update V-Aggregator references                                                 |
| `v-feature.agent.md`            | Minor        | Add isolated memory output; update V-Aggregator references                                                 |
| `r-quality.agent.md`            | Minor        | Add isolated memory output; replace "No Memory Write" section                                              |
| `r-security.agent.md`           | Minor        | Add isolated memory output; update anti-drift anchor                                                       |
| `r-testing.agent.md`            | Minor        | Add isolated memory output; replace "No Memory Write" section                                              |
| `r-knowledge.agent.md`          | Minor        | Add isolated memory output; replace "No Memory Write" section                                              |
| `implementer.agent.md`          | Minor        | Add isolated memory output and write step                                                                  |
| `documentation-writer.agent.md` | Minor        | Add isolated memory output; update memory-write restriction                                                |
| `critical-thinker.agent.md`     | None         | Already deprecated — no changes needed                                                                     |

## Assumptions & Limitations

1. **Assumed:** The initial-request.md's description of "each subagent creates its own isolated memory file" means a file written by the agent as a new output artifact (separate from the existing primary artifact like `ct-review/ct-security.md`). The exact file path pattern is not specified — this research documents what needs to change, not the specific naming convention.
2. **Assumed:** The orchestrator's "merge" operation for isolated memories is a sequential append/update to shared `memory.md`, occurring after each agent completes (or between waves for parallel agents).
3. **Assumed:** The planner's replan mode will read individual V sub-agent artifacts directly instead of `verifier.md`, but the exact format of the task ID failure mapping input needs to be designed (currently provided by V-Aggregator's cross-referencing logic).
4. **Limitation:** This analysis covers file-level and section-level impact, not line-by-line editing instructions. The exact new text for each section is a design/implementation concern.
5. **Limitation:** The `r-knowledge.agent.md` references `verifier.md` via memory navigation (line 25: "Use the Artifact Index in `memory.md` to identify and read only the relevant sections of `feature.md`, `design.md`, `plan.md`, and `verifier.md`"). Since `verifier.md` is removed, this reference needs updating — R-Knowledge would navigate to individual V artifacts instead.

## Open Questions

1. **Memory file naming convention:** What path pattern should isolated memory files use? Options include:
   - `memory/<agent-name>.mem.md` (centralized memory directory)
   - `<output-dir>/<agent-name>.mem.md` (co-located with artifacts)
   - Something else?

2. **Orchestrator complexity management:** With the orchestrator absorbing 3 aggregators' decision logic plus memory merge responsibility, should any of this be extracted into a separate reference document (like `dispatch-patterns.md`) to keep the orchestrator manageable?

3. **Task ID failure mapping for replan:** When V-Aggregator is removed, how does the planner get the cross-referenced task ID failure map? Options:
   - Orchestrator builds it from V memories during merge
   - Planner reads individual V artifacts directly and cross-references itself
   - V-Tasks agent enhances its output to include all needed failure context

4. **R-Knowledge `verifier.md` reference:** Line 25 of `r-knowledge.agent.md` lists `verifier.md` as an artifact accessed via the Artifact Index. With `verifier.md` removed, should R-Knowledge read individual V artifacts, or is the memory sufficient?

5. **Scope of memory isolation for sequential agents:** Currently `spec`, `designer`, and `planner` write to shared `memory.md` directly (as sequential agents). Under the new model, do they also use isolated memory files (branch-merge), or do they retain direct write access since they run sequentially?

## Research Metadata

- confidence_level: high
- coverage_estimate: Complete — all 25 files in `NewAgentsAndPrompts/` were read in full and analyzed for all four reference categories (memory.md, analysis.md, aggregator outputs, synthesis mode). Cross-references validated via grep search.
- gaps: No significant gaps. The only uncovered area is the specific content of new isolated memory file templates and naming conventions, which are design decisions outside the scope of impact research.
