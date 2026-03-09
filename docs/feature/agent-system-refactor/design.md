# Agent System Refactor — Technical Design (Run 3: Targeted Improvements, R2)

## Title & Summary

**Feature:** Targeted Improvements to V2 Multi-Agent Pipeline
**Design Decision Count:** 11 architectural decisions + 2 deviation records
**Run:** 3 (2026-03-09) — implements 10 functional requirements from spec-output.yaml
**Revision Round:** 2 — addresses 1 Critical + 5 Major + 5 Minor findings from architecture-guardian, pragmatic-verifier, and security-sentinel
**Overall Risk:** 🟡 (business logic modification — correcting tool names, git workflow, TDD cycle)

Hybrid targeted modifications (Direction C) across 9 v2 files implementing all 10 improvements. Critical fixes (tool names, git safety, TDD refactor step, exploratory QA) applied inline per agent; shared git safety rule extracted to global-rules.md. No new agent files created. All agents remain under 150-line budgets (architect is tightest at ~145/150 with 5-line headroom — verified line counts). Interactive questioning uses orchestrator-mediated assumptions-first pattern since VS Code subagents cannot directly prompt users. Step 8 decomposed into 5 explicit sub-steps (8a-8e). Git staging uses two-source strategy.

---

## Context & Inputs

| Input                 | Source                                                                                                     |
| --------------------- | ---------------------------------------------------------------------------------------------------------- |
| spec-output.yaml      | Spec Agent — 10 FRs, 27 ACs                                                                                |
| initial-request.md    | User — original feature request                                                                            |
| 5 research YAML files | Researcher — architecture, impact, dependencies, patterns, tool-names-deep-dive                            |
| 8 v2 agent files      | Current v2 system (orchestrator, researcher, architect, planner, implementer, tester, reviewer, knowledge) |
| global-rules.md       | Shared rules document                                                                                      |
| 2 prompt files        | feature-workflow.prompt.md, quick-fix.prompt.md                                                            |
| VS Code official docs | Custom agents, subagents, skills, hooks, cheat sheet (fetched 2026-03-09)                                  |
| R3 review verdicts    | 3 review verdicts from prior design run (2026-03-08)                                                       |

### Key Discovery: Tool Name Crisis

All v2 tool names use invalid `namespace/tool` format (e.g., `read/readFile`, `web/fetch`). VS Code requires flat camelCase names (e.g., `readFile`, `fetch`). Invalid names are **silently ignored** by VS Code, meaning ALL tool restrictions are currently non-functional. This is the highest-priority fix (FR-2).

### Key Discovery: Subagent Isolation

VS Code subagents run in isolated contexts and **cannot directly prompt users**. No built-in "structured question tool" exists. This means architect/planner clarifying questions (FR-3, FR-4) must use an orchestrator-mediated pattern.

---

## High-Level Architecture

**Approach:** Direction C — Hybrid inline + shared extraction (D-1)

- 9 files modified in-place (8 agents + global-rules.md)
- 0 new files created
- 2 prompt files unchanged
- Single shared extraction: git safety rule → global-rules.md
- All other changes inline per agent

```
v2/.github/
├── agents/
│   ├── orchestrator.agent.md  (117→142 lines) — pipeline coordinator
│   ├── researcher.agent.md    (83→84 lines)   — codebase + web research
│   ├── architect.agent.md     (130→145 lines)  — spec + design [🔴 TIGHTEST]
│   ├── planner.agent.md       (105→116 lines)  — DAG task decomposition
│   ├── implementer.agent.md   (99→115 lines)   — TDD implementation
│   ├── tester.agent.md        (129→143 lines)  — testing + exploratory QA
│   ├── reviewer.agent.md      (96→99 lines)    — adversarial review
│   ├── knowledge.agent.md     (107→118 lines)  — post-mortem + doc updates
│   └── global-rules.md        (70→80 lines)    — shared rules
└── prompts/
    ├── feature-workflow.prompt.md (28 lines, unchanged)
    └── quick-fix.prompt.md        (22 lines, unchanged)
```

Total system: 936 → 1,042 lines (106 lines added across all files). Line counts verified via PowerShell `(Get-Content).Count` on 2026-03-09.

---

## Design Decisions

