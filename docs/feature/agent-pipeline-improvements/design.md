# Agent Pipeline Improvements — Technical Design

## Title & Summary

**Feature:** Agent Pipeline Improvements  
**Designer:** designer  
**Date:** 2026-03-05  
**Risk Level:** 🟡 (business logic modifications across 14 files + 1 new)

**Revision Round:** 2 (addressing 5 Major + 8 Minor findings from Round 1 adversarial review)

Eight targeted improvements to the multi-agent pipeline system addressing E2E test gating, approval mode confusion, web fetching capability, terminal redirect enforcement, spec pushback granularity, Copilot Skills integration, SQLite lifecycle management, and fast-track pipeline entry. All changes are to markdown instruction files and YAML schemas — no runtime code.

### R2 Revision Summary

| Finding                                      | Reviewer              | Resolution                                                                                                                                         |
| -------------------------------------------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| S-1: Anti-drift anchor enables Step 7 bypass | security-sentinel     | D-14: Immutable minimum step set {Step 0, Step 7, Step 9} added to anti-drift anchor                                                               |
| A-1: FR-2.4/FR-8.6 spec contradiction        | architecture-guardian | DR-2: Relocated APPROVAL_MODE clarification to orchestrator Step 0; feature-workflow.prompt.md no longer modified                                  |
| PV-A1: Orchestrator baseline 353 not 539     | pragmatic-verifier    | Re-verified: line 539 contains anti-drift anchor (confirmed via direct file read + research/impact.yaml line 410). 353-line measurement incorrect. |
| PV-C1: FR-2.2 per-agent docs missing         | pragmatic-verifier    | All 8 subagent files now include FR-2.2 (approval_mode parameter docs). adversarial-reviewer and knowledge-agent added to inventory.               |
| PV-C2: Dual never-skip-steps gap             | pragmatic-verifier    | Both line 29 and line 539 now amended to same conditional language with minimum step set.                                                          |

---

## Context & Inputs

### Source Artifacts

| Artifact                                                 | Key Content                                                                              |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| [initial-request.md](initial-request.md)                 | 8 user-identified issues from hands-on pipeline experience                               |
| [spec-output.yaml](spec-output.yaml)                     | 38 sub-requirements (FR-1 through FR-8), 22 acceptance criteria, 13 edge cases           |
| [research/architecture.yaml](research/architecture.yaml) | 19-file system structure, 10-step pipeline, E2E triple-gating, approval mode propagation |
| [research/impact.yaml](research/impact.yaml)             | 12/20 files affected, orchestrator at 539/550 lines, EG-10 restructuring options         |
| [research/dependencies.yaml](research/dependencies.yaml) | Cross-reference graph, E2E 7-file cascade, ordering constraints                          |
| [research/patterns.yaml](research/patterns.yaml)         | Uniform completion contracts, anti-drift anchors, no-redirect enforcement gap            |

### Key Constraints

- Orchestrator line budget: 539/550 (NFR-1 exception, re-verified R2: line 539 is anti-drift anchor)
- All changes are markdown/YAML instruction files — no runtime code
- Global immutable safety rules (global-operating-rules.md §9) cannot be weakened
- Existing full pipeline (feature-workflow.prompt.md) must remain functional and unmodified (FR-8.6)
- Tool access enforcement is instruction-based only (no runtime enforcement)
- All pipeline prompts must include minimum step set {Step 0, Step 7, Step 9} (D-14)

---

## High-Level Architecture

### Direction: Hybrid (D-1)

**Selected:** Direction C — Extract complex, inline simple.

Complex additions (fast-track pipeline mode detection, E2E decoupling, DB archive) use compact inline additions to the orchestrator with logic kept minimal. Simple changes (approval mode language, no-redirect, typo fixes) are edited in-place. The fast-track step set is defined in the new prompt file, not duplicated in the orchestrator.

**Rejected alternatives:**

- **Direction A (all in-place):** Orchestrator cannot absorb all additions within 550-line budget
- **Direction B (all reference-doc extraction):** Over-engineers 2-line text fixes with unnecessary indirection

### File Impact Summary

