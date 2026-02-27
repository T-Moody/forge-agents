# Memory: Adversarial Reviewer 1 (gpt-5.3-codex perspective)

## Key Facts Learned

1. **Anvil's `anvil_checks` schema is single-agent-oriented.** It works well for Anvil's single-agent-per-task flow but lacks columns needed for multi-agent multi-round pipelines: `verdict`, `severity`, `round`, `run_id`. Adopting it unchanged into a multi-agent system creates evidence gating bugs.

2. **SQLite concurrent write handling is not optional.** Any design with ≤4 parallel agents writing to the same SQLite file MUST specify WAL mode (`PRAGMA journal_mode=WAL`) and busy timeout (`PRAGMA busy_timeout=5000`). Default journal mode causes SQLITE_BUSY on concurrent writes. This applies to Steps 3b, 5, 6, and 7.

3. **COUNT-based evidence gates need careful WHERE clauses.** Review gates that don't filter on `passed=1` or `verdict='approve'` will pass even when all reviewers found issues. Gates that don't filter by `round` accumulate stale records across revision cycles. Both bugs are silent (gate passes when it shouldn't).

4. **The `orchestrator-tool-restriction` feature explicitly prohibited `run_in_terminal` for the orchestrator.** The next-gen design v3 reverses this. Any reversal should be explicitly justified with a decision record acknowledging the tradeoff change.

5. **Adversarial Reviewer is the only agent without a typed YAML output.** It produces Markdown + SQL, breaking the "typed YAML at every boundary" contract. This creates a resume gap — review verdicts are unrecoverable from persisted state without parsing Markdown.

6. **SQLite initialization should be centralized in Step 0.** Distributed `CREATE TABLE IF NOT EXISTS` across multiple agents creates race conditions during parallel dispatch. Single-point initialization by the orchestrator (which has `run_in_terminal`) eliminates this class of bugs.

7. **`task_id` semantics diverge between per-task operations and feature-level operations.** Anvil uses `task_id` for per-task work. Design review and code review are feature-level. The schema needs a convention for feature-level `task_id` values, or a separate `scope` column.

## Citations

- design.md §Decision 6: `anvil_checks` schema definition (lines ~620-635)
- design.md §Data Storage Strategy: SQLite Operational Notes — "typical pipeline concurrency is low enough for default locking" (line ~1955)
- design.md §Decision 3: Step 3b gate query vs Step 7 gate query asymmetry (lines ~289, ~328)
- design.md §Decision 2: Adversarial Reviewer outputs — no YAML (line ~233)
- docs/feature/orchestrator-tool-restriction/design.md: `run_in_terminal` prohibition (lines ~143-153)
- Anvil/anvil.agent.md: Original `anvil_checks` schema (lines ~62-75)
