# Task 10: Knowledge Agent Definition

## Task Goal

Create `NewAgents/.github/agents/knowledge-agent.agent.md` — the knowledge evolution agent with decision logging, evidence bundle assembly, and cross-session learning.

## depends_on

01, 02

## agent

implementer

## In-Scope

- Agent definition following design.md §Decision 10 template
- Role: knowledge evolution, decision logging, cross-session learning, evidence bundle assembly
- Input: all pipeline outputs (typed YAML), implementation reports, verification evidence, review verdicts, `git diff`
- Output:
  - `knowledge-output.yaml` (typed knowledge-output schema)
  - `decisions.yaml` (append-only decision log updates)
  - VS Code `store_memory` calls (cross-session knowledge)
  - `evidence-bundle.md` (Step 8b — assembled proof-of-quality deliverable)
- Evidence bundle components (v4 H2):
  - Overall confidence rating (High/Medium/Low)
  - Aggregated verification summary (passed/failed/regressions per task)
  - Adversarial review summary (verdicts per model, issues found and fixed)
  - Rollback command: `git revert --no-commit pipeline-baseline-{run_id}..HEAD`
  - Blast radius: files modified, risk classifications, regression count
  - Known issues with severity ratings
- Non-blocking: ERROR from this agent does NOT halt pipeline
- Governed updates: instruction file modifications require explicit approval in interactive mode; logged in autonomous mode
- Safety constraint filter: prevents weakening existing security checks
- Tool access: `read_file`, `list_dir`, `grep_search`, `semantic_search`, `file_search`, `create_file`, `replace_string_in_file`, `memory` (VS Code `store_memory`)
- Completion contract: DONE | ERROR
- Self-verification + anti-drift anchor

## Out-of-Scope

- Schema definition (in schemas.md)
- Pipeline execution (orchestrator's job)
- Telemetry table writes (each agent writes its own telemetry)

## Acceptance Criteria

1. File exists at `NewAgents/.github/agents/knowledge-agent.agent.md`
2. Follows agent definition template with all required sections
3. References `knowledge-output` schema from schemas.md
4. Evidence bundle assembly documented with all 6 components from design.md Step 8b
5. Non-blocking behavior explicitly stated: ERROR does not halt pipeline
6. Decision log append-only format documented
7. Governed updates documented: approval in interactive, logging in autonomous
8. Safety constraint filter documented
9. `store_memory` usage documented for cross-session knowledge persistence
10. Completion contract: DONE | ERROR
11. Self-verification and anti-drift anchor present

## Estimated Effort

Medium

## Test Requirements

- Grep for evidence bundle or evidence-bundle.md
- Grep for non-blocking
- Grep for store_memory
- Grep for governed updates or safety filter
- Verify completion contract and anti-drift

## Implementation Steps

1. Read design.md §Decision 2, Knowledge Agent detail for inputs, outputs, tools, non-blocking, governed, contract
2. Read design.md §Decision 3, Steps 8 and 8b for knowledge capture and evidence bundle assembly (H2)
3. Read design.md §Security for governed updates and safety constraints
4. Reference existing `.github/agents/r-knowledge.agent.md` for knowledge evolution and decision logging patterns
5. Create `NewAgents/.github/agents/knowledge-agent.agent.md` with full template structure including:
   - Evidence bundle assembly workflow (Step 8b sub-step)
   - Decision log append-only format
   - Governed update rules
   - Cross-session store_memory patterns
6. Self-verify all sections present

## Relevant Context from design.md

- §Decision 2 (lines ~310–325) — Knowledge Agent: inputs, outputs, tools, non-blocking, governed, contract
- §Decision 3, Steps 8/8b (lines ~445–470) — Knowledge capture and evidence bundle assembly
- §Security (lines ~1400–1450) — Governed updates, safety constraints
- §Decision 5 (lines ~590–600) — Cross-session knowledge via store_memory

## Completion Checklist

- [x] File created at correct path
- [x] All template sections present
- [x] Evidence bundle assembly (6 components) documented
- [x] Non-blocking explicitly stated
- [x] Decision log format documented
- [x] Governed updates documented
- [x] store_memory usage documented
- [x] Schema reference to schemas.md present
- [x] Anti-drift anchor present

## Implementation Notes

- TDD skipped: configuration-only task (agent definition Markdown file, no behavioral code)
- Validated via grep-based test requirements: evidence bundle, non-blocking, store_memory, governed updates, safety filter, completion contract, anti-drift all present
- Followed formatting conventions from existing agents (researcher.agent.md, designer.agent.md)
- Referenced r-knowledge.agent.md for knowledge-specific patterns (safety filter, decision log, governed updates)
