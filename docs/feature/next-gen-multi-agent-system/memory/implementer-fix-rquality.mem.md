# Memory: implementer-fix-rquality

## Status

DONE: Fixed 1 Blocker + 6 Major findings from R-Quality code review

## Key Findings

- Blocker: Implementer had no SQL INSERT for `phase='baseline'` records — orchestrator evidence gate would always stall. Added explicit INSERT instructions + self-verification checklist item + tool purpose update.
- Major 3: Orchestrator created `pipeline_telemetry` in `verification-ledger.db` but design.md/schemas.md specify separate `pipeline-telemetry.db`. Split init into two separate sqlite3 commands in both orchestrator.agent.md and schemas.md.
- Major 4: Knowledge agent workflow referenced SQL queries but lacked `run_in_terminal`. Changed to read verification YAML reports instead (Option B — consistent with read-focused nature).
- Major 6: Three parallel adversarial reviewers wrote to same `review-verdicts/<scope>.yaml` — only last writer survived. Changed to `review-verdicts/<scope>-<model>.yaml` across all 5 references in adversarial-reviewer.agent.md.
- design.md itself has an internal inconsistency: init script block (line 713) says "Both tables initialized together in Step 0" but directory structure (line 1282) and storage decision table (line 2018) show separate DB files.

## Decisions Made

- Chose Option B for knowledge agent SQL access (read YAML reports instead of adding `run_in_terminal`) to maintain the agent's read-focused design contract.
- Split pipeline_telemetry into separate DB file (matching schemas.md and design.md directory structure/storage table) rather than consolidating both tables into verification-ledger.db.

## Artifact Index

- [NewAgents/.github/agents/implementer.agent.md](NewAgents/.github/agents/implementer.agent.md) — §Baseline Capture Detail (added SQL INSERT section), §Workflow Step 2 (added step 5), §Self-Verification (added checklist item), §Tool Access (updated run_in_terminal purpose)
- [NewAgents/README.md](NewAgents/README.md) — §Per-Task Verifier (fixed tier descriptions), §File Structure (removed Pattern C), §Reference Documents (removed Pattern C)
- [NewAgents/.github/agents/orchestrator.agent.md](NewAgents/.github/agents/orchestrator.agent.md) — §Step 0 init (split into two DB-specific SQL blocks)
- [NewAgents/.github/agents/schemas.md](NewAgents/.github/agents/schemas.md) — §Full Step 0 Init Script (split into two DB-specific blocks), §Schema 6 task example (fixed payload.requirements → payload.functional_requirements)
- [NewAgents/.github/agents/planner.agent.md](NewAgents/.github/agents/planner.agent.md) — §Task example + §Pointer Format (fixed payload.requirements → payload.functional_requirements)
- [NewAgents/.github/agents/knowledge-agent.agent.md](NewAgents/.github/agents/knowledge-agent.agent.md) — §Workflow Step 1 items 6-7 (SQL queries → YAML file reads)
- [NewAgents/.github/agents/adversarial-reviewer.agent.md](NewAgents/.github/agents/adversarial-reviewer.agent.md) — §Output Files, §Produce Output, §Operating Rules, §Self-Verification, §header (all `review-verdicts/<scope>.yaml` → `review-verdicts/<scope>-<model>.yaml`)
