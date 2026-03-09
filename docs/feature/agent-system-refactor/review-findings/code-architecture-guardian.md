# Adversarial Review: code — architecture-guardian

## Review Metadata

- **Perspective:** architecture-guardian
- **Risk Level:** 🔴
- **Round:** 1
- **Run ID:** 2026-03-08T12:00:00Z

## Security Analysis

**Category Verdict:** approve

No security-scoped findings from the architecture lens. Tool access is enforced via YAML frontmatter across all 8 agents. Trust tiers are correctly expressed through tool selection:

- Orchestrator (Tier 1): git-only terminal + agent dispatch
- Implementer (Tier 2): build/test terminal + file editing
- Reviewer/Researcher/Architect/Planner/Knowledge: read + create only (no terminal or limited)

The pre-commit output validation in Step 8 for Knowledge agent outputs is correctly implemented. The `fetch_webpage` gating on Researcher (`web_research_enabled`) and accepted residual risk on Implementer (with reviewer URL audit trail) match the D-10/D-14 trust model from the design.

No security findings.

## Architecture Analysis

**Category Verdict:** needs_revision

### Finding A-1: Verdict signal lives outside completion contract — routing protocol contradiction

- **Severity:** Major
- **Description:** The Reviewer agent's output schema always sets `completion.status: "DONE"` and places its success/failure signal in a separate `verdict: "approve" | "request-changes"` field. The Tester agent follows the same pattern: `completion.status: "DONE"` with `verdict: "pass" | "fail"`. However, the Orchestrator's Constraints section (line 98 of orchestrator.agent.md) states: "completion contracts only — read `completion.status` from YAML." This directly contradicts Step 7 (line 72: "Gate: ≥2 approve + 0 blockers") and Step 6 (line 67: "NEEDS_REVISION → route to Step 5"), which both require reading the `verdict` field — a field that exists OUTSIDE the completion contract.

  The result is contradictory instructions for the LLM executing the orchestrator: the Constraints say "only read completion.status" while the step definitions say "read verdict fields." In an LLM agent system, contradictory instructions are a reliability risk — the model may follow the constraint and ignore verdicts, effectively auto-approving all reviews.

- **Affected artifacts:** `v2/.github/agents/orchestrator.agent.md` (lines 67-72 vs line 98), `v2/.github/agents/reviewer.agent.md` (line 71), `v2/.github/agents/tester.agent.md` (line 107)
- **Recommendation:** Either (a) update the Orchestrator constraint to say "completion contracts + verdict fields for gate evaluation" to explicitly carve out the exception, or (b) change Reviewer/Tester to map failure verdicts to `completion.status: NEEDS_REVISION` so the routing constraint holds as-is. Option (b) is architecturally cleaner — it makes the completion contract the single routing interface.
- **Evidence:** Orchestrator line 98: `completion contracts only — read completion.status from YAML`. Reviewer line 71: `status: "DONE"` (unconditional). Orchestrator Step 7 line 72: `≥2 approve + 0 blockers` (requires reading verdict).

### Finding A-2: Quick-fix pipeline skips Planning but Implementer requires task YAML

- **Severity:** Major
- **Description:** The quick-fix prompt (`quick-fix.prompt.md`, line 19) explicitly skips Step 4 (Planning). But the Implementer agent's Inputs table (implementer.agent.md, line 30-32) requires `tasks/task-XX.yaml` from the Planner, containing `files`, `depends_on`, `acceptance_criteria`, and `relevant_context`. Without the Planner, no task definition exists. The Implementer's file ownership constraint ("MUST NOT modify files not declared in `task.files[]`") becomes unenforceable because `task.files[]` doesn't exist. Neither the quick-fix prompt nor the Orchestrator Step 5 defines a fallback for creating task definitions when the Planner is skipped.

- **Affected artifacts:** `v2/.github/prompts/quick-fix.prompt.md` (line 19), `v2/.github/agents/implementer.agent.md` (lines 30-32, 87-89), `v2/.github/agents/orchestrator.agent.md` (Step 5, lines 59-62)
- **Recommendation:** Add to the Orchestrator's Step 1 (Setup) or Step 5 (Implementation) a clause: "For quick-fix pipelines (no Planner): orchestrator creates a minimal task definition with `id`, `files` (inferred from the feature request), and `acceptance_criteria` before dispatching implementer." Alternatively, document that the quick-fix implementer operates without file ownership constraints and the reviewer must audit file scope instead.
- **Evidence:** quick-fix.prompt.md line 19: "Steps skipped: Research (Step 2), Architecture (Step 3), Planning (Step 4)." Implementer inputs table row 1: `tasks/task-XX.yaml | Planner | Task with files, depends_on, acceptance_criteria, relevant_context`.

### Finding A-3: Orchestrator section structure deviates from 6-section convention

