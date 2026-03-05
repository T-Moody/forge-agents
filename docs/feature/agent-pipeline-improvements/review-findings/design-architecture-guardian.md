# Adversarial Review: design — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🟡
- **Round:** 2
- **Run ID:** 2026-03-05T12:00:00Z

---

## Round 1 Finding Resolution

| R1 Finding                                                                                           | Severity | Status                 | Resolution                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------- | -------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A-1: FR-2.4/FR-8.6 spec contradiction — feature-workflow.prompt.md modified without deviation record | Major    | **Resolved**           | DR-2 added. APPROVAL_MODE clarification relocated to orchestrator Step 0. feature-workflow.prompt.md removed from file inventory. FR-8.6 honored.                                                                             |
| A-2: EG-10(full-tdd-e2e) modification details unspecified — double-gating risk                       | Minor    | **Partially improved** | R2 design.md states "Keep existing lane-based minimum check counts for non-E2E checks" but still does not explicitly specify removal of e2e-test-execution from EG-10(full-tdd-e2e) IN clause or count reduction from 4 to 3. |
| S-1: fetch_webpage scope restriction is vague                                                        | Minor    | **Unchanged**          | Accepted as consistent with system-wide instruction-based enforcement (DR-1).                                                                                                                                                 |
| C-1: "default to autonomous" occurrence count inflated                                               | Minor    | **Resolved**           | D-4 now correctly states "2 times in lines 105 and 109."                                                                                                                                                                      |
| C-2: Fast-track step set supersedes FR-8.1                                                           | Minor    | **Resolved**           | DR-3 deviation record added, justified by FR-8.7 recommendation.                                                                                                                                                              |

---

## Security Analysis

**Category Verdict:** approve

No security findings. The R2 revisions strengthen the security posture:

- **D-14 (Minimum step set):** The immutable minimum step set {Step 0, Step 7, Step 9} in the anti-drift anchor is a sound systemic guarantee. Placing it in the anchor (the highest-impact behavioral enforcement section) rather than a per-prompt property ensures no future prompt can bypass security review. This fully addresses security-sentinel's R1 S-1 finding. From an architectural boundary perspective, the constraint is placed at the correct enforcement layer — the orchestrator's self-governance rules, not external documentation.
- **DR-2 (APPROVAL_MODE relocation):** Moving the APPROVAL_MODE priority statement to orchestrator Step 0 keeps authorization logic within its owning component, consistent with single-responsibility. No new trust boundary concerns introduced.
- **fetch_webpage scope (unchanged):** Residual Minor from R1 — vague scope language is accepted as system-wide technical debt per DR-1 and tool-access-matrix.md §11.

---

## Architecture Analysis

**Category Verdict:** approve

### R1 A-1 Resolution Assessment

The A-1 finding (Major) is **fully resolved**. DR-2 is architecturally the correct resolution — option (a) from my R1 recommendation. The rationale is precise: "The orchestrator Step 0 already owns approval mode selection logic (lines 99-109). Placing the 'initial-request.md APPROVAL_MODE takes absolute priority' statement there is architecturally cleaner than modifying the prompt file. The prompt file defines what steps to run; the orchestrator defines how steps execute." This correctly identifies the separation of concerns: prompts define **which** steps, orchestrator defines **how** steps behave. APPROVAL_MODE is an execution-time concern.

### Finding A-1: Orchestrator at exact budget ceiling (550/550) leaves zero implementation headroom

- **Severity:** Minor
- **Description:** D-11 calculates the orchestrator's post-change line count as ~550/550 — exactly at the NFR-1 budget ceiling. This means any implementation discovery requiring even 1 additional line triggers content extraction to a reference document. The design responsibly identifies the DB archive procedure (~8 lines) as the first extraction candidate and marks D-11 as "Medium confidence." However, zero headroom increases implementation friction and risk of mid-task extraction rework. The line budget arithmetic depends on the 539-line baseline figure, which was disputed by pragmatic-verifier in R1 (measured 353 lines). The designer re-verified at 539 via direct file read and cites research/impact.yaml line 410. If the baseline is correct at 539, the zero-headroom state is architecturally fragile but manageable with the documented extraction fallback.
- **Affected artifacts:** design-output.yaml D-11, design.md NFR-1
- **Recommendation:** No design change required. The extraction fallback (DB archive procedure) is documented and actionable. The implementer should count lines after Task 4 (the 🔴 orchestrator modification) and trigger extraction immediately if over budget, before proceeding to subsequent tasks. This is implementation guidance, not a design deficiency.
- **Evidence:** D-11 states "Net: ~+11 lines → 550/550" and notes "At budget ceiling; if implementation reveals more lines needed, DB archive procedure (~8 lines) is first extraction candidate." D-11 confidence is correctly marked as "Medium."

### Finding A-2: EG-10(full-tdd-e2e) restructuring details remain implicit

