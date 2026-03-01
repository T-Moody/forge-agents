# Adversarial Review: design — claude-opus-4.6

## Review Metadata

- **Review Focus:** architecture
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-01T12:00:00Z

## Overall Verdict

**Verdict:** approve
**Confidence:** High

## Findings

### Finding 1: Schema 6 acceptance_criteria type change misclassified under Schema Evolution Strategy

- **Severity:** Major
- **Category:** design
- **Description:** The design changes `task.acceptance_criteria` from `list of strings` (current Schema 6 definition at schemas.md line 469) to `list of objects` with `{id, text, test_method}`. The design acknowledges this is "technically a breaking change for the field type" (design.md §Migration & Backwards Compatibility). However, it classifies the schema bump as `1.0 → 1.1` (minor/additive) and describes all schema changes as "additive with backward-compatible defaults." This directly contradicts two authoritative sources:
  1. **CR-1** (spec-output.yaml): "All schema changes MUST be additive (new optional fields with sensible defaults). No existing field may be removed or **change type**." Priority: `must`.
  2. **Schema Evolution Strategy** (schemas.md lines 1383-1387): Classifies "remove/rename/**retype** field" as **Breaking** → **Unsafe** → "Update ALL listed consumers in dependency table", with version policy requiring major version increment (`1.0 → 2.0`).

  The mitigation (synchronized update of Planner, Implementer, and Verifier — the 3 agents in Schema 6's producer/consumer dependency row at schemas.md line 51) is adequate for preventing runtime failures. But the version classification is architecturally incorrect and sets a governance precedent where breaking changes can be labeled additive.

- **Affected artifacts:** design-output.yaml D-1, design.md §Migration & Backwards Compatibility, schemas.md Schema Evolution Strategy
- **Recommendation:** Reclassify the Schema 6 change as a **breaking** change per the existing evolution strategy. Bump version to `2.0` instead of `1.1`. Document the synchronized update as the required migration plan (which the design already provides — only the version label needs correction). This preserves semantic accuracy of the versioning scheme without any functional change to the implementation plan.
- **Evidence:**
  - schemas.md line 469: `task.acceptance_criteria | list of strings | Yes`
  - schemas.md line 1386: `Breaking (remove/rename/retype field) | Unsafe | All consumers affected`
  - schemas.md line 1393: `Breaking changes: Increment major version (e.g., "1.1" → "2.0")`
  - spec-output.yaml CR-1: "No existing field may be removed or change type"
  - design.md §Migration: "This is technically a breaking change for the field type"
  - design-output.yaml D-1: "Schema version bumps from 1.0 to 1.1 (additive, per evolution strategy)"

### Finding 2: Verifier Tier 2 expansion approaches stated tool-call context bound

- **Severity:** Minor
- **Category:** design
- **Description:** The verifier agent definition (verifier.agent.md line 17) states it is "dispatched once per completed task to bound context to ~8 tool call outputs per dispatch." Currently, a full verification pass involves: Tier 1 (2 checks: ide-diagnostics, syntax-check), Tier 2 (up to 4 checks: build, type-check, lint, tests), plus baseline cross-check — approximately 7 tool calls. The design adds 2 new Tier 2 checks (behavioral-coverage step 3e, runtime-wiring step 3f), bringing the potential total to 9 tool calls for code tasks with new files. While the ~8 bound is a soft guideline rather than a hard limit, the expansion represents a 50% growth of Tier 2 and should be noted for future context budget planning.
- **Affected artifacts:** design-output.yaml D-3, verifier.agent.md line 17
- **Recommendation:** No action required for this design iteration. Monitor verifier dispatch context usage after implementation. If context pressure becomes an issue, consider making runtime-wiring conditional on new-file tasks only at dispatch time (not just at check time), or splitting the verifier into two dispatches for Large tasks.
- **Evidence:**
  - verifier.agent.md line 17: "dispatched once per completed task to bound context to ~8 tool call outputs per dispatch"
  - verifier.agent.md check_name Quick Reference (lines 112-122): 10 existing check_names across 4 tiers
  - design.md §APIs: 2 new check_names (behavioral-coverage, runtime-wiring)

### Finding 3: EG-7 informational gate lacks measurable promotion criteria

- **Severity:** Minor
- **Category:** design
- **Description:** Decision D-8 makes evidence gate EG-7 informational initially, with promotion to required deferred to "follow-up after behavioral checks prove stable (per CR-9 recommendation)." The rationale is sound for initial rollout and pipeline resume compatibility (EC-4). However, no measurable criteria define what "prove stable" means — e.g., number of successful pipeline runs, false positive rate threshold, or time period. Without promotion criteria, EG-7 risks remaining informational indefinitely, weakening the long-term enforcement guarantee that is the feature's purpose.
- **Affected artifacts:** design-output.yaml D-8, design.md §APIs (EG-7 section)
- **Recommendation:** Add a brief note in the design specifying indicative promotion criteria, such as: "Promote EG-7 to required after 3+ pipeline runs with behavioral-coverage signals present on all code tasks and zero false-positive overrides." This need not be binding but provides a measurable target for the knowledge agent or future feature work.
- **Evidence:**
  - design-output.yaml D-8 rationale: "Promotion to required gate deferred to follow-up after behavioral checks prove stable"
  - spec-output.yaml CR-9: "Raising minimum signal thresholds MAY be done as a separate follow-up change"
  - spec-output.yaml FR-5.1: "This query is informational initially (logged but not blocking)"

### Finding 4: Test selection strategy not shared with reviewer evaluation criteria

- **Severity:** Minor
- **Category:** design
- **Description:** Decision D-7 places the test selection strategy inline in the implementer Operating Rules, citing YAGNI (single consumer). However, the adversarial reviewer's correctness analysis (FR-10.1) evaluates whether tests "invoke production code" and whether "test_method='test' acceptance criterion has a corresponding automated test." The reviewer makes qualitative judgments about test appropriateness without access to the implementer's codified strategy (e.g., "do NOT write tests for what the type system guarantees," "public interface only"). This creates an information asymmetry where the reviewer may flag tests that correctly follow the implementer's strategy, or miss tests that violate it.
- **Affected artifacts:** design-output.yaml D-7, design.md §Tradeoffs D-7
- **Recommendation:** No action required for this iteration — the YAGNI argument holds for the initial scope. If reviewer false positives emerge in practice (reviewer flagging strategy-compliant tests), consider extracting the test selection strategy into a lightweight shared reference or adding a brief cross-reference from the reviewer's correctness guidance to the implementer's strategy section.
- **Evidence:**
  - design-output.yaml D-7: "Single consumer (implementer), so a shared reference document is unnecessary"
  - spec-output.yaml FR-10.1: reviewer must verify "tests invoke production code" and AC coverage
  - spec-output.yaml FR-9.1-9.3: detailed test selection rules for implementer
  - design.md §High-Level Architecture: reviewer gets "Correctness guidance: self-referential detection, AC coverage, wiring" (separate from test selection strategy)

## Summary

The design is architecturally sound. Direction A (structural enforcement within existing architecture) is the correct choice — it preserves the pipeline's structure while adding layered behavioral enforcement across 4 agents (planner, implementer, verifier, reviewer). The 8-file modification scope is manageable because each change is small (10-46 lines), additive, and follows existing agent patterns (self-verification checklists, Tier 2 checks, check_name conventions). Agent boundary responsibilities are preserved — no agent is given duties outside its pipeline role. The synchronized update order is well-specified. The one significant architectural issue is the Schema 6 type change being misclassified as additive when it is breaking per the codebase's own evolution strategy; this requires correcting the version classification from 1.1 to 2.0 but does not affect the implementation approach.