### D-1: Direction C — Hybrid inline + shared extraction (🟢, High)

**Context:** Three directions for implementing 10 improvements to v2.

| Alternative               | Pros                                                   | Cons                               |
| ------------------------- | ------------------------------------------------------ | ---------------------------------- |
| A — Targeted In-Place     | Minimal coupling, self-contained                       | Git safety rule duplicated         |
| B — Targeted + Hooks      | Deterministic enforcement                              | Preview feature, scope creep       |
| **C — Hybrid (SELECTED)** | **Single source of truth for git safety, lean agents** | **global-rules.md grows ~8 lines** |

**Rationale:** Direction C places git safety rule in global-rules.md (single source of truth) while keeping all other changes inline. Practical difference from A is minimal. Direction B adds Preview-feature infrastructure that exceeds scope.

---

### D-2: Individual flat camelCase tool names (🟢, High)

**Context:** V2 uses invalid namespace/tool format. VS Code supports individual names and tool sets.

| Alternative                     | Pros                                          | Cons                       |
| ------------------------------- | --------------------------------------------- | -------------------------- |
| **Individual names (SELECTED)** | **Fine-grained control, matches trust model** | **Slightly more verbose**  |
| Tool sets                       | Concise                                       | Too coarse for trust tiers |

**Rationale:** Trust model (T1/T2/T3) requires fine-grained restrictions. Tool sets are too coarse — `edit` grants createFile AND editFiles AND createDirectory together. Tool names verified against official VS Code cheat sheet.

---

### D-3: Orchestrator-mediated interactive questioning (🟡, High)

**Context:** FR-3/FR-4 require architect and planner to ask users questions. VS Code subagents cannot prompt users.

| Alternative                          | Pros                                   | Cons                              |
| ------------------------------------ | -------------------------------------- | --------------------------------- |
| Embed in subagent                    | Matches spec literally                 | Technically impossible            |
| **Orchestrator mediates (SELECTED)** | **Aligns with VS Code platform model** | **Adds ~8 lines to orchestrator** |
| NEEDS_REVISION loop                  | Uses existing contract                 | Wastes full re-dispatch tokens    |

**Rationale:** Only approach that works within VS Code platform. Uses **assumptions-first flow** (Option B, per PV C-1):

1. Orchestrator dispatches architect with all inputs
2. Architect completes FULLY, documenting assumptions in `assumptions_made[]` output field
3. Architect includes `clarifications_needed[]` for remaining ambiguities
4. Orchestrator reads output; if interactive AND `clarifications_needed` present, presents questions to user via `ask_questions`
5. If user answers **contradict** assumptions → re-dispatch architect with original inputs + `clarification_responses` parameter; architect resumes at Step 2
6. If answers are **compatible** → proceed with existing architecture (no re-dispatch needed)

Same pattern for planner: planner outputs `plan_summary` + `clarifications_needed`; orchestrator mediates approval. Deviations DR-1 and DR-2 document this.

---

### D-4: Keep 'agent' tool set in orchestrator (🟢, High)

**Context:** Orchestrator needs to dispatch subagents. VS Code has both `agent` (tool set) and `runSubagent` (individual tool).

**Selected:** Keep `agent` in tools array — matches existing v2 and VS Code official examples. Body text instructs `runSubagent` usage per FR-2.3.

---

### D-5: Git staging — two-source selective pathspecs (🟡, High)

**Context:** FR-5 removes git add from implementer. FR-6 removes autocommit. Need replacement git workflow.

| Alternative                         | Pros                                                        | Cons                                             |
| ----------------------------------- | ----------------------------------------------------------- | ------------------------------------------------ |
| **Two-source selective (SELECTED)** | **Covers implementation + pipeline artifacts, audit trail** | **Requires parsing reports + directory staging** |
| git add -A in orchestrator          | Simple                                                      | Stages untracked/unrelated files                 |

**Rationale:** Two-source selective staging (revised per AG C-2):

- **Source 1:** Implementation code from reports' `files_modified` lists via `git add <explicit paths>`
- **Source 2:** Pipeline artifacts via `git add docs/feature/<slug>/` (covers knowledge, tester, reviewer, planner, researcher outputs)