| Category                   | Files                                                                          | Risk             |
| -------------------------- | ------------------------------------------------------------------------------ | ---------------- |
| New prompt                 | plan-and-implement.prompt.md                                                   | 🟢               |
| Core agents modified       | orchestrator, planner, verifier, implementer, spec, researcher, designer       | 🟢–🔴            |
| Additional agents modified | adversarial-reviewer, knowledge-agent (FR-2.2 only)                            | 🟢               |
| Reference docs modified    | tool-access-matrix, sql-templates, schemas, e2e-integration, dispatch-patterns | 🟡               |
| **Total**                  | **15 files (1 new + 14 modified)**                                             | **🟡 aggregate** |

---

## Data Models & DTOs

### Schema 6 (task-schema) — e2e_required Field Update (D-2)

**Current:** `e2e_required` is derived deterministically from `workflow_lane`:

```
🟢 → unit-only → e2e_required=false
🟡 → unit-integration → e2e_required=false
🔴 → full-tdd-e2e → e2e_required=true (if contract exists)
```

**New:** `e2e_required` is an independent boolean derived from:

1. **Contract presence:** Does e2e-contract.yaml exist in the project?
2. **Planner assessment:** Does E2E add verification value for this specific task?
3. **Risk override:** For 🔴 tasks, e2e_required=true is mandatory when a contract exists.

```
workflow_lane continues to follow risk:
  🟢 → unit-only
  🟡 → unit-integration
  🔴 → full-tdd-e2e

e2e_required is independent:
  No contract       → false (regardless of risk)
  Contract + 🔴     → true (mandatory)
  Contract + 🟡/🟢  → planner decides (opt-in)
```

### Schema 13 (e2e-contract) — skill_format Field (D-8)

New field added: `skill_format: "playwright-yaml" | "copilot-skill"`

- **playwright-yaml** (existing): Custom YAML skill files translated to Playwright CLI commands
- **copilot-skill** (new): Markdown-based SKILL.md files using copilot-skill:// URI pattern, converted to Playwright CLI interaction steps by the Verifier

### Pushback Log Schema — Per-Concern Responses (D-7)

Updated pushback_log structure in spec-output.yaml:

```yaml
pushback_log:
  concerns_identified: N
  mode: "interactive" | "autonomous"
  concerns:
    - severity: "Major"
      title: "<concern title>"
      user_response: "accept" | "modify" | "dismiss"
      user_direction: "<freeform text if modify>"  # only present for "modify"
```

---

## APIs & Interfaces

### Tool Access Matrix Changes (D-5, D-12)

#### New Row: fetch_webpage

| Agent                | Access | Scope                                                                                                        |
| -------------------- | ------ | ------------------------------------------------------------------------------------------------------------ |
| Orchestrator         | ❌     | —                                                                                                            |
| Researcher           | 🔒     | Interactive mode only. Research architecture patterns, stack updates, best practices, current documentation. |
| Spec                 | 🔒     | Interactive mode only. Research requirement feasibility, current framework capabilities.                     |
| Designer             | 🔒     | Interactive mode only. Stack research, design pattern lookup. Complements Context7 (library API docs).       |
| Planner              | ❌     | —                                                                                                            |
| Implementer          | ❌     | —                                                                                                            |
| Verifier             | ❌     | —                                                                                                            |
| Adversarial Reviewer | ❌     | —                                                                                                            |
| Knowledge Agent      | ❌     | —                                                                                                            |

#### Modified Row: ask_questions

| Agent   | Old | New | Scope                                                                                                            |
| ------- | --- | --- | ---------------------------------------------------------------------------------------------------------------- |
| Planner | ❌  | 🔒  | Fast-track mode AND interactive mode only. Used to ask clarifying questions when spec/design outputs are absent. |

#### Modified: Orchestrator run_in_terminal scope (§2)

Current scope: "SQLite queries (SELECT, DDL) + git operations only."  
New scope: "SQLite queries (SELECT, DDL) + git operations + verification-ledger.db archive rename."

### Fast-Track Prompt Interface (D-10, D-13)

**File:** `NewAgents/.github/prompts/plan-and-implement.prompt.md`

```yaml
---
name: "Plan & Implement (Fast-Track)"
agent: "orchestrator"
---
```

**Pipeline variables passed:**

- `pipeline_mode: "fast-track"` — triggers orchestrator to skip Steps 1-3b
- `{{USER_FEATURE}}` — standard feature description variable

**Step set:** Step 0 → Step 4 → Steps 5-6 (loop) → Step 7 → Step 8 → Step 9

