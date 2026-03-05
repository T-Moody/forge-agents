# Adversarial Review: code — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🟡
- **Round:** 1
- **Run ID:** 2026-03-05T12:00:00Z

## Security Analysis

**Category Verdict:** approve

No security findings. The `fetch_webpage` tool gating (interactive-only across researcher, designer, and spec agents) is consistently enforced in tool-access-matrix.md §3/§4/§5 and each agent's own self-verification guidance. The approval mode safe-default to `interactive` is a security improvement. DB archive rename is properly scope-restricted in tool-access-matrix.md §2.

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: Orchestrator EG-10 description contradicts sql-templates.md (single-source-of-truth violation)

- **Severity:** Major
- **Description:** The orchestrator's verification gate description at line 434 states: `full-tdd-e2e ≥4 (including e2e-test-execution)`. However, sql-templates.md EG-10(full-tdd-e2e) was updated to require ≥3 non-E2E checks only, with E2E verification moved to a separate additive gate EG-10(e2e-additive). The orchestrator's threshold count (4) and included check names are now wrong. Since the orchestrator is an LLM agent that reads its own instructions to determine behavior, the inline description may override the sql-templates.md reference, causing incorrect gate evaluation.
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md#L434), [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L693) (EG-10 full-tdd-e2e)
- **Recommendation:** Update orchestrator line 434 to: `full-tdd-e2e ≥3 passed (baseline, TDD, behavioral coverage). E2E verification handled by additive EG-10(e2e-additive) when e2e_required=true.`
- **Evidence:** Orchestrator line 434: `"full-tdd-e2e ≥4 (including e2e-test-execution)"`. sql-templates.md EG-10(full-tdd-e2e) expected count updated from 4 to 3, with `e2e-test-execution` removed from the check_name IN clause. Design decision D-2 explicitly states "EG-10 restructures to make e2e-test-execution an additive check rather than a lane-specific variant."

### Finding A-2: Orchestrator gate evaluation order missing EG-10(e2e-additive) step

- **Severity:** Major
- **Description:** The orchestrator's gate evaluation order at line 431 states: `EG-1 → EG-10 → EG-7 → EG-8 → EG-9`. The sql-templates.md §6 updated this to: `EG-1 → EG-10 (lane variant) → EG-10(e2e-additive) if applicable → EG-7 → EG-8 → EG-9 → EG-3..EG-6`. The orchestrator is missing the EG-10(e2e-additive) step entirely. Without this, the orchestrator will not evaluate the additive E2E gate for non-🔴 tasks that have `e2e_required=true`, which is the core architectural change of the E2E decoupling feature (D-2).
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md#L431), [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L656)
- **Recommendation:** Update orchestrator line 431 gate evaluation order to: `EG-1 → EG-10 (lane variant) → EG-10(e2e-additive) if applicable → EG-7 → EG-8 → EG-9`.
- **Evidence:** Orchestrator line 431: `"Gate evaluation order: EG-1 → EG-10 → EG-7 → EG-8 → EG-9"`. sql-templates.md line 656: `"EG-1 → EG-10 (lane variant) → EG-10(e2e-additive) if applicable → EG-7 → EG-8 → EG-9 → EG-3..EG-6"`.

### Finding A-3: Orchestrator references non-existent section §11 in sql-templates.md

- **Severity:** Major
- **Description:** Orchestrator line 111 references `[sql-templates.md](sql-templates.md) §11` for the DB archive query. There is no §11 in sql-templates.md — sections go §0 through §9. The correct reference is `§1.1` (Archive Check Query). The notation `§11` (without the dot) will be misinterpreted by an LLM agent as "Section 11" rather than "Section 1.1", causing a failure to locate the archive detection query during Step 0 DB initialization.
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md#L111), [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L199)
- **Recommendation:** Change `§11` to `§1.1` on orchestrator line 111.
- **Evidence:** Orchestrator line 111: `"query per [sql-templates.md](sql-templates.md) §11"`. sql-templates.md section listing: §0, §1, §1.1, §2, §2a, §3, §4, §5, §6, §7, §8, §9. No §11 exists.

