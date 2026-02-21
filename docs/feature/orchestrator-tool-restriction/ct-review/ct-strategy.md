# Critical Review: Strategy (Iteration 3 — Post-Second Revision)

**Context:** Third iteration review. Iteration 2 produced 0 High, 1 Medium, 3 Low findings against the Phase 1 design. The designer addressed the Medium finding (feature.md scope mismatch) by adding §16.5 (Feature Spec Scope Note) and responded to the memory tool contradiction findings from other CT agents by correcting all 5 "via `memory` tool" references to "via subagent delegation" (§3.5–§3.9). The Anti-Drift prohibition rationale was updated to cover both write and read tools. This review evaluates whether the **second design revision** resolves the remaining concerns.

**Iteration history: iter 1 → 2H/4M/2L | iter 2 → 0H/1M/3L | iter 3 → 0H/1M/2L**

---

## Prior Findings Disposition (Iteration 2 → Iteration 3)

| Iter 2 Finding | Severity | Status in Revised Design |
|---|---|---|
| feature.md not scoped to Phase 1 — V cluster risk | Medium | **Partially resolved.** §16.5 added explicit guidance for implementer/verifier. But §16.3/§16.5 incorrectly classify AC-13 and AC-14 as "In scope (no changes needed)" — see Finding 1 below. The broader concern (8 clearly-deferred ACs) is mitigated. |
| YAML `tools:` advisory enforcement | Low | **Unchanged.** Accepted risk with good mitigation. Carried forward. |
| Disambiguation exposes memory tool / memory.md paradox | Low | **Resolved.** All 5 contradicting references corrected: Global Rule 12 → "by dispatching a subagent," Step 0.1 → "Delegate to a setup subagent," Step 8.2 → "dispatches a subagent," Memory Lifecycle Table → "Delegate to a setup subagent," merge step 1.1m wording fixed. The Anti-Drift now says "dispatch subagents to merge" — no paradox remains. |
| Phase 2 scope intentionally light | Low | **Unchanged.** Still valid but truly low impact. Carried forward. |

---

## Findings

### [Severity: Medium] AC-13 and AC-14 Misclassified as Phase 1 "In Scope" — Pass Criteria Require Phase 2 Behavior

- **What:** Design §16.3 marks AC-13 (Cluster Decisions Unaffected) and AC-14 (Self-Verification Preserved) as "In scope (no changes needed)" and §16.5 counts them in the Phase 1 subset (7 ACs). However, the Pass criteria in `feature.md` for these ACs explicitly describe Phase 2 behavior where memory.md logging is removed:

  - **AC-13 Pass:** "CT, V, and R cluster decision flow sections still read `memory/<agent>.mem.md` files via `read_file`, apply decision tables, and dispatch based on results. **Only the self-verification logging to `memory.md` is removed.**" — In Phase 1, self-verification logging to memory.md is NOT removed (memory.md lifecycle is preserved). This Pass condition is unmet.

  - **AC-14 Pass:** "Cluster decision sections retain self-verification logic (evaluate → decide → proceed) but log results to **the orchestrator's context window** (captured by telemetry tracking) **rather than `memory.md`**." — In Phase 1, logging still goes TO memory.md. The "rather than" condition is unmet. **AC-14 Fail:** "Self-verification logic is entirely removed, **or it still references writing to `memory.md`**." — Phase 1 preserves all memory.md write references (corrected to subagent delegation wording, but still writing to memory.md). This Fail condition IS triggered.

  The design instructs the verifier (§16.5) to check 7 ACs including AC-13 and AC-14. If the v-feature agent checks these against feature.md's explicit Pass/Fail text, both ACs will fail, triggering a revision loop — even though Phase 1 isn't supposed to change this behavior.

- **Where:** feature.md AC-13 (L233) — Pass criterion requires self-verification logging removal. feature.md AC-14 (L238) — Pass criterion requires context-window logging "rather than memory.md"; Fail criterion triggered by writing to memory.md. design.md §16.3 (L558) — marks both as "In scope (no changes needed)." design.md §16.5 (L579) — counts "7 ACs" including AC-13 and AC-14.

- **Likelihood:** Medium — the v-feature agent reads both feature.md (Step 2) and design.md (Step 4). The design §7.1 says "Pipeline Flow — Unchanged," which a sophisticated LLM could use to override the feature.md Pass/Fail text. But the explicit Fail criteria in feature.md are unambiguous.

- **Impact:** Medium — if v-feature fails AC-13 and AC-14 based on feature.md's literal Pass/Fail definitions, it triggers a revision loop for 2 ACs that shouldn't be checked in Phase 1. The simplest fix: move AC-13 and AC-14 to "Deferred" in §16.3 and update §16.5 to say "5 ACs" (AC-1, AC-2, AC-3, AC-11, AC-12).

