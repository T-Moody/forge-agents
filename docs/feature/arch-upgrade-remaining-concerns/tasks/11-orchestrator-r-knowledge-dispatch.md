# Task 11: Update Orchestrator r-knowledge Dispatch Row

**Task Goal:** Update the orchestrator's Step 7.2 dispatch table to change r-knowledge's inputs from 6 artifacts to `tier, initial-request.md, memory.md`.

**depends_on:** 08

**agent:** implementer

---

## In-Scope

- Update the r-knowledge row in the Step 7.2 dispatch table in `NewAgentsAndPrompts/orchestrator.agent.md`

## Out-of-Scope

- Modifying other R sub-agent dispatch rows (r-quality, r-security, r-testing — unchanged)
- Modifying `r-knowledge.agent.md` (handled by Tasks 04 and 10)
- Any structural changes to the dispatch table format

---

## Acceptance Criteria

1. The r-knowledge row in Step 7.2 dispatch table lists inputs as `tier, initial-request.md, memory.md`
2. The strings `feature.md`, `design.md`, `plan.md`, `verifier.md` do NOT appear in the r-knowledge row
3. The r-knowledge row outputs remain unchanged: `review/r-knowledge.md` + `review/knowledge-suggestions.md` + `decisions.md`
4. Other R sub-agent rows are unmodified
5. No other files modified

---

## Estimated Effort

Low

---

## Test Requirements

- In Step 7.2 dispatch table, r-knowledge row: inputs include `memory.md` and do NOT include `feature.md`, `design.md`, `plan.md`, `verifier.md`

---

## Implementation Steps

### Step 1: Find the Step 7.2 Dispatch Table (~L332–335)

Locate the R cluster dispatch table in Step 7.2 of `orchestrator.agent.md`. Find the r-knowledge row.

### Step 2: Update r-knowledge Row

**Find (approximately):**

```
| r-knowledge | tier, initial-request.md, feature.md, design.md, plan.md, verifier.md | `review/r-knowledge.md` + `review/knowledge-suggestions.md` + `decisions.md` |
```

**Replace with:**

```
| r-knowledge | tier, initial-request.md, memory.md | `review/r-knowledge.md` + `review/knowledge-suggestions.md` + `decisions.md` |
```

### Step 3: Verify

- Confirm the r-knowledge row uses `memory.md` instead of the 4 removed artifacts
- Confirm other R sub-agent rows are unchanged
- Confirm no other files modified

---

## Completion Checklist

- [x] r-knowledge dispatch row updated — inputs changed to `tier, initial-request.md, memory.md`
- [x] Other R sub-agent rows unmodified
- [x] r-knowledge outputs unchanged
- [x] No other files modified