### Finding A-4: Spec agent workflow summary uses stale pushback option names

- **Severity:** Minor
- **Description:** Spec.agent.md line 258 (Workflow §2 "Evaluate Pushback") still references the old pushback options: `"proceed / modify / abandon"`. The detailed pushback implementation (lines 147–240) was fully updated to the new per-concern structure with options `"Accept as known risk / Modify scope to address / Dismiss"`. This internal inconsistency within the same file could confuse the spec agent about which option set to use.
- **Affected artifacts:** [spec.agent.md](NewAgents/.github/agents/spec.agent.md#L258)
- **Recommendation:** Update line 258 to: `"If concerns exist, present each as a separate question via ask_questions with per-concern options (accept / modify / dismiss). Wait for user responses and aggregate."` to match the detailed section.
- **Evidence:** Line 258: `"present them via ask_questions with structured multiple-choice options (proceed / modify / abandon)"`. Lines 154–172 define the new per-concern options: `"Accept as known risk"`, `"Modify scope to address"`, `"Dismiss"`.

## Correctness Analysis

**Category Verdict:** needs_revision

### Finding C-1: Orchestrator verification threshold mismatch will reject valid tasks

- **Severity:** Major
- **Description:** A direct consequence of A-1/A-2: if the orchestrator evaluates the full-tdd-e2e gate using its own description (≥4 including e2e-test-execution), it will incorrectly reject tasks that correctly passed 3 non-E2E checks as defined by the updated sql-templates.md. The sql-templates.md queries are the source of truth, but the orchestrator's inline summary text may take precedence in agent reasoning since it is presented as the authoritative gate description without caveats.
- **Affected artifacts:** [orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md#L434), [sql-templates.md](NewAgents/.github/agents/sql-templates.md#L693)
- **Recommendation:** Same fix as A-1/A-2 — align the orchestrator's gate descriptions with sql-templates.md.
- **Evidence:** Orchestrator line 434 threshold: ≥4. sql-templates.md EG-10(full-tdd-e2e) COUNT(\*) check: expected 3. A task with exactly 3 non-E2E passed checks would pass the SQL query but fail the orchestrator's inline threshold check.

### Finding C-2: Spec agent autonomous mode pushback log field evolution not reflected in schemas.md

- **Severity:** Minor
- **Description:** The spec.agent.md pushback_log structure was refactored — `auto_selected` and `selected_option` fields were removed, replaced with per-concern `user_response` and `auto_resolved` fields. However, schemas.md does not define the pushback_log schema anywhere, so there's no formal schema to update. This means the pushback_log format is defined only by example in spec.agent.md, with no single canonical schema. Downstream consumers (orchestrator, knowledge agent) have no formal reference for the new field names.
- **Affected artifacts:** [spec.agent.md](NewAgents/.github/agents/spec.agent.md#L190-L230), [schemas.md](NewAgents/.github/agents/schemas.md)
- **Recommendation:** Consider documenting the `pushback_log` structure in schemas.md as part of Schema 3 (spec-output) in a future iteration. For now, the spec.agent.md examples serve as the de facto schema.
- **Evidence:** spec.agent.md defines two pushback_log variants (autonomous with `auto_resolved` field, interactive with `user_freeform` field). grep of schemas.md for `pushback_log` returns 0 matches — no formal schema exists.

## Summary

The implementation is well-structured — E2E decoupling, approval mode fix, fetch_webpage integration, and per-concern pushback are all architecturally sound. However, the orchestrator's verification gate descriptions (EG-10 threshold and evaluation order) were not updated to match the sql-templates.md changes, creating a single-source-of-truth violation that could cause the orchestrator to reject correctly verified tasks. Additionally, a broken §11/§1.1 cross-reference will prevent the orchestrator from finding the archive query template. These 3 Major findings (A-1, A-2, A-3) all affect the orchestrator file and should be addressed together.