**Orchestrator detection:** In Step 0, check if `pipeline_mode` is "fast-track". If yes, after completing Step 0 initialization, jump to Step 4 (skipping Steps 1, 1a, 2, 3, 3b).

---

## Sequence / Interaction Notes

### Issue #1: E2E Decoupling Flow

```
Planner → derives workflow_lane from risk (unchanged)
Planner → derives e2e_required independently:
  ├─ No contract? → e2e_required=false
  ├─ Contract + 🔴? → e2e_required=true (mandatory)
  └─ Contract + 🟢/🟡? → planner assesses value → true/false
Orchestrator → EG-10 checks by lane (unchanged for non-E2E checks)
Orchestrator → EG-9 checks e2e_required=true tasks independently
Verifier → Tier 5 gate: e2e_required=true (removed lane condition)
```

### Issue #7: DB Archive Sequence at Step 0

```
Step 0 initialization:
1. Check: does verification-ledger.db exist?
   ├─ No → proceed to DDL (fresh install)
   └─ Yes → query: SELECT DISTINCT run_id FROM pipeline_telemetry LIMIT 1
      ├─ Query fails (corrupt/missing table) → archive with '-corrupt' suffix
      ├─ Same run_id as current → keep DB (resume case)
      └─ Different run_id → archive with timestamp suffix
2. Archive: Rename-Item verification-ledger.db → verification-ledger-{ISO8601}.db
3. Create fresh DB + run standard DDL from sql-templates.md §1
4. Continue with existing PRAGMA + integrity_check
```

### Issue #8: Fast-Track Pipeline Flow

```
User selects plan-and-implement.prompt.md
  → Orchestrator receives pipeline_mode=fast-track
  → Step 0: normal init (approval mode, DB archive, etc.)
  → Step 0: detect fast-track → set steps_to_skip=[1, 1a, 2, 3, 3b]
  → Step 4: Planner dispatched
    → Planner detects missing spec-output.yaml + design-output.yaml
    → In interactive mode: uses ask_questions to gather requirements
    → In autonomous mode (EC-14): works from initial-request.md only, lower confidence
    → Planner produces plan-output.yaml as normal
  Note: In fast-track mode, Step 4 input is initial-request.md (NOT spec/design artifacts).
  → Steps 5-6: Implementation + Verification loop (unchanged)
  → Step 7: Code review (unchanged)
  → Step 8: Knowledge capture (unchanged)
  → Step 9: Auto-commit (unchanged)
```

### Issue #5: Spec Pushback Per-Concern Flow

```
Spec agent identifies N concerns → builds questions[] array:
  For each concern i:
    questions[i] = {
      header: "Concern {i}: {severity} — {title}",
      question: "{description}",
      options: [
        { label: "Accept as known risk — proceed with this concern noted" },
        { label: "Modify scope to address", description: "Provide direction" },
        { label: "Dismiss — this is not a real concern" }
      ],
      allowFreeformInput: true  # enables freeform text for all options on this question;
                                  # spec agent processes freeform only when "modify" is selected
    }

Interactive mode: call ask_questions with questions[] array
Autonomous mode: auto-resolve each as "Accept as known risk"

Aggregate results:
  All accepted/dismissed → proceed to specification
  Any "modify" → incorporate user direction → re-evaluate
```

---

## Security Considerations

### fetch_webpage Risk (D-5)

- **Threat:** Agents could fetch arbitrary URLs, potentially leaking context or consuming inappropriate content.
- **Mitigation:** Interactive-mode-only restriction means every fetch requires explicit user approval in VS Code. The scope restriction documentation specifies approved use cases (research patterns, stack docs, best practices). Self-verification checklist item confirms no autonomous-mode usage.
- **Residual risk:** Instruction-based enforcement only (accepted system-wide per tool-access-matrix.md §11).

### DB Archive Rename (D-9)

- **Threat:** Archive rename could theoretically target non-DB files if the file path is manipulated.
- **Mitigation:** The orchestrator's expanded run_in_terminal scope is restricted to "verification-ledger.db archive rename" — a specific file, not a general rename capability. The rename target path is deterministic (timestamp-based suffix on the same filename).

### Fast-Track Step Skipping (D-10, D-14)