Pre-stage knowledge validation (renamed from "pre-commit" per AG C-2 terminology fix) checks knowledge `output_paths` BEFORE staging in sub-step 8c. Interactive mode: user chooses Commit/Review/Unstage. Autonomous mode: stage only, no commit.

---

### D-6: Exploratory QA — inline with optional SKILL.md extension (🟡, Medium)

**Context:** FR-9 adds exploratory QA to tester. TourneyPal QA is ~500 lines. Tester at 129 lines (verified). **Note:** Architect (130 lines verified) is the actual tightest agent post-changes (~145, 5 headroom per AG A-2).

| Alternative           | Pros                             | Cons                      |
| --------------------- | -------------------------------- | ------------------------- |
| Inline only           | Self-contained                   | Less detailed             |
| SKILL.md only         | Rich procedures                  | Dependency                |
| **Hybrid (SELECTED)** | **Works with or without skills** | **Slightly more complex** |

**Rationale:** Hybrid keeps tester functional without skills (4-category inline checklist). Tester reaches ~143 lines (8 headroom — comfortable). **Architect at ~145 (5 headroom) is the true budget bottleneck** — its web research step (FR-8) or clarification step (FR-3) should be extracted to a reference doc if budget becomes tight during implementation. The existing `live-qa` category in Step 3 is **subsumed** by the new Exploratory QA phase (PV C-4) — `live-qa` is removed from Step 3's skill categories.

**Medium confidence note:** Skill integration relies on VS Code progressive loading (documented but untested in this system).

---

### D-7: Remove vscode/memory from tools list (🟢, High)

**Context:** Knowledge agent lists `vscode/memory` — no such tool exists. Memory is a VS Code setting.

**Selected:** Remove from frontmatter. Reword workflow step to reference memory as VS Code feature. Knowledge agent still persists learnings via body text guidance.

---

### D-8: E2E skill discovery — user-driven with skill scan (🟢, High)

**Context:** FR-1 requires E2E skill setup guidance. Autodetecting project type is error-prone.

**Selected:** Orchestrator scans skill directories, presents findings, asks user to choose. In autonomous mode, found skills are used; if none found, warning logged per FR-1.3.

---

### D-9: Pre-commit validation scoped to knowledge output paths only (🟡, High)

**Context:** Prior design review (R2/R3) identified Major finding about pre-commit validation scope.

| Alternative                         | Pros                                        | Cons                                    |
| ----------------------------------- | ------------------------------------------- | --------------------------------------- |
| **Knowledge paths only (SELECTED)** | **Correct scoping, resolves Major finding** | **Less protective**                     |
| Full diff validation                | Catches unexpected mods                     | Rejects legitimate implementation files |

**Rationale:** Implementation files validated by tester file ownership audit (Steps 5-6). Orchestrator Step 8c validation applies only to knowledge output paths: evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md.

---

### D-10: Step 8 decomposed into 5 explicit sub-steps 8a-8e (🟢, High)

**Context:** AG A-1 Major: Step 8 expands to 6+ branching sub-workflows with risk of misordering.

| Alternative                     | Pros                                   | Cons                                 |
| ------------------------------- | -------------------------------------- | ------------------------------------ |
| Monolithic Step 8               | Fewer steps                            | Cognitive density, misordering risk  |
| **Decomposed 8a-8e (SELECTED)** | **Clear sequence, isolated branching** | **~3 additional orchestrator lines** |

**Sub-step sequence:**

- **8a: Knowledge Extraction** — dispatch knowledge agent (non-blocking on error)
- **8b: Doc Updates** — interactive only: present [Apply/Review/Skip]; if Apply, re-dispatch knowledge in doc-update mode
- **8c: Knowledge Output Validation** — validate output_paths against allowlist (evidence-bundle.yaml, knowledge-output.yaml, instruction-recommendations.md). Reject unexpected files
- **8d: Selective Staging** — two-source: (1) `git add <paths from reports files_modified>`, (2) `git add docs/feature/<slug>/` for pipeline artifacts
- **8e: Commit/Review Choice** — interactive: [Commit/Review/Unstage] 3-way; autonomous: stage only, no commit

**Rationale:** Explicit sub-steps prevent misordering. Each sub-step has single responsibility. Conditional branches (interactive vs autonomous) isolated to 8b and 8e.

---