- **Assumption at risk:** That AC-13 and AC-14's Pass criteria are achievable without removing memory.md writes — they are not, as written in feature.md.

---

### [Severity: Low] YAML `tools:` Field Remains an Unvalidated Convention

- **What:** Unchanged from iteration 2. The orchestrator would be the first agent to use a `tools:` YAML frontmatter field. No runtime enforcement confirmed. The design transparently acknowledges this (§2.4) with triple-layered mitigation (YAML + prose + Anti-Drift). Consistent with how all 20 other agents manage tool boundaries. Accepted risk.

- **Where:** design.md §2.4.

- **Likelihood:** Low
- **Impact:** Low
- **Assumption at risk:** That VS Code will eventually enforce `tools:` YAML fields.

---

### [Severity: Low] Phase 2 Scope Intentionally Light — May Need Re-Speccing

- **What:** Unchanged from iteration 2. Phase 2 (§15) has no requirements, ACs, or timeline — appropriate for a deferral note. Current `feature.md` cannot serve as Phase 2 spec without incorporating CT-measured evidence (avg 488 tokens per `.mem.md`, 8+ Operating Rule 6 variants, broken keyword heuristic).

- **Where:** design.md §15, feature.md.

- **Likelihood:** Low — Phase 2 "may or may not proceed."
- **Impact:** Low — if Phase 2 proceeds, fresh specification is needed regardless.
- **Assumption at risk:** That `feature.md` is reusable for Phase 2 without significant revision.

---

## Cross-Cutting Observations

- **CT-Maintainability scope:** The memory tool / memory.md paradox that ct-maintainability identified as High in iteration 2 is now fully resolved. All 5 contradicting references corrected to subagent delegation. No remaining maintainability concerns for Phase 1.

- **CT-Security scope:** The memory tool disambiguation is now consistent — no contradicting references remain to create confusion vectors. The advisory-only YAML enforcement is transparently accepted. Phase 1 is a net security improvement.

- **CT-Scalability scope:** No change from iteration 2. Phase 1 has zero performance/scalability impact.

## Requirement Coverage (Phase 1 Scope)

| Requirement | Coverage Status | Notes |
| ----------- | --------------- | ----- |
| FR-1 (Tool Restriction) | Fully covered | YAML field + prose restriction + Anti-Drift Anchor with dual rationale |
| FR-6 (Memory Tool Disambiguation) | Fully covered | 3-location disambiguation + 5 contradiction corrections (§3.5–§3.9) |
| FR-2 through FR-5 (Memory Elimination, Dispatch, Subagents, Supporting Docs) | Explicitly deferred | Deferred to Phase 2 (§14.3) |
| NFR-1 (Backward Compatibility) | Covered | No data migration, no schema changes, memory.md preserved |
| NFR-2 (No New Subagents) | Covered | No new agents; merge wording corrections only |
| NFR-4.1 (YAML/prose consistency) | Covered | Identical 5-tool list in YAML and prose |
| NFR-5 (Cluster Decision Integrity) | Covered | Decision flows unchanged (§7.1) |
| AC-1 (YAML tools field) | In scope — covered | §2.2 specifies exact frontmatter |
| AC-2 (Prose matches YAML) | In scope — covered | §3.3 specifies identical tool list |
| AC-3 (Removed tool refs eliminated) | In scope — covered | §3.1–§3.4 remove all 4 tool references |
| AC-11 (Memory tool disambiguated) | In scope — covered | §3.1, §3.3, §3.4, §3.5–§3.9 |
| AC-12 (Anti-Drift updated) | In scope — covered | §3.4 with dual prohibition rationale |
| AC-13 (Cluster decisions unaffected) | **Misclassified** — should be Deferred | Pass criteria require memory.md logging removal (**Finding 1**) |
| AC-14 (Self-verification preserved) | **Misclassified** — should be Deferred | Fail criteria triggered by memory.md writes (**Finding 1**) |
| AC-4 through AC-10, AC-15 | Deferred to Phase 2 | Correctly mapped in §16.3 |

## Fundamental Approach Assessment

**Is this the right approach?** Yes — more confidently than iteration 2. The second revision resolved the remaining paradox (memory tool contradictions) cleanly, and the merge wording corrections (§3.5–§3.9) fix a pre-existing documentation bug as a bonus. Phase 1 is a well-scoped, low-risk, 2-file change that delivers independent value.

**What's the worst thing that could happen if Phase 1 ships as-is?** The v-feature agent checks AC-13/AC-14 against feature.md's literal Pass/Fail text and fails them, causing a wasted revision cycle. This is avoidable by moving AC-13/AC-14 to "Deferred" in §16.3/§16.5. Outside of this verification-process concern, the design itself has no remaining strategic risks.

**Alternative approaches not considered?** None needed. The design is appropriately minimal. Three iterations of CT review have converged — no fundamental concerns remain.
