# Agent Pipeline Improvements — Feature Specification

## Title & Summary

**Feature:** Agent Pipeline Improvements  
**Summary:** Eight targeted improvements to the multi-agent pipeline system in `NewAgents/.github/agents/` and `NewAgents/.github/prompts/`, addressing E2E test gating, approval mode confusion, web fetching capability, terminal redirect enforcement, spec pushback granularity, Copilot Skills integration, SQLite lifecycle management, and fast-track pipeline entry.

---

## Background & Context

### Research Inputs

- [research/architecture.yaml](research/architecture.yaml) — 14 findings on file structure, pipeline flow, E2E-risk coupling, approval mode propagation
- [research/impact.yaml](research/impact.yaml) — 18 findings on blast radius, breaking changes, file-to-issue impact map
- [research/dependencies.yaml](research/dependencies.yaml) — 13 findings on inter-file cross-references, tool access chains, ordering constraints
- [research/patterns.yaml](research/patterns.yaml) — 19 findings on agent patterns, anti-patterns, enforcement gaps

### System Overview

The agent system comprises 19 files in `NewAgents/.github/`:

- **9 agent definitions** (.agent.md): orchestrator, researcher, spec, designer, planner, implementer, verifier, adversarial-reviewer, knowledge-agent
- **9 shared reference documents** (.md): global-operating-rules, schemas, tool-access-matrix, dispatch-patterns, sql-templates, e2e-integration, review-perspectives, evaluation-schema, severity-taxonomy, context7-integration
- **1 prompt file**: feature-workflow.prompt.md

The pipeline executes 10 steps (Steps 0–9) coordinated by the orchestrator through typed YAML completion contracts and SQL evidence gates.

### Origin

The user identified 8 issues from hands-on experience running the pipeline. See [initial-request.md](initial-request.md) for the verbatim request.

---

## Functional Requirements

### FR-1: E2E Testing Available for All Risk Levels

**Problem:** E2E testing is triple-gated to 🔴 risk only through the planner's workflow_lane derivation, verifier Tier 5 conditional gate, and orchestrator evidence gates. 🟢 and 🟡 tasks cannot have E2E verification.

**Requirements:**

| ID      | Requirement                                                                                                                            | Priority |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-1.1  | Decouple `e2e_required` from `workflow_lane`. Make it an independent boolean based on e2e-contract.yaml presence + planner assessment. | Must     |
| FR-1.2  | Update planner workflow_lane derivation table — lanes remain risk-based but `e2e_required` is set independently.                       | Must     |
| FR-1.3  | Change verifier Tier 5 gate from `e2e_required=true AND workflow_lane=full-tdd-e2e` to `e2e_required=true` only.                       | Must     |
| FR-1.4  | Remove 🔴-risk precondition from orchestrator Step 0 E2E contract discovery.                                                           | Must     |
| FR-1.5  | Restructure EG-10 in sql-templates.md — make e2e-test-execution an additive check gated on `e2e_required=true` regardless of lane.     | Must     |
| FR-1.6  | Update schemas.md `task.e2e_required` field definition to reflect new derivation rule.                                                 | Must     |
| FR-1.7  | E2E remains optional for 🟢/🟡 (planner decides). E2E is mandatory for 🔴 when contract exists.                                        | Must     |
| FR-1.8  | Update planner self-verification check #11 to validate new e2e_required derivation.                                                    | Should   |
| FR-1.9  | Update e2e-integration.md §0 to reflect E2E applicability for all risk levels.                                                         | Should   |
| FR-1.10 | Update dispatch-patterns.md E2E constraints to apply when `e2e_required=true` regardless of risk.                                      | Should   |

**Files affected:** planner.agent.md, verifier.agent.md, orchestrator.agent.md, sql-templates.md, schemas.md, e2e-integration.md, dispatch-patterns.md (7 files)

### FR-2: Approval Mode Clarity and Enforcement

**Problem:** The phrase "default to autonomous" appears 3 times in the orchestrator Step 0 (lines 105–109), biasing agents toward autonomous mode even when the user selected interactive. Approval mode propagation to subagents is implicit.

**Requirements:**