### D-11: Reject disable-model-invocation for worker agents (🟢, High)

**Context:** SS S-1 suggested `disable-model-invocation: true` for workers. Web research revealed this would break the pipeline.

| Alternative                                  | Pros                                     | Cons                                                 |
| -------------------------------------------- | ---------------------------------------- | ---------------------------------------------------- |
| Add disable-model-invocation: true           | Maximum lockdown                         | **BREAKS PIPELINE** — prevents orchestrator dispatch |
| **Rely on user-invocable: false (SELECTED)** | **Pipeline works, trust model enforced** | **Theoretical autonomous invocation risk (low)**     |

**Rationale:** VS Code custom agents docs (fetched 2026-03-09) define `disable-model-invocation` as: _"Optional boolean flag to prevent the agent from being invoked as a subagent by other agents."_ The deprecated `infer` docs confirm: _"disable-model-invocation: true to prevent subagent invocation while keeping it in the picker."_ Adding this to workers would prevent the orchestrator from dispatching them via `runSubagent`. Trust model is already enforced: (1) `user-invocable: false` hides from dropdown, (2) `agents: []` prevents workers from dispatching subagents, (3) `tools:` arrays restrict capabilities per trust tier.

---

## Deviation Records

### DR-1: No structured question tool exists (FR-3.3, AC-9)

Spec requires "structured question tool in tools: frontmatter." No such tool exists in VS Code (verified via official cheat sheet). Design uses orchestrator-mediated interaction instead. Satisfies **intent** of FR-3.

### DR-2: Same for planner (FR-4.4, AC-11)

Same platform constraint as DR-1. Planner outputs `plan_summary`; orchestrator handles interactive plan approval gate.

---

## Tool Name Corrections

| Agent        | Before (invalid)                                                                                                                                                                   | After (valid)                                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| orchestrator | read/readFile, edit/createFile, edit/editFiles, search/listDirectory, search/textSearch, search/fileSearch, execute/runInTerminal, execute/getTerminalOutput, search/changes, todo | readFile, createFile, editFiles, listDirectory, textSearch, fileSearch, runInTerminal, getTerminalOutput, changes, todos     |
| researcher   | read/readFile, search/listDirectory, search/textSearch, search/codebase, search/fileSearch, web/fetch, edit/createFile                                                             | readFile, listDirectory, textSearch, codebase, fileSearch, fetch, createFile                                                 |
| architect    | (same as researcher)                                                                                                                                                               | (same as researcher)                                                                                                         |
| planner      | read/readFile, search/listDirectory, search/textSearch, search/codebase, search/fileSearch, edit/createFile                                                                        | readFile, listDirectory, textSearch, codebase, fileSearch, createFile                                                        |
| implementer  | read/readFile, edit/createFile, edit/editFiles, search/_, execute/_, read/problems                                                                                                 | readFile, createFile, editFiles, listDirectory, textSearch, codebase, fileSearch, runInTerminal, getTerminalOutput, problems |
| tester       | (same as implementer minus editFiles)                                                                                                                                              | readFile, createFile, listDirectory, textSearch, codebase, fileSearch, runInTerminal, getTerminalOutput, problems            |
| reviewer     | read/readFile, search/\*, read/problems, search/changes, edit/createFile                                                                                                           | readFile, listDirectory, textSearch, codebase, fileSearch, problems, changes, createFile                                     |
| knowledge    | read/readFile, search/\*, edit/createFile, **vscode/memory**                                                                                                                       | readFile, listDirectory, textSearch, codebase, fileSearch, createFile (**vscode/memory removed**)                            |

---

## Line Budget

| Agent        | Current (verified) | Estimated | Headroom | Risk                       |
| ------------ | ------------------ | --------- | -------- | -------------------------- |
| orchestrator | 117                | 142       | 8        | 🟡                         |
| researcher   | 83                 | 84        | 66       | 🟢                         |
| architect    | 130                | 145       | **5**    | **🔴 Tightest**            |
| planner      | 105                | 116       | 34       | 🟢                         |
| implementer  | 99                 | 115       | 35       | 🟢                         |
| tester       | 129                | 143       | 7        | 🟡                         |
| reviewer     | 96                 | 99        | 51       | 🟢                         |
| knowledge    | 107                | 118       | 32       | 🟢                         |
| global-rules | 70                 | 80        | N/A      | Shared doc, no agent limit |