- **Threat:** Step skipping could bypass research/design that reveals security concerns. More critically, a future prompt could define a reduced step set that omits Step 7, bypassing all security review.
- **Mitigation:** Fast-track is user-initiated (explicit prompt selection). Code review (Step 7) is preserved in fast-track, providing a security review checkpoint. The amended anti-drift anchor now includes both a conditional step-skip clause AND an immutable minimum step set: "All pipeline prompts MUST include at minimum: Step 0 (initialization), Step 7 (code review), and Step 9 (commit)." This prevents any future prompt from bypassing security review. Both line 29 (Role & Purpose) and line 539 (anti-drift anchor) are amended to this language, preventing contradiction.
- **Minimum step set rationale (D-14):** Step 0 ensures initialization and approval mode selection. Step 7 ensures all code changes are reviewed by adversarial reviewers including the security-sentinel. Step 9 ensures proper git commit with evidence. These three steps form the irreducible security backbone of any pipeline execution.

---

## Failure & Recovery

| Failure Mode                                            | Detection                                                  | Recovery                                                                                                                        |
| ------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| DB archive rename fails (file locked)                   | run_in_terminal returns non-zero exit code                 | Retry once after 1s delay. If still fails, proceed with existing DB (log warning).                                              |
| DB archive query fails (corrupt DB)                     | SQL query returns error                                    | Archive unconditionally with '-corrupt' suffix (EC-9).                                                                          |
| fetch_webpage returns error/timeout                     | Tool returns error response                                | Degrade gracefully — proceed without web research, note gap in output (EC-5).                                                   |
| Planner ask_questions in fast-track gets no response    | ask_questions timeout or empty response                    | Fall back to initial-request.md content only, with lower confidence in plan (log warning).                                      |
| E2E contract missing for 🔴 task                        | Orchestrator Step 0 contract discovery finds no file       | Interactive: prompt user to provide contract (existing behavior preserved). Autonomous: e2e_required=false with warning (EC-2). |
| Copilot skill file found but Playwright CLI unavailable | Verifier Tier 5 cannot execute Playwright commands         | Set e2e verification to "deferred", report skill found but execution unavailable (EC-12).                                       |
| Fast-track + autonomous mode (EC-14)                    | Planner has no upstream artifacts AND cannot ask questions | Plan derived solely from initial-request.md with lower confidence. plan-output.yaml SHOULD indicate reduced planning context.   |
| Orchestrator exceeds 550 lines after changes            | Line count audit during implementation                     | Extract lowest-priority content to reference document. DB archive procedure is first extraction candidate (~8 lines).           |

---

## Non-Functional Requirements

| NFR                                   | Approach                                                                                                                                                                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| NFR-1: Orchestrator ≤550 lines        | Balanced add/remove: -8 lines (removed language) + ~19 lines (added logic) = ~550/550 (D-11). Re-verified R2: baseline is 539 (not 353). At budget ceiling; DB archive procedure is first extraction candidate if needed. |
| NFR-2: Markdown/YAML only             | All changes are to .md and .yaml files. No runtime code.                                                                                                                                                                  |
| NFR-3: Instruction-based enforcement  | fetch_webpage mode restriction follows existing 🔒 pattern (D-5, DR-1)                                                                                                                                                    |
| NFR-4: Updated self-verification      | Each modified agent updates its checklist (tracked per file in implementation)                                                                                                                                            |
| NFR-5: Fast-track ≥50% time reduction | Skipping Steps 1-3b (4 research + spec + design + 3 reviewers = ~8 agent dispatches) saves significant wall-clock time                                                                                                    |

---

## Migration & Backwards Compatibility

- **Full pipeline (feature-workflow.prompt.md):** Unaffected. All 10 steps continue to execute. FR-8.6 honored: feature-workflow.prompt.md is NOT modified (DR-2 — APPROVAL_MODE clarification relocated to orchestrator Step 0).
- **E2E for existing 🔴 workflows:** Backward compatible. 🔴 tasks with contracts still get e2e_required=true (mandatory). Lane assignment unchanged.
- **Existing DBs:** Archived (renamed with timestamp), not deleted. Evidence is preserved.
- **Autonomous mode users:** Approval mode fix (D-4) doesn't break autonomous — users who explicitly set `APPROVAL_MODE: autonomous` will still get autonomous. Only the "no mode specified" fallback changes (from autonomous to interactive).
- **Existing E2E skills:** playwright-yaml format continues to work. New copilot-skill format is additive.