| ID     | Requirement                                                                                                                 | Priority |
| ------ | --------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-2.1 | Remove all "default to autonomous" language. User's APPROVAL_MODE is authoritative. If absent, prompt with no default bias. | Must     |
| FR-2.2 | Explicitly pass `approval_mode` as a named parameter in every subagent dispatch.                                            | Must     |
| FR-2.3 | If approval_mode is undetermined after prompting, treat as interactive (safer default).                                     | Must     |
| FR-2.4 | feature-workflow.prompt.md must clarify APPROVAL_MODE from initial request takes priority.                                  | Should   |
| FR-2.5 | Fix typos in orchestrator.agent.md line 24 ("presetnt", "stoping").                                                         | Should   |

**Files affected:** orchestrator.agent.md, feature-workflow.prompt.md (2 files)

### FR-3: Web Fetching Capability for Agents

**Problem:** `fetch_webpage` is completely absent from the agent system. No agent can research online for architecture patterns, stack updates, or best practices.

**Requirements:**

| ID     | Requirement                                                                                                      | Priority |
| ------ | ---------------------------------------------------------------------------------------------------------------- | -------- |
| FR-3.1 | Add `fetch_webpage` row to tool-access-matrix.md §1. Access: Researcher=🔒, Designer=🔒, Spec=🔒, all others=❌. | Must     |
| FR-3.2 | Scope restriction: interactive mode only. Used for researching patterns, stack info, documentation.              | Must     |
| FR-3.3 | Update researcher.agent.md: tool summary, workflow step for web research.                                        | Must     |
| FR-3.4 | Update designer.agent.md: tool summary, document fetch_webpage vs Context7 usage.                                | Must     |
| FR-3.5 | Update spec.agent.md: tool summary, document usage for requirement feasibility research.                         | Must     |
| FR-3.6 | Agents MUST NOT use fetch_webpage in autonomous mode. Self-verification should check.                            | Must     |

**Note:** `fetch_webpage` complements Context7 integration (library API docs). fetch_webpage is for general web research; Context7 is for specific library documentation.

**Files affected:** tool-access-matrix.md, researcher.agent.md, designer.agent.md, spec.agent.md (4 files)

### FR-4: No Terminal-to-File Redirection Enforcement

**Problem:** The no-file-redirect rule exists in `.github/copilot-instructions.md` and `global-operating-rules.md §9`, but implementer and verifier agent files don't restate it. These agents have full `run_in_terminal` access and are the primary violators.

**Requirements:**

| ID     | Requirement                                                                                         | Priority |
| ------ | --------------------------------------------------------------------------------------------------- | -------- |
| FR-4.1 | Add explicit no-redirect rule to implementer.agent.md Operating Rules with example banned patterns. | Must     |
| FR-4.2 | Add explicit no-redirect rule to verifier.agent.md Operating Rules with example banned patterns.    | Must     |
| FR-4.3 | Add "NEVER redirect terminal output to files" to implementer anti-drift anchor.                     | Must     |
| FR-4.4 | Add "NEVER redirect terminal output to files" to verifier anti-drift anchor.                        | Must     |

**Banned patterns to list:** `> file`, `>> file`, `| tee file`, `2>&1 > file`, `2>&1 | tee file`

**Files affected:** implementer.agent.md, verifier.agent.md (2 files)

### FR-5: Spec Pushback Per-Concern Granularity

**Problem:** The spec agent presents all pushback concerns as a single aggregate question with proceed/modify/abandon options. Users cannot respond to individual concerns.

**Requirements:**

| ID     | Requirement                                                                                                                   | Priority |
| ------ | ----------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-5.1 | In interactive mode, present each concern as a separate question in the `ask_questions` call using the questions[] array.     | Must     |
| FR-5.2 | Per-concern options: "Accept as known risk", "Modify scope to address" (with freeform input), "Dismiss — not a real concern". | Must     |
| FR-5.3 | Aggregate per-concern responses: all accepted/dismissed → proceed; any "modify" → incorporate user direction.                 | Must     |
| FR-5.4 | In autonomous mode, auto-resolve each concern as "Accept as known risk" individually.                                         | Must     |
| FR-5.5 | pushback_log records per-concern `user_response` fields.                                                                      | Should   |

**Files affected:** spec.agent.md (1 file)

### FR-6: E2E via GitHub Copilot Skills Integration