**Total:** 936 → 1,042 lines (of 1,500 budget). Budget remaining: 458 lines.
Line counts verified via PowerShell `(Get-Content).Count` on 2026-03-09 (AG C-1). Prior counts were stale from original v2 system metrics.

---

## File Inventory

| File                       | Action    | Risk | Changes                                                                                                                                                                             |
| -------------------------- | --------- | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| orchestrator.agent.md      | modify    | 🟡   | Tool names, E2E skill scan, clarification mediation (assumptions-first), plan approval, two-source staging, commit choice, doc update, pre-stage validation, Step 8 sub-steps 8a-8e |
| researcher.agent.md        | modify    | 🟢   | Tool names, user-invocable: false                                                                                                                                                   |
| architect.agent.md         | modify    | 🔴   | Tool names, user-invocable: false, assumptions-first clarification step, web research step. **TIGHTEST: 5-line headroom**                                                           |
| planner.agent.md           | modify    | 🟢   | Tool names, user-invocable: false, plan_summary output                                                                                                                              |
| implementer.agent.md       | modify    | 🟡   | Tool names, user-invocable: false, remove git add, REFACTOR step, quality principles                                                                                                |
| tester.agent.md            | modify    | 🟡   | Tool names, user-invocable: false, exploratory QA phase (replaces live-qa), skill references, static audit: remove git add from implementer patterns                                |
| reviewer.agent.md          | modify    | 🟢   | Tool names, user-invocable: false, audit patterns: remove git add from implementer-acceptable, flag git add as Major                                                                |
| knowledge.agent.md         | modify    | 🟡   | Tool names, remove vscode/memory, memory reword, doc-update mode                                                                                                                    |
| global-rules.md            | modify    | 🟢   | Git operations rule, Plan Refinement iteration cap (max 1)                                                                                                                          |
| feature-workflow.prompt.md | no change | 🟢   | —                                                                                                                                                                                   |
| quick-fix.prompt.md        | no change | 🟢   | —                                                                                                                                                                                   |

---

## Security Considerations

- **Tool name fix (D-2):** Currently ALL tool restrictions are non-functional due to invalid names. Fixing names restores VS Code platform-enforced access control.
- **Git safety (D-5, FR-5):** Removing git add from implementer eliminates unsanctioned staging. Only orchestrator can stage/commit. Two-source staging covers both implementation and pipeline artifacts.
- **Pre-stage validation (D-9):** Knowledge agent output paths validated before staging in sub-step 8c. Implementation files validated by tester file ownership audit.
- **Trust model preserved:** T1 (orchestrator), T2 (implementer, tester), T3 (all others) — individual tool names enforce tier boundaries.
- **disable-model-invocation NOT added (D-11):** Web research confirmed this property prevents subagent dispatch, which would break orchestrator→worker routing. Trust model enforced via `user-invocable: false` + `agents: []` + `tools:` arrays instead.
- **Reviewer audit enforcement (PV C-2):** Reviewer command audit patterns updated to flag `git add` in implementer `commands_executed[]` as Major finding.

---

## Failure & Recovery

- **Architect over 150 lines (🔴 Tightest):** If FR-8 web research step or FR-3 clarification step pushes architect past 150, extract web research guidance to a reference doc (e.g., `web-research-guide.md`). This is the primary budget risk (5-line headroom).
- **Tester over 150 lines:** If exploratory QA pushes tester past 150, extract to SKILL.md (D-6 fallback). 7-line headroom is tight but more comfortable than architect.
- **Orchestrator over 150 lines:** If mediation logic exceeds budget, extract to reference doc.
- **Subagent questioning fails:** Assumptions-first flow (D-3) is primary. If assumptions consistently contradict user answers, fall back to two-phase dispatch approach.

---

## Non-Functional Requirements

- All agents ≤ 150 lines (AC-27)
- Total system ≤ 1,500 lines
- Zero § cross-file references
- Completion-contract-only routing
- Zero SQLite dependency

---

## Migration & Backwards Compatibility

No migration needed — v2 files modified in place. Prompt files unchanged. Prior pipeline behavior preserved for steps not affected by the 10 FRs.