---

## Testing Strategy

All acceptance criteria use inspection-based testing (reading modified files and verifying content). Two criteria use grep-based tests:

| Test                                    | Method                                                                 | Criteria                                                            |
| --------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------- |
| AC-5: No "default to autonomous" phrase | `grep -r "default to autonomous" NewAgents/.github/` returns 0 matches | Pass if 0 matches                                                   |
| AC-21: Orchestrator ≤550 lines          | `wc -l NewAgents/.github/agents/orchestrator.agent.md` ≤ 550           | Pass if ≤550                                                        |
| All other ACs (1-4, 6-20, 22)           | File content inspection                                                | Verifier reads modified files and confirms required content present |

---

## Tradeoffs & Alternatives Considered

### D-2: E2E Decoupling Strategy

| Criterion       | Independent boolean (selected)       | New lane values                              | Remove lanes entirely                  |
| --------------- | ------------------------------------ | -------------------------------------------- | -------------------------------------- |
| Complexity      | Low — add one independent field      | High — 6 lane variants × EG-10 queries       | Medium — fundamental restructure       |
| Backward compat | Full — lanes unchanged               | Partial — new lanes need downstream handling | Breaking — removes existing lane logic |
| EG-10 impact    | Minimal — e2e check becomes additive | Major — 6 new SQL variants                   | Major — lane-based queries removed     |
| Planner change  | Small — add assessment logic         | Medium — new derivation rows                 | Large — new derivation model           |

### D-10: Fast-Track Entry Point

| Criterion                 | Separate prompt (selected)          | Dynamic scope detection          |
| ------------------------- | ----------------------------------- | -------------------------------- |
| Clarity                   | High — explicit step set per prompt | Low — heuristic assessment       |
| Implementation complexity | Low — new file + 5-line detection   | High — scope rules engine        |
| Safety                    | High — user explicitly chooses      | Medium — could mis-classify      |
| User control              | Full — user picks prompt            | None — orchestrator decides      |
| Anti-drift risk           | Low — scoped to fast-track prompt   | High — could erode full pipeline |

### D-11: Orchestrator Budget Management

| Criterion            | Balanced add/remove (selected) | Extract to ref doc            | Increase budget          |
| -------------------- | ------------------------------ | ----------------------------- | ------------------------ |
| File count           | 0 new files                    | +1 reference doc              | 0 new files              |
| Orchestrator clarity | Good — all logic visible       | Reduced — cross-reference     | Good — all logic visible |
| Maintenance          | Low                            | Medium — keep ref doc in sync | Low                      |
| Budget headroom      | ~2 lines remaining             | ~15+ lines recovered          | ~50+ lines               |

---

## Implementation Checklist & Deliverables

### Wave 1: Foundation (schemas, reference docs, tool matrix)

These changes establish the data model and tool access foundation upon which agent-level changes depend.

| #   | File                  | Changes                                                                       | Risk | Addresses                      |
| --- | --------------------- | ----------------------------------------------------------------------------- | ---- | ------------------------------ |
| 1   | tool-access-matrix.md | Add fetch_webpage row, Planner ask_questions 🔒, orchestrator scope expansion | 🟡   | FR-3.1, FR-3.2, FR-7.5, FR-8.3 |
| 2   | schemas.md            | Update task.e2e_required definition, add skill_format to e2e-contract         | 🟡   | FR-1.6, FR-6.1                 |
| 3   | sql-templates.md      | Restructure EG-10 e2e check as additive, add archive query                    | 🟡   | FR-1.5, FR-7.2                 |

### Wave 2: Core agent modifications

Changes to agent files that have the widest behavioral impact.