**Problem:** The current E2E skill system uses a custom YAML format translated to Playwright CLI commands. GitHub Copilot Skills (SKILL.md with `copilot-skill://` URIs) are a different, native VS Code system that could enhance app interaction capabilities.

**Requirements:**

| ID     | Requirement                                                                                                                 | Priority |
| ------ | --------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-6.1 | Extend e2e-integration.md §2 with a `skill_format` field: 'playwright-yaml' (existing) and 'copilot-skill' (new).           | Must     |
| FR-6.2 | Researcher checks for existing Copilot skill files in .github/copilot/skills/ and references them in e2e-contract.yaml.     | Should   |
| FR-6.3 | Verifier Tier 5 loads both skill formats: playwright-yaml via CLI translation, copilot-skill via interaction guide parsing. | Should   |
| FR-6.4 | Designer/researcher use fetch_webpage (interactive mode) to research current Copilot Skills format.                         | May      |
| FR-6.5 | Copilot Skills layer on top of Playwright CLI — skills describe WHAT, Playwright CLI is HOW.                                | Must     |

**Design note:** This is an enhancement layered on the existing Playwright infrastructure, not a replacement. Full depth of integration depends on designer research into the Copilot Skills format.

**Files affected:** e2e-integration.md, researcher.agent.md, verifier.agent.md (3 files)

### FR-7: SQLite DB Auto-Archive on New Pipeline Run

**Problem:** The SQLite DB accumulates data indefinitely with no archive strategy. The user wants fresh DBs per pipeline run, with old data archived (not migrated).

**Requirements:**

| ID     | Requirement                                                                                                                                             | Priority |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-7.1 | Orchestrator Step 0 adds an archive check BEFORE DDL execution: if DB has data from a different run_id, rename to `verification-ledger-{timestamp}.db`. | Must     |
| FR-7.2 | Archive check queries `pipeline_telemetry` for existing run_id. If different or query fails, archive.                                                   | Must     |
| FR-7.3 | After archiving, create fresh verification-ledger.db with standard DDL.                                                                                 | Must     |
| FR-7.4 | Expand orchestrator run_in_terminal scope (DR-1) to permit DB file rename.                                                                              | Must     |
| FR-7.5 | Document expanded scope in tool-access-matrix.md §2.                                                                                                    | Must     |
| FR-7.6 | No migration plans — decision is purely run_id mismatch-based.                                                                                          | Must     |

**Files affected:** orchestrator.agent.md, tool-access-matrix.md (2 files, optionally sql-templates.md for the archive check query)

### FR-8: Fast-Track Pipeline Prompt

**Problem:** Only one prompt exists (feature-workflow.prompt.md) which always runs all 10 steps. No shorter entry point exists for tasks that don't need research, specification, or design.

**Requirements:**

| ID     | Requirement                                                                                                                     | Priority |
| ------ | ------------------------------------------------------------------------------------------------------------------------------- | -------- |
| FR-8.1 | Create `plan-and-implement.prompt.md` in `NewAgents/.github/prompts/`. Executes: Step 0 → Step 4 → Steps 5–6 → Step 7 → Step 9. | Must     |
| FR-8.2 | Pass `pipeline_mode=fast-track` variable. Orchestrator detects in Step 0 and skips Steps 1–3b.                                  | Must     |
| FR-8.3 | Planner gains ask_questions access (🔒 fast-track mode only) in tool-access-matrix.md.                                          | Must     |
| FR-8.4 | Planner defines fast-track fallback input: works from initial-request.md + ask_questions responses.                             | Must     |
| FR-8.5 | Amend orchestrator anti-drift anchor: step-skipping is permitted when the active prompt defines a reduced step set.             | Must     |
| FR-8.6 | Existing feature-workflow.prompt.md MUST NOT be modified.                                                                       | Must     |
| FR-8.7 | Fast-track mode should still perform Step 8 (knowledge capture) on success.                                                     | Should   |

**Evaluation — Separate Prompt vs. Dynamic Scope Detection:**

