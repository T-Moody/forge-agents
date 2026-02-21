# Implementation Plan: Orchestrator Tool Restriction (Phase 1)

## Planning Mode

**Initial** — no prior `plan.md` exists.

## Feature Overview

Restrict the orchestrator agent to 5 tools (`agent`, `agent/runSubagent`, `memory`, `read_file`, `list_dir`) by updating prose and YAML in `orchestrator.agent.md`, fix 5 pre-existing "memory tool" contradictions, correct 8 merge step wordings, update `feature-workflow.prompt.md`, and scope `feature.md` to Phase 1 acceptance criteria.

**Phase 1 scope:** 2 production files modified (`orchestrator.agent.md`, `feature-workflow.prompt.md`) + 1 spec file annotated (`feature.md`). No behavioral changes — all edits are prompt text corrections and restriction declarations.

## Success Criteria (Phase 1 ACs)

Per CT-Strategy finding (Medium): AC-13 and AC-14 are **deferred** — their pass/fail criteria require Phase 2 behavior (memory.md elimination). Phase 1 has **5 acceptance criteria**:

| AC | Summary | Design Section |
|----|---------|----------------|
| AC-1 | YAML `tools:` field present with 5 tools | §2 |
| AC-2 | Operating Rule 5 prose matches YAML exactly | §3.3 |
| AC-3 | Zero occurrences of `grep_search`, `semantic_search`, `file_search`, `get_errors` in allowed-tool contexts | §3.1, §3.2, §3.3, §3.4 |
| AC-11 | `memory` tool disambiguated from pipeline memory.md in all references | §3.1, §3.3, §3.4, §3.5–§3.9 |
| AC-12 | Anti-Drift Anchor reflects restricted tool set with expanded prohibited list | §3.4 |

**Deferred (Phase 2):** AC-4–AC-10, AC-13, AC-14, AC-15.

## Ordered Task Index

| # | Task | File(s) Modified | Design Sections | Effort |
|---|------|-----------------|-----------------|--------|
| 01 | Tool restriction: YAML frontmatter + prose updates + Anti-Drift | `orchestrator.agent.md` | §2, §3.1, §3.2, §3.3, §3.4 | Medium |
| 02 | Memory tool disambiguation + merge step wording corrections | `orchestrator.agent.md` | §3.5, §3.6, §3.7, §3.8, §3.9 | Medium |
| 03 | Feature-workflow prompt update | `feature-workflow.prompt.md` | §4 | Low |
| 04 | Feature spec Phase 1 scope annotation | `feature.md` | §16.5 | Low |

## Execution Waves

### Wave 1 (parallel)

- **01-tool-restriction-prose** (depends_on: none) — Primary tool restriction edits to orchestrator.agent.md
- **03-feature-workflow-update** (depends_on: none) — Separate file, no dependency
- **04-feature-spec-phase1-scope** (depends_on: none) — Separate file, no dependency

### Wave 2

- **02-memory-disambiguation-merge-wording** (depends_on: 01) — Edits same file as Task 01; must run after Task 01 completes to avoid concurrent-edit conflicts on `orchestrator.agent.md`

## Dependency Graph

```
Wave 1:  [01] ──┐     [03]     [04]
                 │
Wave 2:  [02] ◄──┘
```

- 01 → 02: Same-file sequencing (both edit `orchestrator.agent.md`)
- 03: Independent (edits `feature-workflow.prompt.md`)
- 04: Independent (edits `feature.md`)

## Dependencies & Schedule

| Task | depends_on | Blocked by | Estimated Lines Changed |
|------|-----------|------------|------------------------|
| 01 | none | — | ~80 |
| 02 | 01 | Task 01 completion | ~100 |
| 03 | none | — | ~15 |
| 04 | none | — | ~40 |

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Line numbers in design.md may have drifted from actual orchestrator.agent.md | Medium | Low | Design provides exact current text (not just line numbers) — implementer matches on text content, not line numbers |
| AC-3 "zero matches" is strict — prohibited tools appear in the MUST NOT list | Low | Medium | AC-3 applies to allowed-tool contexts only; prohibited-tool mentions in the MUST NOT list are expected. Implementer must verify the distinction. |
| AC-12 pass criteria in feature.md says "does not mention memory.md" but Phase 1 preserves memory.md | Medium | Medium | Task 04 adds Phase 1 interpretation that clarifies AC-12 scope — Anti-Drift updated for tool restriction, memory.md references preserved |
| Merge step wording corrections (Task 02) span 8 locations across 300+ lines | Low | Medium | Design §3.9 provides exact current/new text for each step; implementer uses multi-replace |

## Plan Validation

1. **Circular Dependency Check:** 01→02 is the only dependency chain. No cycles. ✓
2. **Task Size Validation:**
   - 01: 1 file, 0 deps, ~80 lines, Medium ✓
   - 02: 1 file, 1 dep, ~100 lines, Medium ✓
   - 03: 1 file, 0 deps, ~15 lines, Low ✓
   - 04: 1 file, 0 deps, ~40 lines, Low ✓
3. **Dependency Existence Check:** Task 02 depends on Task 01 — exists. ✓

## Pre-Mortem Analysis

| Task | Failure Scenario | Likelihood | Impact | Mitigation |
|------|-----------------|-----------|--------|------------|
| 01 | Anti-Drift Anchor `oldString` doesn't match due to prior edits or whitespace | Medium | Medium | Design provides exact current text; implementer should `read_file` the section first to confirm |
| 02 | One or more of the 8 merge steps has slightly different current text than design specifies | Medium | Low | Each merge step edit is independent; partial success is acceptable and remaining can be fixed in a second pass |
| 03 | feature-workflow.prompt.md structure doesn't match expected location for adding the note | Low | Low | Design §4.3 describes where to add; implementer reads file to find correct insertion point |
| 04 | feature.md AC numbering or structure doesn't match expected format | Low | Low | Implementer reads the AC section and adapts the Phase 1 annotation format |

- **Overall Risk Level:** Low — all changes are prompt text edits with exact before/after specifications in the design document. No behavioral, structural, or code changes.
- **Key Assumptions:**
  1. The orchestrator.agent.md file has not been modified since the design document's line-number references were written. (Tasks 01, 02 depend on this — mitigated by text-matching over line-number-matching.)
  2. The VS Code `tools:` YAML frontmatter field is syntactically valid and accepted by the runtime. (Task 01 depends on this — advisory only, so no runtime failure even if not recognized.)
  3. CT-Strategy's recommendation to defer AC-13/AC-14 is accepted. (Task 04 depends on this — explicitly confirmed in user's planning constraints.)

## Deliverables & Acceptance

**Files created/modified:**
- `.github/agents/orchestrator.agent.md` — all tool restriction edits + memory disambiguation fixes (Tasks 01, 02)
- `.github/prompts/feature-workflow.prompt.md` — tool restriction note added (Task 03)
- `docs/feature/orchestrator-tool-restriction/feature.md` — Phase 1 scope annotation (Task 04)

**Completion:** All 5 Phase 1 ACs (AC-1, AC-2, AC-3, AC-11, AC-12) pass verification. Static tests T-1 through T-13 from design §12 pass.
