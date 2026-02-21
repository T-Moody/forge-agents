# Critical Review: Maintainability (Iteration 3)

**Scope:** Re-review after second design revision. Designer corrected all 5 "via memory tool" contradictions (§3.5–§3.9) to "via subagent delegation" and updated Anti-Drift prohibition rationale with dual justification covering both write and read tools.

---

## Prior Findings Disposition (Iterations 1–2)

| # | Prior Finding | Prior Severity | Status |
|---|---------------|---------------|--------|
| 1 | "3 Universal Changes" masks per-agent variation (iter 1) | High | **Resolved** (iter 2). Phase 1 does not touch subagent files. |
| 2 | Post-mortem agent has undocumented memory.md edit points (iter 1) | High | **Resolved** (iter 2). Post-mortem not modified. |
| 3 | Planner undocumented memory.md reference (iter 1) | Medium | **Resolved** (iter 2). Planner not modified. |
| 4 | Context window becomes hidden shared state (iter 1) | Medium | **Resolved** (iter 2). memory.md persistence retained. |
| 5 | Lessons Learned extraction algorithm is fragile (iter 1) | Medium | **Resolved** (iter 2). Broken keyword heuristic dropped. |
| 6 | Invalidation via dispatch prompt weaker than file markers (iter 1) | Medium | **Resolved** (iter 2). File-based `[INVALIDATED]` markers preserved. |
| 7 | "Xc" step suffix has no discoverability (iter 1) | Low | **Resolved** (iter 2). No step renaming in Phase 1. |
| 8 | Memory Lifecycle table removal loses at-a-glance reference (iter 1) | Low | **Resolved** (iter 2). Table preserved. |
| 9 | Memory tool disambiguation contradicts preserved merge references (iter 2) | High | **Resolved** (iter 3). All 5 "via memory tool" references corrected to subagent delegation in §3.5–§3.9. See assessment below. |
| 10 | Anti-Drift prohibition rationale gap — read tools not justified (iter 2) | Medium | **Resolved** (iter 3). Dual justification now covers both write and read tools. See assessment below. |
| 11 | Optional feature-workflow.prompt.md risks documentation drift (iter 2) | Low | **Unchanged.** Still marked optional. Still low risk. Carried forward below. |

---

## Iteration 3 — Assessment of Prior High and Medium Fixes

### [Prior High — Finding 9] Memory Tool Disambiguation Contradiction: RESOLVED

The design now corrects all 5 explicit "via memory tool" references that contradicted the disambiguation statements:

| Location | Prior Text | Corrected Text (§) | Verified in orchestrator.agent.md |
|----------|-----------|---------------------|-----------------------------------|
| Global Rule 12, L44 | "via `memory` tool merging" | "by dispatching a subagent to perform the merge" (§3.5) | L44 confirmed: contains "via `memory` tool merging" |
| Global Rule 12, L46 | "via `memory` tool" | "dispatches a subagent to merge them" (§3.5) | L46 confirmed: contains "via `memory` tool" |
| Step 0.1, L188 | "Use the `memory` tool to create" | "Delegate to a setup subagent via `runSubagent`" (§3.6) | L188 confirmed: contains "Use the `memory` tool to create" |
| Step 8.2, L456 | "using `memory` tool or delegating to subagent" | "dispatches a subagent to merge" (§3.7) | L456 confirmed: contains "using `memory` tool or delegating to subagent" |
| Lifecycle Table Initialize, L503 | "Use `memory` tool to create" | "Delegate to a setup subagent to create" (§3.8) | L503 confirmed: contains "Use `memory` tool to create" |

Additionally, §3.9 corrects all 8 merge step descriptions from "orchestrator merges" to "orchestrator dispatches a subagent to merge" and removes the incorrect line "This is an orchestrator merge operation — no subagent invocation" from Step 1.1m (L237).

After these corrections, the three disambiguation statements (Global Rule 1, Operating Rule 5, Anti-Drift Anchor) saying "memory tool does NOT write to pipeline files" will no longer contradict ANY text in the orchestrator prompt. The prior High finding is **fully resolved**.

### [Prior Medium — Finding 10] Anti-Drift Prohibition Rationale Gap: RESOLVED

The Anti-Drift Anchor prohibition now reads (§3.4):

> "**You MUST NOT use `create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`, `grep_search`, `semantic_search`, `file_search`, or `get_errors` — all file writes are delegated to subagents; search/discovery tools are unnecessary because the orchestrator reads known deterministic paths only.**"

This provides dual justification:
1. Write tools (`create_file`, `replace_string_in_file`, `multi_replace_string_in_file`, `run_in_terminal`): "all file writes are delegated to subagents"
2. Read/search tools (`grep_search`, `semantic_search`, `file_search`, `get_errors`): "search/discovery tools are unnecessary because the orchestrator reads known deterministic paths only"

The logical escape hatch identified in iteration 2 is closed. Both categories of prohibited tools have independent, logically sound rationales. **Fully resolved.**

---

## Remaining and New Findings