| Criterion                 | Separate Prompt File                          | Dynamic Scope Detection                               |
| ------------------------- | --------------------------------------------- | ----------------------------------------------------- |
| Clarity                   | High — each prompt defines exactly what runs  | Low — orchestrator must assess scope at runtime       |
| Implementation complexity | Low — new file + small orchestrator amendment | High — rules for assessing "small" vs "large" tasks   |
| Safety                    | High — prompt-defined step set is explicit    | Medium — heuristic-based step skipping is brittle     |
| User control              | High — user explicitly chooses which prompt   | Low — orchestrator decides for user                   |
| Anti-drift risk           | Low — only active when fast-track prompt used | High — could erroneously skip steps for complex tasks |

**Recommendation:** Separate prompt file (Direction A). Research and user intent both confirm a separate prompt is cleaner, safer, and gives the user explicit control.

**Files affected:** New: plan-and-implement.prompt.md. Modified: orchestrator.agent.md, planner.agent.md, tool-access-matrix.md (3 existing + 1 new)

---

## Non-Functional Requirements

| ID    | Requirement                                                                                                      | Priority |
| ----- | ---------------------------------------------------------------------------------------------------------------- | -------- |
| NFR-1 | Orchestrator line budget ≤550 lines must be maintained. Extract new logic to reference docs if needed.           | Must     |
| NFR-2 | All changes are markdown/YAML instruction files only — no runtime code.                                          | Must     |
| NFR-3 | Tool access enforcement remains instruction-based (accepted risk per tool-access-matrix.md §11).                 | Must     |
| NFR-4 | Modified agents should update self-verification checklists within 1 sprint of the initial change.                | Should   |
| NFR-5 | The fast-track prompt should reduce pipeline wall-clock time by ≥50% compared to full pipeline for simple tasks. | Should   |

---

## Constraints & Assumptions

### Constraints

1. All changes are to markdown instruction files and YAML schemas. No runtime code, executables, or build artifacts.
2. Windows environment, VS Code GitHub Copilot agent framework.
3. Agent line budgets: standard ≤350 lines, orchestrator ≤550 lines (NFR-1 exception).
4. Tool access enforcement is instruction-based only (no runtime enforcement). See tool-access-matrix.md §11.
5. Global immutable safety rules (global-operating-rules.md §9) cannot be weakened.
6. Protected phrases (MUST NOT, NEVER, RESTRICTED, PROHIBITED) cannot be removed from existing rules unless the intent is preserved or strengthened.
7. The existing feature-workflow.prompt.md and full 10-step pipeline must remain functional.

### Assumptions

1. `fetch_webpage` is available as a built-in VS Code Copilot tool in the user's runtime environment.
2. The `ask_questions` tool supports multiple questions in a single call via a `questions[]` array.
3. The user wants E2E to be available (opt-in by planner) for all risk levels, not mandatory for all.
4. GitHub Copilot Skills format uses `.md` files and is documented on the web.
5. The SQLite archive rename operation via `run_in_terminal` works on Windows (using PowerShell `Rename-Item` or `Move-Item`).

---

## Acceptance Criteria