| #   | File                  | Changes                                                                                                                                                                                                                                     | Risk | Addresses                                      |
| --- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | ---------------------------------------------- |
| 4   | orchestrator.agent.md | Approval mode rewrite (2 occurrences at lines 105/109 + line 24), DB archive logic, fast-track detection, E2E gate update, typo fixes, anti-drift amendment (line 539) + line 29 amendment, minimum step set, APPROVAL_MODE priority (DR-2) | 🔴   | FR-1.4, FR-2.1-2.5, FR-7.1-7.4, FR-8.2, FR-8.5 |
| 5   | planner.agent.md      | E2E decoupling in lane derivation, fast-track fallback input, ask_questions workflow, self-verification update, add approval_mode to Orchestrator-Provided Parameters                                                                       | 🟡   | FR-1.1-1.2, FR-1.7-1.8, FR-2.2, FR-8.3-8.4     |
| 6   | verifier.agent.md     | Tier 5 gate simplification, no-redirect rule + anti-drift, copilot-skill format support, add approval_mode to Orchestrator-Provided Parameters                                                                                              | 🟡   | FR-1.3, FR-2.2, FR-4.2, FR-4.4, FR-6.3         |
| 7   | spec.agent.md         | Per-concern pushback restructure, fetch_webpage tool addition, add approval_mode to Orchestrator-Provided Parameters                                                                                                                        | 🟡   | FR-2.2, FR-3.5, FR-5.1-5.5                     |

### Wave 3: Extended agent + reference doc updates

| #   | File                          | Changes                                                                                                     | Risk | Addresses              |
| --- | ----------------------------- | ----------------------------------------------------------------------------------------------------------- | ---- | ---------------------- |
| 8   | implementer.agent.md          | No-redirect rule + anti-drift anchor, add approval_mode to Orchestrator-Provided Parameters                 | 🟢   | FR-2.2, FR-4.1, FR-4.3 |
| 9   | researcher.agent.md           | fetch_webpage tool addition, Copilot skill discovery, add approval_mode to Orchestrator-Provided Parameters | 🟢   | FR-2.2, FR-3.3, FR-6.2 |
| 10  | designer.agent.md             | fetch_webpage tool addition, Context7 disambiguation, add approval_mode to Orchestrator-Provided Parameters | 🟢   | FR-2.2, FR-3.4         |
| 11  | e2e-integration.md            | E2E applicability update, copilot-skill format in §2                                                        | 🟡   | FR-1.9, FR-6.1, FR-6.5 |
| 12  | dispatch-patterns.md          | E2E constraints apply when e2e_required=true regardless of risk                                             | 🟢   | FR-1.10                |
| 13  | adversarial-reviewer.agent.md | Add approval_mode to Orchestrator-Provided Parameters                                                       | 🟢   | FR-2.2                 |
| 14  | knowledge-agent.agent.md      | Add approval_mode to Orchestrator-Provided Parameters                                                       | 🟢   | FR-2.2                 |

### Wave 4: Prompt files

| #   | File                               | Changes                                 | Risk | Addresses              |
| --- | ---------------------------------- | --------------------------------------- | ---- | ---------------------- |
| 15  | plan-and-implement.prompt.md (new) | Fast-track prompt with reduced step set | 🟢   | FR-8.1, FR-8.2, FR-8.6 |

### Acceptance Criteria Coverage

| AC    | Requirement                                            | Covered By                    |
| ----- | ------------------------------------------------------ | ----------------------------- |
| AC-1  | 🟢 task with contract can have e2e_required=true       | D-2, D-3 (Tasks 2, 5)         |
| AC-2  | 🟢 task can have e2e_required=false with contract      | D-3 (Task 5)                  |
| AC-3  | 🔴 task with contract always has e2e_required=true     | D-3 (Task 5)                  |
| AC-4  | APPROVAL_MODE from initial-request.md is authoritative | D-4 (Task 4)                  |
| AC-5  | No "default to autonomous" phrase in system            | D-4 (Task 4)                  |
| AC-6  | fetch_webpage in tool-access-matrix.md §1              | D-5 (Task 1)                  |
| AC-7  | fetch_webpage docs in researcher, designer, spec       | D-5 (Tasks 7, 9, 10)          |
| AC-8  | No-redirect rule in implementer.agent.md               | D-6 (Task 8)                  |
| AC-9  | No-redirect rule in verifier.agent.md                  | D-6 (Task 6)                  |
| AC-10 | Anti-drift anchors include redirect prohibition        | D-6 (Tasks 6, 8)              |
| AC-11 | Per-concern ask_questions in spec pushback             | D-7 (Task 7)                  |
| AC-12 | Per-concern user_response in pushback_log              | D-7 (Task 7)                  |
| AC-13 | skill_format field in e2e-integration.md §2            | D-8 (Task 11)                 |
| AC-14 | DB archive on different run_id                         | D-9 (Task 4)                  |
| AC-15 | Orchestrator scope includes archive rename             | D-9 (Tasks 1, 4)              |
| AC-16 | plan-and-implement.prompt.md exists                    | D-10 (Task 13)                |
| AC-17 | Planner ask_questions 🔒 in tool matrix                | D-12 (Task 1)                 |
| AC-18 | Planner fast-track fallback input path                 | D-12 (Task 5)                 |
| AC-19 | Orchestrator anti-drift amended for fast-track         | D-10, D-14 (Task 4)           |
| AC-20 | feature-workflow.prompt.md structurally unmodified     | DR-2 (no modification needed) |
| AC-21 | Orchestrator ≤550 lines                                | D-11 (Task 4)                 |
| AC-22 | Modified agents have updated self-verification         | All tasks                     |