- **Severity:** Minor
- **Description:** Carried from R1 (originally Minor). The sql-templates.md task description says "make e2e-test-execution an additive check gated on e2e_required=true, independent of workflow_lane." R2 design.md's Data Models section now adds "Keep existing lane-based minimum check counts for non-E2E checks," which improves clarity. However, the explicit implementation instruction — "remove e2e-test-execution from EG-10(full-tdd-e2e) IN clause, reduce expected count from 4 to 3" — is still absent. An implementer could reasonably interpret "additive check" as adding a new gate while leaving EG-10 unchanged, creating double-gating.
- **Affected artifacts:** design-output.yaml sql-templates.md entry, D-2 rationale
- **Recommendation:** The implementer should confirm EG-10(full-tdd-e2e) expected count is reduced to 3 when implementing Task 3 (sql-templates.md). This is implementation guidance that can be caught during code review if misinterpreted. Not blocking.
- **Evidence:** design-output.yaml sql-templates.md description: "Restructure EG-10 — make e2e-test-execution an additive check gated on e2e_required=true, independent of workflow_lane." D-2 says "EG-10 restructures to make e2e-test-execution an additive check rather than a lane-specific variant, leveraging the existing EG-9 independent gate." The removal from EG-10(full-tdd-e2e) is implied but not stated.

### Structural Assessment (No Findings)

- **Wave ordering:** Foundation → Core agents → Extended agents → Prompt. Dependencies flow forward with no circular references. ✅
- **File inventory completeness:** 15 files (1 new + 14 modified). All 9 agents (orchestrator + 8 subagents) properly covered for FR-2.2 after R2 additions. ✅
- **Component boundaries:** Each file's modifications are scoped to its owning concern. No cross-boundary leakage. ✅
- **Deviation records:** 3 deviation records (DR-1, DR-2, DR-3) properly documenting spec departures. DR-2 resolves the R1 A-1 contradiction. ✅
- **D-14 placement:** Minimum step set in anti-drift anchor (not global-operating-rules.md) is the architecturally correct location — highest behavioral enforcement impact, within the orchestrator's self-governance scope. ✅

---

## Correctness Analysis

**Category Verdict:** approve

### R1 Finding Resolution Verification

- **C-1 (Minor, Resolved):** D-4 rationale now states "appearing 2 times in lines 105 and 109" — corrected from the R1 claim of 3 occurrences. ✅
- **C-2 (Minor, Resolved):** DR-3 explicitly records the FR-8.1 deviation with clear justification citing FR-8.7's SHOULD recommendation. The deviation record correctly notes that FR-8.7 recommends Step 8 inclusion, which the design follows. ✅

### Cross-Reviewer Finding Resolution Verification

- **Security-sentinel S-1 (Major, Resolved):** D-14 adds immutable minimum step set {Step 0, Step 7, Step 9}. Both line 29 and line 539 are amended to the same conditional language with the minimum step set constraint. The design.md Security Considerations section now explicitly references D-14 and the immutable constraint. ✅
- **Pragmatic-verifier C-1 (Major, Resolved):** FR-2.2 now tracked in all 8 subagent file inventory entries. adversarial-reviewer.agent.md and knowledge-agent.agent.md added with FR-2.2 coverage. Per_issue_summary for Issue #2 lists all 9 affected files. ✅
- **Pragmatic-verifier C-2 (Major, Resolved):** Both line 29 (Role & Purpose) and line 539 (anti-drift anchor) are now explicitly amended in the orchestrator change description. D-11 line accounting includes "+1 line" for line 29 amendment. ✅

### Contract Consistency Check

| Producer                     | Consumer            | Contract                            | Verified                                           |
| ---------------------------- | ------------------- | ----------------------------------- | -------------------------------------------------- |
| plan-and-implement.prompt.md | orchestrator Step 0 | `pipeline_mode=fast-track` variable | ✅ D-10 specifies detection                        |
| Planner                      | Orchestrator EG-10  | `e2e_required` independent boolean  | ✅ D-2 specifies derivation and gate restructuring |
| Orchestrator                 | All subagents       | `approval_mode` parameter           | ✅ FR-2.2 tracked in all 8 subagent files          |
| Spec pushback                | spec-output.yaml    | Per-concern `user_response` fields  | ✅ D-7 specifies schema                            |
| Researcher                   | e2e-contract.yaml   | `skill_format` field                | ✅ D-8, schemas.md entry                           |

No contract mismatches or schema inconsistencies detected.

---

## Summary

The R2 design adequately addresses all 5 Major findings from Round 1. The primary architecture finding (A-1: FR-2.4/FR-8.6 contradiction) is fully resolved via DR-2 with the architecturally correct approach — relocating APPROVAL_MODE enforcement to the orchestrator, which already owns that concern. D-14 (minimum step set) strengthens the pipeline's structural integrity. Two Minor residual findings remain: (1) orchestrator at exact budget ceiling with zero headroom, mitigated by a documented extraction fallback; (2) EG-10 restructuring details remain implicit, catchable during implementation or code review. The design is well-structured and ready for planning.