| ID    | Criteria                                                                      | Test Method | Pass/Fail Definition                                              |
| ----- | ----------------------------------------------------------------------------- | ----------- | ----------------------------------------------------------------- |
| AC-1  | E2E decoupled from risk: 🟢-risk task + contract → e2e_required=true possible | Inspection  | Planner lane table shows e2e_required independent of risk         |
| AC-2  | E2E optional for low risk: 🟢 + contract → e2e_required=false allowed         | Inspection  | No rule forces e2e_required=true for 🟢                           |
| AC-3  | E2E mandatory for high risk: 🔴 + contract → e2e_required=true always         | Inspection  | Planner rule states 🔴 + contract = e2e_required=true             |
| AC-4  | APPROVAL_MODE from initial request is authoritative                           | Inspection  | Orchestrator Step 0 uses APPROVAL_MODE without re-prompt          |
| AC-5  | No "default to autonomous" phrase in system                                   | Test        | `grep -r "default to autonomous"` returns 0 matches               |
| AC-6  | fetch_webpage in tool-access-matrix.md §1                                     | Inspection  | Row exists with Researcher=🔒, Designer=🔒, Spec=🔒               |
| AC-7  | fetch_webpage documented in 3 agent files                                     | Inspection  | Tool summary + usage docs in researcher, designer, spec           |
| AC-8  | Implementer no-redirect rule in Operating Rules                               | Inspection  | Explicit prohibition with banned pattern examples                 |
| AC-9  | Verifier no-redirect rule in Operating Rules                                  | Inspection  | Explicit prohibition with banned pattern examples                 |
| AC-10 | Anti-drift anchors include no-redirect                                        | Inspection  | Both implementer + verifier anchors mention it                    |
| AC-11 | Per-concern pushback questions in spec interactive mode                       | Inspection  | Pushback section shows questions[] array with per-concern entries |
| AC-12 | pushback_log per-concern responses                                            | Inspection  | Schema shows user_response per concern item                       |
| AC-13 | skill_format field in e2e-integration.md §2                                   | Inspection  | Field distinguishes playwright-yaml and copilot-skill             |
| AC-14 | DB archive in Step 0                                                          | Inspection  | Archive check + rename logic before DDL                           |
| AC-15 | DR-1 scope expansion documented                                               | Inspection  | tool-access-matrix.md §2 includes DB rename permission            |
| AC-16 | plan-and-implement.prompt.md exists                                           | Inspection  | File present in NewAgents/.github/prompts/                        |
| AC-17 | Planner ask_questions in tool matrix                                          | Inspection  | §1 shows 🔒, §6 documents fast-track scope                        |
| AC-18 | Planner fast-track input path                                                 | Inspection  | Minimum Viable Input section has fallback                         |
| AC-19 | Anti-drift anchor amended for prompt-defined step sets                        | Inspection  | Conditional language replaces absolute "NEVER skip"               |
| AC-20 | feature-workflow.prompt.md unchanged                                          | Inspection  | Full pipeline still invokes all steps                             |
| AC-21 | Orchestrator ≤550 lines                                                       | Test        | Line count check                                                  |
| AC-22 | Self-verification checklists updated                                          | Inspection  | New behaviors covered in each modified agent                      |

---

## Edge Cases & Error Handling

| ID    | Input/Condition                                                           | Expected Behavior                                                            | Severity if Missed |
| ----- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------- | ------------------ |
| EC-1  | E2E contract exists + 🟢 trivial doc-only task                            | Planner can set e2e_required=false — no wasted E2E time                      | Medium             |
| EC-2  | No e2e-contract.yaml + 🔴 risk                                            | Orchestrator prompts for contract (interactive) or logs warning (autonomous) | High               |
| EC-3  | APPROVAL_MODE in initial-request.md conflicts with ask_questions response | Initial-request APPROVAL_MODE takes priority                                 | Critical           |
| EC-4  | Agent calls fetch_webpage in autonomous mode                              | Self-verification catches, flags as violation                                | Medium             |
| EC-5  | fetch_webpage returns error/timeout                                       | Agent degrades gracefully, proceeds without web data                         | Low                |
| EC-6  | Spec pushback identifies 0 concerns                                       | Skip ask_questions entirely, log concerns_identified=0                       | Low                |
| EC-7  | Spec pushback identifies >5 concerns                                      | Each gets its own question — no aggregation                                  | Low                |
| EC-8  | verification-ledger.db does not exist at start                            | Skip archive, create fresh DB (normal first-run)                             | Low                |
| EC-9  | verification-ledger.db is corrupt                                         | Archive with '-corrupt' suffix, create fresh                                 | Medium             |
| EC-10 | Fast-track mode but user expects research outputs                         | Planner asks clarifying questions to compensate                              | High               |
| EC-11 | Fast-track mode + 🔴 risk task                                            | E2E still triggered if contract exists — fast-track skips research, not E2E  | High               |
| EC-12 | Copilot skill file found but Playwright unavailable                       | Graceful degradation — report skill found, execution deferred                | Medium             |
| EC-13 | Orchestrator exceeds 550 lines after changes                              | Extract to reference doc before merging                                      | Critical           |

---

## User Stories / Flows

### Story 1: Developer runs E2E on a small bug fix

1. User creates initial-request.md for a 🟢-risk bug fix.
2. User runs feature-workflow.prompt.md (full pipeline).
3. Planner assesses the task, finds e2e-contract.yaml exists, determines E2E would verify the fix.
4. Planner sets e2e_required=true despite 🟢 risk.
5. Verifier Tier 5 executes E2E skills, confirming the bug fix works in the live app.

### Story 2: User selects interactive mode and it sticks