---

## Testing Strategy

- **Static verification:** Verify tool names match VS Code catalog, line counts within budgets, user-invocable set correctly.
- **Post-implementation tool verification (PV S-1):** After implementing tool name corrections, verify restrictions are functional via VS Code Chat Customizations diagnostics view (Configure Chat > Diagnostics). Any misspelled tool name silently fails — same failure mode as current broken state. This MUST be tested, not assumed.
- **Integration verification:** End-to-end pipeline run with modified agents validates routing, questioning, git workflow.
- **Acceptance criteria:** 27 ACs mapped in design-output.yaml ac_mapping section.

---

## Tradeoffs & Alternatives Considered

See individual decision records (D-1 through D-9) for full alternatives analysis. Key tradeoffs:

1. **Individual tools vs tool sets (D-2):** Chose verbosity over coarse-grained control for trust enforcement.
2. **Orchestrator mediation vs embedded questioning (D-3):** Platform constraint forces mediation pattern. Two deviation records document spec divergence.
3. **Inline QA vs SKILL.md (D-6):** Hybrid approach keeps tester self-contained while allowing richer QA when skills exist.
4. **Full diff vs knowledge-only validation (D-9):** Scoped validation avoids false positives on legitimate implementation files.

---

## Implementation Checklist & Deliverables

1. Fix all tool names across 8 agents (FR-2) — highest priority, currently non-functional
2. Add `user-invocable: false` to all 7 worker agents (FR-2.4)
3. Add git operations rule to global-rules.md (FR-5.3) + plan refinement cap (PV C-3)
4. Add E2E skill scan to orchestrator Step 1 (FR-1)
5. Add architect assumptions-first clarification step + orchestrator mediation (FR-3, D-3)
6. Add planner plan_summary + orchestrator approval (FR-4)
7. Remove git add from implementer; add two-source staging to orchestrator Step 8d (FR-5, FR-6, D-5)
8. Decompose orchestrator Step 8 into sub-steps 8a-8e (D-10)
9. Add architect web research step (FR-8)
10. Add tester exploratory QA phase, replace live-qa (FR-9, PV C-4)
11. Add implementer REFACTOR step + quality principles (FR-10)
12. Add knowledge doc-update mode (FR-7)
13. Remove vscode/memory from knowledge tools (D-7)
14. Update reviewer command audit: remove git add from implementer patterns (PV C-2)
15. Update tester static audit: remove git add from implementer patterns (PV C-2)
16. Post-implementation: verify tool names via VS Code diagnostics (PV S-1)

---

## AC Mapping

| AC    | Design Coverage                                          |
| ----- | -------------------------------------------------------- |
| AC-1  | Orchestrator Step 1 skill scan                           |
| AC-2  | Orchestrator interactive E2E prompt                      |
| AC-3  | Orchestrator autonomous warning                          |
| AC-4  | tool_name_corrections section                            |
| AC-5  | Orchestrator tools: agent + agents: 7 workers            |
| AC-6  | Orchestrator body runSubagent instruction                |
| AC-7  | All workers user-invocable: false                        |
| AC-8  | Architect clarification step                             |
| AC-9  | DR-1: No structured question tool; orchestrator mediates |
| AC-10 | Planner plan_summary output                              |
| AC-11 | DR-2: No structured question tool; orchestrator mediates |
| AC-12 | Implementer git removal                                  |
| AC-13 | global-rules.md git rule                                 |
| AC-14 | Orchestrator selective pathspecs (D-5)                   |
| AC-15 | Orchestrator interactive commit choice                   |
| AC-16 | Orchestrator autonomous stage-only                       |
| AC-17 | Orchestrator doc update choice                           |
| AC-18 | Knowledge doc-update constraints                         |
| AC-19 | Architect web research step                              |
| AC-20 | Architect fetch tool name                                |
| AC-21 | Tester exploratory QA phase                              |
| AC-22 | Tester exploratory findings format                       |
| AC-23 | Tester ≤150 lines (estimated 143)                        |
| AC-24 | Implementer REFACTOR step                                |
| AC-25 | Implementer quality principles                           |
| AC-26 | REFACTOR re-run + revert on failure                      |
| AC-27 | All agents ≤150 (see line_budget)                        |