---

## Decision Record Summary

| ID   | Title                                                        | Risk | Confidence | Key Rationale                                                                                                                                              |
| ---- | ------------------------------------------------------------ | ---- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D-1  | Direction: Hybrid (extract complex, inline simple)           | 🟡   | High       | Orchestrator budget constraint (539/550, re-verified R2); simple fixes don't need extraction                                                               |
| D-2  | E2E decoupling: independent boolean                          | 🟡   | High       | Clean factoring; lanes for test tiers, e2e_required for E2E gating                                                                                         |
| D-3  | E2E opt-in for 🟢/🟡, mandatory for 🔴                       | 🟢   | High       | Avoids pipeline bloat; aligns with spec pushback concern                                                                                                   |
| D-4  | Approval mode: user authoritative, interactive safe default  | 🟢   | High       | Flips bias (2 occurrences at lines 105/109) without breaking explicit autonomous users                                                                     |
| D-5  | fetch_webpage: Researcher + Designer + Spec (🔒 interactive) | 🟢   | High       | Three research-oriented agents; follows existing 🔒 pattern                                                                                                |
| D-6  | No-redirect: operating rules + anti-drift anchors            | 🟢   | High       | Global rule insufficient; agent-level reinforcement needed                                                                                                 |
| D-7  | Spec pushback: per-concern in single ask_questions           | 🟢   | High       | API supports questions[] array; optimal UX. allowFreeformInput is per-question (available for all options), spec agent processes freeform only on "modify" |
| D-8  | Copilot Skills: layered on Playwright                        | 🟡   | Medium     | Skills describe WHAT; Playwright is HOW. Full integration depth depends on format research during implementation                                           |
| D-9  | SQLite archive: run_id mismatch rename                       | 🟢   | High       | User rejects migration; deterministic and simple                                                                                                           |
| D-10 | Fast-track: separate prompt file + minimum step set          | 🟡   | High       | Explicit user control; avoids heuristic step skipping; R2: minimum step set prevents Step 7 bypass                                                         |
| D-11 | Orchestrator budget: balanced add/remove (~550/550)          | 🟡   | Medium     | At budget ceiling (539 baseline, re-verified R2). DB archive procedure is first extraction candidate if needed.                                            |
| D-12 | Planner ask_questions: 🔒 fast-track + interactive           | 🟢   | High       | Double restriction prevents prompting in full pipeline or autonomous mode                                                                                  |
| D-13 | Fast-track steps: 0 → 4 → 5-6 → 7 → 8 → 9                    | 🟢   | High       | User wants planning start; code review preserved for safety; Step 8 included per FR-8.7 (DR-3)                                                             |
| D-14 | Minimum required step set: Step 0 + Step 7 + Step 9          | 🟢   | High       | R2 (S-1): Immutable constraint prevents future prompts from bypassing security review                                                                      |

**D-8 (Medium confidence):** The Copilot Skills format integration depth depends on actual SKILL.md format research. The design specifies the layering approach (skills describe WHAT, Playwright is HOW) but implementation may need to adjust based on discovered format details. Evidence that would raise confidence: successful web research into current Copilot Skills format during implementation.

**D-11 (Medium confidence):** The orchestrator budget calculation is based on the re-verified 539-line baseline (R2 confirmation). If implementation reveals the changes require more lines than estimated, content extraction to a reference document will be needed. The DB archive procedure (~8 lines) is the first extraction candidate. Evidence that would raise confidence: verified line counts after implementing Task 4.