1. User sets `APPROVAL_MODE: interactive` in initial-request.md.
2. Orchestrator Step 0 reads this and sets approval_mode=interactive. No "default to autonomous" override.
3. Spec agent receives approval_mode=interactive in dispatch context.
4. Spec agent pushback presents per-concern questions via ask_questions.
5. User responds to each concern individually.

### Story 3: Developer uses fast-track for a quick config change

1. User invokes plan-and-implement.prompt.md with a simple request: "Update the timeout from 5s to 10s."
2. Orchestrator Step 0 detects pipeline_mode=fast-track.
3. Steps 1–3b are skipped. Orchestrator dispatches planner at Step 4.
4. Planner asks: "What file should be modified?", "Are there tests for this configuration?", "Any deployment constraints?"
5. User responds. Planner produces a task plan from initial-request.md + user answers.
6. Steps 5–6 implement and verify. Step 7 reviews. Step 9 commits.

### Story 4: Researcher does web research in interactive mode

1. Pipeline runs in interactive mode. Researcher is dispatched.
2. Researcher identifies a need to understand a new API pattern.
3. Researcher calls fetch_webpage with a relevant URL. VS Code prompts user for approval.
4. User approves. Researcher incorporates web content into findings.

### Story 5: Pipeline archives old DB and starts fresh

1. User starts a new pipeline run. verification-ledger.db exists with data from yesterday's run.
2. Orchestrator Step 0 queries pipeline_telemetry — finds run_id from yesterday.
3. Orchestrator renames: `verification-ledger.db` → `verification-ledger-2026-03-04T15-30-00Z.db`
4. Creates fresh verification-ledger.db with standard DDL.
5. Pipeline proceeds with clean DB.

---

## Test Scenarios

| ID   | Scenario                                                                            | Linked AC | Method                                  |
| ---- | ----------------------------------------------------------------------------------- | --------- | --------------------------------------- |
| T-1  | Verify planner lane derivation table allows e2e_required=true for 🟢 task           | AC-1      | Inspect planner.agent.md                |
| T-2  | Verify verifier Tier 5 gate checks only e2e_required (not workflow_lane)            | AC-1      | Inspect verifier.agent.md               |
| T-3  | Verify planner can set e2e_required=false for 🟢 trivial task with contract         | AC-2      | Inspect planner.agent.md decision logic |
| T-4  | Verify 🔴 + contract → e2e_required=true is enforced                                | AC-3      | Inspect planner.agent.md                |
| T-5  | `grep -ri "default to autonomous" NewAgents/.github/` returns 0 matches             | AC-5      | Terminal test                           |
| T-6  | Verify orchestrator Step 0 uses APPROVAL_MODE from initial-request.md               | AC-4      | Inspect orchestrator.agent.md           |
| T-7  | Verify fetch_webpage row exists in tool-access-matrix.md §1                         | AC-6      | Inspect tool-access-matrix.md           |
| T-8  | Verify researcher.agent.md mentions fetch_webpage with interactive-mode restriction | AC-7      | Inspect researcher.agent.md             |
| T-9  | Verify implementer.agent.md has no-redirect rule with banned patterns               | AC-8      | Inspect implementer.agent.md            |
| T-10 | Verify verifier.agent.md has no-redirect rule with banned patterns                  | AC-9      | Inspect verifier.agent.md               |
| T-11 | Verify spec.agent.md pushback uses questions[] array with per-concern entries       | AC-11     | Inspect spec.agent.md                   |
| T-12 | Verify e2e-integration.md §2 has skill_format field                                 | AC-13     | Inspect e2e-integration.md              |
| T-13 | Verify orchestrator Step 0 has archive check before DDL                             | AC-14     | Inspect orchestrator.agent.md           |
| T-14 | Verify plan-and-implement.prompt.md exists with correct frontmatter                 | AC-16     | Inspect file existence and content      |
| T-15 | Verify planner has ask_questions 🔒 in tool-access-matrix.md                        | AC-17     | Inspect tool-access-matrix.md §1 and §6 |
| T-16 | Verify planner fast-track input path defined                                        | AC-18     | Inspect planner.agent.md                |
| T-17 | Verify orchestrator anti-drift anchor allows prompt-defined step sets               | AC-19     | Inspect orchestrator.agent.md           |
| T-18 | `wc -l orchestrator.agent.md` ≤ 550                                                 | AC-21     | Terminal test                           |
| T-19 | Verify feature-workflow.prompt.md is not modified                                   | AC-20     | Git diff                                |
| T-20 | Verify anti-drift anchors in implementer + verifier mention no-redirect             | AC-10     | Inspect both files                      |