- **Severity:** Minor
- **Description:** Decision D-3 established a consistent 6-section body structure for all agents: Role, Inputs, Workflow, Output Schema, Constraints, Anti-Drift Anchor. All 7 dispatched agents follow this convention. The Orchestrator uses a different structure: Role, Pipeline Steps, DAG Dispatch Algorithm, Constraints, Anti-Drift Anchor — omitting explicit "Inputs" and "Output Schema" sections. While the Orchestrator's unique coordinator role justifies some deviation, the missing sections mean its input expectations (where does it read `plan-output.yaml`? what parameters does it receive?) and output expectations (what does `pipeline-log.yaml` contain?) are implicit rather than documented.

- **Affected artifacts:** `v2/.github/agents/orchestrator.agent.md`
- **Recommendation:** Add a minimal "Inputs" section listing what the orchestrator reads per step (e.g., user feature request, plan-output.yaml, completion YAMLs) and consider documenting `pipeline-log.yaml` structure. This improves self-documentation without adding significant line count.
- **Evidence:** Architect (6 sections), Implementer (6), Planner (6), Researcher (6), Reviewer (6), Tester (6), Knowledge (6), Orchestrator (5 — no Inputs, no Output Schema).

### Finding A-4: Inconsistent verdict vocabularies across agents

- **Severity:** Minor
- **Description:** Three different verdict/status vocabularies exist: completion contract (`DONE/NEEDS_REVISION/ERROR`), reviewer verdict (`approve/request-changes`), tester verdict (`pass/fail`). The orchestrator must understand all three, increasing its parsing complexity. While each vocabulary is locally coherent, the lack of a unified signal vocabulary is a maintenance burden and contributes to the routing contradiction in A-1.

- **Affected artifacts:** `v2/.github/agents/reviewer.agent.md` (line 69), `v2/.github/agents/tester.agent.md` (line 115), `v2/.github/agents/global-rules.md` (line 8)
- **Recommendation:** Standardize on one signal vocabulary. If fixing A-1 via option (b) — mapping verdicts to completion.status — this finding resolves automatically.
- **Evidence:** Reviewer: `approve | request-changes`. Tester: `pass | fail`. Completion: `DONE | NEEDS_REVISION | ERROR`.

## Correctness Analysis

**Category Verdict:** approve

### Finding C-1: Reviewer lacks round-2 awareness guidance

- **Severity:** Minor
- **Description:** The reviewer agent has no workflow guidance for round-2 behavior. The `round` parameter is captured in the output schema, but no instructions tell the reviewer to: check if round-1 findings were addressed, avoid re-reporting resolved findings, or reference previous findings. In round 2, the reviewer may simply re-report all issues from round 1 even if they were fixed, wasting the review cycle.

- **Affected artifacts:** `v2/.github/agents/reviewer.agent.md` (Workflow section, lines 39-46)
- **Recommendation:** Add a workflow step: "If round > 1: read previous round's findings for your perspective. Note which were addressed, partially addressed, or unresolved. Only report unresolved or new findings."
- **Evidence:** Reviewer workflow has 6 numbered steps; none mention round awareness. Output schema includes `round: <int>` but no behavioral guidance is linked to it.

### Verified Correctness Items

- **Line budgets:** All 8 agents range 69–98 lines (≤150 CR-3). Total system: 770 lines (≤1,500 CR-2). 48.7% headroom on system total.
- **Zero § cross-references:** `grep_search` for `§` across v2/.github/\*\* returned 0 matches.
- **Zero SQLite references:** `grep_search` for `sqlite|SQLite|sql-templates|verification-ledger` returned 0 matches.
- **Pipeline coherence:** All 8 steps map to the correct 7 dispatched agents. All agents listed in orchestrator frontmatter (`agents:` field).
- **DAG algorithm:** 7-step algorithm is clear, deterministic, and bounded by task count.
- **`architecture-output.yaml` naming:** Standardized across architect (produces), planner (consumes), reviewer (reads for design review). Zero `design-output.yaml` references in v2/.
- **global-rules.md sole cross-reference:** Only shared document. 5 agents reference it by name only (no section numbers). Zero cross-agent internal references.
- **YAML frontmatter:** All 8 agents and 2 prompts have valid YAML frontmatter with required fields (`name`, `description`, `tools`/`agent`).
- **Feedback loop limits:** global-rules.md defines limits matching orchestrator constraints (3 impl-test, 2 code review, 1 design review). Cycle counter reset documented in both.

## Summary

The v2 clean-slate rebuild is architecturally sound at 770 lines across 11 files — a dramatic 88% reduction from the prior 6,416-line system. Agent decoupling is excellent: zero cross-agent internal references, zero SQLite, zero § cross-references, global-rules.md as the sole shared document. Two Major findings require resolution before approval: (1) the completion contract routing contradiction between the orchestrator's constraint and its gate-evaluation steps, and (2) the quick-fix pipeline's missing task-definition path. Both are addressable with targeted additions of ~5-10 lines.