---

## Decisions Cross-Reference

| ID   | Title                                                             | Risk | Confidence |
| ---- | ----------------------------------------------------------------- | ---- | ---------- |
| D-1  | Direction C — Hybrid inline + shared extraction                   | 🟢   | High       |
| D-2  | Individual flat camelCase tool names                              | 🟢   | High       |
| D-3  | Orchestrator-mediated interactive questioning (assumptions-first) | 🟡   | High       |
| D-4  | Keep 'agent' tool set in orchestrator                             | 🟢   | High       |
| D-5  | Git staging — two-source selective pathspecs                      | 🟡   | High       |
| D-6  | Exploratory QA — inline + optional SKILL.md                       | 🟡   | Medium     |
| D-7  | Remove vscode/memory from tools                                   | 🟢   | High       |
| D-8  | E2E skill discovery — user-driven                                 | 🟢   | High       |
| D-9  | Pre-stage validation — knowledge paths only                       | 🟡   | High       |
| D-10 | Step 8 decomposed into sub-steps 8a-8e                            | 🟢   | High       |
| D-11 | Reject disable-model-invocation (would break pipeline)            | 🟢   | High       |
| DR-1 | Deviation: No structured question tool (FR-3)                     | —    | —          |
| DR-2 | Deviation: No structured question tool (FR-4)                     | —    | —          |

**Medium confidence decision to revisit:**

- **D-6 (Exploratory QA):** Skill integration untested in this system. If progressive loading doesn't work, fall back to inline-only approach.

**Budget risk decision to monitor:**

- **D-6 / architect (AG A-2):** Architect at 130→~145 (5 headroom) is the true tightest agent. If FR-8 or FR-3 additions exceed estimate, extract web research guidance to reference doc.

---

## Revision History

### Round 2 (2026-03-09) — Addresses 1 Critical + 5 Major + 5 Minor

**Reviewers:** architecture-guardian (needs_revision), pragmatic-verifier (needs_revision), security-sentinel (approve)

| Finding                                                | Severity     | Source                | Resolution                                                                                                                                       |
| ------------------------------------------------------ | ------------ | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| AG C-1: Line counts wrong for 5/9 files                | **Critical** | architecture-guardian | All counts updated to PowerShell-verified actuals. Architect identified as tightest (130→145, 5 headroom). Risk reclassified 🔴. Total 936→1042. |
| AG C-2: Selective staging excludes pipeline artifacts  | **Major**    | architecture-guardian | D-5 updated to two-source staging. Pre-commit renamed to pre-stage validation.                                                                   |
| AG A-1: Step 8 cognitive density                       | **Major**    | architecture-guardian | New D-10: Step 8 decomposed into sub-steps 8a-8e.                                                                                                |
| PV C-1: Architect clarification flow underspecified    | **Major**    | pragmatic-verifier    | D-3 updated with assumptions-first flow (Option B). Full data flow specified.                                                                    |
| PV C-2: Reviewer/tester audit patterns not updated     | **Major**    | pragmatic-verifier    | Reviewer + tester file_inventory updated: git add removed from implementer patterns, flagged as Major.                                           |
| SS S-1: Worker agents missing disable-model-invocation | **Major**    | security-sentinel     | New D-11: REJECTED. VS Code docs confirm property breaks subagent dispatch. Trust model via user-invocable:false.                                |
| AG C-3: Question-answer data flow unspecified          | Minor        | architecture-guardian | D-3 rationale includes full assumptions-first flow with re-dispatch on contradiction.                                                            |
| PV S-1: Post-implementation tool name verification     | Minor        | pragmatic-verifier    | Testing strategy updated with VS Code diagnostics verification step.                                                                             |
| PV C-3: Plan refinement iteration cap                  | Minor        | pragmatic-verifier    | Global-rules file_inventory: Plan Refinement row added (max 1 iteration).                                                                        |
| PV C-4: live-qa vs Exploratory QA unclear              | Minor        | pragmatic-verifier    | Tester file_inventory: live-qa subsumed by Exploratory QA. Removed from Step 3.                                                                  |
| AG A-2: Architect is true tightest                     | Minor        | architecture-guardian | D-6 updated. Architect at 5 headroom. Extraction fallback documented.                                                                            |