---

## Dependencies & Risks

### Dependencies

| Dependency            | Type                    | Impact                                                                 |
| --------------------- | ----------------------- | ---------------------------------------------------------------------- |
| tool-access-matrix.md | Shared resource         | 3 issues (#3, #7, #8) modify this file — batch changes                 |
| orchestrator.agent.md | Line-budget constrained | 4 issues (#1, #2, #7, #8) modify this file — extract to reference docs |
| planner.agent.md      | E2E root                | Issue #1 lane derivation is the root of the E2E cascade                |
| e2e-integration.md    | E2E reference           | Issues #1 and #6 both modify E2E integration rules                     |
| fetch_webpage tool    | Runtime dependency      | Must be available in VS Code Copilot runtime                           |
| ask_questions tool    | Runtime dependency      | Must support questions[] array (verified from tool schema)             |
| Copilot Skills format | External dependency     | Issue #6 depends on understanding current format via web research      |

### Risks

| Risk                                                           | Likelihood | Impact              | Mitigation                                                                      |
| -------------------------------------------------------------- | ---------- | ------------------- | ------------------------------------------------------------------------------- |
| Orchestrator exceeds 550-line budget                           | High       | Blocks merge        | Extract fast-track rules to reference doc (e.g., fast-track-pipeline.md)        |
| E2E expansion increases pipeline time for simple tasks         | Medium     | User frustration    | FR-1.7 makes E2E opt-in for 🟢/🟡, mandatory only for 🔴                        |
| Copilot Skills format changes before implementation            | Low        | Rework needed       | Design integration as a thin layer; Playwright CLI remains the execution engine |
| Approval mode fix introduces new edge cases                    | Low        | Mode confusion      | FR-2.3 establishes interactive as the safe fallback                             |
| Multiple concurrent changes to tool-access-matrix.md           | Medium     | Merge conflicts     | Batch all tool-access-matrix changes into one task                              |
| Fast-track planner produces low-quality plans without research | Medium     | Poor implementation | FR-8.3 requires planner to ask clarifying questions via ask_questions           |

### Implementation Ordering Recommendation

**Wave 1 — Independent, Low Risk:**

- Issue #2 (approval mode clarity) — text changes only
- Issue #4 (no-redirect enforcement) — text additions only
- Issue #5 (spec pushback) — single file change
- Issue #7 (SQLite archive) — isolated to Step 0

**Wave 2 — Tool Access and New Capabilities:**

- Issue #3 (fetch_webpage) — new tool row + agent updates
- Issue #8 (fast-track prompt) — new prompt + orchestrator/planner changes
- **Batch** all tool-access-matrix.md changes from Issues #3, #7, #8 into one task

**Wave 3 — High-Cascade E2E Changes:**

- Issue #1 (E2E for all risk levels) — 7-file cascade
- Issue #6 (Copilot Skills) — layered on E2E chain

---

## Pushback Log

| #   | Severity | Concern                                                                  | Recommendation                                                          | Resolution                                                             |
| --- | -------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| 1   | Major    | E2E for all risk levels may increase pipeline duration for trivial tasks | Make E2E opt-in (planner decides) for 🟢/🟡, mandatory only for 🔴      | Auto-accepted (autonomous mode). Addressed in FR-1.7.                  |
| 2   | Major    | GitHub Copilot Skills format requires web research with uncertain scope  | Layer on existing Playwright infrastructure; designer researches format | Auto-accepted (autonomous mode). Addressed in FR-6.5.                  |
| 3   | Major    | Fast-track prompt conflicts with orchestrator "NEVER skip steps" rule    | Amend anti-drift anchor to condition on prompt-defined step sets        | Auto-accepted (autonomous mode). Addressed in FR-8.5.                  |
| 4   | Minor    | Three issues converge on tool-access-matrix.md                           | Batch all tool-access-matrix changes into one task                      | Auto-accepted (autonomous mode). Addressed in ordering recommendation. |