### [Severity: Low] Residual Ambiguous "Merges" Language in Summary-Level References

- **What:** The design corrects all 5 explicit "via memory tool" references and all 8 merge step body descriptions. However, 3 summary-level references still use bare "merges"/"merge" language without specifying subagent delegation:
  1. **Global Rule 6 (L38):** "orchestrator reads the agent's `memory/<agent>.mem.md` and merges Key Findings, Decisions, and Artifact Index into `memory.md`"
  2. **Memory Lifecycle Merge row (L504):** "Orchestrator reads `memory/<agent>.mem.md`, merges Key Findings, Decisions, and Artifact Index into `memory.md`"
  3. **Memory Lifecycle Merge (post-mortem) row (L510):** "Read `memory/post-mortem.mem.md`, merge into `memory.md`"

  The design explicitly declares "All other Global Rules (2–11, 13) are unchanged" (§3.1) and §3.8 targets only the Initialize row of the Lifecycle table. These 3 residual references don't say "via memory tool" (avoiding the contradiction that the prior High finding identified), but they do use "merges" in a way that implies direct orchestrator action on files — inconsistent with the corrected merge step wording ("dispatches a subagent to merge").

- **Where:** orchestrator.agent.md Global Rule 6 (L38), Memory Lifecycle table rows at L504 and L510; design §3.1 (explicitly preserves Global Rule 6) and §3.8 (only corrects Initialize row).
- **Likelihood:** Low — the corrected detailed text (Global Rules 1 and 12, all merge step descriptions, Anti-Drift Anchor) clearly establishes subagent delegation. An LLM encountering "merges" in a summary-level rule alongside "dispatches a subagent to merge" in the adjacent detailed descriptions would likely infer the correct mechanism.
- **Impact:** Low — at worst, the LLM might attempt a direct file write at a merge point, which it still can't execute (no write tools in YAML). The fallback path is subagent delegation.
- **Assumption at risk:** That LLMs always resolve ambiguity in summary-level rules by consulting the adjacent detailed descriptions rather than treating the summary as authoritative.

### [Severity: Low] Optional `feature-workflow.prompt.md` Change Risks Documentation Drift (Carried Forward)

- **What:** The design still marks the `feature-workflow.prompt.md` change as "optional" (§4.3, §16.1 Priority: Low). If the implementer skips it, the pipeline entry-point documentation will not mention the orchestrator's tool restriction.
- **Where:** Design §4.3; `feature-workflow.prompt.md` Rules section.
- **Likelihood:** Medium — "optional" label invites omission.
- **Impact:** Low — drift is cosmetic; orchestrator prompt is authoritative.
- **Assumption at risk:** That developers consult `orchestrator.agent.md` directly rather than the higher-level workflow prompt.

---

## Cross-Cutting Observations

- **Security (CT-Security scope):** The memory tool disambiguation contradiction (prior iter 2 High) is now resolved. The security dimension (stale memory.md from LLM obeying disambiguation and refusing to write) is no longer applicable because no text tells the LLM to use the memory tool for file operations.

- **Strategy (CT-Strategy scope):** The design's characterization of the existing "via memory tool" language as a "pre-existing bug" (§3.5 Rationale, §1.3) is strategically sound — it allows the corrections to be framed as bug fixes rather than behavioral changes, reducing implementation risk. The Phase 2 strategic tension (memory.md viability) remains but is cleanly separated.

---

## Requirement Coverage (Phase 1 Subset)

| Requirement | Coverage Status | Notes |
| ----------- | --------------- | ----- |
| FR-1.1 (YAML tools field) | Adequate | §2 specifies exact YAML. |
| FR-1.2 (Prose matches YAML) | Adequate | §3.3 Operating Rule 5 matches §2. |
| FR-1.3 (Global Rule 1 updated) | Adequate | Contradiction with Global Rule 12 is resolved by §3.5. |
| FR-1.4 (Operating Rule 1 updated) | Adequate | §3.2 removes discovery tool preference cleanly. |
| FR-1.5 (Anti-Drift Anchor updated) | Adequate | Dual justification now covers both write and read tools (§3.4). |
| FR-6.1 (Memory tool disambiguated) | Adequate | Disambiguation in 3 locations + 5 contradicting refs corrected (§3.5–§3.9). Internally consistent. |
| AC-1 (YAML tools field) | Adequate | §2 clear and testable. |
| AC-2 (Prose matches YAML) | Adequate | §3.3 and §2 are consistent. |
| AC-3 (Removed tool refs eliminated) | Adequate | §3.1–§3.4 cover all 4 removed tools. |
| AC-11 (Memory tool disambiguated) | Adequate | All contradictions corrected. 3 residual ambiguous "merges" in summaries are Low risk (see finding). |
| AC-12 (Anti-Drift updated) | Adequate | Dual justification resolves rationale gap. |
| AC-13 (Cluster decisions unaffected) | Adequate | §7.1 correctly notes no change needed. |
| AC-14 (Self-verification preserved) | Adequate | §7.1 preserves existing mechanism. |
